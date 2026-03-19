# Context

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/context)**

Phần mềm thường khởi chạy các quy trình chạy lâu, tốn nhiều tài nguyên (thường là trong các goroutine). Nếu hành động gây ra điều này bị hủy bỏ hoặc thất bại vì lý do nào đó, bạn cần dừng các quy trình này một cách nhất quán trong toàn bộ ứng dụng của mình.

Nếu bạn không quản lý điều này, ứng dụng Go nhanh nhẹn mà bạn rất tự hào có thể bắt đầu gặp các vấn đề về hiệu suất rất khó gỡ lỗi.

Trong chương này, chúng ta sẽ sử dụng package `context` để giúp quản lý các quy trình chạy lâu.

Chúng ta sẽ bắt đầu với một ví dụ kinh điển về một web server, khi được truy cập sẽ khởi chạy một quy trình tiềm ẩn khả năng chạy lâu để lấy một số dữ liệu và trả về trong phản hồi (response).

Chúng ta sẽ thực hiện kịch bản nơi người dùng hủy yêu cầu (request) trước khi dữ liệu được lấy ra, và chúng ta sẽ đảm bảo quy trình đó được thông báo để dừng lại.

Tôi đã thiết lập một số mã nguồn cho kịch bản thành công (happy path) để chúng ta bắt đầu. Đây là mã nguồn server của chúng ta:

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, store.Fetch())
	}
}
```

Hàm `Server` nhận một `Store` và trả về cho chúng ta một `http.HandlerFunc`. Store được định nghĩa là:

```go
type Store interface {
	Fetch() string
}
```

Hàm được trả về gọi phương thức `Fetch` của `store` để lấy dữ liệu và ghi nó vào phản hồi.

Chúng ta có một hàm giám sát (spy) tương ứng cho `Store` mà chúng ta sử dụng trong một bản kiểm thử.

```go
type SpyStore struct {
	response string
}

func (s *SpyStore) Fetch() string {
	return s.response
}

func TestServer(t *testing.T) {
	data := "hello, world"
	svr := Server(&SpyStore{data})

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}
}
```

Bây giờ chúng ta đã có kịch bản thành công, chúng ta muốn tạo một kịch bản thực tế hơn nơi `Store` không thể hoàn thành `Fetch` trước khi người dùng hủy yêu cầu.

## Viết test trước tiên

Handler của chúng ta sẽ cần một cách để thông báo cho `Store` hủy bỏ công việc, vì vậy hãy cập nhật interface:

```go
type Store interface {
	Fetch() string
	Cancel()
}
```

Chúng ta sẽ cần điều chỉnh hàm giám sát của mình để nó tốn một chút thời gian mới trả về `data`, và có một cách để biết nó đã được yêu cầu hủy bỏ. Nó sẽ phải thêm phương thức `Cancel` để triển khai interface `Store`.

```go
type SpyStore struct {
	response  string
	cancelled bool
}

func (s *SpyStore) Fetch() string {
	time.Sleep(100 * time.Millisecond)
	return s.response
}

func (s *SpyStore) Cancel() {
	s.cancelled = true
}
```

Hãy thêm một bản kiểm thử mới nơi chúng ta hủy yêu cầu trước 100 mili giây và kiểm tra store để xem nó có bị hủy hay không.

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)

	cancellingCtx, cancel := context.WithCancel(request.Context())
	time.AfterFunc(5*time.Millisecond, cancel)
	request = request.WithContext(cancellingCtx)

	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if !store.cancelled {
		t.Error("store was not told to cancel")
	}
})
```

