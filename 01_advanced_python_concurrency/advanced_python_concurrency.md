# Advanced Python Concurrency

## 1. GIL và lựa chọn mô hình

GIL (Global Interpreter Lock) là một khóa toàn cục trong CPython. Khi GIL đang bật, tại một thời điểm chỉ một thread được thực thi bytecode Python trong cùng interpreter. Các thread khác vẫn tồn tại và được lập lịch luân phiên, nhưng không thực sự chạy đồng thời mã Python thuần trên nhiều CPU core. Có thể hình dung GIL như một chiếc micro dùng chung: nhiều thread có thể chờ, nhưng chỉ thread đang giữ micro được nói.

GIL tồn tại chủ yếu để đơn giản hóa việc bảo vệ trạng thái nội bộ của CPython. Mỗi object Python có bộ đếm tham chiếu; khi số tham chiếu về 0, object có thể được giải phóng. Nếu nhiều thread cùng cập nhật bộ đếm hoặc các cấu trúc C API mà không có cơ chế bảo vệ, dữ liệu quản lý bộ nhớ có thể bị hỏng. GIL giúp các thao tác này an toàn hơn, nhưng nó không bảo vệ toàn bộ logic nghiệp vụ của ứng dụng. [1]

Ảnh hưởng của GIL phụ thuộc vào workload. Với CPU-bound, chẳng hạn vòng lặp tính toán bằng Python thuần, các thread phải thay phiên giữ GIL nên thêm thread thường không làm chương trình nhanh hơn; chi phí chuyển đổi còn có thể làm chậm hơn. Với I/O-bound, thread thường nhả GIL khi chờ mạng, tệp hoặc thiết bị, nên thread khác có thể chạy trong thời gian chờ. Một số thư viện native như NumPy cũng có thể nhả GIL ở phần tính toán. [1]

GIL không thay thế `threading.Lock`. Ví dụ “kiểm tra số dư rồi trừ tiền” là một invariant gồm nhiều bước; GIL không bảo đảm một thread khác không chen vào giữa các bước đó. Dữ liệu chia sẻ vẫn cần `Lock`, `Queue`, transaction hoặc primitive đồng bộ phù hợp.


| Mô hình | Workload phù hợp | Cách tiến triển | Ví dụ điển hình | Chi phí và giới hạn | Dấu hiệu nên chọn |
|---|---|---|---|---|---|
| `asyncio` | Nhiều tác vụ I/O có thư viện async; mỗi tác vụ thường dành nhiều thời gian chờ mạng, database hoặc socket | Một event loop chạy nhiều task. Task phải nhường quyền tại `await`; trong lúc một task chờ, loop chạy task khác | Gọi đồng thời nhiều API async, WebSocket, async web server | Blocking code, `time.sleep()` hoặc vòng lặp CPU dài sẽ chặn cả loop. Không tự chia Python code lên nhiều core. Cần thư viện async từ đầu đến cuối | Số kết nối lớn, thời gian chờ I/O chiếm phần lớn latency, cần giới hạn task bằng queue hoặc semaphore |
| `threading` / `ThreadPoolExecutor` | I/O blocking, SDK đồng bộ, thư viện không có API async; một số C extension nhả GIL | Hệ điều hành lập lịch nhiều thread trong cùng process. Thread khác có thể chạy khi thread hiện tại chờ I/O hoặc nhả GIL | Gọi SDK cloud đồng bộ, đọc file, truy vấn DB bằng client blocking | Các thread dùng chung heap nên dễ race và deadlock; cần `Lock`, `Queue` hoặc primitive đồng bộ. CPU-bound bằng Python thuần bị giới hạn bởi GIL | Đã có code blocking cần tích hợp nhanh, số worker vừa phải, dữ liệu chia sẻ cần đơn giản |
| `multiprocessing` / `ProcessPoolExecutor` | CPU-bound bằng Python thuần, mỗi job đủ lớn và tương đối độc lập | Mỗi process có interpreter, GIL và không gian bộ nhớ riêng; nhiều process có thể chạy thật sự trên nhiều core | Xử lý ảnh, nén, phân tích số liệu, tính toán batch | Startup và bộ nhớ cao hơn thread; dữ liệu phải serialize qua IPC; payload lớn, job nhỏ hoặc chia sẻ trạng thái nhiều có thể làm mất lợi ích | CPU sử dụng cao, thời gian tính toán lớn hơn rõ rệt so với chi phí gửi job và nhận kết quả |
| Free-threaded threading | CPU-bound hoặc workload hỗn hợp cần thread chạy Python song song và toàn bộ dependency tương thích | Bản dựng CPython free-threaded cho phép nhiều thread thực thi Python code trên nhiều core; các cấu trúc nội bộ dùng cơ chế đồng bộ khác thay cho GIL | Ứng dụng nhiều thread cần chia sẻ dữ liệu và muốn tránh mô hình process | Không phải bản Python mặc định; extension chưa tương thích có thể bật GIL lại. Race, contention và chi phí đồng bộ vẫn tồn tại; phải benchmark và kiểm tra thread safety | Đã kiểm soát dependency, có benchmark chứng minh lợi ích, và workload phù hợp với shared-memory threading |

