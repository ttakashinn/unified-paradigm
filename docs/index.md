# Bản đồ tư duy thống nhất: React Hiện đại, Elixir/OTP, Inversion of Control và Landscape Rộng hơn

> *"Phần mềm tốt không phải là phần mềm dùng công cụ đúng — mà là phần mềm mà mọi lớp của nó đều phục tùng cùng một triết lý."*

---

## Phần 0 — Nền tảng lý thuyết: Ba dòng tư tưởng hội tụ

Để hiểu tại sao React Hooks, Elixir GenServer, và Inversion of Control lại *giống nhau về bản chất*, cần truy ngược về ba dòng tư tưởng độc lập đã hình thành trong 50 năm qua — mỗi dòng xuất phát từ một bài toán khác nhau, nhưng cùng đi đến một câu trả lời:

> **"Code của người dùng là mô tả khai báo của *what*. Runtime/framework là người duy nhất nắm quyền điều phối *when* và *how*."**

---

### Dòng 1 — Tách mô tả khỏi diễn giải: từ lambda calculus đến algebraic effects

#### Bài toán gốc: làm thế nào để lý luận về chương trình có side effects?

Vào thập niên 1930, Alonzo Church xây dựng **lambda calculus** — một hệ thống tối giản mô tả tính toán thuần bằng hàm và ứng dụng hàm. Lambda calculus là nền tảng của mọi ngôn ngữ lập trình hàm, nhưng nó *không có side effects*: một hàm chỉ nhận đầu vào và trả đầu ra, không in ra màn hình, không đọc file, không thay đổi trạng thái toàn cục.

Thực tế thì chương trình cần side effects. Câu hỏi mà cộng đồng lý thuyết CS vật lộn suốt thập niên 1970–1990 là: **làm thế nào để tích hợp side effects vào một ngôn ngữ thuần hàm mà không phá vỡ khả năng lý luận toán học về chương trình?**

#### Denotational semantics: ý nghĩa trước, thực thi sau

Christopher Strachey và Dana Scott (thập niên 1970) đề xuất **denotational semantics**: thay vì định nghĩa ngôn ngữ lập trình bằng cách mô tả *cách máy thực thi từng lệnh* (operational semantics), hãy định nghĩa nó bằng cách gán cho mỗi chương trình một **đối tượng toán học** — denotation — mô tả *ý nghĩa* của nó. Chương trình `2 + 3` có denotation là số `5`, không phải "cộng hai register rồi lưu vào register thứ ba".

Điều này có hệ quả sâu xa: nếu hai chương trình có cùng denotation, chúng **hoán đổi được** mà không thay đổi hành vi quan sát được từ ngoài. Đây là nền tảng của **referential transparency** — tính chất mà functional programming hiện đại kế thừa.

Nhưng side effects thì sao? Hàm `readFile("x.txt")` không thể có denotation là một giá trị cố định, vì nó phụ thuộc vào trạng thái file system. Strachey và Scott giải quyết bằng cách mở rộng denotation: thay vì `String`, denotation của `readFile` là một **hàm từ trạng thái thế giới sang (String, trạng thái thế giới mới)**. State được truyền tường minh qua tham số — không có mutation ngầm.

#### Continuation-Passing Style: đặt tên cho "phần còn lại của tính toán"

**John C. Reynolds** (Carnegie Mellon, 1972) và Guy Steele (1978) hình thức hóa **Continuation-Passing Style (CPS)**: thay vì hàm *trả về* giá trị, nó nhận thêm một tham số `k` (continuation) là "phần còn lại của tính toán" và *gọi `k` với kết quả*. CPS biến mọi hàm thành tail call, lộ rõ luồng điều khiển, và — quan trọng hơn — cho phép *chặn* và *thay thế* continuation.

```
// Trực tiếp
function add(x, y) { return x + y; }

// CPS — caller truyền vào "tôi sẽ làm gì với kết quả"
function add_cps(x, y, k) { k(x + y); }
add_cps(2, 3, result => console.log(result)); // k = phần còn lại
```

Delimited continuations (Felleisen, Danvy, thập niên 1980) cho phép "chặn" continuation tại một điểm cụ thể thay vì toàn bộ stack — đây là primitive tối thiểu cần thiết để implement bất kỳ control flow nào: exceptions, generators, async/await, coroutines, backtracking.

#### Monads: đóng gói "tính toán có ngữ cảnh"

Eugenio Moggi (1989) nhận ra rằng tất cả các dạng side effect (state, exception, IO, non-determinism, continuations) đều có thể được mô hình hóa bằng **monads** — một cấu trúc đại số từ category theory. Philip Wadler (1992) popularize monads vào lập trình hàm qua Haskell.

