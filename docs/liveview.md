# Phần 4 — Phoenix LiveView: Nơi Ba Khái Niệm Hợp Nhất

## 4.1. LiveView là gì và tại sao nó là "kết tinh"

Phoenix LiveView là thư viện cho phép xây dựng giao diện người dùng **real-time, interactive** mà **không cần viết JavaScript**. Nó không phải là trick hay SSR thông thường — nó là một mô hình lập trình mới, được xây dựng trực tiếp trên nền tảng GenServer và WebSocket.

**Điều làm LiveView đặc biệt:** mỗi LiveView mount tạo ra một **GenServer process riêng** cho mỗi client connection. Process đó:

- Giữ state của UI trong heap của nó (như GenServer)
- Nhận events từ browser qua WebSocket (như `handle_cast`)
- Gọi `render/1` để tính HTML mới (như React component function)
- Diff HTML cũ và mới, **gửi chỉ phần thay đổi xuống browser dưới dạng binary term format của Erlang** — không phải JSON text, không phải full HTML — mỗi event thường chỉ tạo payload vài chục bytes (như React reconciliation nhưng diễn ra server-side)
- Được supervisor quản lý lifecycle, tự restart nếu crash

**Binary diff protocol — lý do LiveView scale được:** React diff diễn ra ở client (tốn CPU browser), LiveView diff diễn ra ở server (CPU BEAM) nhưng chỉ truyền kết quả diff xuống wire. Erlang binary term format thường nén structured data hiệu quả hơn JSON đáng kể (mức độ tùy thuộc data shape). Kết quả: một interaction điển hình (update một row trong table, toggle một flag) chỉ tạo payload rất nhỏ qua WebSocket — thường vài chục bytes — so với SSR truyền thống phải gửi lại toàn bộ HTML fragment. Đây là lý do LiveView đạt được latency cảm giác như SPA mà không cần client-side state management. Chris McCord đã công bố các benchmark cụ thể trong các keynote ElixirConf nếu cần số liệu chính xác cho từng workload.

Đây là lần đầu tiên trong lịch sử mainstream có một framework mà **UI declarative + stateful process + IoC** hợp nhất thành **một primitive duy nhất**, không phải ba thứ ghép lại.

```text
LiveView Architecture:
──────────────────────────────────────────────────────────
Browser                    Server (BEAM)
───────                    ─────────────────────────────

HTML initial load    ────► Router → LiveView.mount/3
                                       │
WebSocket connect    ────►             │ GenServer process spawned
                                       │ state = %Socket{assigns: %{...}}
                           ◄──────────►│ (supervised, isolated)
                                       │
phx-click event      ────►    handle_event/3 → assign(socket, ...)
                                       │
                    ◄──── binary diff  │ render/1 → new HTML
                          (minimal     │ diff old vs new
                           payload)    │ send only changed parts
```

## 4.2. LiveView như React: declarative UI pattern

Nhìn vào một LiveView component, bạn sẽ nhận ra ngay cấu trúc React:

```elixir
defmodule MyAppWeb.ShoppingCartLive do
  use Phoenix.LiveView

  # mount/3 = componentDidMount + useState initial value
  def mount(_params, %{"user_id" => user_id}, socket) do
    cart = MyApp.Cart.get(user_id)

    socket =
      socket
      |> assign(:cart, cart)
      |> assign(:user_id, user_id)

    if connected?(socket) do
      # Subscribe to real-time updates (như useEffect với subscription)
      MyApp.Cart.subscribe(user_id)
    end

    {:ok, socket}
  end

  # handle_params/3 = React Router params + useEffect([params])
  def handle_params(%{"filter" => filter}, _uri, socket) do
    {:noreply, assign(socket, :filter, filter)}
  end

  # handle_event/3 = event handlers trong React (onClick, onChange, ...)
  def handle_event("add_item", %{"item_id" => id}, socket) do
    {:ok, cart} = MyApp.Cart.add_item(socket.assigns.user_id, id)
    {:noreply, assign(socket, :cart, cart)}
  end

  def handle_event("remove_item", %{"id" => id}, socket) do
    {:ok, cart} = MyApp.Cart.remove_item(socket.assigns.user_id, id)
    {:noreply, assign(socket, :cart, cart)}
  end

  def handle_event("checkout", _params, socket) do
    case MyApp.Payment.process(socket.assigns.cart) do
      {:ok, order} ->
        {:noreply,
         socket
         |> assign(:cart, %{items: []})
         |> put_flash(:info, "Order #{order.id} placed!")}

      {:error, reason} ->
        {:noreply, put_flash(socket, :error, "Payment failed: #{reason}")}
    end
  end

  # handle_info/3 = useEffect subscription callback
  # Được gọi khi có PubSub message từ server
  def handle_info({:cart_updated, cart}, socket) do
    {:noreply, assign(socket, :cart, cart)}
  end

  # render/1 = React component return (JSX → HEEx template)
  def render(assigns) do
    ~H"""
    <div class="cart">
      <h2>Cart (<%= length(@cart.items) %> items)</h2>

      <%= for item <- @cart.items do %>
        <div class="item" id={"item-#{item.id}"}>
          <span><%= item.name %></span>
          <button phx-click="remove_item" phx-value-id={item.id}>Remove</button>
        </div>
      <% end %>

      <button phx-click="checkout" disabled={Enum.empty?(@cart.items)}>
        Checkout (<%= format_price(@cart.total) %>)
      </button>
    </div>
    """
  end
end
```

