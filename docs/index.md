# Bản đồ tư duy thống nhất: React Hiện đại, Elixir/OTP, Inversion of Control và Landscape Rộng hơn

> *“Phần mềm tốt không phải là phần mềm dùng công cụ đúng — mà là phần mềm mà mọi lớp của nó đều phục tùng cùng một triết lý.”*

-----

## Phần 0 — Nền tảng lý thuyết: Ba dòng tư tưởng hội tụ

React Hooks, Elixir GenServer, và Inversion of Control giống nhau về bản chất — không phải ngẫu nhiên, mà vì chúng đều là câu trả lời cho **cùng một nhóm bài toán cốt lõi** của ngành phần mềm. Để hiểu tại sao, cần truy ngược ba dòng tư tưởng đã hình thành trong khoảng 80–90 năm qua, mỗi dòng khởi phát từ một khủng hoảng kỹ thuật riêng, nhưng cùng hội tụ về một kiến trúc chung.

Ba dòng này phát triển song song trong những cộng đồng học thuật và kỹ thuật khác nhau — đôi khi giao nhau, đôi khi không — nhưng cuối cùng đều cho ra cùng một câu trả lời cho ba câu hỏi tưởng chừng riêng biệt. Sự hội tụ về kiến trúc này không phải là kết quả của một phong trào thống nhất; nó là dấu hiệu rằng *tồn tại một câu trả lời tối ưu duy nhất* cho cả ba bài toán.

> **“Code của người dùng là mô tả khai báo của *what*. Runtime/framework là người duy nhất nắm quyền điều phối *when* và *how*.”**

-----

### Dòng 1 — Tách mô tả khỏi diễn giải: từ lambda calculus đến algebraic effects

#### Bối cảnh: máy tính đầu tiên và câu hỏi nền tảng

Thập niên 1940–1950, máy tính điện tử đầu tiên xuất hiện. Chương trình được viết dưới dạng lệnh máy hoặc assembly — mỗi lệnh là một thao tác cụ thể trên phần cứng: copy register, cộng hai ô nhớ, nhảy đến địa chỉ nào đó. Không có khái niệm “hàm” hay “module” — chỉ là một chuỗi lệnh chạy tuần tự.

Câu hỏi đặt ra ngay từ đầu: **làm thế nào để lý luận về tính đúng đắn của chương trình?** Với assembly, câu trả lời gần như là “không thể” — quá nhiều trạng thái ngầm, quá nhiều lệnh nhảy tuỳ tiện, không có cách nào chứng minh toán học rằng chương trình làm đúng điều nó phải làm.

Alonzo Church đã xây dựng **lambda calculus** từ thập niên 1930 như một hệ thống toán học mô tả tính toán thuần bằng hàm và ứng dụng hàm — không có trạng thái, không có lệnh, không có thứ tự thực thi ngầm. Một hàm trong lambda calculus chỉ làm một việc: nhận đầu vào, trả đầu ra, không có gì khác. Đây là nền tảng toán học cho phép *chứng minh* tính đúng đắn của chương trình.

Nhưng ngay lập tức xuất hiện mâu thuẫn thực tế: chương trình thực sự cần **side effects** — đọc file, in ra màn hình, thay đổi database. Lambda calculus thuần không thể mô hình hoá những việc này. Và nếu thêm side effects vào tuỳ tiện (như C hay Fortran làm), khả năng lý luận toán học về chương trình bị phá vỡ hoàn toàn.

**Đây là khủng hoảng đầu tiên của Dòng 1**: làm sao vừa có tính expressiveness của ngôn ngữ thực tế (side effects), vừa giữ được khả năng lý luận toán học của lambda calculus?

#### Denotational semantics: ý nghĩa trước, thực thi sau (thập niên 1970)

Peter Landin (1966) là người đầu tiên nhìn thấy rõ vấn đề khi ông viết bài “The Next 700 Programming Languages” — lập luận rằng mọi ngôn ngữ lập trình đều có thể được hiểu như một “coating” bên ngoài của một lambda calculus cốt lõi. Nhưng Landin chưa giải quyết được bài toán side effects.

Christopher Strachey và Dana Scott (thập niên 1970) đề xuất **denotational semantics** — một hướng tiếp cận triệt để hơn: thay vì mô tả ngôn ngữ lập trình bằng cách giải thích *cách máy thực thi từng lệnh* (operational semantics), hãy gán cho mỗi chương trình một **đối tượng toán học thuần** — denotation — mô tả *ý nghĩa* của nó.

