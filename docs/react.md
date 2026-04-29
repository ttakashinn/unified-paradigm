# Phần 1 — React hiện đại dưới lăng kính Functional Programming

### 1.1. Mô hình lập trình cốt lõi: UI = f(state)

Triết lý nền tảng của React — được Dan Abramov và đội ngũ React core nhắc đi nhắc lại — là: **giao diện người dùng nên là một hàm thuần của trạng thái**. Component lý tưởng là một pure function: cùng một input (`props`, `state`), luôn trả về cùng một output (cây React elements).

Chính tư tưởng này khiến class component trở nên gượng ép. Lifecycle methods (`componentDidMount`, `componentDidUpdate`, `componentWillUnmount`) phân mảnh logic theo **thời điểm thực thi của instance** thay vì theo **ý nghĩa nghiệp vụ**. Cùng một feature — ví dụ: subscribe vào một event source rồi unsubscribe — bị chẻ ra ba phương thức khác nhau, trong khi hai feature không liên quan lại bị nhồi chung trong `componentDidMount`.

Hooks ra đời (React 16.8, đầu 2019) để giải quyết đúng vấn đề này. Theo Dan Abramov, hooks không phải là cú pháp đường ngọt cho class — chúng là một **mô hình lập trình mới**: thay vì nghĩ theo lifecycle, ta nghĩ theo *đồng bộ hóa giữa state và side effect*.

```
Class Component (tư duy cũ):         Functional Component (tư duy mới):
─────────────────────────────         ──────────────────────────────────
"Khi nào mount? → làm gì"            "State X → effect Y phải đúng"
"Khi nào update? → làm gì"           (Synchronization model)
"Khi nào unmount? → làm gì"

Logic bị phân mảnh theo thời gian    Logic được gom theo ý nghĩa
```

### 1.2. Hooks là "algebraic effects" giả lập trong JavaScript

Đây là điểm tinh tế ít được thảo luận công khai. Dan Abramov đã nhiều lần tham chiếu đến **algebraic effects** như nền tảng lý thuyết cho hooks. Ý tưởng cốt lõi: khi một function `Component()` gọi `useState()`, nó không thực sự *thực hiện* việc cấp phát state — nó chỉ *yêu cầu* (raise an effect). React, đứng cao hơn trong call stack, đảm nhận vai trò "effect handler", quyết định trả về giá trị state nào, lưu trữ ở đâu, và khi nào re-render.

Nhờ đó, **component vẫn có thể được coi là "thuần" về mặt khái niệm**, ngay cả khi nó dùng state. Phần "không thuần" được nâng lên cho runtime. Đây là cùng một tinh thần với cách:
- Haskell xử lý IO qua monads
- Elixir tách logic xử lý message ra khỏi việc spawn/schedule process
- Kotlin coroutines xử lý suspension points

```
Algebraic Effect Model trong React:
────────────────────────────────────

  Component()                    React Reconciler
      │                               │
      │── useState(0) ──► raise ──►   │
      │                     │         │──► lưu vào fiber slot #0
      │◄────────── [0, setter] ◄──────│
      │                               │
      │── useEffect(fn) ──► raise ──► │
      │                               │──► schedule cleanup + run
      │                               │
  (component không biết gì về        (runtime quyết định tất cả)
   việc lưu trữ hay scheduling)
```

Đây chính là lý do hooks chỉ có thể gọi *trong* function component hoặc *trong* custom hook khác: bên ngoài cây gọi của React, "effect handler" không tồn tại nên hook ném lỗi `"Invalid hook call"`.

### 1.3. Closures và lexical scope: cơ chế ngầm dưới hooks

Mỗi lần render, **body của functional component thực thi lại từ đầu**. State "tồn tại" giữa các lần render được lưu trữ trong **fiber node** mà React quản lý nội bộ, được map cứng theo *thứ tự gọi hook* (đây chính là lý do "Rules of Hooks" cấm gọi hook trong điều kiện rẽ nhánh).

```jsx
function Counter() {
  const [count, setCount] = useState(0);          // hook slot #0
  const [name, setName]   = useState('Alice');    // hook slot #1

  // Closure: handleClick "bắt" tham chiếu count tại render hiện tại
  const handleClick = () => setCount(count + 1);

  useEffect(() => {
    document.title = `${name}: ${count}`;
    return () => { /* cleanup khi count/name thay đổi hoặc unmount */ };
  }, [count, name]);

  return <button onClick={handleClick}>{count}</button>;
}
```

**Vấn đề stale closure** — cạm bẫy quan trọng nhất của hooks: mỗi `handleClick` là một **closure mới** trên giá trị `count` của render đó. Nếu capture `count` trong `setInterval` mà quên đưa vào dependency array, bạn kẹt mãi với giá trị cũ:

```jsx
// BUG: count luôn là 0 vì closure được tạo ở render đầu tiên
useEffect(() => {
  const id = setInterval(() => console.log(count), 1000);
  return () => clearInterval(id);
}, []); // ← thiếu count trong dependency array

// FIX: dùng functional updater — không cần capture count
useEffect(() => {
  const id = setInterval(() => setCount(c => c + 1), 1000);
  return () => clearInterval(id);
}, []);
```

Hiểu sâu functional programming — đặc biệt closures và referential transparency — là **điều kiện cần** để gỡ rối hooks ở mức nâng cao. Đây không phải "React bug", đây là hệ quả tất yếu của việc dùng JS closures làm primitive cho state capture.

