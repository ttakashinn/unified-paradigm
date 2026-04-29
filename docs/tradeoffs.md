# Phần 6 — Trade-offs toàn cục và khi nào nên phá vỡ paradigm

## 6.1. Khi IoC gây hại

**Performance-critical hot loops:** message passing trong BEAM có cost (vài microseconds/message). Với in-memory calculation loop cần throughput cao, `:ets` (shared table, no message passing) hay bare functions là đúng hơn GenServer.

```elixir
# Không tốt khi cần throughput cao
def count_words(text), do: GenServer.call(WordCounter, {:count, text})

# Tốt hơn: thuần function, không process overhead
def count_words(text), do: WordCounter.pure_count(text)
```

**React rendering hot path:** đôi khi `useMemo`, `useCallback`, `memo()` là cần thiết — chấp nhận sự phức tạp của dependency tracking để tránh re-render không cần thiết.

## 6.2. Khi nên giữ state local thay vì "lift up"

Không phải mọi state đều cần được IoC-ified. State thuần UI (tooltip open/close, accordion expanded) không cần đi qua context hay GenServer. Quy tắc ngón tay cái: state chỉ cần lift up khi **nhiều hơn một component cần nó**.

## 6.3. Khi LiveView không phải lựa chọn đúng

- **Offline-first app:** LiveView cần WebSocket, nếu offline thì chết.
- **Heavy client-side computation:** video editing, 3D rendering — cần JS, không phải server round-trip.
- **SEO-critical SPA với complex client routing:** React + Next.js vẫn phù hợp hơn.
- **Mobile native app:** LiveView Native đang phát triển nhưng chưa production-ready cho mọi use case.

**Các giải pháp hybrid đang nổi lên trong hệ sinh thái:**

**LiveView Native** (Dockyard) — mở rộng mô hình LiveView ra ngoài browser. Cùng một GenServer process `handle_event` có thể drive **cả web lẫn iOS/Android native UI** đồng thời, chỉ khác ở tầng render (HEEx template vs SwiftUI/Jetpack Compose declarative tree). Về mặt kiến trúc, đây là bằng chứng cực mạnh cho luận điểm của tài liệu này: khi server process là source of truth và render chỉ là projection, thì platform (web/iOS/Android) trở thành chi tiết triển khai hoán đổi được.

```elixir
# Cùng một handle_event phục vụ cả web lẫn native
def handle_event("add_to_cart", %{"id" => id}, socket) do
  {:ok, cart} = MyApp.Cart.add(socket.assigns.user_id, id)
  {:noreply, assign(socket, :cart, cart)}
  # → web: LiveView diff → HEEx → HTML patch qua WebSocket
  # → iOS: LiveView Native → SwiftUI tree update qua WebSocket
end
```

**LiveState** — thư viện tách LiveView state management thành một layer độc lập có thể consume từ bất kỳ client nào (React, Vue, vanilla JS, mobile). Thay vì server render HTML, server chỉ broadcast state changes; client tự render theo paradigm của nó. Đây là pattern trung gian cho team cần giữ React client nhưng muốn server-side state với fault tolerance của OTP.

## 6.4. Khi nên dùng Agent thay vì GenServer trong Elixir

`Agent` là wrapper đơn giản hóa của GenServer cho trường hợp chỉ cần state, không cần custom logic phức tạp:

```elixir
# Agent: khi chỉ cần get/update state đơn giản
{:ok, agent} = Agent.start_link(fn -> %{} end)
Agent.update(agent, &Map.put(&1, :key, "value"))
Agent.get(agent, &Map.get(&1, :key))

# GenServer: khi cần custom message handling, complex transitions, terminate callbacks
```

## 6.5. Anti-patterns: khi paradigm bị lạm dụng

Mỗi paradigm mạnh đều có cách bị lạm dụng đặc trưng. Đây là bốn anti-pattern phổ biến nhất khi developer áp dụng tư duy "pure transition + IoC" mà không dừng lại tự hỏi *có thực sự cần không*.

**Anti-pattern 1: `useEffect` everywhere thay vì derived state**

Đây là anti-pattern phổ biến nhất của hooks. Developer thấy "có cái gì đó cần đồng bộ với state khác" → reflex viết `useEffect`. Nhưng phần lớn các trường hợp đó chỉ là *derived state* — tính toán thuần từ state hiện có, không cần effect:

```jsx
// SAI: useEffect để tính derived value
function UserProfile({ users, userId }) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    setUser(users.find(u => u.id === userId));   // chỉ là filter!
  }, [users, userId]);
  return <div>{user?.name}</div>;
}

// ĐÚNG: derive trực tiếp trong render — không cần state, không cần effect
function UserProfile({ users, userId }) {
  const user = users.find(u => u.id === userId);
  return <div>{user?.name}</div>;
}
```

