# Phần 3 — Inversion of Control: nguyên lý xuyên suốt

Phần 1 và Phần 2 đã cho thấy hai hiện thân cụ thể — React Hooks và Elixir GenServer — đều có cùng cấu trúc: **pure transition function được runtime gọi tại điểm contract định trước**. Phần 3 này tách rời pattern khỏi ngữ cảnh cụ thể, mô tả nó như một nguyên lý kiến trúc trừu tượng tên là Inversion of Control, và chỉ ra các điểm đó xuất hiện ở mọi tầng của ngành phần mềm.

## 3.1. Định nghĩa và lịch sử

**Inversion of Control** (IoC) được Johnson & Foote đặt tên trong bài “Designing Reusable Classes” (OOPSLA 1988) — bài đã được trích dẫn đầy đủ ở [Phần 0](index.md). Bản chất cô đọng: *khi framework điều khiển flow, còn application code chỉ cung cấp các đoạn (callbacks, behaviour implementations, lifecycle methods) được framework gọi tại thời điểm thích hợp*.

**Hollywood Principle** — “Don’t call us, we’ll call you” — là câu đúc kết phổ biến cho cùng nguyên lý, được lan truyền rộng trong cộng đồng phần mềm thập niên 1980-1990 (đáng chú ý nhất qua *Design Patterns* của GoF năm 1994). Như Phần 0 đã làm rõ, nguồn gốc tên gọi này không có một tác giả cụ thể được công nhận rộng rãi.

### So sánh trực tiếp Library vs Framework

```text
LIBRARY (bạn gọi code library):           FRAMEWORK (framework gọi code bạn):
─────────────────────────────              ──────────────────────────────────
// Bạn viết main()                         // Framework đã có sẵn main()
function main() {                          framework_main() {  // bạn không viết
  const data = readFile("x.txt");            init_runtime();
  const result = sortLib(data);              while (running) {
  const json = jsonLib.stringify(result);      event = poll_event();
  console.log(json);                           your_handler(event); ← gọi bạn
}                                              your_render(state); ← gọi bạn
                                             }
                                             cleanup();
                                           }

→ Bạn nắm control flow                    → Framework nắm control flow
→ Bạn import + gọi mỗi dependency          → Bạn chỉ implement các điểm móc
→ Coupling cao với mọi library              được framework định nghĩa trước
→ Test phải mock từng dependency           → Test pure callback đơn giản hơn
```

Đặc tính phân biệt: **nếu bạn gọi nó, đó là library. Nếu nó gọi bạn, đó là framework.** Đây không phải technique, đó là tính chất kiến trúc cơ bản.

### Hai phương ngữ thường bị nhầm lẫn

1. **IoC theo nghĩa flow** (nghĩa gốc, rộng hơn): framework giữ main loop; user code chỉ là callback/handler/lifecycle method. Ví dụ: GUI event loop, web framework, game engine, template method pattern, OTP behaviour, React component function.
1. **Dependency Injection** (DI): một specialization — framework “tiêm” implementation của dependencies vào object thay vì để object tự khởi tạo. Phổ biến trong Spring, Angular, ASP.NET, NestJS.

Cả hai đều phục vụ cùng mục tiêu: **giảm coupling, tăng testability, tăng tính tái sử dụng**, vì code của bạn không còn nắm quyền điều phối — nó chỉ cần thuần và đúng tại điểm móc.

## 3.2. IoC trong React: hooks như “effect handlers” được inject

Nhìn lại React với lăng kính IoC:

|Action                |Ai làm                           |Ý nghĩa IoC            |
|----------------------|---------------------------------|-----------------------|
|Gọi `render` component|React reconciler (không phải bạn)|“Don’t call us”        |
|Gọi cleanup function  |React gọi đúng thời điểm         |Lifecycle callback     |
|Cấp phát slot state   |React quyết định (fiber node)    |Framework-managed state|
|Schedule re-render    |React batching algorithm         |Runtime optimization   |
|Inject context value  |Provider trên cây tự inject      |Dependency Injection   |
|Gọi `useEffect`       |Sau commit phase của React       |Lifecycle hook         |

Tức là: **hooks chính là các điểm inject behaviour vào component**, và bản thân component là **callback mà React điều phối**.