**Quy tắc thực hành:** dùng asyncio cho I/O async; dùng thread pool hữu hạn để tích hợp I/O đồng bộ; cân nhắc process pool cho CPU thuần Python. Đây là điểm bắt đầu để benchmark, không phải quy luật tuyệt đối.

Có thể chọn mô hình theo câu hỏi “thời gian chủ yếu nằm ở đâu?”: nếu chương trình chờ I/O, ưu tiên `asyncio` hoặc thread; nếu chương trình dùng CPU để chạy Python thuần, ưu tiên process; nếu thư viện native nhả GIL, thread có thể vẫn đem lại parallelism. `multiprocessing` vượt giới hạn GIL bằng cách dùng các process có interpreter và không gian bộ nhớ riêng, đổi lại phải trả chi phí khởi tạo, serialization và IPC.

## 2. Multiprocessing và threading

- **Process:** một chương trình đang chạy với không gian bộ nhớ và tài nguyên riêng.
- **Thread:** đơn vị thực thi bên trong một process.
- **Thread dùng chung bộ nhớ:** các thread trong cùng process chia sẻ heap, nên truyền dữ liệu thuận tiện hơn.
- **Rủi ro của thread:** nhiều thread có thể cùng sửa một trạng thái và tạo race condition.
- **Process chạy song song:** các process có bộ nhớ riêng nên có thể chạy CPU-bound song song trên nhiều core và vượt giới hạn GIL.
- **Chi phí của process:** phải trả thêm chi phí khởi tạo, bộ nhớ và giao tiếp liên process.

### Threading

Thread phù hợp với I/O blocking, SDK đồng bộ và các thư viện native có thể nhả GIL. Khi một thread chờ mạng, file hoặc database, thread khác có thể được lập lịch. Dùng `ThreadPoolExecutor` để giới hạn số worker thay vì tạo vô hạn thread:

```python
from concurrent.futures import ThreadPoolExecutor

def fetch(url):
    return requests.get(url).status_code  # blocking I/O

with ThreadPoolExecutor(max_workers=8) as pool:
    results = list(pool.map(fetch, urls))
```

Các thread dùng chung bộ nhớ, vì vậy invariant nhiều bước cần được bảo vệ bằng `Lock`, `Queue` hoặc primitive phù hợp. GIL không thay thế lock nghiệp vụ. CPU-bound bằng Python thuần thường không tăng tốc với nhiều thread trong CPython bật GIL.

### Multiprocessing

Mỗi process có interpreter, GIL và không gian địa chỉ riêng. `multiprocessing.Process` dùng để chạy một hàm trong process con; `start()` bắt đầu và `join()` để process cha chờ kết thúc.

```python
from concurrent.futures import ProcessPoolExecutor

def calculate(value):
    return value * value

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=4) as pool:
        results = list(pool.map(calculate, range(8)))
```

- **Khi nên dùng process pool:** mỗi job đủ lớn, tương đối độc lập và chi phí tính toán lớn hơn chi phí serialize/IPC.
- **`map()`:** truyền từng phần tử của input vào hàm và trả kết quả theo thứ tự input.
- **`starmap()`:** phù hợp khi mỗi phần tử chứa nhiều đối số cần truyền vào hàm.
- **Khi không nên dùng process:** job quá nhỏ hoặc payload quá lớn khiến chi phí truyền dữ liệu lớn hơn phần tính toán.

### Giao tiếp và đồng bộ giữa process

- **Bộ nhớ riêng:** process này không tự nhìn thấy thay đổi trên biến global của process khác.
- **Trao đổi dữ liệu:** dùng `multiprocessing.Queue`, `Pipe`, shared memory hoặc một hệ thống lưu trữ bên ngoài.
- **`Queue`:** phù hợp với mô hình producer–consumer.
- **Giữ thứ tự kết quả:** gắn `identifier` vào mỗi kết quả, sau đó sắp xếp lại sau khi nhận đủ dữ liệu.
- **`multiprocessing.Lock`:** chỉ cho phép một process vào critical section tại một thời điểm.
- **`Semaphore(N)`:** cho phép tối đa `N` process cùng sử dụng một tài nguyên.

