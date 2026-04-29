# Phần 2 — Elixir, Actor Model và GenServer

## 2.1. Bối cảnh: BEAM, immutability, và process

Elixir compile xuống bytecode chạy trên **BEAM VM** (kế thừa từ Erlang, ra đời ở Ericsson những năm 1980 cho hệ thống viễn thông đòi hỏi 99.9999999% uptime — "nine nines"). Hai đặc tính cốt lõi không thể tách rời:

**Immutability tuyệt đối:** mọi data structure đều immutable. State không bao giờ được mutate tại chỗ — nó được "chuyển hóa" qua các function trả về giá trị mới. Điều này loại bỏ toàn bộ lớp lỗi liên quan đến shared mutable state.

**Process isolation triệt để:** process trong BEAM không phải OS thread mà là lightweight process do scheduler của VM quản lý. Một node có thể chạy hàng triệu process. Mỗi process có **mailbox riêng**, **heap riêng**, không chia sẻ memory với process khác. Giao tiếp duy nhất là qua **message passing bất đồng bộ**.

```text
BEAM Process Model vs OS Thread:
──────────────────────────────────────────────────
OS Thread:                     BEAM Process:
┌────────────┐                 ┌────────────────┐
│ Shared     │◄──── LOCK ────► │ PID: #PID<0.1> │
│ Memory     │                 │ Mailbox: []    │
│ RACE       │                 │ Heap: isolated │
│ CONDITIONS │                 └────────┬───────┘
└────────────┘                          │ message
                                        ▼
                               ┌────────────────┐
                               │ PID: #PID<0.2> │
                               │ Mailbox: [msg] │
                               │ Heap: isolated │
                               └────────────────┘
                               (Zero shared state)
```

Đây là hiện thân thuần khiết nhất của **Actor Model** (Carl Hewitt, 1973): actor có ID, có mailbox, có thể (a) gửi message tới actor khác, (b) spawn actor mới, và (c) thay đổi hành vi cho message kế tiếp.

## 2.2. State trong thế giới immutable: recursive loop + tham số

Vì không thể mutate, làm sao một process "giữ" state? Câu trả lời là **đệ quy đuôi (tail recursion) trên tham số trạng thái**:

```elixir
defmodule Counter do
  def start(initial), do: spawn(fn -> loop(initial) end)

  defp loop(state) do
    receive do
      {:inc, from} ->
        new_state = state + 1
        send(from, {:ok, new_state})
        loop(new_state)              # gọi lại chính mình với state mới

      {:get, from} ->
        send(from, {:value, state})
        loop(state)                  # state không thay đổi

      :stop ->
        :ok                          # thoát loop, process kết thúc
    end
  end
end
```

State "sống" trong **stack frame của hàm `loop/1`**. Mỗi message làm hàm gọi lại chính nó với tham số đã được transform. Đây là cơ chế "stateful không cần biến mutable". BEAM tối ưu tail call nên đây không phải stack overflow — mỗi call thay thế frame hiện tại.

```text
State evolution trong recursive loop:
──────────────────────────────────────
loop(0)       ← initial
  receive {:inc, ...}
loop(1)       ← new frame
  receive {:inc, ...}
loop(2)       ← new frame
  receive {:get, ...}
loop(2)       ← không thay đổi, vẫn là frame mới
```

## 2.3. GenServer: behaviour kết tinh pattern client-server

Pattern recursive-receive-loop quá phổ biến đến mức OTP cung cấp một **behaviour** đóng gói nó: `GenServer`. Behaviour trong Elixir gần giống interface trong Java, nhưng mạnh hơn — nó còn cung cấp implementation mặc định, supervision integration, hot code swapping, telemetry, và distributed process registry.

Một module dùng `GenServer` chỉ cần khai báo các callback nó quan tâm; khung sườn còn lại (receive loop, dispatching, error reporting, monitoring, timeout handling) do OTP lo:

```elixir
defmodule ShoppingCart do
  use GenServer

  # ===== Client API (chạy trong process gọi) =====
  def start_link(user_id) do
    GenServer.start_link(__MODULE__, %{user_id: user_id, items: []},
      name: via(user_id))
  end

  def add_item(user_id, item),   do: GenServer.cast(via(user_id), {:add, item})
  def remove_item(user_id, id),  do: GenServer.cast(via(user_id), {:remove, id})
  def get_cart(user_id),         do: GenServer.call(via(user_id), :get)
  def checkout(user_id),         do: GenServer.call(via(user_id), :checkout, 10_000)

  defp via(user_id), do: {:via, Registry, {MyApp.Registry, {:cart, user_id}}}

  # ===== Server callbacks (chạy trong process GenServer) =====

  @impl true
  def init(state), do: {:ok, state, {:continue, :load_from_db}}

  @impl true
  def handle_continue(:load_from_db, state) do
    items = MyApp.DB.load_cart(state.user_id)
    {:noreply, %{state | items: items}}
  end

  @impl true
  def handle_call(:get, _from, state) do
    {:reply, state, state}          # trả về state, giữ nguyên state
  end

  @impl true
  def handle_call(:checkout, _from, state) do
    case MyApp.Payment.process(state) do
      {:ok, order} ->
        {:reply, {:ok, order}, %{state | items: []}}   # clear cart
      {:error, reason} ->
        {:reply, {:error, reason}, state}               # giữ nguyên
    end
  end

  @impl true
  def handle_cast({:add, item}, state) do
    new_items = [item | state.items]
    {:noreply, %{state | items: new_items}}
  end

  @impl true
  def handle_cast({:remove, id}, state) do
    new_items = Enum.reject(state.items, &(&1.id == id))
    {:noreply, %{state | items: new_items}}
  end

  @impl true
  def handle_info(:timeout_cleanup, state) do
    # Được gọi bởi Process.send_after/3 sau X milliseconds
    {:stop, :normal, state}         # tự shutdown sau idle timeout
  end

  @impl true
  def terminate(reason, state) do
    # Cleanup khi bị shutdown (supervisor stop hoặc crash)
    MyApp.DB.persist_cart(state.user_id, state.items)
    :ok
  end
end
```

**Quan sát kiến trúc quan trọng:**

- `init/1` thiết lập state ban đầu — **tương tự initial value của `useState`/`useReducer`**
- `handle_call/3` xử lý request đồng bộ và trả về `{:reply, response, newState}` — **tương tự reducer trong `useReducer`**
- `handle_cast/2` xử lý message bất đồng bộ (fire-and-forget), trả về `{:noreply, newState}`
- `handle_info/2` xử lý các "system message" tự do — timeout, monitor DOWN, external port
- `terminate/2` — **tương tự cleanup function của `useEffect`**

GenServer **về bản chất là một state machine**: mỗi callback là một transition function `(state, message) → new_state`, hoàn toàn pure trên giá trị state, dù được orchestrate trong một process có lifecycle.

## 2.4. Supervisor và supervision tree: "let it crash"

Triết lý Joe Armstrong: **đừng viết defensive code; hãy để process crash, rồi khôi phục về trạng thái sạch**. Process crash không làm sập hệ thống vì supervisor sẽ phát hiện và restart theo strategy đã định. Đây là **fault isolation kiến trúc** — không phải defensive programming bằng try/catch.

```elixir
defmodule MyApp.Application do
  use Application

  def start(_type, _args) do
    children = [
      # Lõi: Database pool — nếu crash, restart toàn bộ app
      {MyApp.Repo, []},

      # Registry cho dynamic processes
      {Registry, keys: :unique, name: MyApp.Registry},

      # DynamicSupervisor: quản lý shopping carts (spawn on-demand)
      {DynamicSupervisor, name: MyApp.CartSupervisor, strategy: :one_for_one},

      # Web endpoint
      MyAppWeb.Endpoint
    ]

    Supervisor.start_link(children, strategy: :one_for_one, name: MyApp.Supervisor)
  end
end

# Spawn cart process khi cần
def get_or_create_cart(user_id) do
  case Registry.lookup(MyApp.Registry, {:cart, user_id}) do
    [{pid, _}] -> pid
    [] ->
      {:ok, pid} = DynamicSupervisor.start_child(
        MyApp.CartSupervisor,
        {ShoppingCart, user_id}
      )
      pid
  end
end
```