```jsx
// IoC qua context — inject service vào component
const LoggerContext = createContext(null);
const ApiContext    = createContext(null);

// Component không biết gì về implementation
function UserDashboard() {
  const api    = useContext(ApiContext);      // DI: framework inject
  const logger = useContext(LoggerContext);   // DI: framework inject

  const { data } = useFetch(api.dashboardUrl);

  useEffect(() => {
    logger.log('dashboard_viewed');    // không import logger trực tiếp
  }, []);

  return <Dashboard data={data}/>;
}

// Production
const app = (
  <LoggerContext.Provider value={datadogLogger}>
    <ApiContext.Provider value={productionApi}>
      <UserDashboard/>
    </ApiContext.Provider>
  </LoggerContext.Provider>
);

// Test — hoàn toàn swap implementation
const testApp = (
  <LoggerContext.Provider value={mockLogger}>
    <ApiContext.Provider value={{ dashboardUrl: '/mock/dashboard' }}>
      <UserDashboard/>
    </ApiContext.Provider>
  </LoggerContext.Provider>
);
```

## 3.3. IoC trong Elixir/OTP: behaviour + supervisor

OTP là một trong những hiện thân **trong sạch nhất** của IoC trong toàn ngành. Bằng chứng:

**(a) Behaviour như callback module — Hollywood Principle thuần khiết:**

Khi bạn viết `use GenServer`, bạn không gọi message loop — OTP gọi. Bạn chỉ cung cấp callbacks. Đây chính xác là Hollywood Principle.

```elixir
defmodule MyWorker do
  use GenServer   # "Don't call us (the receive loop), we'll call you"

  # Bạn chỉ implement các "hook" tại điểm OTP quy định
  @impl true
  def init(arg),                       do: {:ok, arg}

  @impl true
  def handle_call(:ping, _from, s),    do: {:reply, :pong, s}

  @impl true
  def handle_cast({:work, data}, s),   do: {:noreply, process(s, data)}

  @impl true
  def handle_info(:timeout, s),        do: {:stop, :normal, s}

  @impl true
  def terminate(reason, s),            do: persist(s)
end
```

**(b) Hooks ↔ Behaviour callbacks là cùng một pattern:**

