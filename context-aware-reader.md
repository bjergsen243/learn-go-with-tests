# Context-aware Reader

**[Bạn có thể tìm thấy toàn bộ mã nguồn tại đây](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/context-aware-reader)**

Chương này hướng dẫn cách dùng TDD để xây dựng một `io.Reader` có khả năng nhận biết context, dựa trên bài viết của Mat Ryer và David Hernandez trên [The Pace Dev Blog](https://pace.dev/blog/2020/02/03/context-aware-ioreader-for-golang-by-mat-ryer).

## Context aware reader là gì?

Trước hết, hãy cùng tóm lược nhanh về `io.Reader`.

Nếu đã đọc các chương trước, bạn đã gặp `io.Reader` khi mở file, encode JSON và nhiều tác vụ khác. Đây là một abstraction đơn giản cho việc đọc dữ liệu từ _một nguồn nào đó_.

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

Dùng `io.Reader` giúp bạn tận dụng rất nhiều code có sẵn trong thư viện chuẩn, vì đây là abstraction cực kỳ phổ biến (cùng với `io.Writer`).

### Context-aware nghĩa là gì?

[Trong chương trước](context.md), chúng ta đã tìm hiểu cách dùng `context` để hủy bỏ (cancel) các tác vụ. Điều này hữu ích khi bạn có tác vụ tốn tài nguyên và muốn dừng giữa chừng.

Với `io.Reader`, bạn không biết trước tốc độ đọc — có thể mất 1 nano giây hoặc hàng trăm giờ. Khả năng hủy tác vụ đọc rất hữu ích, và đó là điều Mat và David đã giải quyết.

Họ kết hợp hai abstraction đơn giản (`context.Context` và `io.Reader`) để xử lý vấn đề này.

Hãy thử áp dụng TDD cho một số chức năng để chúng ta có thể bao bọc một `io.Reader` sao cho nó có thể bị hủy bỏ.

Việc kiểm thử điều này đặt ra một thách thức thú vị. Thông thường khi sử dụng một `io.Reader`, bạn thường cung cấp nó cho một hàm khác và bạn không thực sự bận tâm đến các chi tiết; chẳng hạn như `json.NewDecoder` hoặc `io.ReadAll`.

Những gì chúng ta muốn chứng minh là thứ gì đó tương tự như:

> Cho một `io.Reader` với nội dung "ABCDEF", khi tôi gửi tín hiệu hủy bỏ ở giữa quá trình đọc, thì khi tôi cố gắng tiếp tục đọc, tôi sẽ không nhận được thêm gì nữa, vì vậy tất cả những gì tôi nhận được chỉ là "ABC".

Hãy xem lại interface một lần nữa.

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

Phương thức `Read` của `Reader` sẽ đọc nội dung nó có vào một `[]byte` mà chúng ta cung cấp.

Thay vì đọc mọi thứ, chúng ta có thể:

 - Cung cấp một mảng byte có kích thước cố định không chứa hết toàn bộ nội dung.
 - Gửi một tín hiệu hủy bỏ.
 - Thử đọc lại và việc này sẽ trả về một lỗi với 0 byte được đọc.

Hiện tại, hãy viết một bài kiểm thử "happy path" (trường hợp thuận lợi) nơi không có sự hủy bỏ, chỉ để chúng ta làm quen với vấn đề mà chưa cần viết bất kỳ mã nguồn thực tế nào.

```go
func TestContextAwareReader(t *testing.T) {
	t.Run("hãy xem một reader thông thường hoạt động như thế nào", func(t *testing.T) {
		rdr := strings.NewReader("123456")
		got := make([]byte, 3)
		_, err := rdr.Read(got)

		if err != nil {
			t.Fatal(err)
		}

		assertBufferHas(t, got, "123")

		_, err = rdr.Read(got)

		if err != nil {
			t.Fatal(err)
		}

		assertBufferHas(t, got, "456")
	})
}

func assertBufferHas(t testing.TB, buf []byte, want string) {
	t.Helper()
	got := string(buf)
	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

- Tạo một `io.Reader` từ một chuỗi với một số dữ liệu.
- Một mảng byte để đọc vào, có kích thước nhỏ hơn nội dung của reader.
- Gọi read, kiểm tra nội dung, lặp lại.

Từ ví dụ này, chúng ta có thể hình dung việc gửi một loại tín hiệu hủy bỏ nào đó trước lần đọc thứ hai để thay đổi hành vi.

Bây giờ chúng ta đã thấy nó hoạt động như thế nào, chúng ta sẽ áp dụng TDD cho các chức năng còn lại.

## Viết bài kiểm thử trước

Chúng ta muốn có thể kết hợp một `io.Reader` với một `context.Context`.

Với TDD, tốt nhất là bắt đầu bằng việc hình dung API mong muốn và viết một bài kiểm thử cho nó.

Từ đó, hãy để trình biên dịch và kết quả kiểm thử thất bại dẫn dắt chúng ta đến một giải pháp.

```go
t.Run("hoạt động giống như một reader thông thường", func(t *testing.T) {
	rdr := NewCancellableReader(strings.NewReader("123456"))
	got := make([]byte, 3)
	_, err := rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "123")

	_, err = rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "456")
})
```

## Thử chạy bài kiểm thử

```
./cancel_readers_test.go:12:10: undefined: NewCancellableReader
```

## Viết lượng mã tối thiểu để bài kiểm thử có thể chạy và kiểm tra kết quả lỗi

Chúng ta sẽ cần định nghĩa hàm này và nó nên trả về một `io.Reader`.

```go
func NewCancellableReader(rdr io.Reader) io.Reader {
	return nil
}
```

Nếu bạn thử chạy nó:

```
=== RUN   TestCancelReaders
=== RUN   TestCancelReaders/behaves_like_a_normal_reader
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
	panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x10f8fb5]