## 3. asyncio internals

Lập trình bất đồng bộ phù hợp với các tác vụ I/O-bound như gọi API, truy vấn database, đọc/ghi file và xử lý kết nối mạng. Thay vì chờ tuần tự từng tác vụ, chương trình tạm dừng tác vụ đang chờ I/O để xử lý tác vụ khác. Mục tiêu là giảm thời gian chờ; `asyncio` không tự làm phép tính CPU chạy nhanh hơn.

### Coroutine và `async`/`await`

Coroutine là hàm có thể tạm dừng và tiếp tục. Dùng `async def` để định nghĩa coroutine và `await` để chờ kết quả mà không chặn thread hiện tại:

```python
async def fetch_data(client, url):
    response = await client.get(url)
    return response.json()
```

Gọi hàm `async def` chỉ tạo coroutine object; coroutine thực sự chạy khi được `await` hoặc được đóng gói thành `Task`. Khi gặp `await`, coroutine hiện tại nhường quyền cho event loop. `Task` quản lý vòng đời coroutine, còn `Future` đại diện cho một kết quả sẽ hoàn tất sau, thường là cầu nối giữa I/O/callback và coroutine.

### Event loop

Event loop là bộ điều phối các coroutine và task. Quy trình khái quát:

1. Lên lịch task đang sẵn sàng.
2. Chạy task cho tới khi hoàn tất, lỗi hoặc gặp `await` phải chờ.
3. Theo dõi socket, timer và các hoạt động I/O.
4. Khi I/O hoàn tất, đưa task tương ứng trở lại hàng đợi để chạy tiếp.

Ví dụ, task A đang chờ response mạng thì event loop chuyển sang task B. Khi response của A sẵn sàng, loop tiếp tục A. Vì vậy không cần một thread riêng cho mỗi request.

```python
import asyncio

async def fetch_like(name):
    print(name, "start")
    await asyncio.sleep(0.2)  # mô phỏng I/O không chặn
    print(name, "done")

async def main():
    async with asyncio.TaskGroup() as group:
        group.create_task(fetch_like("A"))
        group.create_task(fetch_like("B"))

if __name__ == "__main__":
    asyncio.run(main())
```

Các lỗi cần tránh:

- Quên `await` khi gọi coroutine: chỉ tạo coroutine object mà chưa lấy kết quả.
- Gọi thư viện blocking như `requests` hoặc `time.sleep()` trực tiếp trong `async def`: làm nghẽn event loop.
- Chạy CPU-bound trực tiếp trên event loop: các task khác không được chạy cho tới khi phép tính kết thúc.
- Không đóng event loop: dùng `asyncio.run(main())` để tự quản lý vòng đời loop.

Nếu bắt buộc dùng hàm blocking trong chương trình async, chuyển nó sang thread:

```python
result = await asyncio.to_thread(blocking_function, arg)
```

`to_thread()` giúp tránh chặn event loop cho I/O blocking, nhưng không tạo CPU parallelism cho Python thuần trên CPython đang bật GIL. [3][4]

## 4. Backpressure

Backpressure là cơ chế báo ngược rằng bên xử lý đang bận, để bên tạo việc giảm tốc độ nhận vào, giới hạn việc đang chờ hoặc từ chối việc mới. Nếu producer tạo 100 job/giây nhưng consumer chỉ xử lý được 20 job/giây, backlog sẽ tăng 80 job/giây nếu không có giới hạn. Queue dài chỉ làm tăng RAM và thời gian chờ, không tạo thêm năng lực xử lý.

### Ví dụ không có backpressure

```python
async def producer():
    tasks = []
    for job in range(1000):
        tasks.append(asyncio.create_task(process(job)))
    await asyncio.gather(*tasks)
```

Đoạn code tạo 1.000 task gần như ngay lập tức. Với input rất lớn, các task đang chờ sẽ chiếm bộ nhớ và làm hệ thống khó kiểm soát.

### Queue hữu hạn và worker cố định

```python
async def worker(queue):
    while True:
        job = await queue.get()
        try:
            await process(job)
        finally:
            queue.task_done()

async def producer(queue, total_jobs):
    for job in range(total_jobs):
        await queue.put(job)

async def main():
    queue = asyncio.Queue(maxsize=5)
    workers = [asyncio.create_task(worker(queue)) for _ in range(2)]
    await producer(queue, 20)
    await queue.join()
    for task in workers:
        task.cancel()
```