Chương trình `2 + 3` có denotation là số nguyên `5`. Hàm `double` có denotation là hàm toán học `x → 2x`. Hai chương trình có cùng denotation thì hoán đổi được — đây là **referential transparency**, nền tảng của functional programming hiện đại.

Nhưng side effects thì sao? Hàm `readFile("x.txt")` không thể có denotation là một giá trị cố định vì nó phụ thuộc vào file system tại thời điểm gọi. Strachey và Scott giải quyết bằng cách mở rộng mô hình: denotation của `readFile` là một **hàm nhận vào toàn bộ trạng thái hiện tại của hệ thống và trả về cặp `(kết quả, trạng thái mới)`**. State được truyền tường minh qua tham số thay vì ẩn trong mutation toàn cục — không có gì ngầm, không có gì bí ẩn.

Đây là bước ngoặt quan trọng về mặt tư duy: **side effect không còn là thứ “xảy ra” bên ngoài hàm — nó trở thành một phần của kiểu trả về**. Hàm không “làm” I/O; nó *mô tả* một phép tính mà khi được diễn giải bởi runtime, sẽ thực hiện I/O.

#### Continuation-Passing Style: lộ rõ luồng điều khiển (1972–1978)

Thập niên 1970, khi ngôn ngữ bậc cao (LISP, Algol) bắt đầu phổ biến, xuất hiện một bài toán mới: làm thế nào để implement exceptions, coroutines, generators — những cấu trúc điều khiển phức tạp — trong một ngôn ngữ chỉ có hàm và gọi hàm?

**John C. Reynolds** (1972) và Guy Steele (1978) nhận ra rằng mọi chương trình đều có thể được biến đổi sang **Continuation-Passing Style (CPS)**: thay vì hàm *trả về* giá trị, nó nhận thêm một tham số `k` (continuation) — “phần còn lại của tính toán sau bước này” — và *gọi `k` với kết quả*.

```javascript
// Trực tiếp
function add(x, y) { return x + y; }
const result = add(2, 3);           // result = 5
doSomethingElse(result);

// CPS — "phần còn lại" được truyền tường minh
function add_cps(x, y, k) { k(x + y); }
add_cps(2, 3, result => doSomethingElse(result));
```

Thoạt nhìn CPS có vẻ phức tạp hơn không cần thiết. Nhưng hệ quả sâu xa của nó là: **luồng điều khiển trở thành first-class value**. Khi continuation là một tham số như mọi tham số khác, bạn có thể lưu trữ nó, truyền đi, hoặc gọi nó nhiều lần — đây là cơ chế nền tảng để implement exceptions (không gọi k), generators (gọi k nhiều lần), async/await (gọi k sau khi I/O hoàn thành), và backtracking.

Felleisen và Danvy (thập niên 1980) tinh chỉnh thêm với **delimited continuations** — cho phép “chặn” continuation tại một điểm cụ thể trong stack thay vì toàn bộ — và chứng minh rằng đây là primitive tối thiểu đủ để implement bất kỳ control flow nào. Đây là nền tảng lý thuyết cho `async/await` trong JavaScript, coroutines trong Python, và về mặt khái niệm, cho cả React Fiber.

#### Monads: đóng gói tính toán có ngữ cảnh (1989–1992)

Thập niên 1980, Haskell đang được thiết kế như một ngôn ngữ purely functional. Vấn đề: một ngôn ngữ không có side effects thì làm thế nào để đọc input từ keyboard hay ghi file? Mọi chương trình hữu ích đều cần I/O.

Eugenio Moggi (1989) nhận ra câu trả lời từ category theory: tất cả các dạng side effect — state, exception, I/O, non-determinism, continuations — đều có thể được mô hình hoá bằng một cấu trúc đại số gọi là **monad**. Philip Wadler (1992) “dịch” khái niệm này sang ngôn ngữ lập trình thực tế và giúp Haskell community hiểu cách dùng nó.

Ý tưởng cốt lõi: thay vì `f: A → B` (hàm thuần), hàm có side effect có kiểu `f: A → M B` trong đó `M` là “container” đóng gói hiệu ứng:

- `IO String` — tính toán trả về `String` thông qua I/O
- `Maybe Int` — tính toán có thể fail và trả về `Int` nếu thành công
- `State S Int` — tính toán thay đổi state kiểu `S` và trả về `Int`
- `Either Error Int` — tính toán có thể throw `Error` hoặc trả về `Int`

**Hệ quả quan trọng**: một hàm kiểu `IO String` trong Haskell không *thực hiện* I/O — nó *mô tả* một computation mà khi được diễn giải bởi Haskell runtime (trong `main`), sẽ thực hiện I/O. Đây chính là IoC ở mức kiểu: *runtime gọi code của bạn, bạn không gọi I/O trực tiếp*.

