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

_Keep the discipline!_ You don't need to know anything new right now to make the test fail properly.

All you need to do right now is enough to make it compile so you can check your test is written well.

```go
package iteration

func Repeat(character string) string {
	return ""
}
```

Isn't it nice to know you already know enough Go to write tests for some basic problems? This means you can now play with the production code as much as you like and know it's behaving as you'd hope.

`repeat_test.go:10: expected 'aaaaa' but got ''`

## Viết đủ code để test chạy thành công

The `for` syntax is very unremarkable and follows most C-like languages.

```go
func Repeat(character string) string {
	var repeated string
	for i := 0; i < 5; i++ {
		repeated = repeated + character
	}
	return repeated
}
```

Unlike other languages like C, Java, or JavaScript there are no parentheses surrounding the three components of the for statement and the braces `{ }` are always required. You might wonder what is happening in the row

```go
	var repeated string
```

as we've been using `:=` so far to declare and initializing variables. However, `:=` is simply [short hand for both steps](https://gobyexample.com/variables). Here we are declaring a `string` variable only. Hence, the explicit version. We can also use `var` to declare functions, as we'll see later on.

Run the test and it should pass.

Additional variants of the for loop are described [here](https://gobyexample.com/for).

## Refactor

Now it's time to refactor and introduce another construct `+=` assignment operator.

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

`+=` called _"the Add AND assignment operator"_, adds the right operand to the left operand and assigns the result to left operand. It works with other types like integers.

### Benchmarking

Writing [benchmarks](https://golang.org/pkg/testing/#hdr-Benchmarks) in Go is another first-class feature of the language and it is very similar to writing tests.

```go
func BenchmarkRepeat(b *testing.B) {
	for i := 0; i < b.N; i++ {
		Repeat("a")
	}
}
```

You'll see the code is very similar to a test.

The `testing.B` gives you access to the cryptically named `b.N`.

When the benchmark code is executed, it runs `b.N` times and measures how long it takes.

The amount of times the code is run shouldn't matter to you, the framework will determine what is a "good" value for that to let you have some decent results.

To run the benchmarks do `go test -bench=.` (or if you're in Windows Powershell `go test -bench="."`)

```text
goos: darwin
goarch: amd64
pkg: github.com/quii/learn-go-with-tests/for/v4
10000000           136 ns/op
PASS
```

What `136 ns/op` means is our function takes on average 136 nanoseconds to run \(on my computer\). Which is pretty ok! To test this it ran it 10000000 times.

**Chú ý:** By default benchmarks are run sequentially.

**Chú ý:** Sometimes, Go can optimize your benchmarks in a way that makes them inaccurate, such as eliminating the function being benchmarked. Check your benchmarks to see if the values make sense. If they seem overly optimized, you can follow the strategies in this **[blog post](https://dave.cheney.net/2013/06/30/how-to-write-benchmarks-in-go)**.

## Bài tập thực hành

-   Chỉnh sửa bài kiểm thử để người dùng có thể chỉ định số lần lặp lại ký tự, sau đó sửa mã để đáp ứng yêu cầu.
-   Viết `ExampleRepeat` để tài liệu hóa hàm của bạn.
-   Xem qua gói [strings](https://golang.org/pkg/strings). Tìm các hàm hữu ích và thử nghiệm với chúng bằng cách viết các bài kiểm thử như chúng ta đã làm. Đầu tư thời gian tìm hiểu thư viện tiêu chuẩn sẽ rất có ích về lâu dài.

## Tổng kết

-   Thực hành TDD nhiều hơn
-   Học cách sử dụng vòng lặp `for`
-   Học cách viết benchmark để đo hiệu suất