**Đối chiếu trực tiếp LiveView ↔ React:**

| Concept | React | Phoenix LiveView |
| --- | --- | --- |
| **Khởi tạo state** | `useState(initialValue)` | `assign(socket, key, value)` trong `mount/3` |
| **State container** | Fiber node (browser memory) | `%Socket{assigns: %{...}}` (server process heap) |
| **Update state** | `setState` / `dispatch` | `assign(socket, ...)` trả về socket mới |
| **Event handler** | `onClick`, `onChange` | `handle_event("name", params, socket)` |
| **Server message** | Không có native equivalent | `handle_info(message, socket)` |
| **Side effects** | `useEffect` với cleanup | `mount + connected?/1`, `handle_info` |
| **Template** | JSX | HEEx template |
| **Diffing** | Virtual DOM (client-side) | HTML diff (server-side, binary protocol) |
| **Reconciliation** | React reconciler (browser) | LiveView diff engine (BEAM) |
| **Lifecycle cleanup** | `useEffect` return function | `terminate/2` của GenServer |

## 4.3. LiveView như GenServer: process isolation và fault tolerance

Mỗi LiveView mount là một **GenServer process được supervisor quản lý**. Điều này mang lại toàn bộ OTP guarantees:

```elixir
# LiveView process tree (auto-managed bởi Phoenix)
Phoenix.LiveView.Supervisor
├── LiveView{conn_1, ShoppingCartLive}   # user 1 tab 1
├── LiveView{conn_2, ShoppingCartLive}   # user 1 tab 2
├── LiveView{conn_3, ProductListLive}    # user 2
└── LiveView{conn_N, ...}

# Nếu conn_1 crash: chỉ user 1 tab 1 bị ảnh hưởng
# Supervisor restart process → user thấy page reload
# Tất cả connections khác không bị ảnh hưởng
```

**"Let it crash" trong LiveView context:**

```elixir
def handle_event("process_payment", params, socket) do
  # Không defensive try/catch ở đây
  # Nếu crash → process restart → user thấy reload
  # Tốt hơn là silent failure với corrupted state
  order = MyApp.Payment.process!(params)   # có thể raise
  {:noreply, assign(socket, :order, order)}
end
```

**Stateful WebSocket — không cần session store:**

Vì state sống trong process heap, LiveView không cần Redis session store hay JWT refresh logic cho realtime state. State được tự động persist trong suốt lifetime của connection, và tự động cleanup khi user disconnect:

```elixir
def terminate(_reason, socket) do
  # Tự động gọi khi WebSocket disconnect
  MyApp.Analytics.track_session_end(socket.assigns.user_id)
  :ok
end
```

## 4.4. LiveView như IoC: Hollywood Principle hoàn hảo nhất

LiveView là hiện thân cực đoan nhất của Hollywood Principle. Bạn không:

- Gọi WebSocket manually
- Manage connection lifecycle
- Diff HTML manually
- Gọi `render` khi nào cần
- Handle reconnection logic
- Manage message queue

Phoenix LiveView làm tất cả. Bạn chỉ implement các callbacks tại điểm được định nghĩa:

```text
LiveView Hollywood Principle:
─────────────────────────────────────────────────────
"Don't call us..."          "We'll call you when..."
────────────────            ────────────────────────
(bạn không gọi)             (LiveView gọi callback của bạn)

WebSocket.connect()    ──►  mount/3
DOM.diff()             ──►  render/1 (sau mỗi assign)
Event.listen()         ──►  handle_event/3
PubSub.subscribe()     ──►  handle_info/2
Session.cleanup()      ──►  terminate/2
```

## 4.5. LiveView Streams: giải quyết bài toán large list

LiveView Streams (Phoenix 0.19+) là pattern giải quyết vấn đề render list lớn — tương tự virtualization trong React nhưng khác cơ chế:

```elixir
def mount(_params, _session, socket) do
  socket = stream(socket, :messages, MyApp.Messages.list_recent(100))
  {:ok, socket}
end

def handle_event("load_more", _params, socket) do
  more = MyApp.Messages.load_more(socket.assigns.cursor)
  {:noreply, stream_insert(socket, :messages, more, at: -1)}
end

def handle_info({:new_message, msg}, socket) do
  # Prepend mới nhất — LiveView chỉ gửi diff của 1 element mới
  {:noreply, stream_insert(socket, :messages, msg)}
end
```