React’s `useState`/`useEffect`/`useContext` và OTP’s `init/1`/`handle_call/3`/Application env là cùng một mô hình IoC ở hai ngữ cảnh khác nhau — runtime gọi user code tại điểm contract đã định, user code không gọi runtime. Bảng đối chiếu chi tiết đầy đủ ở [Phần 5 — Bản đồ tư duy thống nhất](unified-map.md#52-bang-oi-chieu-pattern-hoan-chinh).

**(c) Supervisor tree là IoC ở tầng lifecycle:**

Bạn không quyết định khi nào worker được start, restart, hay shutdown — supervisor làm điều đó dựa trên strategy bạn khai báo. Code của bạn chỉ trả lời: “đây là child_spec của tôi, đây là cách tôi init, đây là cách tôi terminate.”

**(d) Application behaviour — IoC ở tầng khởi động hệ thống:**

`Application.start/2` được BEAM gọi khi node khởi động. Bạn không gọi nó.

## 3.4. IoC trong cả hai: dependency injection ngầm

```text
React DI:                              Elixir DI:
─────────────────────────────          ──────────────────────────────
Context Provider                       Registry + :via tuple
  ↓ inject                               ↓ inject
useContext() trong component           Application.get_env/3 trong module
  ↓ swap cho test                        ↓ swap cho test
<Provider value={mock}>                Mox behaviour mock
```

Cả hai đều đẩy “ai cung cấp implementation nào” ra khỏi business logic, để cùng một code chạy trong production, trong test, hoặc trong môi trường mock-up.

## 3.5. Ba pattern Dependency Injection cần phân biệt

DI là specialization quan trọng của IoC, nhưng không phải mọi cách inject dependency đều tương đương. Có ba pattern chính, mỗi pattern có trade-off riêng:

**Constructor Injection** — dependency được truyền qua constructor, lưu vào instance variable:

```typescript
class UserService {
  constructor(
    private repo: UserRepository,
    private mailer: EmailService
  ) {}

  createUser(data) { /* dùng this.repo, this.mailer */ }
}
```

Đây là pattern **được khuyến nghị nhất**: dependency lộ ra ngay ở signature, compiler/IDE tự catch missing deps, không thể tạo object ở trạng thái không hợp lệ. Dùng trong Spring, Angular, NestJS, ASP.NET Core.

**Setter Injection** — dependency được set sau khi khởi tạo qua setter method:

```typescript
class UserService {
  private repo: UserRepository;
  setRepo(r: UserRepository) { this.repo = r; }
}
```

**Trade-off**: object có thể tồn tại ở trạng thái không hợp lệ (chưa set dep), runtime null check rải rác. Chỉ dùng khi dependency là **optional** hoặc cần thay đổi trong vòng đời object.

**Interface Injection** — class phải implement interface khai báo dependency mà nó cần:

```typescript
interface NeedsLogger {
  injectLogger(l: Logger): void;
}
```

Hiếm gặp trong code hiện đại — quá nhiều ceremony cho ít lợi ích. Đáng nhắc đến vì nó là pattern thuần tuý nhất về mặt lý thuyết, nhưng implementation overhead khiến nó bị bỏ rơi.

**Trong React, IoC qua hooks tương đương với một dạng DI implicit qua context — không có ceremony của constructor/setter injection nhưng vẫn đảm bảo dependency tường minh tại điểm cần.** Trong OTP, DI thường qua `Application.get_env/2` hoặc Mox behaviour — cũng implicit, runtime-dispatched.

## 3.6. Trade-offs của IoC

IoC không phải free lunch. Mỗi lợi ích đi kèm một cost:

|Lợi ích                                                                   |Cái giá phải trả                                                        |
|--------------------------------------------------------------------------|------------------------------------------------------------------------|
|**Decoupling** — swap implementation dễ dàng cho test, prod, mock         |**“Magic”** — flow ngầm khó debug khi mới đến framework                 |
|**Testability** — pure callback test thuần như unit test                  |**Stack trace nhiễu** — đi qua frame của framework dày đặc              |
|**Framework optimization** (batching, scheduling, hot reload)             |**Phụ thuộc convention** — phải tuân thủ quy ước framework đặt ra       |
|**Plugin-ability** — mở rộng không sửa core framework                     |**Vendor lock-in** — khó thoát khỏi framework khi yêu cầu thay đổi      |
|**Hot code swap** (OTP), **concurrent rendering** (React Fiber)           |**Learning curve** — phải học “ngôn ngữ” và mental model của framework  |
|**Lifecycle guarantees** — framework đảm bảo cleanup, init, error recovery|**Loss of control** — không thể can thiệp khi framework điều phối khác ý|

**Khi IoC trở thành cản trở:**

- **Hot loops cần performance cực cao**: overhead của message passing (BEAM), virtual DOM diff (React), DI lookup (Spring) có thể chiếm phần lớn thời gian thực thi. Lúc này phải rời IoC, gọi trực tiếp.
- **Logic flow quá phức tạp**: khi business logic là một state machine 50 trạng thái với rules phức tạp giữa transitions, dùng plain function với explicit dispatch có khi rõ hơn callback rải rác qua framework.
- **Debugging deep call stack**: khi production bug xảy ra qua 15 layer của framework, stack trace dài 200 dòng và khó tìm code của mình ở đâu.

Phần 6 sẽ đi sâu vào những trường hợp cụ thể này.

-----

Đến đây đã rõ: IoC không phải technique riêng của một framework, mà là **đặc tính kiến trúc cơ bản** — phân biệt framework với library, phân biệt code có thể tái sử dụng với code bị khoá vào implementation cụ thể. React, OTP, Spring đều áp dụng IoC theo cách riêng. Phần 4 sẽ chỉ ra một framework đặc biệt — Phoenix LiveView — nơi IoC, Pure Transition (Dòng 1), và Actor Model (Dòng 2) hội tụ thành một primitive UI duy nhất, không phải ba thứ ghép lại.

-----

**Trước:** [← Phần 2 — Elixir/OTP](elixir-otp.md) | **Tiếp theo:** [Phần 4 — Phoenix LiveView →](liveview.md)