Ý tưởng cốt lõi của monad: thay vì `f: A -> B`, hàm có side effect có kiểu `f: A -> M B` trong đó `M` là "container" đóng gói hiệu ứng. `M = IO` cho I/O, `M = Maybe` cho nullable, `M = State S` cho stateful computation, `M = Either E` cho exception. Hai phép toán `return` (đưa giá trị thuần vào container) và `bind/>>= ` (chuỗi hai computation có effect) là đủ để compose mọi computation.

**Hệ quả quan trọng**: IO monad trong Haskell có nghĩa là một hàm `IO String` không *thực hiện* I/O — nó *mô tả* một I/O action. Haskell runtime là entity duy nhất thực sự thực thi nó, khi nó xuất hiện trong `main`. Đây là IoC ở mức kiểu — *framework (runtime Haskell) gọi code của bạn, không phải bạn gọi I/O trực tiếp*.

Tuy nhiên, monads có nhược điểm khi kết hợp: hai monads không tự nhiên composable. `IO (State S A)` và `State S (IO A)` là khác nhau và không dễ ghép với nhau. Monad transformers (`mtl`) giải quyết một phần nhưng tạo ra boilerplate phức tạp. Đây là vấn đề mà algebraic effects giải quyết triệt để hơn.

#### Algebraic Effects & Handlers: tổng quát hóa cuối cùng

Gordon Plotkin và John Power (2002) nhận ra rằng các computational effects (state, exception, non-determinism, I/O...) đều là **free algebras** — cấu trúc đại số tối giản sinh ra từ một tập operations. Plotkin và Matija Pretnar (2009) thêm **handlers**: một effect handler là homomorphism — ánh xạ từ algebraic structure của effects sang một interpretation cụ thể.

Thực hành, điều này có nghĩa:

```
// Code khai báo effect (perform operation):
perform State.get()        // "tôi cần giá trị state hiện tại"
perform State.set(42)      // "tôi muốn đặt state = 42"
perform Log.write("msg")   // "tôi muốn ghi log"

// Handler ở tầng trên quyết định nghĩa của mỗi effect:
handler {
  State.get()   -> resume(current_state)          // trả về state hiện tại, tiếp tục
  State.set(s)  -> current_state = s; resume()    // cập nhật state, tiếp tục
  Log.write(m)  -> println(m); resume()           // in ra stdout, tiếp tục
}
```

**Điểm mấu chốt**: `perform` không thực hiện gì cả — nó *yêu cầu* (raise) một effect. Handler *giải thích* (interpret) effect đó. Cùng một code có thể chạy với nhiều handler khác nhau: handler production thực sự ghi vào database; handler test chỉ ghi vào in-memory list; handler replay đọc từ event log. Đây là **dependency injection ở mức ngữ nghĩa**, không cần container, không cần reflection.

Quan trọng hơn: handlers có thể *resume* computation nhiều lần (non-determinism), hoặc không resume (exception), hoặc resume với delay (async). Điều này cho phép implement mọi control flow bằng một primitive duy nhất.

**Liên hệ trực tiếp với React Hooks**: Sebastian Markbåge viết công khai *"Conceptually, Hooks are algebraic effects."* Khi component gọi `useState(0)`, nó không tạo state — nó *performs* một effect "tôi cần slot state thứ k". React fiber (handler) giải quyết bằng cách trả về giá trị từ slot đã lưu. Vì JavaScript không có delimited continuations, React giả lập bằng dispatcher mutable + thứ tự gọi cố định (Rules of Hooks). `useEffect` là `perform Effect.subscribe(fn, deps)` — React handler schedule cleanup và re-run đúng lúc.

Elm Architecture `Cmd msg` và `Sub msg` là algebraic effects ở dạng **explicit values**: thay vì perform một effect, code *trả về* một data structure mô tả effect. Elm runtime là handler duy nhất. Đây là dạng "Free Monad" — biểu diễn effect như AST rồi interpret sau.

Các ngôn ngữ implement algebraic effects native: **Koka** (Daan Leijen, Microsoft Research), **Eff** (Bauer & Pretnar), **Frank**, **Effekt**, **OCaml 5.x** (từ phiên bản 5.3, effect handlers là cú pháp chính thức), **Unison**. Rust's `async/await` là một dạng hạn chế — `Future` là description của async computation, executor là handler.

---

### Dòng 2 — Tính toán qua trao đổi thông điệp: từ bài toán concurrency đến actor model

#### Bài toán gốc: làm thế nào để lý luận về concurrency?