Tuy nhiên, monads có một nhược điểm nghiêm trọng: **không compose tự nhiên**. `IO (State S A)` và `State S (IO A)` là hai thứ khác nhau và không ghép được dễ dàng. Khi cần kết hợp nhiều effects (vừa có I/O vừa có state vừa có exception), phải dùng “monad transformer stack” — một cơ chế phức tạp, verbose, và khó debug. Đây là vấn đề mà cộng đồng Haskell vật lộn suốt thập niên 2000.

#### Algebraic Effects & Handlers: tổng quát hoá cuối cùng (2002–nay)

Gordon Plotkin và John Power (2002) tiếp cận bài toán từ góc độ đại số thuần tuý hơn: họ nhận ra rằng các computational effects đều là **free algebras** — cấu trúc đại số được sinh ra từ một tập operations, không cần thêm gì khác. Plotkin và Matija Pretnar (2009) bổ sung **handlers**: cơ chế để *diễn giải* các operations đó theo nhiều cách khác nhau.

Về mặt thực hành, điều này có nghĩa là bạn tách rời hai việc hoàn toàn:

```javascript
// Phần 1: Code CHỈ khai báo nó cần gì — không thực thi gì cả
perform State.get()        // "tôi cần giá trị state hiện tại"
perform State.set(42)      // "tôi muốn cập nhật state thành 42"
perform Log.write("msg")   // "tôi muốn ghi một dòng log"

// Phần 2: Handler ở tầng trên QUYẾT ĐỊNH mỗi operation có nghĩa là gì
handler {
  // Với handler production: ghi thật vào database
  State.get()   → resume(db.read())
  State.set(s)  → db.write(s); resume()

  // Với handler test: ghi vào in-memory map
  State.get()   → resume(testMap.get())
  State.set(s)  → testMap.put(s); resume()

  // Với handler replay: đọc từ event log đã record
  State.get()   → resume(eventLog.next())
}
```

`perform` không thực hiện gì cả — nó *yêu cầu* (raise) một effect và dừng lại chờ handler quyết định. Handler nhận yêu cầu, quyết định cách xử lý, rồi `resume` để tiếp tục computation với kết quả. Quan trọng: handler có thể `resume` nhiều lần (non-determinism), không `resume` (exception), hoặc `resume` sau một khoảng thời gian (async). Một primitive duy nhất giải quyết tất cả.

So với monads, algebraic effects **compose tự nhiên**: cùng một computation có thể dùng nhiều effects (State + Log + Async) mà không cần monad transformer stack. Effects là row — có thể thêm bớt độc lập như thêm bớt cột trong một bảng.

**Liên hệ với React**: Sebastian Markbåge viết công khai *“Conceptually, Hooks are algebraic effects.”* Khi component gọi `useState(0)`, nó `perform` một effect “tôi cần slot state thứ k”. React fiber (handler) quyết định trả về giá trị nào từ slot đó. Vì JavaScript không có delimited continuations native, React giả lập bằng dispatcher mutable + thứ tự gọi cố định (Rules of Hooks). `useEffect` là `perform Effect.subscribe(fn, deps)` — React handler schedule cleanup và re-run đúng thời điểm sau mỗi render.

Elm Architecture `Cmd msg` và `Sub msg` là algebraic effects ở dạng explicit values: code không `perform` mà *trả về* một data structure mô tả effect (như AST), Elm runtime là handler duy nhất interpret nó. Đây là cách tiếp cận “Free Monad” — an toàn hơn về mặt toán học, nhưng verbose hơn về mặt API.

-----

### Dòng 2 — Tính toán qua trao đổi thông điệp: từ bài toán concurrency đến actor model

#### Bối cảnh: máy tính đa nhiệm và khủng hoảng shared memory (thập niên 1960–1970)

Thập niên 1960, máy tính trở nên đủ mạnh để chạy nhiều chương trình cùng lúc — **time-sharing systems** xuất hiện. Đây là bước tiến lớn về mặt utilization, nhưng ngay lập tức bộc lộ một lớp bug hoàn toàn mới mà trước đây không tồn tại: **race conditions**, **deadlocks**, **livelocks**.

Nguồn gốc của tất cả là **shared mutable memory**: nhiều thread cùng đọc/ghi vào một vùng nhớ chung. Nếu thread A đang đọc một giá trị trong khi thread B đang ghi, kết quả là undefined. Nếu thread A giữ lock L1 và chờ lock L2, trong khi thread B giữ lock L2 và chờ lock L1, cả hai thread đứng chờ nhau mãi mãi.