> **Đúc kết về dependency array:** `[count, name]` trong `useEffect` không phải "danh sách những thứ tao muốn watch". Đó là **bản hợp đồng bạn ký với React runtime**: *"Closure trong effect này đang capture những giá trị này — hãy làm mới effect khi chúng thay đổi."* Vi phạm hợp đồng (khai báo thiếu hoặc thừa) là vi phạm callback contract với framework — cùng một họ lỗi với việc `handle_cast` trả sai tuple trong GenServer, hay `@impl` không đúng signature trong OTP behaviour. Stale closure là IoC contract bị phá vỡ ở tầng closure.

**Cơ chế lưu trữ fiber (để rõ ràng hơn):**

```
Fiber Node của <Counter />:
┌─────────────────────────────┐
│  memoizedState (linked list)│
│  ┌─────────────────────┐    │
│  │ slot #0: count = 3  │    │
│  └──────────┬──────────┘    │
│             ▼               │
│  ┌─────────────────────┐    │
│  │ slot #1: name='Bob' │    │
│  └──────────┬──────────┘    │
│             ▼               │
│  ┌─────────────────────┐    │
│  │ slot #2: effect deps│    │
│  └─────────────────────┘    │
└─────────────────────────────┘
     ↑
     React dùng thứ tự gọi hook
     để biết slot nào là slot nào
```

### 1.4. Custom hooks: composition over inheritance

Custom hook là vũ khí thật sự của paradigm này. Nó cho phép **composition của stateful logic** mà các pattern cũ (HOC, render props, mixins) không làm sạch được.

```jsx
// useFetch.js — custom hook đóng gói logic fetch + cache + cancellation
function useFetch(url) {
  const [data, setData]       = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError]     = useState(null);

  useEffect(() => {
    const ctrl = new AbortController();
    setLoading(true);

    fetch(url, { signal: ctrl.signal })
      .then(r => r.json())
      .then(setData)
      .catch(e => { if (e.name !== 'AbortError') setError(e); })
      .finally(() => setLoading(false));

    return () => ctrl.abort();     // cleanup = abort khi url thay đổi
  }, [url]);

  return { data, loading, error };
}

// useDebounce.js — custom hook độc lập, có thể kết hợp
function useDebounce(value, delay) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);
  }, [value, delay]);
  return debounced;
}

// Component dùng — declarative, không lo lifecycle
function UserSearch() {
  const [query, setQuery] = useState('');
  const debouncedQuery    = useDebounce(query, 300);     // compose
  const { data, loading } = useFetch(`/api/users?q=${debouncedQuery}`); // compose

  return (
    <>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      {loading ? <Spinner/> : <List data={data}/>}
    </>
  );
}
```

Logic nghiệp vụ ("fetch và quản lý vòng đời request", "debounce input") được tách hoàn toàn khỏi cả lifecycle lẫn presentation. Mỗi hook là một **đơn vị có thể test độc lập**, có thể tái sử dụng trong nhiều component, không conflict nhau vì mỗi invocation giữ slot state riêng trong fiber của component đó.

### 1.5. `useReducer` và `useContext`: pure reducers + dependency injection

`useReducer` đưa Redux pattern — **pure reducer function** — vào component-local state:

```jsx
// Reducer là pure function: (state, action) -> newState
// Dễ test, dễ replay, dễ debug bằng time-travel
function counterReducer(state, action) {
  switch (action.type) {
    case 'increment': return { ...state, count: state.count + 1 };
    case 'decrement': return { ...state, count: state.count - 1 };
    case 'reset':     return { ...state, count: action.payload ?? 0 };
    default:          throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(counterReducer, { count: 0 });

  return (
    <div>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <span>{state.count}</span>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'reset', payload: 0 })}>Reset</button>
    </div>
  );
}
```

**Khi nào dùng `useReducer` thay vì `useState`:** khi có nhiều sub-values liên quan đến nhau, khi next state phụ thuộc vào previous state theo nhiều nhánh điều kiện, hoặc khi cần khả năng test reducer riêng biệt.

`useContext` là **dependency injection thuần khiết**: component không tạo provider, không lookup, không pass-through prop drilling — provider ở trên cây tự inject xuống.

```jsx
// Định nghĩa "interface" (contract)
const ApiContext = createContext(null);

// Hook: component chỉ khai báo "tôi cần một api"
function useApi() {
  const api = useContext(ApiContext);
  if (!api) throw new Error('Missing ApiProvider');
  return api;
}

// Component hoàn toàn decoupled khỏi implementation cụ thể
function UserList() {
  const api = useApi();
  const { data, loading } = useFetch(api.usersEndpoint);
  return loading ? <Spinner/> : <List data={data}/>;
}

// Production: inject real implementation
<ApiContext.Provider value={productionApi}>
  <UserList/>
</ApiContext.Provider>

// Test: inject mock implementation
<ApiContext.Provider value={mockApi}>
  <UserList/>
</ApiContext.Provider>
```

### 1.6. Tại sao React chuyển từ class sang functional: tổng kết

| Vấn đề của class component | Giải pháp của functional + hooks |
|---|---|
| `this` binding tạo lỗi tinh vi | Không có `this`, closure rõ ràng hơn |
| Logic reuse qua HOC/render props tạo "wrapper hell" | Custom hooks: compose không wrap |
| Lifecycle methods phân mảnh logic theo thời gian | `useEffect` gom logic theo ý nghĩa |
| `componentDidMount` nhồi chung nhiều concerns không liên quan | Mỗi `useEffect` = một concern |
| Khó minify (class syntax giới hạn tree-shaking) | Function declarations: tối ưu tốt hơn |
| Test phức tạp vì phải mount component | Custom hook test được thuần như unit test |
| Mental model "instance vòng đời" không khớp React thực | Mental model "state → UI" đơn giản hơn |

---


---

**Trước:** [← Phần 0 — Nền tảng lý thuyết](index.md) | **Tiếp theo:** [Phần 2 — Elixir, Actor Model và GenServer →](elixir-otp.md)