Vào thập niên 1960–1970, khi các hệ thống máy tính bắt đầu chạy nhiều tác vụ đồng thời, xuất hiện một lớp lỗi hoàn toàn mới: **race conditions**, **deadlocks**, **livelocks**. Shared mutable state — vùng nhớ mà nhiều luồng cùng đọc/ghi — là nguồn gốc của tất cả.

Giải pháp truyền thống: **locks** (mutex, semaphore). Nhưng locks có vấn đề cơ bản: composition bị phá vỡ. Hai đoạn code, mỗi đoạn đúng khi dùng lock riêng, có thể deadlock khi ghép lại nếu lấy lock theo thứ tự ngược nhau. Không có công cụ toán học nào để lý luận về một chương trình dùng locks phức tạp — chỉ có thể test và hy vọng.

#### Actor Model: cô lập là giải pháp, không phải lock

Carl Hewitt, Peter Bishop, và Richard Steiger (1973) đề xuất **Actor Model** như một câu trả lời triệt để: **không chia sẻ memory; giao tiếp chỉ qua message**. Mỗi *actor* có:

- **Mailbox** — hàng đợi message, chỉ actor đó có thể đọc
- **Behavior** — hàm xử lý message kế tiếp, có thể thay đổi cho message sau
- **Address** — định danh để nhận message, không phải pointer vào memory
- Ba khả năng khi nhận message: gửi message tới actors khác, spawn actors mới, thay đổi behavior cho message kế tiếp

**Điểm then chốt**: không có cơ chế nào để một actor đọc trực tiếp state của actor khác. Cách duy nhất là gửi message hỏi, và actor kia có thể *chọn không trả lời*. Không có shared memory → không có race condition → không cần lock.

Gul Agha (1986) xây dựng **transition semantics** cho actor model — nền tảng toán học để lý luận về liveness (hệ thống cuối cùng sẽ tiến về phía trước) và safety (hệ thống không bao giờ ở trạng thái xấu). Robin Milner sau đó chỉ ra mối quan hệ giữa actor model và **π-calculus** — một hệ thống process algebra có thể mô hình hóa communication channels như values có thể truyền đi.

#### Erlang/OTP: actor model gặp fault tolerance

Năm 1986, Joe Armstrong, Robert Virding và Mike Williams tại Ericsson bắt đầu xây dựng Erlang — ngôn ngữ cho hệ thống viễn thông đòi hỏi **99.9999999% uptime** (nine nines = dưới 31ms downtime/năm). Yêu cầu này định hình toàn bộ thiết kế:

- **Không thể dùng shared memory**: một bug trong một cuộc gọi điện thoại không được ảnh hưởng đến cuộc gọi khác. Vì vậy mỗi cuộc gọi là một process riêng biệt với heap riêng.
- **Không thể viết defensive code đủ tốt**: phần cứng hỏng, mạng gián đoạn, memory corruption xảy ra. Giải pháp: *để process crash, supervisor restart về trạng thái sạch*. Armstrong gọi đây là **"let it crash"** philosophy.
- **Cần hot code swap**: không thể dừng hệ thống viễn thông để upgrade phần mềm. Vì vậy BEAM hỗ trợ thay thế code đang chạy mà không dừng process.

OTP (Open Telecom Platform) là bộ thư viện và conventions đóng gói các pattern tái sử dụng được từ kinh nghiệm thực tế: `gen_server` (client-server state machine), `gen_statem` (finite state machine), `supervisor` (fault recovery tree), `application` (lifecycle của cả ứng dụng).

**`gen_server` là hiện thân của actor model với một pattern rõ ràng**: bạn implement các callback thuần `init/1`, `handle_call/3`, `handle_cast/2`, `handle_info/2` — OTP runtime lo mọi thứ còn lại (receive loop, timeout, monitoring, distributed registration). Callback của bạn là *mô tả khai báo*; OTP là *handler*.

**Liên hệ với Dòng 1**: OTP behaviour chính là IoC (Dòng 3) áp dụng cho actor model — framework gọi callback của bạn khi nhận message, bạn không gọi receive loop. Và mỗi callback `handle_call/3` là một pure function `(msg, state) -> (reply, newState)` — tách mô tả (what) khỏi diễn giải (when/how) theo đúng tinh thần của algebraic effects.

#### Tại sao không phải locks + threads?

Lý do actor model vượt trội trong fault-tolerant distributed systems:

**Locks không compose**: như đã nói, đúng từng phần không có nghĩa là đúng khi ghép lại. Actor model compose tự nhiên — hai actor đúng vẫn đúng khi giao tiếp với nhau, vì họ chỉ trao đổi message.

