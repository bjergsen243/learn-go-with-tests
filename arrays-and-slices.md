# Arrays và slices

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/arrays)**

Arrays cho phép bạn lưu trữ nhiều phần tử cùng kiểu trong một biến theo thứ tự nhất định.

Khi làm việc với arrays, việc phải lặp qua chúng là rất phổ biến. Vì vậy, hãy dùng [kiến thức mới về `for`](iteration.md) để tạo một hàm `Sum`. Hàm `Sum` sẽ nhận một mảng số và trả về tổng của chúng.

Hãy áp dụng kỹ năng TDD của chúng ta!

## Viết test trước tiên

Tạo một thư mục mới để làm việc. Tạo một file mới có tên `sum_test.go` và thêm đoạn sau:

```go
package main

import "testing"

func TestSum(t *testing.T) {

	numbers := [5]int{1, 2, 3, 4, 5}

	got := Sum(numbers)
	want := 15

	if got != want {
		t.Errorf("got %d want %d given, %v", got, want, numbers)
	}
}
```

Arrays có _kích thước cố định_ mà bạn xác định khi khai báo biến. Chúng ta có thể khởi tạo array theo hai cách:

* \[N\]type{value1, value2, ..., valueN} ví dụ: `numbers := [5]int{1, 2, 3, 4, 5}`
* \[...\]type{value1, value2, ..., valueN} ví dụ: `numbers := [...]int{1, 2, 3, 4, 5}`

Đôi khi việc in giá trị đầu vào của hàm vào thông báo lỗi sẽ rất hữu ích. Ở đây, chúng ta dùng định dạng `%v` để in ra định dạng "mặc định", hoạt động tốt với arrays.

