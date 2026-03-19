# Select

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/select)**

Bạn được yêu cầu tạo một hàm gọi là `WebsiteRacer` nhận vào hai URL và cho chúng "đua" với nhau bằng cách gọi HTTP GET đến cả hai, sau đó trả về URL nào phản hồi trước. Nếu không có URL nào phản hồi trong vòng 10 giây thì nó nên trả về một `error`.

Để làm điều này, chúng ta sẽ sử dụng:

- `net/http` để thực hiện các cuộc gọi HTTP.
- `net/http/httptest` để giúp chúng ta kiểm thử.
- goroutines.
- `select` để đồng bộ hóa các quy trình.

## Viết test trước tiên

Hãy bắt đầu với một cách tiếp cận đơn giản nhất có thể.

```go
func TestRacer(t *testing.T) {
	slowURL := "http://www.facebook.com"
	fastURL := "http://www.quii.dev"

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

Chúng ta biết điều này chưa hoàn hảo và còn nhiều vấn đề, nhưng đó là một điểm khởi đầu. Điều quan trọng là đừng quá sa đà vào việc làm mọi thứ hoàn hảo ngay từ lần đầu tiên.

## Thử chạy test

`./racer_test.go:14:9: undefined: Racer`

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func Racer(a, b string) (winner string) {
	return
}
```

`racer_test.go:25: got '', want 'http://www.quii.dev'`

## Viết đủ code để test chạy thành công

```go
func Racer(a, b string) (winner string) {
	startA := time.Now()
	http.Get(a)
	aDuration := time.Since(startA)

	startB := time.Now()
	http.Get(b)
	bDuration := time.Since(startB)

	if aDuration < bDuration {
		return a
	}

	return b
}
```

Đối với mỗi URL:

1. Chúng ta sử dụng `time.Now()` để ghi lại thời điểm ngay trước khi thử truy cập `URL`.
1. Sau đó, chúng ta sử dụng [`http.Get`](https://golang.org/pkg/net/http/#Client.Get) để thực hiện yêu cầu HTTP `GET` đối với `URL`. Hàm này trả về một [`http.Response`](https://golang.org/pkg/net/http/#Response) và một `error`, nhưng hiện tại chúng ta chưa quan tâm đến các giá trị này.
1. `time.Since` nhận thời gian bắt đầu và trả về một `time.Duration` cho sự chênh lệch thời gian.

Sau khi thực hiện điều này, chúng ta chỉ đơn giản là so sánh các khoảng thời gian để xem cái nào nhanh nhất.

### Vấn đề

Cách này có thể giúp test của bạn vượt qua hoặc không. Vấn đề là chúng ta đang kết nối đến các trang web thực tế để kiểm thử logic của chính mình.

Kiểm thử mã nguồn sử dụng HTTP là việc rất phổ biến nên Go có các công cụ trong thư viện chuẩn để giúp bạn thực hiện điều đó.

Trong các chương về mocking và dependency injection, chúng ta đã đề cập rằng lý tưởng nhất là chúng ta không muốn dựa vào các dịch vụ bên ngoài để kiểm thử mã của mình vì chúng có thể:

- Chậm
- Không ổn định (Flaky)
- Không thể kiểm thử được các trường hợp đặc biệt (edge cases)

Trong thư viện chuẩn, có một package gọi là [`net/http/httptest`](https://golang.org/pkg/net/http/httptest/) cho phép người dùng dễ dàng tạo một máy chủ HTTP giả lập (mock HTTP server).

Hãy thay đổi các bản kiểm thử của chúng ta để sử dụng bản giả lập sao cho chúng ta có các máy chủ đáng tin cậy để kiểm thử và có thể kiểm soát được.

```go
func TestRacer(t *testing.T) {

	slowServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(20 * time.Millisecond)
		w.WriteHeader(http.StatusOK)
	}))

	fastServer := httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}

	slowServer.Close()
	fastServer.Close()
}
```

Cú pháp trông có vẻ hơi rắc rối nhưng hãy bình tĩnh quan sát.

`httptest.NewServer` nhận một `http.HandlerFunc`, cái mà chúng ta đang gửi vào thông qua một *hàm ẩn danh*.

`http.HandlerFunc` là một kiểu dữ liệu có dạng: `type HandlerFunc func(ResponseWriter, *Request)`.

Tất cả những gì nó thực sự nói là nó cần một hàm nhận vào một `ResponseWriter` và một `Request`, điều này không quá ngạc nhiên đối với một máy chủ HTTP.

Hóa ra không có phép màu gì ở đây cả, **đây cũng chính là cách bạn viết một máy chủ HTTP *thật sự* trong Go**. Điểm khác biệt duy nhất là chúng ta đang bọc nó trong một `httptest.NewServer`, giúp việc sử dụng trong kiểm thử trở nên dễ dàng hơn, vì nó sẽ tìm một cổng mở để lắng nghe và sau đó bạn có thể đóng nó lại khi hoàn thành bản kiểm thử của mình.

Bên trong hai máy chủ của chúng ta, chúng ta làm cho cái chậm có một khoảng nghỉ ngắn `time.Sleep` khi nhận được yêu cầu để làm cho nó chậm hơn cái còn lại. Cả hai máy chủ sau đó đều gửi phản hồi `OK` bằng `w.WriteHeader(http.StatusOK)` về cho người gọi.

Nếu bạn chạy lại test, nó chắc chắn sẽ vượt qua ngay bây giờ và sẽ nhanh hơn. Hãy thử nghịch các giá trị sleep này để cố tình làm hỏng bản kiểm thử.

## Refactor

Chúng ta thấy có sự lặp lại trong cả mã triển khai và mã kiểm thử.

```go
func Racer(a, b string) (winner string) {
	aDuration := measureResponseTime(a)
	bDuration := measureResponseTime(b)

	if aDuration < bDuration {
		return a
	}

	return b
}

func measureResponseTime(url string) time.Duration {
	start := time.Now()
	http.Get(url)
	return time.Since(start)
}
```

Việc áp dụng nguyên tắc DRY (Don't Repeat Yourself) này làm cho mã `Racer` của chúng ta dễ đọc hơn nhiều.

```go
func TestRacer(t *testing.T) {

	slowServer := makeDelayedServer(20 * time.Millisecond)
	fastServer := makeDelayedServer(0 * time.Millisecond)

	defer slowServer.Close()
	defer fastServer.Close()

	slowURL := slowServer.URL
	fastURL := fastServer.URL

	want := fastURL
	got := Racer(slowURL, fastURL)

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}

func makeDelayedServer(delay time.Duration) *httptest.Server {
	return httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		time.Sleep(delay)
		w.WriteHeader(http.StatusOK)
	}))
}
```

Chúng ta đã cấu trúc lại việc tạo máy chủ giả lập thành một hàm gọi là `makeDelayedServer` để tách những đoạn mã không thú vị ra khỏi bản kiểm thử và giảm sự lặp lại.

### `defer`

Bằng cách đặt từ khóa `defer` trước một lời gọi hàm, hàm đó sẽ được gọi *ở cuối hàm chứa nó*.

Đôi khi bạn sẽ cần giải phóng tài nguyên, chẳng hạn như đóng một file hoặc trong trường hợp của chúng ta là đóng một máy chủ để nó không tiếp tục chiếm dụng cổng lắng nghe.

Bạn muốn việc này thực thi ở cuối hàm, nhưng hãy giữ chỉ thị này ở gần nơi bạn tạo máy chủ để những người đọc mã sau này dễ dàng theo dõi.

Việc tái cấu trúc này là một sự cải tiến và là một giải pháp hợp lý dựa trên các tính năng của Go đã học được cho đến nay, nhưng chúng ta có thể làm cho giải pháp trở nên đơn giản hơn nữa.

### Đồng bộ hóa các quy trình (Synchronising processes)

- Tại sao chúng ta lại đi kiểm tra tốc độ của các website lần lượt từng cái một trong khi Go rất mạnh về lập trình đồng thời (concurrency)? Chúng ta nên kiểm tra cả hai cùng một lúc.
- Chúng ta không thực sự quan tâm đến *thời gian phản hồi chính xác* của các yêu cầu, chúng ta chỉ muốn biết cái nào phản hồi trước.

Để làm điều này, chúng ta sẽ giới thiệu một cấu trúc mới gọi là `select`, giúp chúng ta đồng bộ hóa các quy trình một cách dễ dàng và rõ ràng.

```go
func Racer(a, b string) (winner string) {
	select {
	case <-ping(a):
		return a
	case <-ping(b):
		return b
	}
}

func ping(url string) chan struct{} {
	ch := make(chan struct{})
	go func() {
		http.Get(url)
		close(ch)
	}()
	return ch
}
```

#### `ping`

Chúng ta đã định nghĩa một hàm `ping` để tạo một `chan struct{}` và trả về nó.

Trong trường hợp này, chúng ta không *quan tâm* kiểu dữ liệu nào được gửi vào channel, *chúng ta chỉ muốn phát tín hiệu rằng mình đã xong* và việc đóng channel hoạt động một cách hoàn hảo cho mục đích đó!

Tại sao lại là `struct{}` mà không phải một kiểu khác như `bool`? Có thể nói `chan struct{}` là kiểu dữ liệu nhỏ nhất xét về mặt bộ nhớ, vì vậy chúng ta không tốn bộ nhớ so với `bool`. Vì chúng ta chỉ đóng channel chứ không gửi bất kỳ dữ liệu gì qua nó, tại sao lại phải phân bổ bộ nhớ làm gì?

Bên trong cùng một hàm, chúng ta bắt đầu một goroutine để gửi một tín hiệu vào channel đó sau khi đã hoàn thành việc gọi `http.Get(url)`.

##### Luôn luôn dùng `make` để tạo channels

Lưu ý cách chúng ta sử dụng `make` khi tạo channel; thay vì chỉ khai báo `var ch chan struct{}`. Khi bạn sử dụng `var`, biến sẽ được khởi tạo với giá trị "zero" của kiểu đó. Đối với `string` là `""`, `int` là 0, v.v.

Đối với channels, giá trị zero là `nil` và nếu bạn cố gửi dữ liệu vào nó bằng `<-`, nó sẽ chặn (block) mãi mãi vì bạn không thể gửi vào một channel `nil`.

[Bạn có thể thấy điều này đang hoạt động trong The Go Playground](https://play.golang.org/p/IIbeAox5jKA)

#### `select`

Bạn còn nhớ từ chương lập trình đồng thời rằng bạn có thể đợi các giá trị được gửi vào channel bằng `myVar := <-ch`. Đây là một lời gọi *chặn* (blocking), vì bạn đang đợi một giá trị.

`select` cho phép bạn đợi trên *nhiều* channels cùng lúc. Channel nào gửi giá trị về trước sẽ "thắng" và đoạn mã dưới `case` đó sẽ được thực thi.

Chúng ta sử dụng `ping` trong `select` của mình để tạo hai channels, mỗi cái cho một `URL`. Cái nào ghi vào channel của nó trước sẽ khiến mã của nó được thực thi trong `select`, dẫn đến việc `URL` đó được trả về (và trở thành người thắng cuộc).

Sau các thay đổi này, ý định đằng sau mã nguồn của chúng ta trở nên rất rõ ràng và việc triển khai thực tế thậm chí còn đơn giản hơn.

### Timeouts (Hết thời gian chờ)

Yêu cầu cuối cùng của chúng ta là trả về một lỗi nếu `Racer` tốn nhiều hơn 10 giây.

## Viết test trước tiên

```go
func TestRacer(t *testing.T) {
	t.Run("compares speeds of servers, returning the url of the fastest one", func(t *testing.T) {
		slowServer := makeDelayedServer(20 * time.Millisecond)
		fastServer := makeDelayedServer(0 * time.Millisecond)

		defer slowServer.Close()
		defer fastServer.Close()

		slowURL := slowServer.URL
		fastURL := fastServer.URL

		want := fastURL
		got, _ := Racer(slowURL, fastURL)

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("returns an error if a server doesn't respond within 10s", func(t *testing.T) {
		serverA := makeDelayedServer(11 * time.Second)
		serverB := makeDelayedServer(12 * time.Second)

		defer serverA.Close()
		defer serverB.Close()

		_, err := Racer(serverA.URL, serverB.URL)

		if err == nil {
			t.Error("expected an error but didn't get one")
		}
	})
}
```

Chúng ta đã làm cho các máy chủ kiểm thử mất hơn 10 giây để phản hồi nhằm thử nghiệm kịch bản này và chúng ta kỳ vọng `Racer` sẽ trả về hai giá trị, URL chiến thắng (mà chúng ta bỏ qua trong test này bằng `_`) và một `error`.

Lưu ý rằng chúng ta cũng đã xử lý việc trả về lỗi trong bản kiểm thử ban đầu của mình, hiện tại chúng ta sử dụng `_` để đảm bảo các bản kiểm thử vẫn chạy được.

## Thử chạy test

`./racer_test.go:37:10: assignment mismatch: 2 variables but Racer returns 1 value`

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func Racer(a, b string) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	}
}
```

Thay đổi signature của `Racer` để trả về người chiến thắng và một `error`. Trả về `nil` cho các trường hợp thành công.

Compiler sẽ phàn nàn về *test đầu tiên* của bạn vì nó chỉ mong đợi một giá trị, vì vậy hãy đổi dòng đó thành `got, err := Racer(slowURL, fastURL)`, lưu ý rằng chúng ta nên kiểm tra xem mình *không* nhận được lỗi trong kịch bản thành công.

Nếu bạn chạy nó bây giờ, sau 11 giây nó sẽ thất bại.

```
--- FAIL: TestRacer (12.00s)
    --- FAIL: TestRacer/returns_an_error_if_a_server_doesn't_respond_within_10s (12.00s)
        racer_test.go:40: expected an error but didn't get one
```

## Viết đủ code để test chạy thành công

```go
func Racer(a, b string) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(10 * time.Second):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

`time.After` là một hàm rất tiện dụng khi sử dụng `select`. Mặc dù nó không xảy ra trong trường hợp của chúng ta, nhưng bạn có khả năng viết mã nguồn bị chặn mãi mãi nếu các channels bạn đang lắng nghe không bao giờ trả về giá trị. `time.After` trả về một `chan` (giống như `ping`) và sẽ gửi một tín hiệu qua đó sau một khoảng thời gian bạn chỉ định.

Đối với chúng ta, điều này thật hoàn hảo; nếu `a` hoặc `b` phản hồi kịp, họ sẽ thắng, nhưng nếu đạt đến 10 giây thì `time.After` sẽ gửi một tín hiệu và chúng ta sẽ trả về một lỗi `error`.

### Các bản kiểm thử chậm chạp

Vấn đề chúng ta gặp phải là bản kiểm thử này mất 10 giây để chạy. Đối với một đoạn logic đơn giản như vậy, điều này có vẻ không ổn lắm.

Những gì chúng ta có thể làm là làm cho thời gian chờ (timeout) có thể cấu hình được. Vì vậy, trong bản kiểm thử của mình, chúng ta có thể đặt một thời gian chờ rất ngắn và sau đó khi mã nguồn được sử dụng trong thế giới thực, nó có thể được đặt là 10 giây.

```go
func Racer(a, b string, timeout time.Duration) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

Các test của chúng ta bây giờ sẽ không biên dịch được vì chúng ta không cung cấp timeout.

Trước khi vội vàng thêm giá trị mặc định này vào cả hai bản kiểm thử, hãy *lắng nghe chúng*.

- Chúng ta có quan tâm đến timeout trong bản kiểm thử "thành công" không?
- Các yêu cầu đã nói rõ ràng về timeout.

Với kiến thức này, hãy thực hiện một chút tái cấu trúc để thân thiện với cả các bản kiểm thử và những người sử dụng mã nguồn của chúng ta.

```go
var tenSecondTimeout = 10 * time.Second

func Racer(a, b string) (winner string, error error) {
	return ConfigurableRacer(a, b, tenSecondTimeout)
}

func ConfigurableRacer(a, b string, timeout time.Duration) (winner string, error error) {
	select {
	case <-ping(a):
		return a, nil
	case <-ping(b):
		return b, nil
	case <-time.After(timeout):
		return "", fmt.Errorf("timed out waiting for %s and %s", a, b)
	}
}
```

Người dùng của chúng ta và bản kiểm thử đầu tiên có thể sử dụng `Racer` (hàm này sử dụng `ConfigurableRacer` bên dưới) và bản kiểm thử trường hợp lỗi của chúng ta có thể sử dụng `ConfigurableRacer`.

```go
func TestRacer(t *testing.T) {

	t.Run("compares speeds of servers, returning the url of the fastest one", func(t *testing.T) {
		slowServer := makeDelayedServer(20 * time.Millisecond)
		fastServer := makeDelayedServer(0 * time.Millisecond)

		defer slowServer.Close()
		defer fastServer.Close()

		slowURL := slowServer.URL
		fastURL := fastServer.URL

		want := fastURL
		got, err := Racer(slowURL, fastURL)

		if err != nil {
			t.Fatalf("did not expect an error but got one %v", err)
		}

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("returns an error if a server doesn't respond within the specified time", func(t *testing.T) {
		server := makeDelayedServer(25 * time.Millisecond)

		defer server.Close()

		_, err := ConfigurableRacer(server.URL, server.URL, 20*time.Millisecond)

		if err == nil {
			t.Error("expected an error but didn't get one")
		}
	})
}
```

Tôi đã thêm một lần kiểm tra cuối cùng vào bản kiểm thử đầu tiên để xác minh rằng chúng ta không nhận được lỗi `error`.

## Tổng kết

### `select`

- Giúp bạn đợi trên nhiều channels cùng lúc.
- Đôi khi bạn sẽ muốn bao gồm `time.After` trong một trong các `cases` của mình để ngăn hệ thống bị chặn mãi mãi.

### `httptest`

- Một cách thuận tiện để tạo các máy chủ kiểm thử nhằm có được các bản kiểm thử đáng tin cậy và có thể kiểm soát được.
- Sử dụng cùng các interface như các máy chủ `net/http` "thật", điều này giúp nhất quán và giảm bớt những thứ bạn cần phải học.