```

Đúng như dự đoán.

## Viết đủ mã để bài kiểm thử vượt qua

Hiện tại, chúng ta sẽ chỉ trả về `io.Reader` mà chúng ta truyền vào.

```go
func NewCancellableReader(rdr io.Reader) io.Reader {
	return rdr
}
```

Bài kiểm thử bây giờ sẽ vượt qua.

Tôi biết, tôi biết, điều này có vẻ kỳ quặc và máy móc, nhưng trước khi bắt tay vào phần việc phức tạp, điều quan trọng là chúng ta phải có một số sự xác nhận rằng chúng ta không làm hỏng hành vi "thông thường" của một `io.Reader` và bài kiểm thử này sẽ mang lại cho chúng ta sự tự tin khi tiến xa hơn.

## Viết bài kiểm thử trước

Tiếp theo, chúng ta cần thử thực hiện việc hủy bỏ.

```go
t.Run("dừng đọc khi bị hủy", func(t *testing.T) {
	ctx, cancel := context.WithCancel(context.Background())
	rdr := NewCancellableReader(ctx, strings.NewReader("123456"))
	got := make([]byte, 3)
	_, err := rdr.Read(got)

	if err != nil {
		t.Fatal(err)
	}

	assertBufferHas(t, got, "123")

	cancel()

	n, err := rdr.Read(got)

	if err == nil {
		t.Error("mong đợi một lỗi sau khi hủy nhưng không nhận được")
	}

	if n > 0 {
		t.Errorf("mong đợi 0 byte được đọc sau khi hủy nhưng đã đọc được %d byte", n)
	}
})
```

Chúng ta ít nhiều có thể sao chép bài kiểm thử đầu tiên nhưng bây giờ chúng ta đang làm thêm các việc:
- Tạo một `context.Context` với khả năng hủy bỏ để chúng ta có thể gọi `cancel` sau lần đọc đầu tiên.
- Để mã của chúng ta hoạt động, chúng ta cần truyền `ctx` vào hàm của mình.
- Sau đó, chúng ta khẳng định (assert) rằng sau khi gọi `cancel`, không có gì được đọc thêm.

## Thử chạy bài kiểm thử

```
./cancel_readers_test.go:33:30: too many arguments in call to NewCancellableReader
	have (context.Context, *strings.Reader)
	want (io.Reader)
```

## Viết lượng mã tối thiểu để bài kiểm thử có thể chạy và kiểm tra kết quả lỗi

Trình biên dịch đang cho chúng ta biết phải làm gì; hãy cập nhật chữ ký hàm (signature) của chúng ta để chấp nhận một context.

```go
func NewCancellableReader(ctx context.Context, rdr io.Reader) io.Reader {
	return rdr
}
```

(Bạn cũng sẽ cần cập nhật bài kiểm thử đầu tiên để truyền vào `context.Background`.)

Bây giờ bạn sẽ thấy kết quả bài kiểm thử thất bại rất rõ ràng:

```
=== RUN   TestCancelReaders
=== RUN   TestCancelReaders/stops_reading_when_cancelled
--- FAIL: TestCancelReaders (0.00s)
    --- FAIL: TestCancelReaders/stops_reading_when_cancelled (0.00s)
        cancel_readers_test.go:48: expected an error but didn't get one
        cancel_readers_test.go:52: expected 0 bytes to be read after cancellation but 3 were read