[Đọc thêm về format strings](https://golang.org/pkg/fmt/)

## Thử chạy test

Nếu bạn khởi tạo go mod với `go mod init main`, bạn sẽ gặp lỗi
`_testmain.go:13:2: cannot import "main"`. Đây là vì theo thông lệ, package main chỉ nên chứa tích hợp của các package khác, không phải code có thể unit test được. Do đó Go không cho phép import một package tên `main`.

Để sửa, hãy đổi tên main module trong `go.mod` sang bất kỳ tên nào khác.

Sau khi sửa lỗi trên, nếu bạn chạy `go test` thì compiler sẽ thất bại với lỗi quen thuộc `./sum_test.go:10:15: undefined: Sum`. Giờ chúng ta có thể tiến hành viết code thực sự để test.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Trong `sum.go`

```go
package main

func Sum(numbers [5]int) int {
	return 0
}
```

Test của bạn giờ sẽ thất bại với _thông báo lỗi rõ ràng_

`sum_test.go:13: got 0 want 15 given, [1 2 3 4 5]`

## Viết đủ code để test chạy thành công

```go
func Sum(numbers [5]int) int {
	sum := 0
	for i := 0; i < 5; i++ {
		sum += numbers[i]
	}
	return sum
}
```

Để lấy giá trị từ một array tại một vị trí cụ thể, dùng cú pháp `array[index]`. Ở đây, chúng ta dùng `for` để lặp 5 lần, tính tổng từng phần tử vào biến `sum`.

## Refactor

Hãy dùng [`range`](https://gobyexample.com/range) để làm sạch code hơn

```go
func Sum(numbers [5]int) int {
	sum := 0
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

`range` cho phép bạn lặp qua một array. Mỗi lần lặp, `range` trả về hai giá trị — chỉ số và giá trị. Chúng ta bỏ qua giá trị chỉ số bằng cách dùng `_` ([blank identifier](https://golang.org/doc/effective_go.html#blank)).

### Arrays và kiểu dữ liệu của chúng

Một tính chất thú vị của arrays là kích thước được mã hóa vào kiểu của nó. Nếu bạn cố truyền `[4]int` vào hàm nhận `[5]int`, nó sẽ không biên dịch được. Chúng là các kiểu khác nhau, giống như truyền `string` vào hàm nhận `int`.

Bạn có thể thấy arrays có kích thước cố định khá bất tiện và hầu hết thời gian bạn sẽ không dùng chúng!

Go có _slices_, không mã hóa kích thước của tập hợp và có thể có kích thước bất kỳ.

Yêu cầu tiếp theo là tính tổng các tập hợp có kích thước biến đổi.

## Viết test trước tiên

Chúng ta sẽ dùng [kiểu slice][slice] cho phép có các tập hợp với kích thước bất kỳ. Cú pháp rất giống array, chỉ cần bỏ qua kích thước khi khai báo.

`mySlice := []int{1,2,3}` thay vì `myArray := [3]int{1,2,3}`

```go
func TestSum(t *testing.T) {

	t.Run("collection of 5 numbers", func(t *testing.T) {
		numbers := [5]int{1, 2, 3, 4, 5}

		got := Sum(numbers)
		want := 15

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

	t.Run("collection of any size", func(t *testing.T) {
		numbers := []int{1, 2, 3}

		got := Sum(numbers)
		want := 6

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

}
```

## Thử chạy test

Đoạn code này không biên dịch được

`./sum_test.go:22:13: cannot use numbers (type []int) as type [5]int in argument to Sum`

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Vấn đề ở đây là chúng ta có thể:

* Phá vỡ API hiện tại bằng cách thay đổi đối số của `Sum` thành slice thay vì array. Khi làm vậy, test _khác_ của chúng ta sẽ không còn biên dịch được!
* Tạo một hàm mới

Trong trường hợp này, không ai khác đang dùng hàm của chúng ta, vì vậy thay vì duy trì hai hàm, hãy chỉ giữ một.

```go
func Sum(numbers []int) int {
	sum := 0
	for _, number := range numbers {
		sum += number
	}
	return sum
}
```

Nếu bạn thử chạy test, chúng vẫn chưa biên dịch được — bạn cần đổi test đầu tiên để truyền slice thay vì array.

## Viết đủ code để test chạy thành công

Hóa ra việc sửa lỗi biên dịch là tất cả những gì chúng ta cần và test đã pass!

## Refactor

Chúng ta đã refactor `Sum` — chỉ thay thế arrays bằng slices, không cần thay đổi gì thêm. Nhớ rằng chúng ta không được bỏ qua code test trong giai đoạn refactoring — hãy cải thiện thêm test `Sum`.

```go
func TestSum(t *testing.T) {

	t.Run("collection of 5 numbers", func(t *testing.T) {
		numbers := []int{1, 2, 3, 4, 5}

		got := Sum(numbers)
		want := 15

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

	t.Run("collection of any size", func(t *testing.T) {
		numbers := []int{1, 2, 3}

		got := Sum(numbers)
		want := 6

		if got != want {
			t.Errorf("got %d want %d given, %v", got, want, numbers)
		}
	})

}
```

Điều quan trọng cần đặt câu hỏi về giá trị của các test. Mục tiêu không phải là có càng nhiều test càng tốt, mà là có _sự tự tin_ càng nhiều càng tốt đối với codebase. Có quá nhiều test có thể là vấn đề thực sự và chỉ tốn thêm công bảo trì. **Mỗi test đều có chi phí**.

Trong trường hợp này, bạn có thể thấy rằng có hai test cho hàm này là dư thừa. Nếu nó hoạt động với slice kích thước một, rất có thể nó sẽ hoạt động với slice bất kỳ kích thước nào.

Bộ công cụ testing tích hợp của Go có [công cụ coverage](https://blog.golang.org/cover). Dù không nên coi 100% coverage là mục tiêu cuối, công cụ này có thể giúp xác định các phần code chưa được kiểm thử. Nếu bạn tuân thủ TDD, bạn sẽ gần đạt 100% coverage ngay từ đầu.

Thử chạy

`go test -cover`

Bạn sẽ thấy

```bash
PASS
coverage: 100.0% of statements
```

Bây giờ hãy xóa một trong các test và kiểm tra coverage lại.

Khi bạn đã hài lòng với hàm được kiểm thử kỹ càng, hãy commit thành quả của bạn trước khi tiếp tục thử thách tiếp theo.

Chúng ta cần một hàm mới gọi là `SumAll` nhận số lượng slice biến đổi, trả về một slice mới chứa tổng của mỗi slice được truyền vào.

Ví dụ:

`SumAll([]int{1,2}, []int{0,9})` sẽ trả về `[]int{3, 9}`

hoặc

`SumAll([]int{1,1,1})` sẽ trả về `[]int{3}`

## Viết test trước tiên

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := []int{3, 9}

	if got != want {
		t.Errorf("got %v want %v", got, want)
	}
}
```

## Thử chạy test

`./sum_test.go:23:9: undefined: SumAll`

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Chúng ta cần định nghĩa `SumAll` theo những gì test mong đợi.

Go cho phép bạn viết [_hàm variadic_](https://gobyexample.com/variadic-functions) nhận số lượng đối số biến đổi.

```go
func SumAll(numbersToSum ...[]int) []int {
	return nil
}
```

Đoạn code này hợp lệ, nhưng test vẫn không biên dịch!

`./sum_test.go:26:9: invalid operation: got != want (slice can only be compared to nil)`

Go không cho phép dùng toán tử bình đẳng với slices. Bạn _có thể_ viết hàm để lặp qua từng phần tử của `got` và `want` để kiểm tra giá trị, nhưng tiện hơn là dùng [`reflect.DeepEqual`][deepEqual] để xem hai biến có giống nhau không.

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := []int{3, 9}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

\(Nhớ `import reflect` ở đầu file để truy cập `DeepEqual`\)

Lưu ý quan trọng là `reflect.DeepEqual` không "type safe" — code vẫn biên dịch dù bạn làm gì đó sai. Để thấy điều này, hãy tạm thời đổi test thành:

```go
func TestSumAll(t *testing.T) {

	got := SumAll([]int{1, 2}, []int{0, 9})
	want := "bob"

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

Chúng ta đang cố so sánh một `slice` với một `string`. Điều này không có nghĩa gì, nhưng test vẫn biên dịch được! Vì vậy, dù `reflect.DeepEqual` là cách tiện lợi để so sánh slices, bạn cần cẩn thận khi dùng nó.

(Từ Go 1.21, package chuẩn [slices](https://pkg.go.dev/slices#pkg-overview) cung cấp hàm [slices.Equal](https://pkg.go.dev/slices#Equal) để shallow compare trên slices, không cần lo về kiểu như trường hợp trên. Lưu ý hàm này yêu cầu các phần tử phải [comparable](https://pkg.go.dev/builtin#comparable), do đó không áp dụng được cho slices chứa phần tử không so sánh được như 2D slices.)

Đổi test lại và chạy. Bạn sẽ thấy kết quả kiểu như:

`sum_test.go:30: got [] want [3 9]`

## Viết đủ code để test chạy thành công

Chúng ta cần lặp qua varargs, tính tổng bằng hàm `Sum` hiện có, rồi thêm vào slice sẽ trả về.

```go
func SumAll(numbersToSum ...[]int) []int {
	lengthOfNumbers := len(numbersToSum)
	sums := make([]int, lengthOfNumbers)

	for i, numbers := range numbersToSum {
		sums[i] = Sum(numbers)
	}

	return sums
}
```

Nhiều điều mới cần học!

Có một cách mới để tạo slice. `make` cho phép bạn tạo slice với dung lượng ban đầu bằng `len` của `numbersToSum`. Độ dài của slice là số phần tử nó đang giữ `len(mySlice)`, trong khi dung lượng là số phần tử nó có thể giữ trong mảng bên dưới `cap(mySlice)`, ví dụ: `make([]int, 0, 5)` tạo slice với độ dài 0 và dung lượng 5.

Bạn có thể truy cập phần tử của slices như arrays với `mySlice[N]` để lấy giá trị hoặc gán giá trị mới bằng `=`.

Test giờ sẽ pass.

## Refactor

Như đã đề cập, slices có dung lượng. Nếu bạn có slice với dung lượng 2 và cố làm `mySlice[10] = 1`, bạn sẽ gặp _lỗi runtime_.

Tuy nhiên, bạn có thể dùng hàm `append` nhận một slice và một giá trị mới, sau đó trả về slice mới với tất cả các phần tử.

```go
func SumAll(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		sums = append(sums, Sum(numbers))
	}

	return sums
}
```

Trong cách triển khai này, chúng ta ít lo hơn về dung lượng. Chúng ta bắt đầu với slice rỗng `sums` và append kết quả của `Sum` vào khi lặp qua varargs.

Yêu cầu tiếp theo là đổi `SumAll` thành `SumAllTails`, tính tổng của "tails" của mỗi slice. Tail của một tập hợp là tất cả các phần tử ngoại trừ phần tử đầu tiên \(the "head"\).

## Viết test trước tiên

```go
func TestSumAllTails(t *testing.T) {
	got := SumAllTails([]int{1, 2}, []int{0, 9})
	want := []int{2, 9}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}
```

## Thử chạy test

`./sum_test.go:26:9: undefined: SumAllTails`

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Đổi tên hàm thành `SumAllTails` và chạy lại test

`sum_test.go:30: got [3 9] want [2 9]`

## Viết đủ code để test chạy thành công

```go
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		tail := numbers[1:]
		sums = append(sums, Sum(tail))
	}

	return sums
}
```

Slices có thể được cắt! Cú pháp là `slice[low:high]`. Nếu bạn bỏ qua giá trị ở một bên của `:`, nó sẽ lấy tất cả về phía đó. Trong trường hợp này, chúng ta nói "lấy từ 1 đến cuối" với `numbers[1:]`. Bạn nên dành thời gian viết thêm test về slices và thử nghiệm toán tử slice để làm quen hơn.

## Refactor

Không có nhiều thứ cần refactor lần này.

Bạn nghĩ điều gì sẽ xảy ra nếu truyền một slice rỗng vào hàm? "Phần đuôi" của một slice rỗng là gì? Điều gì xảy ra khi bạn bảo Go lấy tất cả phần tử từ `myEmptySlice[1:]`?

## Viết test trước tiên

```go
func TestSumAllTails(t *testing.T) {

	t.Run("make the sums of some slices", func(t *testing.T) {
		got := SumAllTails([]int{1, 2}, []int{0, 9})
		want := []int{2, 9}

		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	})

	t.Run("safely sum empty slices", func(t *testing.T) {
		got := SumAllTails([]int{}, []int{3, 4, 5})
		want := []int{0, 9}

		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	})

}
```

## Thử chạy test

```text
panic: runtime error: slice bounds out of range [recovered]
    panic: runtime error: slice bounds out of range
```

Ôi không! Quan trọng là dù test _đã biên dịch_, nó _có lỗi runtime_.

Lỗi biên dịch là bạn của chúng ta vì chúng giúp chúng ta viết phần mềm hoạt động đúng; còn lỗi runtime là kẻ thù vì chúng ảnh hưởng đến người dùng.

## Viết đủ code để test chạy thành công

```go
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		if len(numbers) == 0 {
			sums = append(sums, 0)
		} else {
			tail := numbers[1:]
			sums = append(sums, Sum(tail))
		}
	}

	return sums
}
```

## Refactor

Các test của chúng ta có code lặp lại về assertions, vậy hãy tách chúng ra thành một hàm.

```go
func TestSumAllTails(t *testing.T) {

	checkSums := func(t testing.TB, got, want []int) {
		t.Helper()
		if !reflect.DeepEqual(got, want) {
			t.Errorf("got %v want %v", got, want)
		}
	}

	t.Run("make the sums of tails of", func(t *testing.T) {
		got := SumAllTails([]int{1, 2}, []int{0, 9})
		want := []int{2, 9}
		checkSums(t, got, want)
	})

	t.Run("safely sum empty slices", func(t *testing.T) {
		got := SumAllTails([]int{}, []int{3, 4, 5})
		want := []int{0, 9}
		checkSums(t, got, want)
	})

}
```

Chúng ta có thể tạo hàm mới `checkSums` như thông thường, nhưng ở đây chúng ta đang giới thiệu một kỹ thuật mới — gán function vào biến. Trông có vẻ lạ, nhưng không khác gì gán biến kiểu `string` hay `int`, vì về bản chất function cũng là giá trị.

Kỹ thuật này hữu ích khi bạn muốn gắn function với các biến cục bộ khác trong "scope" (ví dụ: trong một khối `{}`). Nó cũng giúp thu hẹp phạm vi API.

Bằng cách định nghĩa hàm này bên trong test, nó không thể được dùng bởi các hàm khác trong package. Việc ẩn các biến và hàm không cần export là một quyết định thiết kế quan trọng.

Một tác dụng phụ có lợi là điều này thêm một chút type-safety vào code. Nếu developer vô tình thêm test mới với `checkSums(t, got, "dave")`, compiler sẽ dừng họ lại.

```bash
$ go test
./sum_test.go:52:21: cannot use "dave" (type string) as type []int in argument to checkSums
```

## Tổng kết

Những gì chúng ta đã học:

* Arrays
* Slices
  * Các cách khác nhau để tạo chúng
  * Chúng có dung lượng _cố định_ nhưng bạn có thể tạo slice mới từ slice cũ bằng `append`
  * Cách cắt slices!
* `len` để lấy độ dài của array hoặc slice
* Công cụ kiểm tra code coverage
* `reflect.DeepEqual` và tại sao nó hữu ích nhưng có thể làm giảm type-safety của code

Chúng ta đã dùng slices và arrays với số nguyên, nhưng chúng hoạt động với bất kỳ kiểu nào khác, kể cả arrays/slices chính nó. Vì vậy bạn có thể khai báo biến kiểu `[][]string` nếu cần.

[Xem bài viết trên blog Go về slices][blog-slice] để tìm hiểu sâu hơn. Hãy thử viết thêm test để củng cố những gì bạn học từ bài viết đó.

Một cách khác để thử nghiệm với Go ngoài việc viết test là Go playground. Bạn có thể thử hầu hết mọi thứ và chia sẻ code dễ dàng khi cần hỏi. [Tôi đã tạo một go playground với slice để bạn thử nghiệm.](https://play.golang.org/p/ICCWcRGIO68)

[Đây là một ví dụ](https://play.golang.org/p/bTrRmYfNYCp) về cách cắt array và cách thay đổi slice ảnh hưởng đến array gốc; nhưng "bản sao" của slice sẽ không ảnh hưởng đến array gốc. [Một ví dụ khác](https://play.golang.org/p/Poth8JS28sc) về lý do tại sao nên tạo bản sao của một slice sau khi cắt từ slice rất lớn.

[for]: ../iteration.md#
[blog-slice]: https://blog.golang.org/go-slices-usage-and-internals
[deepEqual]: https://golang.org/pkg/reflect/#DeepEqual
[slice]: https://golang.org/doc/effective_go.html#slices
