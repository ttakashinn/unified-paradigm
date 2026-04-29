# Phần 8 — Phụ lục: Đọc sâu hơn

### Pragmatic / Engineering (đọc trước)

- Gary Bernhardt — *"Boundaries"* (talk SCNA 2012) và *"Functional Core, Imperative Shell"* (Destroy All Software screencast). Đây là cách giải thích ngắn nhất có thể cho toàn bộ tài liệu này.
- Dan Abramov — *"Algebraic Effects for the Rest of Us"* (overreacted.io, 2019). Đọc ngay sau Bernhardt.
- Martin Fowler — *"Inversion of Control"* (martinfowler.com/bliki/InversionOfControl.html).
- Saša Jurić — *"Elixir in Action"* (Manning, 2nd ed). Chương 5–10 về OTP là giáo trình GenServer tốt nhất hiện tại.
- Joe Armstrong — *"Programming Erlang"* (Pragmatic Bookshelf, 2nd ed). Chương về error handling và supervision.
- Robert Nystrom — *"Game Programming Patterns"* (gameprogrammingpatterns.com). Game Loop, Component, Event Queue patterns.

### Foundational / Academic (đọc sau khi đã có intuition)

- Gordon Plotkin & John Power — *"Notions of Computation Determine Monads"* (FoSSaCS 2002).
- Gordon Plotkin & Matija Pretnar — *"Handlers of Algebraic Effects"* (ESOP 2009). Bài báo định nghĩa thuật ngữ effect handlers.
- Daan Leijen — *"Type Directed Compilation of Row-Typed Algebraic Effects"* (POPL 2017). Koka language.
- Carl Hewitt, Peter Bishop, Richard Steiger — *"A Universal Modular Actor Formalism"* (IJCAI 1973).
- Gul Agha — *"Actors: A Model of Concurrent Computation in Distributed Systems"* (MIT Press, 1986).
- Conal Elliott & Paul Hudak — *"Functional Reactive Animation"* (ICFP 1997). Bài báo gốc FRP.
- Conal Elliott — *"Denotational Design with Type Class Morphisms"* (2009). Triết lý thiết kế từ ngữ nghĩa.
- Eugenio Moggi — *"Notions of Computation and Monads"* (Information and Computation, 1991).
- Sylvan Clebsch et al. — *"Deny Capabilities for Safe, Fast Actors"* (AGERE 2015). Pony language.

### Modern Syntheses

- Martin Kleppmann — *"Designing Data-Intensive Applications"*. Chương 11 về stream processing = Kafka qua lens algebraic-effects-at-scale.
- Nathaniel J. Smith — *"Notes on structured concurrency"* (vorpus.org, 2018). Tại sao `async/await` cần structured scoping giống OTP supervision.
- Aleksey Kladov (matklad) — blog về type systems cho concurrency và async (matklad.github.io).

---

## Kết luận

Ba khái niệm ban đầu — React hooks, Elixir GenServer/OTP, và Inversion of Control — không phải ba chủ đề rời rạc. Chúng là **ba cú chiếu của cùng một hình học bốn chiều**:

> *Code do người dùng viết là mô tả khai báo của what (algebraic effects); trên các thực thể trạng thái cô lập (actor model); với runtime/framework/scheduler nắm trọn quyền điều phối (Hollywood Principle); được thiết kế xuất phát từ ngữ nghĩa chứ không phải từ chi tiết thực thi (denotational design).*

Hình học bốn chiều này xuất hiện ở **khắp nơi trong ngành**: từ Unity MonoBehaviour đến Kubernetes controller, từ AWS Lambda đến Temporal workflow, từ Elm TEA đến Bevy ECS, từ Haskell IO monad đến OCaml 5 effect handlers, từ Unix pipe đến Kafka consumer group. Mỗi lần là cùng một thuật toán ở một quy mô và ngôn ngữ khác nhau.

**React hooks** dạy bạn nghĩ về UI như một hàm thuần của state, dùng custom hook để compose stateful logic, để framework điều phối re-render.

**Elixir GenServer** dạy bạn cùng tư duy đó ở quy mô concurrent/distributed: pure transition trong process bị cô lập, supervisor lo failure.

**Inversion of Control** là cái khung triết học chung từ thập niên 1980, lý giải vì sao cả hai đều đáng tin cậy và tái sử dụng được.

**Phoenix LiveView** là nơi ba khái niệm hợp nhất thành một primitive — bằng chứng rằng đây không phải ba thứ song hành mà là một thứ được nhìn từ ba góc.

**Landscape rộng hơn** ([Phần 7](landscape.md)) cho thấy đây không phải xu hướng của một cộng đồng: từ Ericsson thập niên 1980 đến Apple Swift Concurrency 2021, từ Unix pipe đến Kubernetes 2014, cùng một triết lý được tái phát hiện độc lập lần lượt vì nó là câu trả lời đúng cho bài toán "làm thế nào để viết phần mềm đáng tin cậy trong thế giới concurrent và có lỗi."

Senior developer nắm vững hình học này có một lợi thế quyết định: **họ không học các framework — họ đọc các framework như những hiện thân khác nhau của cùng một bộ nguyên tắc bất biến**, và do đó học mọi framework mới nhanh hơn, thiết kế hệ thống mới đúng hơn, debug các hệ thống cũ sâu hơn người chỉ biết một mảnh.

---

*Tài liệu này là living document. Các phần có thể mở rộng tiếp: distributed Elixir (horde, libcluster), Temporal workflow deep-dive, OCaml 5 effect handlers thực chiến, và Bevy ECS như case study game engine.*

---

**Trước:** [← Phần 7 — Landscape toàn cục](landscape.md) | [↑ Quay về đầu](index.md)