Edsger Dijkstra (1965) đề xuất **semaphores** — primitive đầu tiên để synchronize threads. Tony Hoare (1974) đề xuất **monitors** — một cách tổ chức lock ở mức cao hơn. Nhưng cả hai đều gặp phải cùng một vấn đề cốt lõi mà không giải pháp nào dựa trên lock có thể thoát ra:

**Lock không compose**: hai đoạn code, mỗi đoạn đúng khi dùng lock riêng của nó, có thể deadlock khi ghép lại nếu chúng lấy lock theo thứ tự ngược nhau. Không có công cụ toán học nào để phát hiện hay ngăn chặn điều này ở compile time — chỉ có thể test và hy vọng.

Dijkstra đề xuất “Dining Philosophers Problem” (1965) như một bài toán minh hoạ cổ điển: năm triết gia ngồi quanh bàn tròn, mỗi người cần hai chiếc đũa (shared) để ăn, và nếu mỗi người cầm một đũa bên trái rồi chờ đũa bên phải, cả năm người đứng chờ nhau mãi mãi. Đây là deadlock được hình thức hoá thành bài toán học thuật — nhưng version thực tế của nó xảy ra hàng ngày trong các hệ thống production.

**Đây là khủng hoảng đầu tiên của Dòng 2**: không có mô hình lý luận toán học nào cho phép bạn viết chương trình concurrent đúng đắn theo cách có thể chứng minh được.

#### Hai con đường thoát: Actor Model và CSP (1973–1978)

Đứng trước khủng hoảng này, hai nhà khoa học máy tính đề xuất hai giải pháp triệt để khác nhau — cả hai đều dựa trên cùng một insight cốt lõi: **bỏ shared memory hoàn toàn, chỉ giao tiếp qua message**. Nhưng họ implement insight này theo hai cách rất khác nhau.

**Carl Hewitt (MIT, 1973) đề xuất Actor Model.** Bài báo “A Universal Modular Actor Formalism for Artificial Intelligence” (Hewitt, Bishop & Steiger) định nghĩa **Actor**: một thực thể tính toán có *mailbox riêng*, *heap riêng*, và *địa chỉ duy nhất* (không phải pointer). Actor xử lý message tuần tự từng cái một, có thể spawn actor mới, và có thể thay đổi behavior cho message kế tiếp. Quan trọng: bạn gửi message *cho một actor cụ thể* (định danh bằng địa chỉ), và message xếp hàng trong mailbox của actor đó.

```text
Actor Model — định danh PROCESS, async by default:

   Actor A                  Actor B
   ┌──────────┐    send     ┌──────────┐
   │ Mailbox  │   ────►     │ Mailbox  │
   │ [m1, m2] │             │ [msg]    │
   │ Heap     │             │ Heap     │
   └──────────┘             └──────────┘
   sender không block         receiver xử lý khi sẵn sàng
```

Bối cảnh của paper Hewitt rất quan trọng: ông đang nghiên cứu AI, cụ thể là cách mô hình hoá các agent thông minh trong hệ thống phân tán. Ông không chủ yếu nghĩ về “viết web server concurrent” — ông nghĩ về “làm thế nào để nhiều agent thông minh có thể cộng tác mà không tạo ra contradictions”. Nhưng giải pháp của ông hoá ra có tính phổ quát vượt xa domain AI.

**Tony Hoare (Oxford, 1978) đề xuất CSP** (Communicating Sequential Processes) — bài báo cùng tên. Hoare cũng từ chối shared memory, nhưng khác với Hewitt, ông không dùng mailbox. Thay vào đó, ông định nghĩa **channel**: một kênh giao tiếp được đặt tên, mà nhiều process có thể cùng đọc/ghi vào. Process gửi `c ! value` (gửi vào channel `c`), process khác nhận `c ? variable` (nhận từ channel `c`). Mặc định là **đồng bộ (rendezvous)**: sender block cho đến khi receiver sẵn sàng nhận.

```text
CSP — định danh CHANNEL, sync by default (rendezvous):

   Process P                Process Q
       │      channel c        │
       │      ──────────       │
       │  ──► c ! 42           │
       │  (block)              │
       │           c ? x  ◄──  │
       │  (unblock, both done) │
```

**Khác biệt cốt lõi:**

- **Actor**: định danh *process*, message bất đồng bộ qua mailbox, sender không biết khi nào receiver xử lý
- **CSP**: định danh *channel*, rendezvous đồng bộ, sender và receiver “gặp nhau” tại channel

Hai approach dẫn đến hai dòng kế thừa khác nhau: Actor Model → Erlang/OTP → Akka, Pony. CSP → occam → Go (channels của Go là CSP, không phải Actor).

