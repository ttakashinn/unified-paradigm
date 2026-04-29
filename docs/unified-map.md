# Phần 5 — Bản đồ tư duy thống nhất

### 5.1. Ba trụ kiến trúc (kết tinh từ ba dòng tư tưởng ở Phần 0)

Lưu ý phân biệt thuật ngữ: **"ba dòng tư tưởng"** ở [Phần 0](index.md) là *lịch sử ý tưởng* (algebraic effects, actor model, Hollywood Principle) — đây là *nơi paradigm ra đời*. **"Ba trụ kiến trúc"** dưới đây là *kết quả triết học cụ thể* mà developer thấy trong code hàng ngày — đây là *cách paradigm biểu hiện*.

Đặt bốn entity (React, Elixir/OTP, IoC, LiveView) cạnh nhau, ta thấy chúng cùng đứng trên ba trụ:

**Trụ 1 — Tách "what" khỏi "when"**

Cả ba đều khuyến khích viết các hàm/callback thuần mô tả *cái gì xảy ra với state khi có một sự kiện*, để runtime/framework lo *khi nào và như thế nào* sự kiện đó được trigger.

```
React:      reducer(state, action) → newState
Elixir:     handle_call(msg, from, state) → {:reply, result, newState}
LiveView:   handle_event(name, params, socket) → {:noreply, newSocket}
IoC:        callback(context) → result  (framework quyết định khi nào gọi)
```

**Trụ 2 — Pure Core, Impure Shell**

Logic nghiệp vụ (transition function) là pure. Side effect (DOM mutation, network IO, process spawning, supervisor restart) bị đẩy ra rìa, do framework/runtime điều phối.

```
"Functional Core, Imperative Shell" (Gary Bernhardt)
         ┌──────────────────────────┐
         │     IMPERATIVE SHELL     │
         │  (framework, runtime,    │
         │   side effects, IO)      │
         │  ┌────────────────────┐  │
         │  │  FUNCTIONAL CORE   │  │
         │  │  (pure reducers,   │  │
         │  │   pure handlers,   │  │
         │  │   pure callbacks)  │  │
         │  └────────────────────┘  │
         └──────────────────────────┘
```

**Trụ 3 — State qua tham số, không qua mutation**

Cả `useReducer` lẫn GenServer đều ép developer trả về `newState` thay vì mutate. Việc state "tồn tại giữa các sự kiện" là chuyện của runtime.

### 5.2. Bảng đối chiếu pattern hoàn chỉnh

| Concept | React | Elixir/OTP | Phoenix LiveView | IoC tổng quát |
|---|---|---|---|---|
| **Đơn vị tính toán** | Function component | GenServer process | LiveView process | Callback object |
| **State container** | Fiber node | Process heap | `%Socket{assigns}` | Framework-managed |
| **State transition** | Reducer | `handle_call/cast` | `handle_event` | Pure handler |
| **Khởi tạo** | `useState(init)` | `init/1` | `mount/3` | Constructor callback |
| **Side effects** | `useEffect` | `handle_info`, Task | `handle_info` | Lifecycle callback |
| **Cleanup** | `useEffect` return fn | `terminate/2` | `terminate/2` | Destructor callback |
| **Error handling** | Error boundary | Supervisor | Supervisor | Framework handler |
| **Composition** | Custom hooks | GenServer + Agent | Components + Hooks | Higher-order callback |
| **Dependency provision** | `useContext` / Provider | Registry, App env | Assigns, PubSub | Dependency Injection |
| **Contract** | Hook rules | `@behaviour` + `@impl` | Callback signatures | Interface |
| **Runtime entity** | Reconciler + Scheduler | BEAM Scheduler | LiveView diffing | Framework engine |
| **Diffing** | Virtual DOM (client) | N/A | HTML diff (server) | N/A |
| **Realtime** | useEffect + WebSocket | PubSub, GenServer | PubSub + handle_info | Event subscription |

### 5.3. Bộ công cụ đọc framework bất kỳ: 5 vai trò universal

Khi gặp **bất kỳ** framework/runtime nào thuộc họ này — Temporal, Bevy ECS, SwiftUI, Unity, AWS Lambda, Kubernetes operator — ánh xạ về 5 vai trò sau:

| Vai trò | React | OTP/GenServer | Spring DI | Elm TEA | Kubernetes | AWS Lambda |
|---|---|---|---|---|---|---|
| **Pure description (what)** | Component fn body | `handle_call/3`, `init/1` | Bean methods | `update`, `view` | YAML manifest | Handler fn |
| **Effect operations (request)** | `useState`, `useEffect`, `Suspense throw` | `:reply/:noreply`, `Process.send` | `@Inject`, `@Autowired` | `Cmd msg`, `Sub msg` | `spec:` block | Return value |
| **State (held by runtime)** | Fiber slots | Process heap | Container singleton | Model | etcd | Execution context |
| **Handler / Orchestrator** | React Reconciler | BEAM scheduler + Supervisor | IoC container | Elm runtime | kube-controller-manager | Lambda service |
| **Failure recovery** | Error boundary | Supervisor restart strategy | (ít hỗ trợ) | Runtime exception | Controller reconcile loop | Retry + DLQ |

**5 câu hỏi cần hỏi với mọi framework mới:**

1. *Effect signature* gồm những operations gì? (state, async, exception, message-send, …)
2. *Handler* sống ở đâu? Lifetime của nó là gì? Có bao nhiêu handler nested?
3. *Failure model* là gì? (exception bubble, supervisor restart, replay từ event log, retry policy)
4. *Identity và lifetime* của state holder là gì? (unmount = lost? crash = lost? persistent?)
5. *Ranh giới pure/impure* được vẽ ở đâu? Có thể test pure phần riêng không?

