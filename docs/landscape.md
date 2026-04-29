# Phần 7 — Landscape toàn cục: cùng một con voi

Trước khi đi vào từng mảnh của landscape, hãy nhìn bức tranh toàn cục: **cùng một thuật toán** — pure description + orchestrating runtime — xuất hiện ở mọi tầng của stack, từ pixel trên màn hình đến container trên cluster:

```text
SCALE: UI LAYER → PROCESS LAYER → INFRASTRUCTURE LAYER
═══════════════════════════════════════════════════════════════════════════

  UI LAYER (ms latency, single machine)
  ┌─────────────────────────────────────────────────────────┐
  │  React / Elm / SwiftUI / Jetpack Compose / LiveView     │
  │                                                         │
  │  State:    Fiber node / Model / @State / Socket assigns │
  │  Trigger:  User event (click, input, scroll)            │
  │  Pure fn:  reducer / update / body / handle_event       │
  │  Runtime:  Reconciler / Elm runtime / SwiftUI engine    │
  │  Diff:     Virtual DOM / HTML patch / native tree diff  │
  │  Output:   Pixels on screen                             │
  └─────────────────────┬───────────────────────────────────┘
                        │ cùng pattern, quy mô lớn hơn
                        ▼
  PROCESS LAYER (μs–ms latency, single node → cluster)
  ┌─────────────────────────────────────────────────────────┐
  │  Elixir GenServer / Akka Actor / Swift Actor / Go CSP   │
  │                                                         │
  │  State:    Process heap / Actor state / Goroutine       │
  │  Trigger:  Message / call / cast / channel send         │
  │  Pure fn:  handle_call / receive / Behavior[Cmd]        │
  │  Runtime:  BEAM scheduler / JVM / Go runtime            │
  │  Diff:     N/A (state mutation via new state return)    │
  │  Output:   Reply message / side effect via shell        │
  │                                                         │
  │  Supervisor / errgroup / structured concurrency         │
  │  → Failure recovery at process level                    │
  └─────────────────────┬───────────────────────────────────┘
                        │ cùng pattern, quy mô lớn hơn
                        ▼
  INFRASTRUCTURE LAYER (ms–s latency, distributed)
  ┌─────────────────────────────────────────────────────────┐
  │  Kubernetes / Temporal / AWS Lambda / Kafka / ECS       │
  │                                                         │
  │  State:    etcd / Temporal history / S3 / Kafka offset  │
  │  Trigger:  Event / HTTP req / message / cron / webhook  │
  │  Pure fn:  Reconcile loop / Workflow fn / Handler fn    │
  │  Runtime:  kube-controller / Temporal service / Lambda  │
  │  Diff:     Desired vs actual state / event replay       │
  │  Output:   Pod lifecycle / workflow step / response     │
  │                                                         │
  │  Supervisor equivalent: restart policy / retry + DLQ   │
  │  → Failure recovery at infrastructure level             │
  └─────────────────────────────────────────────────────────┘

PATTERN XA SUỐT CÁC TẦNG:
─────────────────────────────────────────────────────────────
  pure_fn(current_state, trigger) → (new_state, effects)
  runtime.apply(effects)
  runtime.observe() → next trigger
  [repeat]
```

Kubernetes reconciliation loop, React render loop, GenServer message loop — là cùng một vòng lặp, chỉ khác đơn vị thời gian (nanosecond vs millisecond vs second) và đơn vị state (fiber node vs process heap vs etcd key). Nắm vững bất kỳ một tầng nào, bạn đã có 70% mental model cho tầng còn lại.

## 7.1. Elm Architecture — Rosetta Stone của triết lý

**The Elm Architecture (TEA)** là biểu hiện **tối giản và trong sạch nhất** của toàn bộ triết lý này. Xuất hiện 2012 (trước React Hooks 7 năm), được Dan Abramov công khai thừa nhận là nguồn gốc của Redux:

```elm
type alias Program flags model msg =
    { init   : flags -> (model, Cmd msg)          -- pure constructor, trả về effect description
    , update : msg -> model -> (model, Cmd msg)    -- pure reducer = GenServer handle_call/3
    , view   : model -> Html msg                  -- UI = f(state)
    , subscriptions : model -> Sub msg            -- khai báo input stream, runtime lo subscribe
    }
```

`Cmd msg` và `Sub msg` là **first-class values mô tả effect** — code không thực hiện HTTP call, code *trả về AST của HTTP call* để Elm runtime interpret. Đây chính xác là Free Monad pattern và là algebraic effects thuần khiết nhất trong các UI framework mainstream. Dùng TEA như Rosetta Stone: mọi khái niệm trong React và OTP đều ánh xạ trực tiếp về đây.

## 7.2. Ngôn ngữ và runtime: cùng triết lý, accent khác nhau

