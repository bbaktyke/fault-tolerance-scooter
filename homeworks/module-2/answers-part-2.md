# Модуль 2, часть 2: «Можно удалить `terminated`?»

**Короткий ответ коллеге:** нет. Kubernetes умеет остановить pod, но без флага readiness оставалась бы `200`, пока сервер закрывается. Pod продолжал бы получать новые запросы, и пользователи видели бы ошибки. Флаг заранее выводит pod из трафика, а сервер в это время дорабатывает уже принятые запросы.

## 1. Что означает флаг и чем отличается от `started`?

`terminated` означает «процесс начал штатную остановку, новый трафик сюда больше не слать». Сам флаг ничего не завершает и соединения не закрывает: это делают функции Closer'а (см. п. 4).

| Флаг | Когда 0 → 1 | Смысл для readiness |
|---|---|---|
| `started` | в конце `Run`, когда запущены серверы, шина и inbox-воркеры ([app.go:153](../../services/pricing/internal/app.go#L153)) | можно принимать трафик |
| `terminated` | при начале остановки ([init.go:85](../../services/pricing/internal/init.go#L85), [init.go:244](../../services/pricing/internal/init.go#L244)) | трафик больше не принимать |

Оба флага меняются только в одну сторону и обратно не сбрасываются. Итоговая readiness = `started && тарифы загружены && !terminated`. Поэтому поздняя загрузка данных или `uncordon` не вернут завершающийся процесс в готовность.

## 2. Кто его изменяет и читает? Почему доступ должен быть безопасным?

- **Пишут** closer-функции `publicCloser`: [init.go:85](../../services/pricing/internal/init.go#L85) (graceful-остановка сервера) и [init.go:244](../../services/pricing/internal/init.go#L244) (логирование сигнала). При инициализации функции только регистрируются, а срабатывают при `CloseAll()`: по сигналу или при падении сервера / inbox-воркера ([app.go:139](../../services/pricing/internal/app.go#L139), [app.go:148](../../services/pricing/internal/app.go#L148)).
- **Читает** readiness-проверка `"termination"` ([init.go:229](../../services/pricing/internal/init.go#L229)). Она вызывается на каждый запрос `/healthcheck/ready` в горутине admin HTTP-сервера.

Пишут горутины, которые запускает `CloseAll` (каждая функция в своей горутине), а читают горутины HTTP-обработчиков. С обычным `int32` это data race: `go test -race` его ловит, а модель памяти Go не гарантирует, что читатель увидит запись. `atomic.StoreInt32` и `atomic.LoadInt32` дают безопасный доступ без мьютекса.

## 3. Как обрабатываются `SIGTERM` и `SIGINT`? Почему нельзя обработать `SIGKILL`?

- `closer.New(syscall.SIGTERM, syscall.SIGINT)` ([app.go:74](../../services/pricing/internal/app.go#L74)) запускает горутину с `signal.Notify` ([closer.go](../../services/pricing/internal/pkg/closer/closer.go), `New`).
- По первому сигналу вызываются `signal.Stop` и затем `CloseAll()`, то есть выполняются все функции `publicCloser`. Повторный Ctrl+C уже обрабатывается по умолчанию и убивает процесс сразу.
- `Run` ждёт на `publicCloser.Wait()`, затем закрывает глобальный `closer` (например, `conn.Close` из [init.go:184](../../services/pricing/internal/init.go#L184)), а последним `adminCloser` ([app.go:161-165](../../services/pricing/internal/app.go#L161-L165)).
- `SIGTERM` присылает Kubernetes при удалении pod'а, `SIGINT` присылает Ctrl+C при локальном запуске.

`SIGKILL` ядро процессу не доставляет: его нельзя перехватить, заблокировать или проигнорировать (как и `SIGSTOP`), и `signal.Notify` его не получит. Поэтому Kubernetes сначала шлёт `SIGTERM`, ждёт `terminationGracePeriodSeconds` (по умолчанию 30 с) и только потом `SIGKILL`. Graceful-остановка должна уложиться в это время, здесь `graceful.timeout: 10s` ([config.yaml](../../services/pricing/config/config.yaml)).

## 4. Что флаг делает при graceful shutdown, а кто завершает уже принятые запросы?

1. `terminated = 1`, и `/healthcheck/ready` начинает отвечать `503`.
2. Kubelet помечает pod как NotReady, Kubernetes убирает его из Endpoints, и gateway/envoy перестают слать на него **новые** запросы.
3. `time.Sleep(Graceful.Delay)` (3 с, [init.go:90](../../services/pricing/internal/init.go#L90)) нужен, чтобы шаг 2 успел распространиться до закрытия сервера.
4. **Уже принятые запросы завершает сервер, а не флаг:** `a.mainServer.Stop(gracefulCtx)` ([init.go:120](../../services/pricing/internal/init.go#L120)) дожидается текущих запросов, но не дольше `Graceful.Timeout`. Параллельно останавливаются inbox-воркеры и шина ([init.go:299](../../services/pricing/internal/init.go#L299), [init.go:306](../../services/pricing/internal/init.go#L306)).
5. Liveness флаг не трогает и остаётся `200`, поэтому Kubernetes не считает штатную остановку поломкой и не перезапускает контейнер.

## 5. Гарантирует ли порядок регистрации функций в `Closer` порядок выполнения?

**Нет.** `Add` сохраняет порядок в слайсе (`append`), но `CloseAll` запускает каждую функцию в **отдельной горутине** (`go func(f) { errs <- f() }(f)`), и выполняются они одновременно.

Поэтому `terminated = 1` выставляется в двух местах: graceful-closer ставит флаг первой строкой сам и не рассчитывает, что логирующий closer успеет раньше.

Гарантированный порядок есть только между разными Closer'ами: `Run` вызывает их последовательно, сначала `publicCloser`, потом глобальный `closer`, потом `adminCloser`. Admin-сервер с health-check'ами закрывается последним, поэтому пробы отвечают на протяжении всей остановки.

## Подтверждение: `./bin/scooterctl homework module-2 run shutdown`

```
OK termination снимает readiness: pricing-d7c48bb75-9hzp5
OK Liveness во время shutdown: live=200 при ready=503
OK Uncordon не оживляет shutdown: termination сохраняется
Журнал остановки:
09:27:13.494690 INFO Ожидание прекращения нового трафика delay=3s
09:27:13.494803 INFO Получен сигнал остановки timeout=10s
09:27:16.519594 ERROR Ошибка основного сервера error="http: Server closed"
09:27:16.519885 INFO Основной сервер остановлен
```

Порядок событий по коду:
- **Сигнал и начало завершения (09:27:13).** `SIGTERM` → `publicCloser.CloseAll()`. Обе closer-функции стартуют одновременно и ставят `terminated = 1`, readiness → `503`, liveness остаётся `200`.
- **Порядок строк подтверждает п. 5.** «Ожидание прекращения…» ([init.go:89](../../services/pricing/internal/init.go#L89)) напечаталось раньше, чем «Получен сигнал остановки» ([init.go:236](../../services/pricing/internal/init.go#L236)), хотя closer'ы зарегистрированы в обратном порядке.
- **Остановка сервера (09:27:16).** Ровно через `Graceful.Delay` = 3 с вызывается `mainServer.Stop`. Строка `ERROR … http: Server closed` — штатный результат: `mainServer.Run` вернул `ErrServerClosed` ([app.go:138](../../services/pricing/internal/app.go#L138)), а повторный `publicCloser.CloseAll()` ничего не делает благодаря `sync.Once`.
- Сам статус `Terminating` ничего не доказывает. Работу флага подтверждают `ready=503` при `live=200` в первых строках вывода.