Queue chỉ chứa tối đa 5 job đang chờ và có 2 worker xử lý. Khi queue đầy, `await queue.put(job)` phải chờ; producer tự động chậm lại theo tốc độ consumer. Đây là backpressure.

### Các cơ chế thường dùng

| Cơ chế | Giới hạn |
|---|---|
| `asyncio.Queue(maxsize=N)` | Số job đang chờ |
| Worker cố định | Số job đang xử lý |
| `Semaphore(N)` | Số thao tác đồng thời |
| Rate limit | Số request trong một khoảng thời gian |
| Timeout/deadline | Thời gian chờ tối đa |
| Reject hoặc load shedding | Từ chối job mới khi quá tải |

Semaphore chỉ giới hạn số thao tác đang giữ permit. Nếu tạo hàng triệu task cùng chờ semaphore, bộ nhớ vẫn có thể tăng; vì vậy nên kết hợp semaphore với queue hữu hạn hoặc worker cố định.

### Dấu hiệu thiếu backpressure

- Queue tăng liên tục.
- RAM tăng theo thời gian.
- Request có latency ngày càng cao.
- Có quá nhiều task hoặc thread đang chờ.
- Downstream bị quá tải.
- Timeout và lỗi retry tăng mạnh.

Backpressure không làm hệ thống xử lý nhanh hơn. Nó giúp tốc độ nhận việc phù hợp với năng lực xử lý, giữ backlog trong giới hạn và ngăn hệ thống tự quá tải. [8][9]

## 5. Cancellation