Clinger (1981) bổ sung nền tảng toán học với denotational semantics cho actors. Agha (1986) xây dựng transition semantics đầy đủ. Robin Milner phát triển **π-calculus** — process algebra tổng quát có thể mô hình hoá channels như first-class values có thể truyền đi — và chứng minh π-calculus có thể nhúng cả Actor lẫn CSP, làm cầu nối lý thuyết giữa hai dòng.

#### Ericsson và Erlang: khi bài toán viễn thông gặp actor model (1986–1998)

Thập niên 1980, hệ thống viễn thông Ericsson đang phải đối mặt với ba yêu cầu tưởng như mâu thuẫn nhau:

1. **Concurrency cực cao**: một tổng đài điện thoại phải xử lý hàng nghìn cuộc gọi đồng thời
1. **Fault isolation tuyệt đối**: một bug trong một cuộc gọi không được làm sập cuộc gọi khác
1. **Zero downtime**: không thể dừng hệ thống để deploy phần mềm mới

Joe Armstrong, Robert Virding và Mike Williams ban đầu thử nghiệm với **Prolog** — ngôn ngữ logic mà Ericsson Computer Science Lab đang dùng làm research platform. Họ phát hiện Prolog có hai đặc tính quan trọng tình cờ giúp giải quyết bài toán: (1) immutability mặc định, (2) pattern matching mạnh trên message. Nhưng Prolog quá chậm cho production. Nhóm quyết định thiết kế lại từ đầu một ngôn ngữ kế thừa các đặc tính tốt của Prolog, thêm vào primitive cho concurrency dựa trên **Actor Model của Hewitt** — đó là Erlang.

Cùng với đó, BEAM VM được thiết kế từ đầu để hỗ trợ:

- Mỗi cuộc gọi điện thoại là một process riêng biệt với heap riêng → bug trong một cuộc gọi không ảnh hưởng cuộc gọi khác
- Process crash không làm sập hệ thống → supervisor process phát hiện và restart process con
- Hot code swap → thay thế module đang chạy mà không dừng VM

Yêu cầu **99.9999999% uptime** (nine nines = dưới 31ms downtime/năm) không phải là marketing — đó là SLA thực tế cho hệ thống viễn thông. Yêu cầu đó ép buộc mọi quyết định thiết kế phải tính đến failure từ đầu, không phải là afterthought. Armstrong sau đó đúc kết thành triết lý **“let it crash”**: đừng viết defensive code phức tạp để ngăn crash — hãy để crash xảy ra và khôi phục về trạng thái sạch.

State trong Erlang không lưu trong biến mutable — nó “sống” trong tham số của một recursive loop:

```erlang
%% Actor Erlang giữ state qua đệ quy đuôi (tail recursion)
counter_loop(State) ->
    receive
        {inc, From} ->
            NewState = State + 1,
            From ! {ok, NewState},
            counter_loop(NewState);          %% gọi lại với state mới
        {get, From} ->
            From ! {value, State},
            counter_loop(State);             %% state không đổi
        stop ->
            ok
    end.
```

Đây chính xác là pure transition function `(state, message) → new_state` được đóng gói trong một loop bất tận — mỗi iteration là một transition step. State không bao giờ mutate; nó được “chuyển hoá” qua đệ quy.

OTP (Open Telecom Platform) là kết tinh của nhiều năm kinh nghiệm thực chiến: `gen_server`, `gen_statem`, `supervisor`, `application` — mỗi behaviour là một pattern đã được kiểm chứng qua nhiều hệ thống production. Quan trọng hơn, OTP là hiện thân của IoC (Dòng 3) áp dụng vào actor model: bạn viết callbacks thuần, OTP runtime lo mọi việc còn lại.

**Điểm giao với Dòng 1 (qua functional programming nói chung)**: Erlang được thiết kế với immutability bắt buộc và pure functions là default — kế thừa truyền thống functional programming có gốc rễ từ lambda calculus. Mặc dù Erlang phát triển song song với Haskell (cả hai cùng thập niên 1986–1990) chứ không phải kế thừa Haskell, nhưng cả hai đều áp dụng cùng nguyên lý nền tảng: tách mô tả khỏi diễn giải, dùng pure transition function. Mỗi `handle_call/3` của OTP về bản chất là cùng cấu trúc với pure function trong denotational semantics mà Strachey/Scott đề xuất thập niên trước.

#### Tại sao actor model thắng cho distributed systems

Khi internet bùng nổ thập niên 1990–2000, bài toán concurrency không còn là “nhiều thread trên một máy” mà là “nhiều máy kết nối với nhau”. Shared memory không còn khả thi — bạn không thể share memory qua network. Locks trở nên vô nghĩa.

