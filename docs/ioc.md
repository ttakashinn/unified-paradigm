# Phần 3 — Inversion of Control: nguyên lý xuyên suốt

### 3.1. Định nghĩa, Hollywood Principle và lịch sử

**Inversion of Control** (IoC) là khái niệm xuất hiện trong bài "Designing Reusable Classes" (Johnson & Foote, 1988) và được Martin Fowler hệ thống hóa lại. Định nghĩa gốc: *khi framework điều khiển flow, còn application code chỉ cung cấp các đoạn được framework gọi tại thời điểm thích hợp*.

Slogan nổi tiếng: **"Don't call us, we'll call you"** — Hollywood Principle.

```
Flow truyền thống (không IoC):        IoC (framework điều khiển):
─────────────────────────────          ──────────────────────────────
main()                                 Framework.run()
  → khởi tạo dependencies               → gọi App.init()
  → gọi service A                        → nhận event
  → gọi service B                        → gọi App.handle_event()
  → cleanup                              → gọi App.cleanup()

Application nắm control flow          Framework nắm control flow
```

IoC có hai phương ngữ thường bị nhầm lẫn:

1. **IoC theo nghĩa flow** (nghĩa gốc, rộng hơn): framework giữ main loop; user code chỉ là callback/handler/lifecycle method. Ví dụ: GUI event loop, web framework, game engine, template method pattern.

2. **Dependency Injection** (DI): một specialization — framework "tiêm" implementation của dependencies vào object thay vì để object tự khởi tạo. Phổ biến trong Spring, Angular, ASP.NET, NestJS.

Cả hai đều phục vụ cùng mục tiêu: **giảm coupling, tăng testability, tăng tính tái sử dụng**, vì code của bạn không còn nắm quyền điều phối — nó chỉ cần thuần và đúng tại điểm móc.

### 3.2. IoC trong React: hooks như "effect handlers" được inject

Nhìn lại React với lăng kính IoC:

| Action | Ai làm | Ý nghĩa IoC |
|---|---|---|
| Gọi `render` component | React reconciler (không phải bạn) | "Don't call us" |
| Gọi cleanup function | React gọi đúng thời điểm | Lifecycle callback |
| Cấp phát slot state | React quyết định (fiber node) | Framework-managed state |
| Schedule re-render | React batching algorithm | Runtime optimization |
| Inject context value | Provider trên cây tự inject | Dependency Injection |
| Gọi `useEffect` | Sau commit phase của React | Lifecycle hook |

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

### 3.3. IoC trong Elixir/OTP: behaviour + supervisor

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

React's `useState`/`useEffect`/`useContext` và OTP's `init/1`/`handle_call/3`/Application env là cùng một mô hình IoC ở hai ngữ cảnh khác nhau — runtime gọi user code tại điểm contract đã định, user code không gọi runtime. Bảng đối chiếu chi tiết đầy đủ ở [Phần 5 — Bản đồ tư duy thống nhất](unified-map.md#52-bang-oi-chieu-pattern-hoan-chinh).

**(c) Supervisor tree là IoC ở tầng lifecycle:**

Bạn không quyết định khi nào worker được start, restart, hay shutdown — supervisor làm điều đó dựa trên strategy bạn khai báo. Code của bạn chỉ trả lời: "đây là child_spec của tôi, đây là cách tôi init, đây là cách tôi terminate."

**(d) Application behaviour — IoC ở tầng khởi động hệ thống:**

`Application.start/2` được BEAM gọi khi node khởi động. Bạn không gọi nó.

### 3.4. IoC trong cả hai: dependency injection ngầm

```
React DI:                              Elixir DI:
─────────────────────────────          ──────────────────────────────
Context Provider                       Registry + :via tuple
  ↓ inject                               ↓ inject
useContext() trong component           Application.get_env/3 trong module
  ↓ swap cho test                        ↓ swap cho test
<Provider value={mock}>                Mox behaviour mock
```

Cả hai đều đẩy "ai cung cấp implementation nào" ra khỏi business logic, để cùng một code chạy trong production, trong test, hoặc trong môi trường mock-up.

### 3.5. Trade-offs của IoC

| Lợi ích | Cái giá phải trả |
|---|---|
| Decoupling — swap implementation dễ dàng | "Magic" — flow ngầm khó debug |
| Testability — mock dependencies | Stack trace đi qua framework dày đặc |
| Framework-driven optimization (batching, scheduling) | Phụ thuộc vào quy ước của framework |
| Plugin-ability — mở rộng không sửa core | Khó thoát khỏi framework khi cần |
| Hot code swap (OTP), concurrent updates (React) | Học thêm "ngôn ngữ" của framework |

---


---

**Trước:** [← Phần 2 — Elixir/OTP](elixir-otp.md) | **Tiếp theo:** [Phần 4 — Phoenix LiveView →](liveview.md)