**Locks không survive failure**: khi một thread giữ lock rồi crash, lock không bao giờ được release — deadlock toàn hệ thống. Actor crash không ảnh hưởng actor khác — mailbox của nó đơn giản biến mất, sender nhận thông báo DOWN nếu đang monitor.

**Locks không distribute**: trên nhiều máy không có shared memory, lock là khái niệm vô nghĩa. Actor model hoạt động y hệt dù actor ở cùng node hay khác node — message passing là primitive của cả single-node và distributed.

---

### Dòng 3 — Đảo ngược luồng điều khiển: từ module coupling đến Hollywood Principle

#### Bài toán gốc: làm thế nào để tái sử dụng code?

Suốt thập niên 1970–1980, cộng đồng kỹ thuật phần mềm vật lộn với một câu hỏi thực tế: tại sao code tái sử dụng được (reusable) lại khó viết đến vậy? Một library sort, một library HTTP client — những thứ này tái sử dụng được. Nhưng một framework UI, một framework web server — những thứ này thường *không* tái sử dụng được: bạn bị kẹt trong một kiến trúc nhất định.

Michael Jackson (không phải ca sĩ) nhận ra trong **Jackson Structured Programming** (thập niên 1970) rằng khi cấu trúc dữ liệu đầu vào và đầu ra không khớp, chương trình phải có một điểm "inversion" — nơi control flow bị lật ngược. Đây là quan sát sớm nhất về IoC, dù chưa có tên.

#### Sự phân biệt giữa library và framework

Thập niên 1980, khi object-oriented programming nổi lên và các framework lớn đầu tiên xuất hiện (Smalltalk MVC, MacApp, ET++), nhà nghiên cứu Ralph Johnson và Brian Foote viết bài *"Designing Reusable Classes"* (1988) — lần đầu tiên đặt tên **Inversion of Control**:

> *"One important characteristic of a framework is that the methods defined by the user to customize the framework will often be called from within the framework itself, rather than from the user's application code. The framework often plays the role of the main program in coordinating and sequencing application activity. This inversion of control gives frameworks the power to serve as extensible skeletons."*

Đây là phân biệt then chốt giữa **library** và **framework**:
- **Library**: bạn gọi code của library. Bạn nắm control flow.
- **Framework**: framework gọi code của bạn. Framework nắm control flow.

IoC không phải technique — đó là **đặc tính định nghĩa** của framework. Nếu bạn gọi framework, đó là library. Nếu framework gọi bạn, đó là framework.

#### Hollywood Principle: "Don't call us, we'll call you"

Michael Sweet (Mesa language, Xerox PARC, 1983) đặt tên dễ nhớ hơn: **Hollywood's Law** — ví von như cách Hollywood casting: "Đừng gọi cho chúng tôi, chúng tôi sẽ gọi cho bạn."

Martin Fowler sau đó hệ thống hóa và popularize thuật ngữ **Inversion of Control** trong cộng đồng Java/enterprise thập niên 2000, phân biệt hai specializations:

**IoC ở tầng flow** (nghĩa gốc): framework drive main loop, user code là callback. Ví dụ điển hình:
- GUI event loop (Windows WndProc, Swing ActionListener, React onClick)
- Web framework request handler (Rails action, Phoenix controller, Express route handler)
- Game engine lifecycle (Unity Awake/Start/Update)
- Template Method pattern (định nghĩa skeleton, subclass override steps)

**Dependency Injection** (specialization): thay vì user code tạo dependencies, framework *tiêm* chúng vào. Martin Fowler giải thích rằng DI là một *cách thực hiện* IoC, không phải IoC nói chung — nhưng cộng đồng Java (đặc biệt Spring) đã chiếm dụng thuật ngữ IoC để chỉ riêng DI, gây nhầm lẫn kéo dài đến nay.

#### Tại sao IoC tạo ra reusability?

Lý do sâu hơn: khi user code gọi dependencies trực tiếp, nó bị **coupled** với implementation cụ thể. Coupling này ngăn test (không thể swap real database với in-memory), ngăn extension (phải sửa source để thay đổi behavior), ngăn reuse (mang code sang context khác phải mang theo cả dependency tree).

IoC giải quyết bằng cách **đảo ngược dependency**: thay vì code biết về framework, framework biết về interface của code. Code chỉ implement interface (callback signature, behaviour protocol, hook contract) — không import, không khởi tạo, không call. Framework lookup implementation tại runtime và inject hoặc invoke.