Quy tắc Dan Abramov: **"You might not need an effect."** Nếu effect chỉ để tính từ props/state hiện có, hoặc để adjust state theo prop change — đó là dấu hiệu code đang fight với React thay vì dùng nó. Effect chỉ nên dùng cho *side effect thật*: subscribe/unsubscribe external system, sync với DOM thủ công, gửi analytics event.

### Anti-pattern 2: GenServer cho mọi state — over-actor-ification

Developer mới đến từ OOP background nhìn GenServer như một class — và biến mọi struct thành GenServer:

```elixir
# SAI: GenServer chỉ để lưu một map cấu hình tĩnh đọc bởi nhiều process
defmodule MyApp.Config do
  use GenServer
  def start_link(_), do: GenServer.start_link(__MODULE__, %{}, name: __MODULE__)
  def get(key), do: GenServer.call(__MODULE__, {:get, key})
  def init(_), do: {:ok, %{api_key: "...", timeout: 5000}}
  def handle_call({:get, key}, _from, state), do: {:reply, Map.get(state, key), state}
end
```

GenServer là một process serialize tất cả request — mọi `Config.get/1` từ 1000 process khác nhau phải xếp hàng qua một message loop, tạo bottleneck nghiêm trọng cho dữ liệu chỉ đọc. Đúng phải là `:persistent_term`, `:ets`, hoặc Application env (`Application.get_env/2`):

```elixir
# ĐÚNG: persistent_term cho config tĩnh, đọc đồng thời từ mọi process không lock
:persistent_term.put({MyApp, :config}, %{api_key: "...", timeout: 5000})
def get(key), do: Map.get(:persistent_term.get({MyApp, :config}), key)
```

Quy tắc: **GenServer là cho state mutable + có business logic phức tạp, không phải cho mọi cái có state**. Nếu chỉ cần shared read-only data → `:persistent_term`. Nếu cần concurrent read/write nhưng không cần ordering → `:ets`. Nếu cần một single source of truth với transition rules → mới đến GenServer.

### Anti-pattern 3: DI container biến thành Service Locator

Spring/NestJS framework rất dễ trượt vào pattern này. Thay vì inject dependency tường minh qua constructor/parameter, developer inject một "container" rồi pull mọi dependency từ đó:

```typescript
// SAI: Service Locator pattern trá hình dưới vỏ DI
class UserService {
  constructor(private container: ServiceContainer) {}

  createUser(data) {
    const repo    = this.container.get('UserRepository');     // hidden dependency
    const mailer  = this.container.get('EmailService');       // hidden dependency
    const logger  = this.container.get('Logger');             // hidden dependency
    // ...
  }
}

// ĐÚNG: DI tường minh, type signature là contract
class UserService {
  constructor(
    private repo: UserRepository,
    private mailer: EmailService,
    private logger: Logger
  ) {}
  createUser(data) { /* dependencies rõ ràng từ signature */ }
}
```

Service Locator phá vỡ mọi lợi ích của IoC: dependency không còn lộ ra ở signature → không thể static-analyze, không thể test riêng, không thể swap mock cho từng dependency. **Nguyên tắc**: nếu type signature của hàm/constructor không nói lên *toàn bộ* dependency mà nó cần, đó là service locator chứ không phải DI.

### Anti-pattern 4: Over-supervision — supervisor tree 5 tầng cho ứng dụng đơn giản

Developer mới học OTP đôi khi build supervision tree quá sâu vì "best practice nói cần có supervisor":

```text
MyApp.Supervisor
└── MyApp.SubSystemSupervisor
    └── MyApp.FeatureSupervisor
        └── MyApp.WorkerPoolSupervisor
            └── MyApp.Worker
```

Mỗi tầng supervisor chỉ có một child duy nhất → không có giá trị fault isolation, chỉ thêm complexity. Supervision tree nên phản ánh *failure boundary thực tế*: hai process nên chung supervisor nếu một crash thì cái kia *cũng* nên restart; nên tách supervisor nếu chúng phục hồi độc lập.

**Quy tắc**: Bắt đầu với cấu trúc phẳng nhất (`one_for_one` ở root), chỉ thêm supervisor con khi có lý do cụ thể — ví dụ một subsystem cần restart strategy khác (`one_for_all` cho group dependent processes), hoặc một dynamic pool.

**Tổng kết bốn anti-pattern:**

Cả bốn đều có chung một origin — *áp dụng paradigm như công thức thay vì như công cụ*. Senior developer hiểu paradigm phải biết khi nào *không* dùng nó. Với mỗi quyết định kiến trúc, câu hỏi cần hỏi không phải "GenServer/useEffect/DI có thể giải quyết việc này không" (gần như luôn là *có*), mà là **"Có giải pháp đơn giản hơn không?"** — derived state thay effect, persistent_term thay GenServer, constructor injection thay container, supervisor phẳng thay nested.

---

**Trước:** [← Phần 5 — Bản đồ thống nhất](unified-map.md) | **Tiếp theo:** [Phần 7 — Landscape toàn cục →](landscape.md)
