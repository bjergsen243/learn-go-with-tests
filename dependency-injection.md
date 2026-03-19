# Dependency Injection

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/di)**

Chương này giả định rằng bạn đã đọc [phần structs](./structs-methods-and-interfaces.md) trước đó vì chúng ta sẽ cần hiểu biết về interfaces.

Có *rất nhiều* hiểu lầm xung quanh Dependency Injection (tiêm phụ thuộc) trong cộng đồng lập trình. Hy vọng hướng dẫn này sẽ cho bạn thấy rằng:

* Bạn không cần một framework nào cả.
* Nó không làm thiết kế của bạn quá phức tạp.
* Nó hỗ trợ việc kiểm thử (testing).
* Nó cho phép bạn viết các hàm tổng quát, tuyệt vời.

Chúng ta muốn viết một hàm để chào hỏi ai đó, giống như chúng ta đã làm trong chương hello-world, nhưng lần này chúng ta sẽ kiểm thử việc *in thực tế*.

Để nhắc lại, đây là giao diện của hàm đó:

```go
func Greet(name string) {
	fmt.Printf("Hello, %s", name)
}
```

Nhưng làm sao chúng ta có thể kiểm thử điều này? Việc gọi `fmt.Printf` sẽ in ra stdout (đầu ra tiêu chuẩn), điều này khá khó để chúng ta nắm bắt bằng framework testing.

Những gì chúng ta cần làm là có thể **inject** (tiêm - thực chất chỉ là một từ hoa mỹ cho việc truyền vào) phụ thuộc của việc in ấn.

**Hàm của chúng ta không cần quan tâm việc in ấn diễn ra *ở đâu* hay *như thế nào*, vì vậy chúng ta nên chấp nhận một *interface* thay vì một kiểu cụ thể.**

Nếu chúng ta làm vậy, chúng ta có thể thay đổi implementation (cách triển khai) để in ra một thứ gì đó mà chúng ta kiểm soát để có thể kiểm thử nó. Trong "đời thực", bạn sẽ tiêm vào một thứ gì đó ghi ra stdout.