**Các supervision strategy:**

```text
:one_for_one    — chỉ restart process bị crash (processes độc lập)
:one_for_all    — restart tất cả child khi một child crash (processes phụ thuộc nhau)
:rest_for_one   — restart child crash + tất cả child đứng sau nó trong list
```

**Cây supervisor (supervision tree) tạo nên kiến trúc fault-isolation phân tầng:**

```text
MyApp.Supervisor (:one_for_one)
├── MyApp.Repo                    ← critical, nếu crash → app restart
├── MyApp.Registry                ← critical
├── MyApp.CartSupervisor          ← DynamicSupervisor
│   ├── ShoppingCart{user_1}      ← crash không ảnh hưởng user_2
│   ├── ShoppingCart{user_2}
│   └── ShoppingCart{user_N}
└── MyAppWeb.Endpoint
```

**So sánh với Kubernetes:** cây supervisor này là điều mà Kubernetes/Docker Swarm phải mô phỏng ở tầng OS process. OTP làm điều tương tự ở tầng ứng dụng, với latency microseconds thay vì seconds.

## 2.5. So sánh trực diện: GenServer vs `useState`/`useReducer`

| Khía cạnh | React `useReducer` | Elixir `GenServer` |
| --- | --- | --- |
| **Đơn vị state** | Component instance (fiber node) | Lightweight process (PID) |
| **Cập nhật state** | `dispatch(action)` → reducer | `call/cast` → `handle_*` |
| **Bản chất hàm cập nhật** | Pure: `(state, action) -> newState` | Pure: `(msg, state) -> {reply, newState}` |
| **Đồng bộ vs bất đồng bộ** | `dispatch` luôn sync; effect async | `call` sync (block); `cast` async |
| **Xử lý lỗi** | Error boundary (component cha) | Supervisor (process cha) |
| **Cô lập state** | Cô lập theo subtree React (share JS heap) | Cô lập tuyệt đối (heap riêng) |
| **Lifecycle** | Mount/update/unmount của React | `init/1`, `terminate/2`, `code_change/3` |
| **Scope** | Trong process render của 1 tab | Toàn node BEAM, có thể distributed |
| **Failure recovery** | Error boundary re-render | Supervisor restart với clean state |
| **Test** | Render hook + assert state | GenServer.call mock process |

**Trade-off cốt lõi:**

- React reducer: chạy đồng bộ trong JS thread → đơn giản hơn, nhưng không có fault isolation thật sự. Một exception trong render subtree có thể leak nếu thiếu error boundary.
- GenServer: chạy trong process riêng → concurrency thật, fault isolation, distribution, nhưng phải trả giá bằng overhead message passing và cognitive cost của asynchronous reasoning.

## 2.6. GenStage và Flow: backpressure-aware pipeline

Ngoài GenServer, OTP cung cấp **GenStage** — behaviour chuyên biệt cho data pipeline với built-in backpressure. Đây là điểm mà Elixir vượt xa các runtime khác trong việc xây dựng streaming systems:

```elixir
# Producer: phát ra events
defmodule MyProducer do
  use GenStage

  def handle_demand(demand, state) do
    events = fetch_events(demand)    # chỉ fetch đúng số lượng demand
    {:noreply, events, state}
  end
end

# Consumer: tiêu thụ events (rate tự điều chỉnh qua backpressure)
defmodule MyConsumer do
  use GenStage

  def handle_events(events, _from, state) do
    Enum.each(events, &process/1)
    {:noreply, [], state}
  end
end
```

Pattern này không có equivalent trực tiếp trong React ecosystem (RxJS approach gần nhất), nhưng cho thấy breadth của tư duy "declarative pipeline" trong OTP.

## 2.7. Ecto Changeset: Pure Core / Impure Shell trong thực chiến

Nếu GenServer là ví dụ rõ nhất của pure transition function trong OTP, thì **Ecto Changeset** là ví dụ rõ nhất của "Functional Core, Imperative Shell" áp dụng cho tầng data và nghiệp vụ.

Bài toán quen thuộc: validate và transform dữ liệu đầu vào (form submission, API payload) trước khi lưu xuống database. Cách imperative truyền thống: trộn lẫn validation logic với database call, khiến test phải mock database dù chỉ muốn test "địa chỉ email có đúng format không".