Từ [Go Blog: Context](https://blog.golang.org/context):

> Package context cung cấp các hàm để tạo ra các giá trị Context mới từ các giá trị hiện có. Các giá trị này tạo thành một cái cây: khi một Context bị hủy, tất cả các Context được tạo ra từ nó cũng bị hủy.

Điều quan trọng là bạn phải tạo ra các context kế thừa để việc hủy bỏ được lan truyền (propagate) trong suốt stack lời gọi cho một yêu cầu nhất định.

Những gì chúng ta làm là tạo ra một `cancellingCtx` mới từ `request`, hàm này trả về cho chúng ta một hàm `cancel`. Sau đó, chúng ta lên lịch để hàm đó được gọi sau 5 mili giây bằng cách sử dụng `time.AfterFunc`. Cuối cùng, chúng ta sử dụng context mới này trong yêu cầu của mình bằng cách gọi `request.WithContext`.

## Thử chạy test

Bản kiểm thử thất bại như chúng ta mong đợi.

```
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.00s)
    	context_test.go:62: store was not told to cancel
```

## Viết đủ code để test chạy thành công

Hãy nhớ kỷ luật với TDD. Viết lượng mã *nhỏ nhất* để làm cho bản kiểm thử của chúng ta vượt qua.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		store.Cancel()
		fmt.Fprint(w, store.Fetch())
	}
}
```

Điều này làm cho test vượt qua nhưng cảm giác không ổn chút nào phải không! Chúng ta chắc chắn không nên gọi `Cancel()` trước khi fetch trong *mọi yêu cầu*.

Bằng cách kỷ luật, nó đã làm nổi bật một lỗ hổng trong các bản kiểm thử của chúng ta, đây là một điều tốt!

Chúng ta sẽ cần cập nhật bản kiểm thử kịch bản thành công để xác nhận rằng nó không bị hủy bỏ.

```go
t.Run("returns data from store", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}

	if store.cancelled {
		t.Error("it should not have cancelled the store")
	}
})
```

Chạy cả hai test và kịch bản thành công hiện tại sẽ thất bại, và bây giờ chúng ta bị buộc phải thực hiện một cách triển khai hợp lý hơn.

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		ctx := r.Context()

		data := make(chan string, 1)

		go func() {
			data <- store.Fetch()
		}()

		select {
		case d := <-data:
			fmt.Fprint(w, d)
		case <-ctx.Done():
			store.Cancel()
		}
	}
}
```

Chúng ta đã làm gì ở đây?

`context` có một phương thức `Done()` trả về một channel, channel này sẽ nhận được một tín hiệu khi context bị "xong" hoặc "bị hủy". Chúng ta muốn lắng nghe tín hiệu đó và gọi `store.Cancel` nếu nhận được, nhưng chúng ta muốn bỏ qua nó nếu `Store` của chúng ta hoàn thành việc `Fetch` trước đó.

Để quản lý điều này, chúng ta chạy `Fetch` trong một goroutine và nó sẽ ghi kết quả vào một channel mới `data`. Sau đó, chúng ta sử dụng `select` để thực hiện một cuộc đua hiệu quả giữa hai quy trình không đồng bộ và sau đó chúng ta viết một phản hồi hoặc gọi `Cancel`.

## Refactor

Chúng ta có thể tái cấu trúc mã kiểm thử của mình một chút bằng cách tạo các phương thức xác nhận trên hàm giám sát:

```go
type SpyStore struct {
	response  string
	cancelled bool
	t         *testing.T
}

func (s *SpyStore) assertWasCancelled() {
	s.t.Helper()
	if !s.cancelled {
		s.t.Error("store was not told to cancel")
	}
}

func (s *SpyStore) assertWasNotCancelled() {
	s.t.Helper()
	if s.cancelled {
		s.t.Error("store was told to cancel")
	}
}
```

Hãy nhớ truyền vào `*testing.T` khi tạo hàm giám sát.

```go
func TestServer(t *testing.T) {
	data := "hello, world"

	t.Run("returns data from store", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)
		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		if response.Body.String() != data {
			t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
		}

		store.assertWasNotCancelled()
	})

	t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
		store := &SpyStore{response: data, t: t}
		svr := Server(store)

		request := httptest.NewRequest(http.MethodGet, "/", nil)

		cancellingCtx, cancel := context.WithCancel(request.Context())
		time.AfterFunc(5*time.Millisecond, cancel)
		request = request.WithContext(cancellingCtx)

		response := httptest.NewRecorder()

		svr.ServeHTTP(response, request)

		store.assertWasCancelled()
	})
}
```

