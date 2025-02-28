# Vòng lặp

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/for)**

Để thực hiện một hành động lặp lại trong Go, bạn sẽ cần sử dụng `for`. Trong Go không có các từ khóa như `while`, `do`, `until`, bạn chỉ có thể dùng `for` và điều này thực sự hữu ích!

Hãy viết một bài kiểm thử cho một hàm lặp lại một ký tự 5 lần.

Không có gì mới ở đây, vì vậy hãy thử tự viết nó để thực hành.

## Viết test trước tiên

```go
package iteration

import "testing"

func TestRepeat(t *testing.T) {
	repeated := Repeat("a")
	expected := "aaaaa"

	if repeated != expected {
		t.Errorf("expected %q but got %q", expected, repeated)
	}
}
```

## Thử chạy test

`./repeat_test.go:6:14: undefined: Repeat`

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

_Giữ kỷ luật_! Bạn chưa cần học thêm điều gì mới để làm cho test thất bại đúng cách.

Lúc này, bạn chỉ cần viết đủ code để nó biên dịch được, sau đó kiểm tra xem test của bạn đã được viết đúng chưa.

```go
package iteration

func Repeat(character string) string {
	return ""
}
```

Thật tuyệt khi biết rằng bạn đã nắm đủ kiến thức về Go để viết test cho một số bài toán cơ bản! Điều này có nghĩa là bạn có thể tự do chỉnh sửa code mà vẫn đảm bảo nó hoạt động đúng như mong đợi.

`repeat_test.go:10: expected 'aaaaa' but got ''`

## Viết đủ code để test chạy thành công

Cú pháp `for` trong Go rất đơn giản và tương tự như hầu hết các ngôn ngữ lập trình kiểu C.

```go
func Repeat(character string) string {
	var repeated string
	for i := 0; i < 5; i++ {
		repeated = repeated + character
	}
	return repeated
}
```

Không giống như các ngôn ngữ khác như C, Java hay JavaScript, câu lệnh for trong Go không cần dấu ngoặc tròn `()` bao quanh ba thành phần của vòng lặp, nhưng dấu ngoặc nhọn `{}` luôn bắt buộc phải có. Bạn có thể tự hỏi điều gì đang xảy ra trong dòng này.

```go
	var repeated string
```

Từ trước đến giờ, chúng ta đã sử dụng `:=` để khai báo và khởi tạo biến. Tuy nhiên, `:=` chỉ là một cách viết tắt cho cả hai bước này ([xem chi tiết tại đây](https://gobyexample.com/variables)). Trong trường hợp này, chúng ta chỉ khai báo một biến kiểu `string`, vì vậy cần viết tường minh hơn.

Ngoài ra, chúng ta cũng có thể sử dụng `var` để khai báo hàm, điều này sẽ được đề cập sau.

Các biến thể khác của vòng lặp for được mô tả [tại đây](https://gobyexample.com/for).

## Refactor

Bây giờ là lúc refactor và giới thiệu thêm một toán tử gán khác: `+=`.

```go
const repeatCount = 5

func Repeat(character string) string {
	var repeated string
	for i := 0; i < repeatCount; i++ {
		repeated += character
	}
	return repeated
}
```

Toán tử `+=`, được gọi là _“toán tử cộng và gán”_, sẽ cộng toán hạng bên phải vào toán hạng bên trái và gán kết quả cho toán hạng bên trái. Toán tử này cũng hoạt động với các kiểu dữ liệu khác như số nguyên.


### Benchmarking

Viết [benchmark](https://golang.org/pkg/testing/#hdr-Benchmarks) trong Go là một tính năng quan trọng của ngôn ngữ và nó có cách viết rất giống với viết test.

```go
func BenchmarkRepeat(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Repeat("a")
	}
}
```

Bạn sẽ thấy mã benchmark rất giống với một bài test.

`testing.B` cung cấp quyền truy cập vào `b.N`, một giá trị được framework quyết định.

Khi mã benchmark chạy, nó sẽ lặp `b.N` lần và đo thời gian thực hiện.

Số lần chạy không quan trọng với bạn, vì Go sẽ tự động xác định giá trị phù hợp để đưa ra kết quả chính xác.

Để chạy benchmark, dùng lệnh:

- Trên Linux/macOS: `go test -bench=.`
- Trên Windows PowerShell: go test -bench="."

```text
goos: darwin
goarch: amd64
pkg: github.com/quii/learn-go-with-tests/for/v4
10000000           136 ns/op
PASS
```

Ở đây `136 ns/op` có nghĩa là trung bình mỗi lần chạy hàm mất 136 nanosecond \(trên máy của tôi\). Để đo được kết quả này, Go đã chạy nó 10000000 lần.

**Chú ý:** Mặc định, các benchmark chạy tuần tự.

**Chú ý:** Đôi khi, Go có thể tối ưu hóa quá mức, làm cho benchmark không chính xác, chẳng hạn như loại bỏ hoàn toàn hàm đang được đo. Nếu thấy kết quả quá tốt để tin, hãy tham khảo các chiến lược trong **[bài viết này](https://dave.cheney.net/2013/06/30/how-to-write-benchmarks-in-go)** để đảm bảo benchmark phản ánh đúng hiệu suất thực tế.
## Bài tập thực hành

-   Chỉnh sửa bài kiểm thử để người dùng có thể chỉ định số lần lặp lại ký tự, sau đó sửa mã để đáp ứng yêu cầu.
-   Viết `ExampleRepeat` để tài liệu hóa hàm của bạn.
-   Xem qua gói [strings](https://golang.org/pkg/strings). Tìm các hàm hữu ích và thử nghiệm với chúng bằng cách viết các bài kiểm thử như chúng ta đã làm. Đầu tư thời gian tìm hiểu thư viện tiêu chuẩn sẽ rất có ích về lâu dài.

## Tổng kết

-   Thực hành TDD nhiều hơn
-   Học cách sử dụng vòng lặp `for`
-   Học cách viết benchmark để đo hiệu suất