Ecto tách hoàn toàn:

```elixir
defmodule MyApp.Accounts.User do
  use Ecto.Schema
  import Ecto.Changeset

  schema "users" do
    field :email, :string
    field :name, :string
    field :age, :integer
    field :role, Ecto.Enum, values: [:user, :admin, :moderator]
    timestamps()
  end

  # Changeset là pure function hoàn toàn — không đụng database
  # Input: struct + params chưa validated
  # Output: %Changeset{} mô tả kết quả validation + transformation
  def registration_changeset(user, params) do
    user
    |> cast(params, [:email, :name, :age, :role])      # whitelist fields
    |> validate_required([:email, :name])              # required check
    |> validate_format(:email, ~r/@/)                  # format check
    |> validate_number(:age, greater_than: 0)          # range check
    |> validate_inclusion(:role, [:user, :moderator])  # enum check
    |> unique_constraint(:email)                       # DB constraint (deferred)
    |> put_change(:role, :user)                        # transformation
  end

  def admin_changeset(user, params) do
    user
    |> cast(params, [:email, :name, :age, :role])
    |> validate_required([:email, :name, :role])
    # admin có thể set bất kỳ role nào
  end
end

# Functional core: pure, testable không cần database
changeset = User.registration_changeset(%User{}, %{email: "bad", name: ""})
changeset.valid?          # => false
changeset.errors          # => [name: {"can't be blank", ...}, email: {"has invalid format", ...}]

# Impure shell: chỉ gọi Repo khi changeset hợp lệ
case Repo.insert(changeset) do
  {:ok, user}    -> {:ok, user}
  {:error, cs}   -> {:error, cs}  # DB-level errors (unique constraint) merge vào changeset
end
```

**Điểm then chốt về kiến trúc:**

`cast → validate → transform` là một **pipeline của pure functions** — không có side effect, không có DB call, không có network. `Repo.insert/update/delete` là *toàn bộ* impure shell, được tách hoàn toàn. Kết quả:

- Test validation logic là `assert changeset.valid? == false` — không cần database, không cần mock, chạy nhanh milliseconds.
- Cùng một changeset reuse được ở nhiều context: API endpoint, admin panel, background job — mỗi context gọi đúng changeset function tương ứng.
- Business rules (validation) và persistence (Repo) thay đổi độc lập: đổi database provider không ảnh hưởng một dòng validation.

**Ánh xạ sang React:**

| Ecto | React |
| --- | --- |
| `%Changeset{}` (mô tả transformation + errors) | `state` trong `useReducer` |
| `cast/validate/transform` pipeline (pure) | `reducer(state, action) -> newState` (pure) |
| `Repo.insert(changeset)` (impure shell) | `useEffect(() => fetch(...))` (impure shell) |
| `unique_constraint` (deferred DB check) | Server-side validation error từ API response |
| `changeset.valid?` | `formState.isValid` trong React Hook Form |

`unique_constraint` trong Ecto là một ví dụ tinh tế: đây là constraint chỉ database mới biết (hai request đồng thời cùng insert email giống nhau), nhưng Ecto vẫn "merge" lỗi đó vào `%Changeset{}` struct sau khi DB trả về — giữ toàn bộ error state trong một object nhất quán, thay vì throw exception ở đâu đó trong code.

---

Từ [Phần 1 (React)](react.md) đến Phần 2 (Elixir/OTP), ta đã thấy hai ngữ cảnh hoàn toàn khác nhau — UI trên trình duyệt và process trên BEAM — đều áp dụng cùng một cấu trúc: **pure transition function được orchestrate bởi runtime**. Câu hỏi tự nhiên đặt ra: cơ chế trừu tượng nào ngầm chi phối cả hai? Tại sao framework gọi code của mày, không phải mày gọi framework? Đó chính là Inversion of Control — nguyên lý kiến trúc mà [Phần 3](ioc.md) sẽ mổ xẻ.

---

**Trước:** [← Phần 1 — React](react.md) | **Tiếp theo:** [Phần 3 — Inversion of Control →](ioc.md)