Actor model lại hoạt động tự nhiên: message passing qua network về cơ bản giống hệt message passing trong cùng một node. PID (process identifier) hoạt động xuyên node trong Erlang cluster mà không cần thay đổi code. Đây là tính chất không một giải pháp dựa trên lock nào có thể có.

CSP (Go channels) lại mạnh hơn cho synchronization patterns trong cùng một node, nhưng kém tự nhiên cho distributed — channel khó scale ra ngoài một process boundary so với PID/address của Actor. Đây là lý do Erlang/OTP thống trị trong fault-tolerant distributed systems, trong khi Go thống trị trong concurrent server applications trên một node.

-----

### Dòng 3 — Đảo ngược luồng điều khiển: từ module coupling đến Hollywood Principle

#### Bối cảnh: structured programming và bài toán reuse (thập niên 1960–1970)

Thập niên 1960, Dijkstra và Wirth đang dẫn dắt phong trào **structured programming** — thay thế GOTO bằng if/while/function, tổ chức code thành modules có interfaces rõ ràng. Đây là bước tiến lớn về mặt tổ chức code.

Nhưng ngay khi modules xuất hiện, một bài toán mới nảy sinh: **tại sao code tái sử dụng được lại khó viết đến vậy?**

Quan sát thực tế: một library `sort` hay `math` thì dễ tái sử dụng. Nhưng một framework GUI, một framework web server — những thứ có state phức tạp, lifecycle, event handling — gần như không thể viết theo cách tái sử dụng được. Bạn dùng framework A thì bị khóa vào kiến trúc của framework A, không thể dễ dàng chuyển sang framework B.

Vấn đề nằm ở chỗ: trong mô hình truyền thống, **code của bạn gọi framework**. Bạn gọi `drawButton()`, bạn gọi `sendEmail()`, bạn gọi `queryDatabase()`. Mỗi lần gọi như vậy là một điểm coupling với implementation cụ thể. Muốn test thì phải mock cái gì đó. Muốn thay đổi implementation thì phải sửa ở khắp nơi.

```text
Library pattern (code của bạn gọi):       Framework pattern (framework gọi code bạn):
─────────────────────────────────         ─────────────────────────────────────────
your_main() {                              framework_main() {  // bạn không viết
    init_database()                            init_event_loop()
    init_logger()                              while (running) {
    while (running) {                              event = wait_for_event()
        event = poll_event()                       your_handler(event)  ← gọi bạn
        handle(event)                          }
    }                                          cleanup()
    cleanup()                              }
}

Bạn nắm control flow                       Framework nắm control flow
Bạn coupling với mọi dependency             Bạn chỉ implement contract (handler)
```

Michael Jackson (không phải ca sĩ — ông là kỹ sư phần mềm người Anh) nhận ra vấn đề này khi ông phát triển **Jackson Structured Programming** (JSP) thập niên 1970. Ông nhận thấy khi cấu trúc của dữ liệu đầu vào và dữ liệu đầu ra không khớp nhau, chương trình bắt buộc phải có một điểm mà tại đó control flow bị “đảo ngược” — không còn đi theo một hướng tuyến tính nữa. Ông gọi đây là **structure clash** và đây là quan sát sớm nhất về hiện tượng mà sau này được đặt tên là IoC.

**Đây là khủng hoảng đầu tiên của Dòng 3**: làm sao viết được framework có thể tái sử dụng cho nhiều ứng dụng khác nhau, mà không buộc ứng dụng phải sửa code khi muốn thay đổi behavior?

#### GUI frameworks và sự đảo ngược control flow (thập niên 1980)

Thập niên 1980 chứng kiến sự bùng nổ của GUI frameworks: Smalltalk-80 MVC (1980), MacApp (1985), ET++ (1988), và nhiều frameworks khác. Đây là lần đầu tiên trong lịch sử, các framework lớn xuất hiện dưới dạng “application skeletons” — bộ khung mà developer điền vào, thay vì library mà developer gọi.

Sự khác biệt rất cụ thể: với MacApp, bạn không viết `main()` và gọi framework. Thay vào đó, framework đã có sẵn `main()` — nó khởi động event loop, quản lý window, và *gọi code của bạn* khi user click hay type. Bạn chỉ override các method mà framework quy định.

Ralph Johnson và Brian Foote (University of Illinois) nhận ra đây là một thay đổi có tính nền tảng về kiến trúc. Bài báo **“Designing Reusable Classes”** (OOPSLA 1988) lần đầu tiên đặt tên và định nghĩa chính xác:

> *“One important characteristic of a framework is that the methods defined by the user to customize the framework will often be called from within the framework itself, rather than from the user’s application code… This inversion of control gives frameworks the power to serve as extensible skeletons.”*

**Inversion of Control** — tên gọi chính thức từ đây. Johnson và Foote cũng chứng minh rằng chính sự đảo ngược này tạo ra reusability: vì framework nắm control flow, nhiều ứng dụng khác nhau có thể dùng cùng một framework mà không cần thay đổi framework.

Cùng giai đoạn đó, **Hollywood Principle** (“Don’t call us, we’ll call you”) được phổ biến trong cộng đồng phần mềm như một câu đúc kết dễ nhớ cho cùng ý tưởng. Thuật ngữ này không có một tác giả cụ thể được công nhận rộng rãi — nó xuất hiện trong nhiều tài liệu thiết kế phần mềm thập niên 1980-1990 (đáng chú ý nhất là qua *Design Patterns* của GoF năm 1994). Ví von từ cách Hollywood casting: studio không để diễn viên gọi cho họ; họ gọi cho diễn viên khi có vai phù hợp. Framework không để code của bạn gọi nó; nó gọi code của bạn khi cần.

#### Dependency Injection: IoC đến thế giới enterprise (thập niên 2000)

Thập niên 1990, object-oriented programming đã hoàn toàn thống trị. GoF (Gang of Four) book (1994) hệ thống hoá các design patterns — Template Method, Observer, Strategy — đều là các pattern IoC dưới các tên gọi khác nhau.

Nhưng vấn đề mới xuất hiện: trong các hệ thống enterprise lớn với hàng trăm classes, làm thế nào để quản lý dependencies giữa các object? Nếu `UserService` cần `UserRepository`, `EmailService`, `Logger` — ai sẽ tạo và inject chúng?

Martin Fowler (2004) hệ thống hoá điều này thành **Dependency Injection** — một specialization của IoC trong đó framework (container) tự động tạo và inject dependencies vào object. Spring Framework (Rod Johnson, 2003) cho Java, và nhiều DI containers khác sau đó, đưa pattern này vào mainstream.

Tuy nhiên, Fowler cũng cảnh báo: DI chỉ là *một cách thực hiện* IoC, không phải IoC nói chung. Cộng đồng Java (đặc biệt Spring) sau đó dùng “IoC container” như synonym cho “DI container”, tạo ra nhầm lẫn kéo dài đến nay — nhiều developer nghĩ IoC chỉ là DI, bỏ qua ý nghĩa rộng hơn của nó.

**Điểm giao với Dòng 1 và Dòng 2**: IoC là cơ chế cho phép algebraic effects và actor model hoạt động trong thực tế. Khi OTP runtime gọi `handle_call/3` của GenServer — đó là IoC. Khi React reconciler gọi component function — đó là IoC. Khi Elm runtime interpret `Cmd msg` — đó là IoC. IoC là cơ sở hạ tầng mà Dòng 1 và Dòng 2 đứng trên khi chuyển từ lý thuyết sang implementation thực tế.

-----

### Sự hội tụ: từ Erlang (1990s) đến mainstream (2010s)

Ba dòng tư tưởng xuất phát từ ba khủng hoảng kỹ thuật khác nhau:

- **Dòng 1** (1930s–nay): *Làm sao lý luận toán học về chương trình có side effects mà không mất đi tính đúng đắn có thể chứng minh?*
- **Dòng 2** (1960s–nay): *Làm sao xây dựng hệ thống concurrent và distributed mà không có race condition, không có deadlock, và tự phục hồi sau lỗi?*
- **Dòng 3** (1960s–nay): *Làm sao viết code có thể tái sử dụng, test được, và mở rộng được mà không bị coupled với implementation?*

**Erlang/OTP là điểm hội tụ đầu tiên trong lịch sử (1986–1998)** — không phải thập niên 2010 như có thể nhầm tưởng. Khi Armstrong và đội thiết kế Erlang, họ vô tình giải quyết đồng thời cả ba bài toán:

- Pure transition function trong recursive loop = câu trả lời cho Dòng 1
- Process isolation + message passing + supervision = câu trả lời cho Dòng 2
- OTP behaviour (callbacks được runtime gọi) = câu trả lời cho Dòng 3

Hội tụ này xảy ra trong một niche cụ thể (viễn thông) suốt thập niên 1990 — nhưng phần còn lại của ngành phần mềm vẫn đang giải từng bài toán riêng lẻ với các công cụ tách biệt: Java + threads cho concurrency, Spring DI cho IoC, Haskell cho purity. Mỗi cộng đồng giải một bài toán nhưng không ai đồng thời giải cả ba ngoài Erlang.

