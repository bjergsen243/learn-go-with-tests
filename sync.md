# Sync

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/sync)**

Chúng ta muốn tạo một bộ đếm (counter) an toàn để sử dụng đồng thời (concurrently).

Chúng ta sẽ bắt đầu với một bộ đếm không an toàn và xác minh hành vi của nó hoạt động trong môi trường đơn luồng (single-threaded).

Sau đó, chúng ta sẽ thử nghiệm tính không an toàn của nó với nhiều goroutine cùng cố gắng sử dụng bộ đếm thông qua một bản kiểm thử, và sau đó khắc phục nó.

## Viết test trước tiên

Chúng ta muốn API của mình cung cấp một phương thức để tăng bộ đếm và sau đó lấy giá trị của nó.

```go
func TestCounter(t *testing.T) {
	t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
		counter := Counter{}
		counter.Inc()
		counter.Inc()
		counter.Inc()

		if counter.Value() != 3 {
			t.Errorf("got %d, want %d", counter.Value(), 3)
		}
	})
}
```

## Thử chạy test

```
./sync_test.go:9:14: undefined: Counter
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Hãy định nghĩa `Counter`.

```go
type Counter struct {
}
```

Thử lại và nó thất bại với lỗi sau:

```
./sync_test.go:14:10: counter.Inc undefined (type Counter has no field or method Inc)
./sync_test.go:18:13: counter.Value undefined (type Counter has no field or method Value)
```

Vì vậy, để cuối cùng làm cho test chạy được, chúng ta có thể định nghĩa các phương thức đó:

```go
func (c *Counter) Inc() {

}

func (c *Counter) Value() int {
	return 0
}
```

Bây giờ nó sẽ chạy và thất bại:

```
=== RUN   TestCounter
=== RUN   TestCounter/incrementing_the_counter_3_times_leaves_it_at_3
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/incrementing_the_counter_3_times_leaves_it_at_3 (0.00s)
    	sync_test.go:27: got 0, want 3
```

## Viết đủ code để test chạy thành công

Điều này hẳn là tầm thường đối với các chuyên gia Go như chúng ta. Chúng ta cần giữ một số trạng thái cho bộ đếm trong kiểu dữ liệu của mình và sau đó tăng nó lên sau mỗi lần gọi `Inc`.

```go
type Counter struct {
	value int
}

func (c *Counter) Inc() {
	c.value++
}

func (c *Counter) Value() int {
	return c.value
}
```

## Refactor

Không có nhiều thứ để tái cấu trúc nhưng vì chúng ta sẽ viết thêm nhiều test cho `Counter`, chúng ta sẽ viết một hàm xác nhận nhỏ `assertCounter` để bản kiểm thử dễ đọc hơn một chút.

```go
t.Run("incrementing the counter 3 times leaves it at 3", func(t *testing.T) {
	counter := Counter{}
	counter.Inc()
	counter.Inc()
	counter.Inc()

	assertCounter(t, counter, 3)
})
```

```go
func assertCounter(t testing.TB, got Counter, want int) {
	t.Helper()
	if got.Value() != want {
		t.Errorf("got %d, want %d", got.Value(), want)
	}
}
```

## Các bước tiếp theo

Việc đó khá dễ dàng nhưng bây giờ chúng ta có yêu cầu rằng nó phải an toàn để sử dụng trong môi trường đồng thời. Chúng ta sẽ cần viết một bản kiểm thử thất bại để thử nghiệm điều này.

## Viết test trước tiên

```go
t.Run("it runs safely concurrently", func(t *testing.T) {
	wantedCount := 1000
	counter := Counter{}

	var wg sync.WaitGroup
	wg.Add(wantedCount)

	for i := 0; i < wantedCount; i++ {
		go func() {
			counter.Inc()
			wg.Done()
		}()
	}
	wg.Wait()

	assertCounter(t, counter, wantedCount)
})
```

Đoạn mã này sẽ lặp qua `wantedCount` của chúng ta và khởi động một goroutine để gọi `counter.Inc()`.

Chúng ta đang sử dụng [`sync.WaitGroup`](https://golang.org/pkg/sync/#WaitGroup), đây là một cách thuận tiện để đồng bộ hóa các quy trình đồng thời.

> Một WaitGroup chờ một tập hợp các goroutine kết thúc. Goroutine chính gọi Add để thiết lập số lượng goroutine cần chờ. Sau đó, mỗi goroutine chạy và gọi Done khi kết thúc. Đồng thời, Wait có thể được sử dụng để chặn cho đến khi tất cả các goroutine kết thúc.

Bằng cách chờ `wg.Wait()` kết thúc trước khi đưa ra các xác nhận, chúng ta có thể chắc chắn rằng tất cả các goroutine của mình đã cố gắng gọi `Inc` cho `Counter`.

## Thử chạy test

```
=== RUN   TestCounter/it_runs_safely_in_a_concurrent_envionment
--- FAIL: TestCounter (0.00s)
    --- FAIL: TestCounter/it_runs_safely_in_a_concurrent_envionment (0.00s)
    	sync_test.go:26: got 939, want 1000