| Ngôn ngữ / Runtime | Đặc điểm nổi bật | Vị trí trong landscape |
| --- | --- | --- |
| **Haskell** | `IO` monad là handler implicit của toàn bộ I/O. `do`-notation là CPS-style chaining có đường. `STM` monad cô lập transaction. | Tổ tiên trực tiếp — algebraic effects được generalize từ đây |
| **OCaml 5.x** | Effect handlers thành cú pháp chính thức (5.3+). `Eio`, `Miou` xây async runtime trên primitive này thay vì "magic". | First-class algebraic effects trong ngôn ngữ mainstream |
| **Koka / Eff / Frank / Effekt** | Effect set được track trong type: `fun divide(x: int, y: int): exn int`. Compiler từ chối effect không có handler. | Research languages — "đáp án cuối cùng" mà industry đang tiến về |
| **Rust** | `Future` là *description* của async computation; `.await` là yield-point cho executor (Tokio). Ownership = compile-time supervisor — eliminate data race không bằng GC mà bằng kiểu. | Zero-cost algebraic effects qua type system thay vì runtime |
| **Pony** | Reference capabilities (`iso`, `val`, `ref`, `box`, `tag`, `trn`) — formal proof (Clebsch et al.) cho phép truyền mutable data giữa actors không data race. Per-actor GC, không stop-the-world. | Actor model + type-level capabilities = BEAM với compile-time safety |
| **Kotlin Coroutines + Structured Concurrency** | `CoroutineScope { }` enforce: child phải hoàn thành/cancel trước khi parent thoát. Tương đồng mạnh với OTP supervisor tree nhưng lifetime-scoped thay vì process-scoped. | OTP supervision trong JVM ecosystem, nhưng không có crash isolation ở mức process |
| **Swift Concurrency (5.5+)** | `actor` keyword first-class, `@MainActor` là global handler cho UI effects, `Sendable` types ≈ Pony capabilities. | Actor model + algebraic effects vào Apple ecosystem |
| **Go** | Goroutines + channels = CSP (Hoare 1978). Channel direction trong type (`chan<-`, `<-chan`). Không có supervisor first-class — phải tự build với `errgroup`/`context.Context`. | CSP thay vì actor — cùng triết lý không share memory, khác biệt cốt lõi ở đơn vị định danh và timing |

**CSP (Go) vs Actor Model (Erlang/Elixir) — khác biệt cốt lõi:**

Cả hai đều từ chối shared memory và dùng message passing, nhưng khác nhau ở hai điểm then chốt:

```text
Actor Model:                          CSP (Go):
────────────────────────────          ──────────────────────────────
Định danh PROCESS (PID)               Định danh CHANNEL
  → "Gửi message cho process X"         → "Gửi vào channel C"
  → Receiver luôn có mailbox             → Channel có thể buffered hoặc không

Bất đồng bộ mặc định:                Đồng bộ mặc định (rendezvous):
  sender không block                    sender block cho đến khi receiver ready
  message xếp hàng trong mailbox        unbuffered channel = hai bên phải gặp nhau

Decoupling:                           Coupling:
  sender biết địa chỉ receiver          sender và receiver biết cùng một channel
  nhưng không biết receiver đang làm gì  phải coordinate qua channel chung
```

Hệ quả thực tế: Actor model dễ build distributed system hơn (PID hoạt động xuyên node), CSP dễ express synchronization patterns hơn (select trên nhiều channels). Go chọn CSP vì Rob Pike đã làm Newsqueak/Limbo trước đó; Erlang chọn Actor vì yêu cầu fault isolation tuyệt đối giữa các cuộc gọi điện thoại. Cả hai đều tránh được race condition, nhưng theo cơ chế khác nhau.

**Quan sát quan trọng:** Các runtime *trẻ nhất* (Swift Concurrency 2021, Kotlin structured concurrency 2019, OCaml 5 effects 2022) đều **chủ động** thiết kế quanh ba trục algebraic effects + actor + IoC, vì cộng đồng đã nhận ra nhiều bug concurrency/lifecycle/DI là triệu chứng của cùng một bệnh gốc.

## 7.3. UI Frameworks: `UI = f(state)` xuyên platform