**Sự lan toả ra mainstream xảy ra thập niên 2010** khi internet và mobile bùng nổ. Một ứng dụng web production điển hình bây giờ phải:

- Xử lý hàng nghìn request đồng thời (Dòng 2)
- Có UI reactive theo dõi state phức tạp (Dòng 1 + 3)
- Dễ test, dễ maintain, dễ deploy không downtime (Dòng 3)
- Fault-tolerant khi database chậm hoặc service phụ thuộc fail (Dòng 2)

Không có hệ thống nào có thể giải quyết chỉ một bài toán và bỏ qua hai cái kia. Các paradigm trước đây hội tụ trong Erlang nay được tái phát hiện và phổ biến rộng:

- React Hooks (2019) — Dòng 1 + Dòng 3 cho UI layer
- Elixir + Phoenix (2012–nay) — Erlang/OTP đến với Web developer mainstream
- Phoenix LiveView (2018) — điểm mà cả ba hội tụ thành một primitive duy nhất, server-rendered UI có process isolation
- Rust Tokio, Swift Concurrency, Kotlin Coroutines (2018–2021) — Dòng 1 + 2 cho native applications

**Lý do ba dòng buộc phải hội tụ về cùng một kiến trúc** — pure description + orchestrating runtime — không phải ngẫu nhiên. Đó là kiến trúc tối thiểu cần thiết để đồng thời:

- Cho phép lý luận toán học (Dòng 1 đòi hỏi purity)
- Cô lập failure (Dòng 2 đòi hỏi isolation)
- Cho phép tái sử dụng (Dòng 3 đòi hỏi decoupling)

Bất kỳ giải pháp nào thiếu một trong ba thành phần này sẽ thất bại trên một mặt: thiếu purity → khó debug, khó test, khó replay; thiếu isolation → một bug làm sập cả hệ thống; thiếu decoupling → không thể swap implementation, không thể mở rộng.

Ba yêu cầu này cùng trỏ về một điểm: **code của người dùng phải là mô tả thuần, runtime phải nắm toàn bộ quyền điều phối**.

```text
╔══════════════════════════════════════════════════════════════════════╗
║              CÙNG MỘT KIẾN TRÚC NHÌN TỪ BA GÓC                       ║
╠══════════════════════════════════════════════════════════════════════╣
║                                                                      ║
║   Dòng 1 (Lý thuyết PL)                                              ║
║      pure function  +  handler diễn giải                             ║
║                                                                      ║
║   Dòng 2 (Concurrency)                                               ║
║      pure callback  +  runtime điều phối message                     ║
║                                                                      ║
║   Dòng 3 (Kỹ nghệ phần mềm)                                          ║
║      pure callback  +  framework gọi đúng lúc                        ║
║                          ↓                                           ║
║              ┌──────────────────────┐                                ║
║              │  PURE DESCRIPTION    │                                ║
║              │           +          │                                ║
║              │ ORCHESTRATING RUNTIME│                                ║
║              └──────────┬───────────┘                                ║
║                         │                                            ║
║       ┌─────────────────┼─────────────────┐                          ║
║       ▼                 ▼                 ▼                          ║
║  ┌──────────┐    ┌──────────────┐   ┌──────────────────┐             ║
║  │  REACT   │    │  ELIXIR/OTP  │   │  Phoenix LiveView│             ║
║  │  HOOKS   │    │  GENSERVER   │   │  (cả 3 hội tụ    │             ║
║  │ (Dòng 1+3│    │ (Dòng 1+2+3) │   │ trong 1 primitive│             ║
║  │ cho UI)  │    │  cho server) │   │   năm 2018)      │             ║
║  └──────────┘    └──────────────┘   └──────────────────┘             ║
║                                                                      ║
║   BA TRỤ KIẾN TRÚC XUYÊN SUỐT TÀI LIỆU:                              ║
║   [1] Tách "WHAT" khỏi "WHEN":   Pure logic ↔ Runtime orchestration  ║
║   [2] Pure Core, Impure Shell:   Business ↔ Side effects (rìa ngoài) ║
║   [3] State qua tham số:         newState ↔ Không mutation tại chỗ   ║
╚══════════════════════════════════════════════════════════════════════╝
```

Phần còn lại của tài liệu đi vào implementation cụ thể: React (Dòng 1+3 cho UI), Elixir/OTP (Dòng 1+2+3 cho server), IoC như nguyên lý thống nhất, và LiveView như điểm hội tụ.

-----

**Tiếp theo:** [Phần 1 — React hiện đại dưới lăng kính Functional Programming →](react.md)