Cách tiếp cận này ổn, nhưng nó có phải là chuẩn mực (idiomatic)?

Liệu có hợp lý khi web server của chúng ta phải quan tâm đến việc hủy bỏ `Store` một cách thủ công? Điều gì xảy ra nếu `Store` cũng phụ thuộc vào các quy trình chạy chậm khác? Chúng ta sẽ phải đảm bảo rằng `Store.Cancel` lan truyền chính xác việc hủy bỏ đến tất cả các bên phụ thuộc của nó.

Một trong những mục đích chính của `context` là nó cung cấp một cách nhất quán để thực hiện việc hủy bỏ.

[Từ Go doc](https://golang.org/pkg/context/):

> Các yêu cầu đến một server nên tạo một Context, và các cuộc gọi đi đến các server khác nên chấp nhận một Context. Chuỗi các cuộc gọi hàm giữa chúng phải lan truyền Context, có thể thay thế nó bằng một Context phái sinh được tạo bằng WithCancel, WithDeadline, WithTimeout hoặc WithValue. Khi một Context bị hủy, tất cả các Context phái sinh từ nó cũng bị hủy.

Từ [Go Blog: Context](https://blog.golang.org/context):

> Tại Google, chúng tôi yêu cầu các lập trình viên Go truyền một tham số Context làm đối số đầu tiên cho mọi hàm trên đường dẫn cuộc gọi giữa các yêu cầu đến và đi. Điều này cho phép mã Go được phát triển bởi nhiều nhóm khác nhau có thể tương tác tốt. Nó cung cấp sự kiểm soát đơn giản đối với các khoảng thời gian chờ (timeouts) và việc hủy bỏ, đồng thời đảm bảo rằng các giá trị quan trọng như thông tin bảo mật được truyền qua các chương trình Go một cách chính xác.

(Dừng lại một chút và nghĩ về những hệ quả khi mọi hàm phải nhận vào một context, và tính tiện dụng của việc đó.)

Cảm thấy hơi khó chịu? Tốt. Tuy nhiên, hãy thử làm theo cách đó và thay vào đó truyền `context` qua cho `Store` của chúng ta và để nó tự chịu trách nhiệm. Bằng cách đó, nó cũng có thể truyền `context` qua cho các bên phụ thuộc của nó và họ cũng có thể tự chịu trách nhiệm dừng lại.

## Viết test trước tiên

Chúng ta sẽ phải thay đổi các bản kiểm thử hiện có vì trách nhiệm của chúng đang thay đổi. Điều duy nhất handler của chúng ta chịu trách nhiệm lúc này là đảm bảo nó gửi một context qua cho `Store` ở hạ nguồn và nó xử lý lỗi sẽ đến từ `Store` khi bị hủy.

Hãy cập nhật interface `Store` để thể hiện các trách nhiệm mới.

```go
type Store interface {
	Fetch(ctx context.Context) (string, error)
}
```

Xóa mã bên trong handler của chúng ta ngay bây giờ:

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
	}
}
```

Cập nhật `SpyStore` của chúng ta:

```go
type SpyStore struct {
	response string
	t        *testing.T
}