**Liên hệ với Dòng 1**: khi OTP behaviour `use GenServer` transform module của bạn, nó đang làm IoC ở tầng flow — OTP sẽ gọi `init/1`, `handle_call/3` của bạn đúng lúc, bạn không gọi receive loop. Đây chính là Hollywood Principle. Và như đã phân tích ở Dòng 1, mỗi callback là một pure function — tách mô tả khỏi diễn giải.

**Liên hệ với Dòng 2**: Actor model và IoC giải cùng một vấn đề coupling từ hai góc. Actor model: đừng share memory, share message thay vào đó — coupling qua interface (message type), không qua implementation. IoC: đừng gọi framework, để framework gọi bạn — coupling qua interface (callback contract), không qua orchestration logic.

---

### Hội tụ: tại sao cả ba dòng đến cùng một câu trả lời

Ba dòng tư tưởng xuất phát từ ba bài toán khác nhau:
- Dòng 1 hỏi: *làm thế nào để lý luận toán học về chương trình có side effects?*
- Dòng 2 hỏi: *làm thế nào để xây dựng hệ thống concurrent/distributed không có race condition và tự phục hồi sau lỗi?*
- Dòng 3 hỏi: *làm thế nào để viết code tái sử dụng được và dễ mở rộng?*

Ba câu trả lời hội tụ về **cùng một kiến trúc**:

```
Tách "mô tả khai báo" (pure, composable, testable)
khỏi "thực thi có side effect" (runtime, framework, scheduler)
bằng cách giao toàn bộ quyền điều phối cho một handler/runtime/orchestrator,
để code của người dùng chỉ là pure function tại các điểm móc nối được định nghĩa trước.
```

Khi Sebastian Markbåge gọi Hooks là "algebraic effects" (Dòng 1), khi Joe Armstrong giải thích triết lý "let it crash" qua actor model (Dòng 2), khi Martin Fowler mô tả framework qua Hollywood Principle (Dòng 3) — họ đang mô tả *cùng một hình học*, chỉ khác ngôn ngữ và ngữ cảnh.

React Hooks là Dòng 1 + Dòng 3 áp dụng cho UI layer trong single-threaded JavaScript. Elixir GenServer là Dòng 1 + Dòng 2 + Dòng 3 áp dụng cho concurrent distributed system trên BEAM. Phoenix LiveView là điểm mà cả ba hội tụ thành một primitive duy nhất.

Phần còn lại của tài liệu đi vào implementation cụ thể của từng hội tụ này.

---

## Sơ đồ tổng quan

```
╔══════════════════════════════════════════════════════════════════════╗
║                    TRIẾT LÝ LẬP TRÌNH THỐNG NHẤT                    ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║      ┌─────────────┐         ┌─────────────┐                        ║
║      │    REACT    │         │  ELIXIR/OTP │                        ║
║      │   HOOKS     │         │  GENSERVER  │                        ║
║      │             │         │             │                        ║
║      │ • useState  │         │ • init/1    │                        ║
║      │ • useReducer│         │ • handle_*  │                        ║
║      │ • useEffect │    ▼    │ • Supervisor│                        ║
║      │ • useContext│         │ • Behaviour │                        ║
║      └──────┬──────┘         └──────┬──────┘                        ║
║             │                       │                                ║
║             └──────────┬────────────┘                                ║
║                        │                                             ║
║                        ▼                                             ║
║              ┌─────────────────┐                                     ║
║              │  GIAO ĐIỂM CỐT  │                                     ║
║              │      LÕI        │                                     ║
║              │                 │                                     ║
║              │ Pure Transition │◄── Inversion of Control             ║
║              │ + Framework-    │    (Hollywood Principle)            ║
║              │ Driven Control  │                                     ║
║              └────────┬────────┘                                     ║
║                       │                                              ║
║                       ▼                                              ║
║            ┌──────────────────────┐                                  ║
║            │  Phoenix LiveView    │                                  ║
║            │  (Tổng hợp hoàn hảo) │                                  ║
║            │ GenServer + React UI │                                  ║
║            │   trong 1 primitive  │                                  ║
║            └──────────────────────┘                                  ║
╚══════════════════════════════════════════════════════════════════════╝

    BA TRỤ TRIẾT LÝ XUYÊN SUỐT
    ────────────────────────────
    [1] Tách "WHAT" khỏi "WHEN":   Pure logic  ←→  Runtime orchestration
    [2] Pure Core, Impure Shell:   Business    ←→  Side effects (rìa ngoài)
    [3] State qua tham số:         newState    ←→  Không mutation tại chỗ
```

---

---

**Tiếp theo:** [Phần 1 — React hiện đại dưới lăng kính Functional Programming →](react.md)