Năm câu này là bộ đo lường thống nhất — trả lời được cho một framework, bạn hiểu kiến trúc của nó sâu hơn hầu hết tài liệu hướng dẫn chính thức.

### 5.4. Vì sao hiểu cả ba (+ LiveView) tạo ra tư duy kiến trúc vượt trội

**Năng lực 1: Nhận diện pattern xuyên domain**

Khi đọc Vue Composition API, Svelte stores, Jetpack Compose, SwiftUI — bạn lập tức nhận ra "đây cũng là IoC + pure transition". Không cần học lại từ đầu, chỉ cần map sang mental model đã có.

**Năng lực 2: Thiết kế full-stack cùng triết lý**

Một hệ thống đẹp có client React/LiveView + server GenServer, cùng một mental model: *state là kết quả tích lũy của events qua pure transition*. Event sourcing, CQRS trở nên tự nhiên thay vì exotic.

**Năng lực 3: Gỡ rối ở tầng đúng**

Một bug "state không update" có thể là:
- Dependency array sai → `useEffect` không chạy lại (React)
- `handle_cast` không trả `{:noreply, newState}` mà trả sai tuple (Elixir)
- `assign` được gọi nhưng `render` không đọc đúng key (LiveView)
- DI container scope sai → wrong implementation injected (IoC)

Cả bốn đều là cùng một họ lỗi: **callback contract bị vi phạm**. Hiểu IoC giúp debug ở mức triết học, không chỉ syntax.

**Năng lực 4: Đánh giá trade-off xác đáng**

Biết khi nào nên:
- Dùng `useReducer` thay vì 5 `useState` lồng nhau
- Đẩy state lên server làm GenServer thay vì giữ ở client
- Dùng LiveView thay vì React + API cho realtime feature
- Phá vỡ IoC để có control flow rõ ràng (imperative animation, manual transaction)

**Năng lực 5: Tránh anti-pattern phổ biến**

- Giữ state mutable trong closure React → stale closure
- Gọi `GenServer.call` đệ quy → deadlock
- Lạm dụng DI container thành service locator
- Dùng `assigns` của LiveView như global state không kiểm soát

Tất cả đều tránh được nếu đã nội hóa: *pure transition function + framework-driven control + dependency injection tường minh*.

### 5.5. Hệ quả với testing: pure transition → test là value assertion

Đây là hệ quả tất yếu của paradigm chưa được nói thẳng: khi tách "mô tả khai báo" khỏi "thực thi có side effect", **test business logic trở thành so sánh giá trị thuần** — không mock, không setup, không teardown phức tạp.

**React reducer:**
```javascript
// Test: gọi pure function, assert output — không mount component, không render DOM
test('increment action', () => {
  const state    = { count: 0 };
  const newState = counterReducer(state, { type: 'increment' });
  expect(newState).toEqual({ count: 1 });
});

// Custom hook test với react-hooks-testing-library
test('useFetch returns loading then data', async () => {
  const { result, waitForNextUpdate } = renderHook(() => useFetch('/api/users'));
  expect(result.current.loading).toBe(true);
  await waitForNextUpdate();
  expect(result.current.data).toBeDefined();
});
```

**Elixir GenServer callback:**
```elixir
# Test handle_call trực tiếp — không start process, không message passing
test "increment increases count" do
  {:reply, :ok, new_state} = ShoppingCart.handle_cast({:add, item}, %{items: []})
  assert length(new_state.items) == 1
end

# Test Ecto Changeset — không cần database
test "rejects invalid email" do
  changeset = User.registration_changeset(%User{}, %{email: "not-an-email", name: "Alice"})
  refute changeset.valid?
  assert {:email, _} = List.keyfind(changeset.errors, :email, 0)
end
```

**Elm / TEA:**
```elm
test "increment msg updates model" =
    let
        (newModel, _) = update Increment { count = 0 }
    in
    Expect.equal newModel.count 1
```

**Điểm mấu chốt:** Trong tất cả các ví dụ trên, test chỉ là `f(input) == expected_output`. Không có database connection. Không có HTTP mock. Không có `before/afterEach` setup phức tạp. Tốc độ test suite tính bằng milliseconds thay vì phút.

Phần impure (HTTP call, DB write, DOM mutation) được test riêng ở tầng integration/e2e — và vì nó mỏng (chỉ là shell gọi framework), số lượng test cần ít hơn nhiều. Đây là lý do các codebase Elm và Elixir production thường đạt coverage cao với effort test thấp hơn so với codebase OOP imperative cùng quy mô.

**So sánh với paradigm imperative:**
```
Imperative (class + DI + mock):        Pure transition:
────────────────────────────────        ──────────────────────────────
@Mock UserRepository repo               test "reducer handles action" do
@InjectMocks UserService service          state  = %{count: 0}
                                          result = MyReducer.call(state, :inc)
@BeforeEach                               assert result.count == 1
void setup() {                          end
  when(repo.findById(1L))
    .thenReturn(Optional.of(user));     # 3 dòng test, 0 mock
}

@Test                                   # so với
void testGetUser() {                    # 15+ dòng setup + mock
  User result = service.getUser(1L);   # cho cùng một assertion
  assertEquals("Alice", result.name());
}
```

---


---

**Trước:** [← Phần 4 — LiveView](liveview.md) | **Tiếp theo:** [Phần 6 — Trade-offs →](tradeoffs.md)