```heex
<div id="messages" phx-update="stream">
  <%= for {id, message} <- @streams.messages do %>
    <div id={id}>
      <%= message.content %>
    </div>
  <% end %>
</div>
```

LiveView Streams không giữ toàn bộ list trong server-side state — chỉ gửi operations (insert/delete/update) xuống client, client tự apply. Đây là pattern cực kỳ efficient cho realtime feed.

## 4.6. LiveView Components: composable stateless + stateful

LiveView cung cấp hai loại component, mirror chính xác React's class vs functional components nhưng với semantic khác:

```elixir
# Stateless Function Component (như React functional component)
# Không có process riêng, không có lifecycle
defmodule MyAppWeb.ButtonComponent do
  use Phoenix.Component

  attr :label, :string, required: true
  attr :on_click, :string, required: true
  attr :disabled, :boolean, default: false

  def primary_button(assigns) do
    ~H"""
    <button
      phx-click={@on_click}
      disabled={@disabled}
      class="btn btn-primary">
      <%= @label %>
    </button>
    """
  end
end

# Stateful LiveComponent (có process riêng, có lifecycle)
# Tương tự như React component với state riêng nhưng vẫn là server-side
defmodule MyAppWeb.CartItemComponent do
  use Phoenix.LiveComponent

  def update(%{item: item}, socket) do
    {:ok, assign(socket, :item, item)}
  end

  def handle_event("toggle_select", _params, socket) do
    {:noreply, update(socket, :selected, &(!&1))}
  end

  def render(assigns) do
    ~H"""
    <div class={[@selected && "selected"]}>
      <%= @item.name %>
      <button phx-click="toggle_select" phx-target={@myself}>Select</button>
    </div>
    """
  end
end
```

## 4.7. JavaScript Hooks: escape hatch giống `useRef` + imperative handle

Đôi khi cần JS native (animation, charting library, clipboard). LiveView JS Hooks là escape hatch:

```javascript
// app.js — tương tự useRef + useImperativeHandle trong React
let Hooks = {}

Hooks.Chart = {
  mounted() {
    // this.el = DOM element
    this.chart = new Chart(this.el, {
      type: 'line',
      data: JSON.parse(this.el.dataset.chartData)
    });
  },

  updated() {
    // LiveView cập nhật data-chart-data attribute
    this.chart.data = JSON.parse(this.el.dataset.chartData);
    this.chart.update();
  },

  destroyed() {
    this.chart.destroy();
  }
}

let liveSocket = new LiveSocket("/live", Socket, { hooks: Hooks });
```

```heex
<canvas id="revenue-chart"
        phx-hook="Chart"
        data-chart-data={Jason.encode!(@chart_data)}>
</canvas>
```

**Pattern này mirror chính xác React's escape hatch:**

| React | LiveView |
| --- | --- |
| `useRef(null)` | `phx-hook="HookName"` |
| `useEffect(() => { /* setup */ }, [])` | `mounted()` callback |
| `useEffect cleanup` | `destroyed()` callback |
| `useEffect` khi deps thay đổi | `updated()` callback |
| `useImperativeHandle` | `this.pushEvent`, `this.handleEvent` |

## 4.8. Realtime multi-user: PubSub + LiveView

Đây là điểm LiveView làm được mà React (without WebSocket library) không có native:

```elixir
defmodule MyAppWeb.BoardLive do
  use Phoenix.LiveView

  def mount(%{"board_id" => board_id}, _session, socket) do
    if connected?(socket) do
      # Subscribe vào board-specific topic
      Phoenix.PubSub.subscribe(MyApp.PubSub, "board:#{board_id}")
    end

    board = MyApp.Board.get!(board_id)
    {:ok, assign(socket, :board, board)}
  end

  def handle_event("move_card", %{"card_id" => id, "to_col" => col}, socket) do
    {:ok, board} = MyApp.Board.move_card(id, col)

    # Broadcast tới tất cả users đang xem board này
    Phoenix.PubSub.broadcast(
      MyApp.PubSub,
      "board:#{board.id}",
      {:board_updated, board}
    )

    {:noreply, assign(socket, :board, board)}
  end

  # Được gọi khi bất kỳ user nào broadcast update
  def handle_info({:board_updated, board}, socket) do
    {:noreply, assign(socket, :board, board)}
  end
end
```

Với ~20 dòng code, ta có Kanban board realtime multi-user, không cần Redux, không cần WebSocket client library, không cần API endpoint riêng cho realtime.

---

**Trước:** [← Phần 3 — IoC](ioc.md) | **Tiếp theo:** [Phần 5 — Bản đồ tư duy thống nhất →](unified-map.md)