| Framework | Cơ chế state/effect | Đặc điểm phân biệt |
| --- | --- | --- |
| **Elm TEA** | `Model`, `Cmd msg`, `Sub msg` — pure, no hidden state | Algebraic effects thuần khiết nhất; runtime là handler duy nhất |
| **Redux / Flux** | Pure reducer `(state, action) -> state`; middleware là effect handler | TEA cho JS, mất type safety. Saga dùng generator = algebraic effect handler |
| **SolidJS** | Signals: `createSignal`, `createEffect`, `createMemo` | Fine-grained reactivity — component chạy một lần, DOM update tại điểm signal consume |
| **Vue Composition API** | `ref`, `reactive`, `watchEffect`, `onMounted` | Reactivity qua Proxy; gần SolidJS hơn React về tracking |
| **Angular Signals** | Từ Zone.js (implicit IoC) sang explicit signals (explicit reactivity) | Rời bỏ monkey-patch, về gần algebraic effects hơn |
| **Svelte 5 Runes** | `$state`, `$derived`, `$effect` — compiler-driven | Signals compile thành DOM update, không VDOM overhead |
| **SwiftUI** | `@State`, `@Binding`, `@ObservableObject`, `@EnvironmentObject`, `.task {}` | DI qua environment = `useContext`; `.task` = `useEffect` với structured concurrency |
| **Jetpack Compose** | `mutableStateOf`, `remember`, `LaunchedEffect`, `DisposableEffect` | Unidirectional Data Flow + State Hoisting = TEA dưới tên Android |
| **Flutter** | `StatefulWidget` + `setState`; Riverpod/BLoC cho DI | Imperative-leaning nhưng ecosystem push về unidirectional |

## 7.4. Distributed Systems & Infrastructure: cùng thuật toán, quy mô khác

**Thuật toán lõi** chạy xuyên suốt Kubernetes, React, OTP supervisor, AWS Lambda, Temporal:

```text
loop {
  desired = pure_function(current_state, inbox/events)
  diff    = compute_diff(current_actual, desired)
  apply_effects(diff)          // managed by runtime/orchestrator
  current_actual = observe_world()
}
```

| Hệ thống | Biểu hiện của thuật toán |
| --- | --- |
| **Kafka + Consumer Groups** | Partition = mailbox actor; offset = persistent state; Kafka Streams KTable = event sourcing pipeline |
| **AWS Lambda / Cloud Functions** | `(event, context) -> response` = pure handler; Lambda service = orchestrator; retry/DLQ = supervisor strategy |
| **Kubernetes Controllers** | Desired state (YAML) = declarative description; reconciliation loop = render loop của React; etcd = fiber state |
| **Temporal / Cadence** | Workflow code là pure deterministic; Temporal runtime lo persistent execution, replay, timer, retry. **Algebraic effects + event sourcing + supervision** đóng gói thành infrastructure product |
| **Event Sourcing + CQRS** | `(state, event) -> state` = GenServer model scale lên persistence; read model = `view` của TEA |
| **Microsoft Durable Functions** | Generator `yield` per step = explicit continuation để runtime checkpoint — CPS tường minh |
| **Akka / Akka Typed** | `Behavior[Command]` — typed actor; ZIO/Cats Effect đem algebraic effects vào Scala |

## 7.5. Game Engines: Hollywood Principle ở 60FPS

Unity MonoBehaviour có thể được dạy như "GenServer cho GameObject":

| Unity | GenServer / OTP |
| --- | --- |
| `Awake()` | `init/1` |
| `Start()` | `handle_continue/2` |
| `Update()` | `handle_info(:tick, state)` |
| `OnDestroy()` | `terminate/2` |
| `GetComponent<T>()` | `useContext` / Registry lookup |
| GameObject hierarchy | Supervision tree |

**Bevy ECS (Rust)** đi xa hơn: systems là pure functions over component sets; framework drive scheduler (parallel, ordered). Về triết lý rất gần Elm Architecture — data-oriented, system functions thuần, app runner là handler duy nhất.

**PLC / IEC 61131-3** (industrial control từ thập niên 1970): bạn mô tả ladder logic (declarative), runtime PLC scan loop drives theo chu kỳ cố định. Đây là Hollywood Principle ở embedded trước khi thuật ngữ tồn tại.

## 7.6. Unix Philosophy: tiền thân kiến trúc từ 1978

Doug McIlroy (1978): *"Write programs that do one thing and do it well. Write programs to work together. Write programs to handle text streams."*

Unix pipe (`|`) là **inter-process message-passing thuần khiết**: stdin/stdout là mailbox; mỗi tiến trình là actor; kernel là supervisor. Microservices, Kafka, Kubernetes pods là Unix philosophy mở rộng lên quy mô phân tán. Shell là framework; tiến trình của bạn là user code; shell quyết định fork, exec, reap — bạn không control nó.

Joe Armstrong thiết kế Erlang công khai theo Unix philosophy: *"The world is concurrent. Things in the world don't share data. Things communicate with messages. Things fail."* — tổng quát hóa từ tiến trình OS (MB) xuống tiến trình BEAM (300 bytes).

---

**Trước:** [← Phần 6 — Trade-offs](tradeoffs.md) | **Tiếp theo:** [Phần 8 — Phụ lục →](bibliography.md)