```

## Viết đủ mã để bài kiểm thử vượt qua

Tại thời điểm này, chúng ta sẽ sao chép từ bài viết gốc của Mat và David nhưng chúng ta vẫn sẽ thực hiện chậm rãi và theo từng bước lặp.

Chúng ta biết mình cần có một kiểu dữ liệu đóng gói `io.Reader` mà chúng ta đọc từ đó và `context.Context`, vì vậy hãy tạo kiểu đó và thử trả về nó từ hàm của chúng ta thay vì trả về `io.Reader` gốc.

```go
func NewCancellableReader(ctx context.Context, rdr io.Reader) io.Reader {
	return &readerCtx{
		ctx:      ctx,
		delegate: rdr,
	}
}

type readerCtx struct {
	ctx      context.Context
	delegate io.Reader
}
```

Như tôi đã nhấn mạnh nhiều lần trong cuốn sách này, hãy đi chậm và để trình biên dịch giúp bạn.

```
./cancel_readers_test.go:60:3: cannot use &readerCtx literal (type *readerCtx) as type io.Reader in return argument:
	*readerCtx does not implement io.Reader (missing Read method)
```

Sự trừu tượng hóa có vẻ đúng, nhưng nó chưa triển khai interface mà chúng ta cần (`io.Reader`), vì vậy hãy thêm phương thức đó vào.

```go
func (r *readerCtx) Read(p []byte) (n int, err error) {
	panic("implement me")
}
```

Chạy các bài kiểm thử và chúng sẽ được _biên dịch_ thành công nhưng sẽ gây ra lỗi panic. Đây vẫn là một bước tiến.

Hãy làm cho bài kiểm thử đầu tiên vượt qua bằng cách chỉ cần _ủy nhiệm (delegate)_ lệnh gọi cho `io.Reader` bên dưới.

```go
func (r readerCtx) Read(p []byte) (n int, err error) {
	return r.delegate.Read(p)
}
```

Tại thời điểm này, bài kiểm thử happy path của chúng ta đã vượt qua trở lại và có cảm giác như chúng ta đã trừu tượng hóa mọi thứ một cách ổn thỏa.

Để làm cho bài kiểm thử thứ hai vượt qua, chúng ta cần kiểm tra `context.Context` để xem nó đã bị hủy bỏ hay chưa.

```go
func (r readerCtx) Read(p []byte) (n int, err error) {
	if err := r.ctx.Err(); err != nil {
		return 0, err
	}
	return r.delegate.Read(p)
}
```

Bây giờ tất cả các bài kiểm thử sẽ vượt qua. Bạn sẽ nhận thấy cách chúng ta trả về lỗi từ `context.Context`. Điều này cho phép những người gọi mã kiểm tra các lý do khác nhau dẫn đến việc hủy bỏ đã xảy ra và điều này được đề cập kỹ hơn trong bài viết gốc.

## Tổng kết

- Các interface nhỏ là tốt và dễ dàng kết hợp (compose).
- Khi muốn mở rộng khả năng của một thứ (ví dụ `io.Reader`), hãy xem xét [delegation pattern](https://en.wikipedia.org/wiki/Delegation_pattern).

> Trong kỹ thuật phần mềm, mẫu ủy nhiệm là một mẫu thiết kế hướng đối tượng cho phép kết hợp đối tượng để đạt được khả năng tái sử dụng mã tương tự như kế thừa.

- Một cách dễ dàng để bắt đầu loại công việc này là bao bọc đối tượng ủy nhiệm của bạn và viết một bài kiểm thử khẳng định rằng nó hoạt động giống như cách đối tượng ủy nhiệm thường làm trước khi bạn bắt đầu kết hợp các phần khác để thay đổi hành vi. Điều này sẽ giúp bạn duy trì mọi thứ hoạt động chính xác khi bạn viết mã hướng tới mục tiêu của mình.