Trong Python với thư viện `asyncio`, cancellation cho phép dừng một `Task` trước khi nó hoàn thành. Đây là yêu cầu hợp tác: Python không cắt ngang coroutine ngay tại thời điểm gọi `.cancel()`, mà đưa `asyncio.CancelledError` vào coroutine tại điểm `await` tiếp theo. [1](https://www.c-sharpcorner.com/article/python-asyncio-cancellation-how-to-stop-tasks-without-leaving-resources-open)

### 1. Hủy một Task bằng `task.cancel()`

- Gọi `.cancel()` trên đối tượng `asyncio.Task` để gửi tín hiệu hủy.
- Coroutine cần đi tới một suspension point như `await` để nhận `CancelledError`.
- Sau khi nhận tín hiệu, task nên đóng connection, file, lock hoặc tài nguyên khác trong `finally`.
- Sau khi gọi `.cancel()`, nên `await task` để chờ task hoàn tất cleanup.

```python
import asyncio

async def worker():
    try:
        while True:
            print("Worker đang chạy...")
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print("Worker đã nhận được tín hiệu hủy!")
        raise
    finally:
        print("Đang dọn dẹp tài nguyên...")

async def main():
    task = asyncio.create_task(worker())
    await asyncio.sleep(3)

    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("Task đã dừng hoàn toàn.")

asyncio.run(main())
```

Nếu coroutine đang chạy vòng lặp CPU dài hoặc hàm blocking không có `await`, nó có thể không phản hồi cancellation kịp thời.

### 2. Công cụ hỗ trợ cancellation

`asyncio.wait_for()` bọc một coroutine hoặc task với thời gian giới hạn. Khi quá timeout, công việc được yêu cầu hủy và bên gọi nhận `TimeoutError`:

```python
try:
    result = await asyncio.wait_for(slow_operation(), timeout=3)
except TimeoutError:
    print("Tác vụ quá thời gian cho phép")
```

`asyncio.TaskGroup` (Python 3.11+) quản lý vòng đời một nhóm task. Khi một task trong group phát sinh lỗi thông thường, group hủy các task còn lại, chờ cleanup rồi báo lỗi theo nhóm. Một task bị hủy có chủ đích thường được coi là kết thúc do cancellation; không nên mặc định rằng mọi cancellation đơn lẻ sẽ hủy toàn bộ group. [2](https://www.pythontutorial.net/python-concurrency/python-asyncio-wait_for/) [3](https://www.c-sharpcorner.com/article/python-asyncio-cancellation-how-to-stop-tasks-without-leaving-resources-open)

### 3. Những lưu ý quan trọng

- **Không nuốt lỗi quá mức:** nếu bắt `CancelledError` để log hoặc cleanup, thường phải `raise` lại để trạng thái hủy được truyền tiếp.
- **Không gọi `.cancel()` rồi bỏ mặc:** luôn `await task` trong `try...except` để đảm bảo `finally` chạy xong.
- **Cleanup phải có cấu trúc:** dùng `finally` hoặc async context manager cho connection, file và lock.
- **Timeout không phải hard real-time:** cleanup có thể khiến thời gian kết thúc thực tế dài hơn deadline.
- **Không hủy được thread tùy ý:** hủy coroutine đang chờ `asyncio.to_thread()` không tự dừng hàm đã bắt đầu trong thread; thread cần cơ chế dừng hợp tác riêng.

## 6. Quiz: Advanced Python Concurrency

### Câu hỏi

**Câu 1.** GIL trong CPython giới hạn điều gì khi GIL đang bật?

A. Số lượng process được tạo  
B. Số thread cùng thực thi Python bytecode trong một interpreter  
C. Số coroutine trong event loop  
D. Số file được mở

**Câu 2.** Mô hình nào thường phù hợp nhất với tác vụ CPU-bound viết bằng Python thuần?

A. `multiprocessing`  
B. `asyncio`  
C. Tạo thêm nhiều coroutine  
D. Chỉ dùng một `Queue`

**Câu 3.** `threading` thường phù hợp với loại công việc nào?

A. Tính toán CPU nặng bằng Python thuần  
B. I/O blocking như gọi API hoặc đọc file  
C. Chỉ xử lý dữ liệu trong process khác  
D. Thay thế hoàn toàn `asyncio`

**Câu 4.** Các thread trong cùng một process có đặc điểm nào?

A. Luôn có bộ nhớ hoàn toàn riêng  
B. Không thể trao đổi dữ liệu  
C. Dùng chung bộ nhớ của process  
D. Mỗi thread có một GIL riêng

**Câu 5.** Gọi một hàm được khai báo bằng `async def` sẽ trả về gì trước khi được `await`?

A. Một process mới  
B. Một coroutine object  
C. Kết quả cuối cùng ngay lập tức  
D. Một thread mới

**Câu 6.** Vai trò chính của event loop là gì?

A. Tạo một process cho mỗi request  
B. Điều phối các task và tiếp tục task khi I/O sẵn sàng  
C. Làm mọi phép tính CPU chạy song song  
D. Thay thế database

**Câu 7.** Điều gì xảy ra nếu gọi `time.sleep()` trực tiếp trong một coroutine đang chạy trên event loop?

A. Event loop có thể bị chặn  
B. Coroutine tự chuyển sang process khác  
C. Chương trình tự tạo thêm thread  
D. Không có ảnh hưởng nào

**Câu 8.** Backpressure dùng để làm gì?

A. Tạo vô hạn task để xử lý nhanh hơn  
B. Điều chỉnh tốc độ nhận việc theo khả năng xử lý  
C. Xóa tất cả job đang chờ  
D. Tăng số CPU core của máy

**Câu 9.** `asyncio.Queue(maxsize=5)` có ý nghĩa gì?

A. Chỉ tạo tối đa 5 process  
B. Queue chứa tối đa 5 item đang chờ trước khi `put()` phải đợi  
C. Mỗi task chỉ chạy 5 giây  
D. Event loop chỉ chạy tối đa 5 lần

**Câu 10.** Sau khi gọi `task.cancel()`, cách xử lý phù hợp là gì?

A. Bỏ mặc task và kết thúc chương trình ngay  
B. Xóa biến `task`  
C. `await task` và để coroutine cleanup tài nguyên trong `finally`  
D. Gọi `task.cancel()` liên tục

### Đáp án

1. **B** — GIL chỉ cho phép một thread thực thi Python bytecode tại một thời điểm trong cùng interpreter khi GIL đang bật.
2. **A** — Nhiều process có interpreter và GIL riêng nên có thể chạy trên nhiều CPU core.
3. **B** — Thread phù hợp khi phần lớn thời gian được dùng để chờ network, file hoặc I/O blocking khác.
4. **C** — Các thread dùng chung bộ nhớ nên trao đổi dữ liệu thuận tiện nhưng có nguy cơ race condition.
5. **B** — Gọi hàm `async def` tạo coroutine object; coroutine chạy khi được `await` hoặc tạo thành Task.
6. **B** — Event loop điều phối task, theo dõi I/O và chạy lại task khi công việc đang chờ đã sẵn sàng.
7. **A** — `time.sleep()` là hàm blocking nên chặn thread đang chạy event loop và làm các task khác phải chờ.
8. **B** — Backpressure giữ lượng việc chờ trong giới hạn và làm producer chậm lại khi consumer đang quá tải.
9. **B** — Khi queue đã có 5 item, `await queue.put(...)` phải chờ đến khi có chỗ trống.
10. **C** — Cancellation là hợp tác; cần chờ task hoàn tất để code cleanup trong `finally` được chạy.