FAIL
```

Bản kiểm thử *có lẽ* sẽ thất bại với một con số khác, nhưng dù sao nó cũng chứng minh rằng nó không hoạt động khi nhiều goroutine cố gắng thay đổi giá trị của bộ đếm cùng một lúc.

## Viết đủ code để test chạy thành công

Một giải pháp đơn giản là thêm một khóa (lock) vào `Counter` của chúng ta, đảm bảo chỉ có một goroutine có thể tăng bộ đếm tại một thời điểm. [`Mutex`](https://golang.org/pkg/sync/#Mutex) của Go cung cấp một loại khóa như vậy:

> Một Mutex là một khóa loại trừ tương hỗ (mutual exclusion lock). Giá trị zero của một Mutex là một mutex chưa được khóa.

```go
type Counter struct {
	mu    sync.Mutex
	value int
}

func (c *Counter) Inc() {
	c.mu.Lock()
	defer c.mu.Unlock()
	c.value++
}
```

Điều này có nghĩa là bất kỳ goroutine nào gọi `Inc` sẽ giành được khóa trên `Counter` nếu họ đến trước. Tất cả các goroutine khác sẽ phải đợi cho đến khi nó được `Unlock` trước khi được quyền truy cập.

Nếu bạn chạy lại test ngay bây giờ, nó sẽ vượt qua vì mỗi goroutine phải đợi đến lượt mình trước khi thực hiện thay đổi.

## Tôi đã thấy các ví dụ khác nơi `sync.Mutex` được nhúng (embedded) vào struct.

Bạn có thể thấy các ví dụ như thế này:

```go
type Counter struct {
	sync.Mutex
	value int
}
```

Có thể lập luận rằng nó làm cho mã nguồn trông thanh thoát hơn một chút.

```go
func (c *Counter) Inc() {
	c.Lock()
	defer c.Unlock()
	c.value++
}
```

Điều này *trông* có vẻ ổn nhưng mặc dù lập trình là một kỷ luật mang tính chủ quan cao, điều này là **tồi tệ và sai lầm**.

Đôi khi mọi người quên rằng việc nhúng (embedding) các kiểu dữ liệu có nghĩa là các phương thức của kiểu dữ liệu đó trở thành *một phần của public interface*; và bạn thường không muốn điều đó. Hãy nhớ rằng chúng ta nên rất cẩn thận với các public API của mình, thời điểm chúng ta công khai một thứ gì đó là thời điểm mã nguồn khác có thể liên kết (couple) với nó. Chúng ta luôn muốn tránh sự liên kết không cần thiết.

Việc để lộ `Lock` và `Unlock` là điều gây bối rối, nhưng tệ hơn nữa là nó có khả năng gây hại rất lớn cho phần mềm của bạn nếu những người sử dụng kiểu dữ liệu của bạn bắt đầu gọi các phương thức này.

![Hình ảnh minh họa một người dùng API này có thể thay đổi trạng thái khóa một cách sai lầm](https://i.imgur.com/SWYNpwm.png)

*Đây có vẻ là một ý tưởng thực sự tồi tệ*

## Sao chép các mutexes

Bản kiểm thử của chúng ta đã vượt qua nhưng mã nguồn của chúng ta vẫn còn một chút nguy hiểm.

Nếu bạn chạy `go vet` cho mã nguồn của mình, bạn sẽ nhận được một lỗi như sau:

```
sync/v2/sync_test.go:16: call of assertCounter copies lock value: v1.Counter contains sync.Mutex
sync/v2/sync_test.go:39: assertCounter passes lock by value: v1.Counter contains sync.Mutex
```

Nhìn vào tài liệu của [`sync.Mutex`](https://golang.org/pkg/sync/#Mutex) sẽ cho chúng ta biết lý do tại sao:

> Một Mutex không được sao chép sau lần sử dụng đầu tiên.

Khi chúng ta truyền `Counter` của mình (theo tham trị - by value) vào `assertCounter`, nó sẽ cố gắng tạo một bản sao của mutex.

Để giải quyết vấn đề này, chúng ta nên truyền vào một con trỏ tới `Counter` thay thế, vì vậy hãy thay đổi signature của `assertCounter`:

```go
func assertCounter(t testing.TB, got *Counter, want int)
```

Các bản kiểm thử của chúng ta sẽ không còn biên dịch được nữa vì chúng ta đang cố gắng truyền vào một `Counter` thay vì `*Counter`. Để giải quyết vấn đề này, tôi thích tạo một constructor để cho người đọc API của bạn thấy rằng tốt hơn là không nên tự khởi tạo kiểu dữ liệu này.

```go
func NewCounter() *Counter {
	return &Counter{}
}
```

Sử dụng hàm này trong các bản kiểm thử khi khởi tạo `Counter`.

## Tổng kết

Chúng ta đã học một vài thứ từ [package sync](https://golang.org/pkg/sync/):

- `Mutex` cho phép chúng ta thêm các khóa (lock) vào dữ liệu của mình.
- `WaitGroup` là một phương tiện để đợi các goroutine hoàn thành công việc.

### Khi nào nên sử dụng khóa thay cho channel và goroutine?

[Chúng ta đã đề cập đến goroutine trong chương lập trình đồng thời đầu tiên](concurrency.md), nó cho phép chúng ta viết mã đồng thời an toàn, vậy tại sao bạn lại sử dụng khóa?
[Trang go wiki có một trang dành riêng cho chủ đề này; Mutex Hay Channel](https://go.dev/wiki/MutexOrChannel)

> Một sai lầm phổ biến của những người mới học Go là sử dụng quá mức channel và goroutine chỉ vì có thể, và/hoặc vì nó thú vị. Đừng ngại sử dụng sync.Mutex nếu điều đó phù hợp nhất với vấn đề của bạn. Go rất thực tế trong việc cho phép bạn sử dụng các công cụ giải quyết vấn đề của mình tốt nhất và không ép buộc bạn vào một phong cách viết mã duy nhất.

Nói một cách khác:

- **Sử dụng channel khi chuyển quyền sở hữu dữ liệu (passing ownership of data)**
- **Sử dụng mutex để quản lý trạng thái (managing state)**

### go vet

Hãy nhớ sử dụng `go vet` trong các tập lệnh build của bạn vì nó có thể cảnh báo bạn về một số lỗi tinh vi trong mã của mình trước khi chúng ảnh hưởng đến người dùng tội nghiệp của bạn.

### Đừng sử dụng nhúng (embedding) chỉ vì nó tiện lợi

- Hãy nghĩ về tác động của việc nhúng đối với public API của bạn.
- Bạn có *thực sự* muốn để lộ các phương thức này và để mọi người liên kết mã của chính họ với chúng không?
- Đối với mutexes, điều này có khả năng gây thảm họa theo những cách rất khó đoán và kỳ lạ, hãy tưởng tượng một đoạn mã xấu xa nào đó mở khóa một mutex khi lẽ ra không nên; điều này sẽ gây ra một số lỗi rất lạ lùng và khó truy vết.