func (s *SpyStore) Fetch(ctx context.Context) (string, error) {
	data := make(chan string, 1)

	go func() {
		var result string
		for _, c := range s.response {
			select {
			case <-ctx.Done():
				log.Println("spy store got cancelled")
				return
			default:
				time.Sleep(10 * time.Millisecond)
				result += string(c)
			}
		}
		data <- result
	}()

	select {
	case <-ctx.Done():
		return "", ctx.Err()
	case res := <-data:
		return res, nil
	}
}
```

Chúng ta phải làm cho hàm giám sát hoạt động giống như một phương thức thực thụ làm việc với `context`.

Chúng ta đang mô phỏng một quy trình chậm nơi chúng ta xây dựng kết quả một cách từ từ bằng cách thêm từng ký tự vào chuỗi trong một goroutine. Khi goroutine hoàn thành công việc, nó sẽ ghi chuỗi vào channel `data`. Goroutine lắng nghe `ctx.Done` và sẽ dừng công việc nếu có tín hiệu được gửi vào channel đó.

Cuối cùng, mã sử dụng một `select` khác để đợi goroutine đó hoàn thành công việc hoặc xảy ra việc hủy bỏ.

Nó tương tự như cách tiếp cận trước đây của chúng ta, chúng ta sử dụng các nguyên hàm đồng thời (concurrency primitives) của Go để tạo ra hai quy trình không đồng bộ đua với nhau để xác định những gì chúng ta trả về.

Bạn sẽ thực hiện một cách tiếp cận tương tự khi viết các hàm và phương thức của riêng mình có chấp nhận `context`, vì vậy hãy đảm bảo bạn hiểu chuyện gì đang xảy ra.

Cuối cùng, chúng ta có thể cập nhật các bản kiểm thử của mình. Hãy tạm đóng (comment out) bản kiểm thử hủy bỏ để chúng ta có thể sửa test kịch bản thành công trước.

```go
t.Run("returns data from store", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data, t: t}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)
	response := httptest.NewRecorder()

	svr.ServeHTTP(response, request)

	if response.Body.String() != data {
		t.Errorf(`got "%s", want "%s"`, response.Body.String(), data)
	}
})
```

## Thử chạy test

```
=== RUN   TestServer/returns_data_from_store
--- FAIL: TestServer (0.00s)
    --- FAIL: TestServer/returns_data_from_store (0.00s)
    	context_test.go:22: got "", want "hello, world"
```

## Viết đủ code để test chạy thành công

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, _ := store.Fetch(r.Context())
		fmt.Fprint(w, data)
	}
}
```

Kịch bản thành công của chúng ta nên... thành công. Bây giờ chúng ta có thể sửa bản kiểm thử còn lại.

## Viết test trước tiên

Chúng ta cần kiểm thử rằng mình không viết bất kỳ loại phản hồi nào khi xảy ra lỗi. Đáng buồn là `httptest.ResponseRecorder` không có cách nào để xác định điều này, vì vậy chúng ta sẽ phải tự tạo hàm giám sát của riêng mình để kiểm tra.

```go
type SpyResponseWriter struct {
	written bool
}

func (s *SpyResponseWriter) Header() http.Header {
	s.written = true
	return nil
}

func (s *SpyResponseWriter) Write([]byte) (int, error) {
	s.written = true
	return 0, errors.New("not implemented")
}

func (s *SpyResponseWriter) WriteHeader(statusCode int) {
	s.written = true
}
```

`SpyResponseWriter` của chúng ta triển khai `http.ResponseWriter` để chúng ta có thể sử dụng nó trong bản kiểm thử.

```go
t.Run("tells store to cancel work if request is cancelled", func(t *testing.T) {
	data := "hello, world"
	store := &SpyStore{response: data, t: t}
	svr := Server(store)

	request := httptest.NewRequest(http.MethodGet, "/", nil)

	cancellingCtx, cancel := context.WithCancel(request.Context())
	time.AfterFunc(5*time.Millisecond, cancel)
	request = request.WithContext(cancellingCtx)

	response := &SpyResponseWriter{}

	svr.ServeHTTP(response, request)

	if response.written {
		t.Error("a response should not have been written")
	}
})
```

## Thử chạy test

```
=== RUN   TestServer
=== RUN   TestServer/tells_store_to_cancel_work_if_request_is_cancelled
--- FAIL: TestServer (0.01s)
    --- FAIL: TestServer/tells_store_to_cancel_work_if_request_is_cancelled (0.01s)
    	context_test.go:47: a response should not have been written
```

## Viết đủ code để test chạy thành công