Nếu bạn nhìn vào mã nguồn của [`fmt.Printf`](https://pkg.go.dev/fmt#Printf), bạn có thể thấy một cách để chúng ta can thiệp:

```go
// Nó trả về số byte đã ghi và bất kỳ lỗi ghi nào gặp phải.
func Printf(format string, a ...interface{}) (n int, err error) {
	return Fprintf(os.Stdout, format, a...)
}
```

Thú vị! Bên dưới lớp vỏ, `Printf` chỉ gọi `Fprintf` và truyền vào `os.Stdout`.

Chính xác thì `os.Stdout` *là* cái gì? `Fprintf` mong đợi tham số đầu tiên là gì?

```go
func Fprintf(w io.Writer, format string, a ...interface{}) (n int, err error) {
	p := newPrinter()
	p.doPrintf(format, a)
	n, err = w.Write(p.buf)
	p.free()
	return
}
```

Đó là một `io.Writer`:

```go
type Writer interface {
	Write(p []byte) (n int, err error)
}
```

Từ đây chúng ta có thể suy ra rằng `os.Stdout` triển khai `io.Writer`; `Printf` truyền `os.Stdout` cho `Fprintf`, nơi mong đợi một `io.Writer`.

Khi bạn viết nhiều mã Go hơn, bạn sẽ thấy interface này xuất hiện rất nhiều vì đây là một interface tổng quát tuyệt vời cho việc "đưa dữ liệu này đến đâu đó".

Vì vậy, chúng ta biết rằng thực tế chúng ta đang sử dụng `Writer` để gửi lời chào của mình đến một nơi nào đó. Hãy sử dụng trừu tượng (abstraction) hiện có này để làm cho mã của chúng ta có thể kiểm thử được và dễ tái sử dụng hơn.

## Viết test trước tiên

```go
func TestGreet(t *testing.T) {
	buffer := bytes.Buffer{}
	Greet(&buffer, "Chris")

	got := buffer.String()
	want := "Hello, Chris"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

Kiểu `Buffer` từ package `bytes` triển khai interface `Writer`, vì nó có phương thức `Write(p []byte) (n int, err error)`.

Vì vậy, chúng ta sẽ sử dụng nó trong test để truyền vào dưới dạng `Writer` của mình, sau đó chúng ta có thể kiểm tra những gì đã được ghi vào đó sau khi gọi `Greet`.

## Thử chạy test

Test sẽ không biên dịch được:

```text
./di_test.go:10:2: undefined: Greet
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

*Hãy lắng nghe compiler* và khắc phục vấn đề.

```go
func Greet(writer *bytes.Buffer, name string) {
	fmt.Printf("Hello, %s", name)
}
```

`Hello, Chris di_test.go:16: got '' want 'Hello, Chris'`

Test thất bại. Lưu ý rằng tên đang được in ra, nhưng nó đi ra stdout.

## Viết đủ code để test chạy thành công

Sử dụng writer để gửi lời chào đến buffer trong test của chúng ta. Hãy nhớ `fmt.Fprintf` giống như `fmt.Printf` nhưng thay vào đó nhận một `Writer` để gửi chuỗi đến, trong khi `fmt.Printf` mặc định gửi đến stdout.

```go
func Greet(writer *bytes.Buffer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}
```

Test bây giờ đã vượt qua.

## Refactor

Trước đó compiler đã bảo chúng ta truyền vào một con trỏ tới `bytes.Buffer`. Điều này về mặt kỹ thuật là đúng nhưng không hữu ích lắm.

Để chứng minh điều này, hãy thử kết nối hàm `Greet` vào một ứng dụng Go nơi chúng ta muốn nó in ra stdout.

```go
func main() {
	Greet(os.Stdout, "Elodie")
}
```

`./di.go:14:7: cannot use os.Stdout (type *os.File) as type *bytes.Buffer in argument to Greet`

Như đã thảo luận trước đó, `fmt.Fprintf` cho phép bạn truyền vào một `io.Writer`, mà chúng ta biết cả `os.Stdout` và `bytes.Buffer` đều triển khai.

Nếu chúng ta thay đổi mã của mình để sử dụng interface tổng quát hơn, chúng ta có thể sử dụng nó trong cả test và trong ứng dụng của mình.

```go
package main

import (
	"fmt"
	"io"
	"os"
)

func Greet(writer io.Writer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}

func main() {
	Greet(os.Stdout, "Elodie")
}
```

## Tìm hiểu thêm về io.Writer

Những nơi khác chúng ta có thể ghi dữ liệu bằng `io.Writer` là gì? Hàm `Greet` của chúng ta tổng quát đến mức nào?

### Internet

Chạy đoạn sau:

```go
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
)

func Greet(writer io.Writer, name string) {
	fmt.Fprintf(writer, "Hello, %s", name)
}

func MyGreeterHandler(w http.ResponseWriter, r *http.Request) {
	Greet(w, "world")
}

func main() {
	log.Fatal(http.ListenAndServe(":5001", http.HandlerFunc(MyGreeterHandler)))
}
```

Chạy chương trình và truy cập [http://localhost:5001](http://localhost:5001). Bạn sẽ thấy hàm chào hỏi của mình đang được sử dụng.

HTTP server sẽ được đề cập trong một chương sau nên đừng lo lắng quá nhiều về chi tiết.

Khi bạn viết một HTTP handler, bạn được cung cấp một `http.ResponseWriter` và `http.Request` được sử dụng để tạo request. Khi bạn triển khai server của mình, bạn *ghi* phản hồi của mình bằng cách sử dụng writer.

Bạn có lẽ có thể đoán rằng `http.ResponseWriter` cũng triển khai `io.Writer`, vì vậy đây là lý do tại sao chúng ta có thể tái sử dụng hàm `Greet` của mình bên trong handler.

## Tổng kết

Đoạn mã đầu tiên của chúng ta không dễ kiểm thử vì nó ghi dữ liệu đến nơi mà chúng ta không thể kiểm soát.

*Được thúc đẩy bởi các bản kiểm thử*, chúng ta đã cấu trúc lại mã để có thể kiểm soát được *nơi* dữ liệu được ghi bằng cách **injecting a dependency** (tiêm một phụ thuộc), điều này cho phép chúng ta:

* **Kiểm thử mã của mình**: Nếu bạn không thể kiểm thử một hàm *dễ dàng*, thường là do các phụ thuộc (dependencies) được gắn cứng vào một hàm *hoặc* do trạng thái toàn cục (global state). Ví dụ: nếu bạn có một pool kết nối cơ sở dữ liệu toàn cục được sử dụng bởi một lớp dịch vụ nào đó, nó có thể sẽ khó kiểm thử và sẽ chạy chậm. DI sẽ thúc đẩy bạn tiêm phụ thuộc cơ sở dữ liệu vào (thông qua một interface), sau đó bạn có thể mock bằng một thứ gì đó mà bạn có thể kiểm soát trong test.
* **Tách biệt các mối quan tâm (Separate concerns)**: Tách rời việc *dữ liệu đi đâu* và *cách tạo ra dữ liệu*. Nếu bạn từng cảm thấy một phương thức/hàm có quá nhiều trách nhiệm (vừa tạo dữ liệu *vừa* ghi vào cơ sở dữ liệu? vừa xử lý HTTP request *vừa* thực hiện logic nghiệp vụ?), DI có lẽ là công cụ bạn cần.
* **Cho phép mã của chúng ta được tái sử dụng trong các ngữ cảnh khác nhau**: Ngữ cảnh "mới" đầu tiên mã của chúng ta có thể được sử dụng là bên trong các bản kiểm thử. Nhưng xa hơn nữa, nếu ai đó muốn thử một điều gì đó mới với hàm của bạn, họ có thể tiêm các phụ thuộc của riêng họ.

### Còn về mocking? Tôi nghe nói bạn cần điều đó cho DI và nó cũng là "quỷ dữ"

Mocking sẽ được đề cập chi tiết sau (và nó không phải quỷ dữ). Bạn sử dụng mocking để thay thế những thứ thực tế bạn tiêm vào bằng một phiên bản giả định mà bạn có thể kiểm soát và kiểm tra trong test. Tuy nhiên trong trường hợp của chúng ta, thư viện chuẩn đã có sẵn thứ gì đó để chúng ta sử dụng.

### Thư viện chuẩn của Go thực sự rất tốt, hãy dành thời gian để nghiên cứu nó

Bằng cách làm quen với interface `io.Writer`, chúng ta có thể sử dụng `bytes.Buffer` trong test của mình dưới dạng `Writer` và sau đó chúng ta có thể sử dụng các `Writer` khác từ thư viện chuẩn để sử dụng hàm của mình trong một ứng dụng dòng lệnh hoặc trong máy chủ web.

Càng quen thuộc với thư viện chuẩn, bạn sẽ càng thấy nhiều interface tổng quát này, sau đó bạn có thể tái sử dụng chúng trong mã của chính mình để làm cho phần mềm của bạn có thể tái sử dụng trong nhiều ngữ cảnh khác nhau.

Ví dụ này chịu ảnh hưởng lớn từ một chương trong [The Go Programming Language](https://www.amazon.co.uk/Programming-Language-Addison-Wesley-Professional-Computing/dp/0134190440), vì vậy nếu bạn thích điều này, hãy mua cuốn sách đó!