```go
func Server(store Store) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		data, err := store.Fetch(r.Context())

		if err != nil {
			return // todo: log error however you like
		}

		fmt.Fprint(w, data)
	}
}
```

Sau việc này, chúng ta có thể thấy mã nguồn server đã trở nên đơn giản hơn vì nó không còn chịu trách nhiệm rõ ràng về việc hủy bỏ nữa, nó chỉ đơn giản là truyền `context` qua và dựa vào các hàm hạ nguồn để tôn trọng bất kỳ sự hủy bỏ nào có thể xảy ra.

## Tổng kết

### Những gì chúng ta đã học

- Cách kiểm thử một HTTP handler khi yêu cầu bị hủy bởi client.
- Cách sử dụng context để quản lý việc hủy bỏ.
- Cách viết một hàm chấp nhận `context` và sử dụng nó để tự hủy bằng cách dùng goroutine, `select` và channel.
- Tuân theo các hướng dẫn của Google về cách quản lý việc hủy bỏ bằng cách lan truyền context phạm vi yêu cầu (request scoped context) qua stack lời gọi.
- Cách tự tạo hàm giám sát cho `http.ResponseWriter` nếu bạn cần.

### Còn context.Value thì sao?

[Michal Štrba](https://faiface.github.io/post/context-should-go-away-go2/) và tôi có quan điểm tương tự.

> Nếu bạn sử dụng ctx.Value trong công ty (không tồn tại) của tôi, bạn bị đuổi việc.

Một số kỹ sư đã ủng hộ việc truyền các giá trị qua `context` vì nó *cảm thấy tiện lợi*.

Sự tiện lợi thường là nguyên nhân gây ra mã nguồn tồi.

Vấn đề với `context.Values` là nó chỉ là một map không được định nghĩa kiểu (untyped map) nên bạn không có tính an toàn kiểu (type-safety) và bạn phải xử lý trường hợp nó không thực sự chứa giá trị của bạn. Bạn phải tạo ra sự liên kết giữa các map keys từ module này sang module khác và nếu ai đó thay đổi điều gì đó, mọi thứ bắt đầu bị hỏng.

Tóm lại, **nếu một hàm cần một số giá trị, hãy đặt chúng làm các tham số có kiểu dữ liệu rõ ràng thay vì cố gắng lấy chúng từ `context.Value`**. Điều này giúp nó được kiểm tra tĩnh và được tài liệu hóa cho mọi người xem.

#### Nhưng...

Mặt khác, có thể hữu ích khi bao gồm thông tin trực giao (orthogonal) với một yêu cầu trong một context, chẳng hạn như trace id. Có khả năng thông tin này sẽ không cần thiết cho mọi hàm trong stack lời gọi của bạn và sẽ làm cho các signature hàm của bạn trở nên rất lộn xộn.

[Jack Lindamood nói **Context.Value nên mang tính thông tin, không phải điều khiển**](https://medium.com/@cep21/how-to-correctly-use-context-context-in-go-1-7-8f2c0fafdf39):

> Nội dung của context.Value là dành cho những người bảo trì chứ không phải người dùng. Nó không bao giờ được là đầu vào bắt buộc cho các kết quả được tài liệu hóa hoặc mong đợi.

### Tài liệu bổ sung

- Tôi thực sự thích đọc bài [Context should go away for Go 2 của Michal Štrba](https://faiface.github.io/post/context-should-go-away-go2/). Lập luận của ông là việc phải truyền `context` ở mọi nơi là một dấu hiệu xấu (smell), nó chỉ ra một thiếu sót của ngôn ngữ trong việc hủy bỏ. Ông nói rằng sẽ tốt hơn nếu điều này được giải quyết ở cấp độ ngôn ngữ theo cách nào đó, thay vì ở cấp độ thư viện. Cho đến khi điều đó xảy ra, bạn sẽ cần `context` nếu muốn quản lý các quy trình chạy lâu.
- [Go blog mô tả thêm động lực khi làm việc với `context` và có một số ví dụ](https://blog.golang.org/context).
