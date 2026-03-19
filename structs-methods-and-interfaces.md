# Structs, methods & interfaces

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/structs)**

Giả sử chúng ta cần một số code hình học để tính chu vi của hình chữ nhật với chiều cao và chiều rộng cho trước. Chúng ta có thể viết hàm `Perimeter(width float64, height float64)`, trong đó `float64` dành cho các số dấu phẩy động như `123.45`.

Chu trình TDD giờ chắc hẳn đã quen thuộc với bạn.

## Viết test trước tiên

```go
func TestPerimeter(t *testing.T) {
	got := Perimeter(10.0, 10.0)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

Chú ý format string mới? Chữ `f` là cho `float64` và `.2` có nghĩa là in 2 chữ số thập phân.

## Thử chạy test

`./shapes_test.go:6:9: undefined: Perimeter`

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func Perimeter(width float64, height float64) float64 {
	return 0
}
```

Kết quả: `shapes_test.go:10: got 0.00 want 40.00`.

## Viết đủ code để test chạy thành công

```go
func Perimeter(width float64, height float64) float64 {
	return 2 * (width + height)
}
```

Đến đây, khá đơn giản. Giờ hãy tạo hàm `Area(width, height float64)` trả về diện tích hình chữ nhật.

Hãy tự thực hiện theo chu trình TDD.

Cuối cùng bạn sẽ có test như sau:

```go
func TestPerimeter(t *testing.T) {
	got := Perimeter(10.0, 10.0)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}

func TestArea(t *testing.T) {
	got := Area(12.0, 6.0)
	want := 72.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

Và code như sau:

```go
func Perimeter(width float64, height float64) float64 {
	return 2 * (width + height)
}

func Area(width float64, height float64) float64 {
	return width * height
}
```

## Refactor

Code của chúng ta đúng, nhưng không có gì rõ ràng về hình chữ nhật. Một developer thiếu cẩn thận có thể truyền chiều rộng và chiều cao của tam giác vào các hàm này mà không nhận ra sẽ nhận được kết quả sai.

Chúng ta có thể đặt tên hàm cụ thể hơn như `RectangleArea`. Nhưng giải pháp gọn hơn là định nghĩa _kiểu_ riêng tên `Rectangle` để đóng gói khái niệm cho chúng ta.

Chúng ta có thể tạo kiểu đơn giản bằng **struct**. [Struct](https://golang.org/ref/spec#Struct_types) là một tập hợp các trường có tên để lưu dữ liệu.

Khai báo struct như sau:

```go
type Rectangle struct {
	Width  float64
	Height float64
}
```

Giờ hãy refactor test để dùng `Rectangle` thay vì `float64` thuần túy.

```go
func TestPerimeter(t *testing.T) {
	rectangle := Rectangle{10.0, 10.0}
	got := Perimeter(rectangle)
	want := 40.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}

func TestArea(t *testing.T) {
	rectangle := Rectangle{12.0, 6.0}
	got := Area(rectangle)
	want := 72.0

	if got != want {
		t.Errorf("got %.2f want %.2f", got, want)
	}
}
```

Nhớ chạy test trước khi cố sửa. Test sẽ hiển thị lỗi hữu ích như:

```text
./shapes_test.go:7:18: not enough arguments in call to Perimeter
    have (Rectangle)
    want (float64, float64)
```

Bạn có thể truy cập các trường của struct với cú pháp `myStruct.field`.

Sửa hai hàm để test pass:

```go
func Perimeter(rectangle Rectangle) float64 {
	return 2 * (rectangle.Width + rectangle.Height)
}

func Area(rectangle Rectangle) float64 {
	return rectangle.Width * rectangle.Height
}
```

Hy vọng bạn đồng ý rằng truyền `Rectangle` vào hàm thể hiện ý định rõ ràng hơn, nhưng structs còn có nhiều lợi ích khác sẽ được đề cập sau.

Yêu cầu tiếp theo là viết hàm `Area` cho hình tròn.

## Viết test trước tiên

```go
func TestArea(t *testing.T) {

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		got := Area(rectangle)
		want := 72.0

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		got := Area(circle)
		want := 314.1592653589793

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

}
```

Như bạn thấy, `f` đã được thay bằng `g`, và có lý do. Dùng `g` sẽ in số thập phân chính xác hơn trong thông báo lỗi \([fmt options](https://golang.org/pkg/fmt/)\). Ví dụ, với bán kính 1.5 trong tính diện tích hình tròn, `f` sẽ hiển thị `7.068583` trong khi `g` sẽ hiển thị `7.0685834705770345`.

## Thử chạy test

`./shapes_test.go:28:13: undefined: Circle`

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Chúng ta cần định nghĩa kiểu `Circle`.

```go
type Circle struct {
	Radius float64
}
```

Bây giờ thử chạy test lại

`./shapes_test.go:29:14: cannot use circle (type Circle) as type Rectangle in argument to Area`

Một số ngôn ngữ lập trình cho phép làm như sau:

```go
func Area(circle Circle) float64       {}
func Area(rectangle Rectangle) float64 {}
```

Nhưng trong Go thì không thể

`./shapes.go:20:32: Area redeclared in this block`

Chúng ta có hai lựa chọn:

* Có thể khai báo các hàm cùng tên trong các _package_ khác nhau. Vì vậy chúng ta có thể tạo `Area(Circle)` trong một package mới, nhưng điều đó có vẻ quá mức ở đây.
* Chúng ta có thể định nghĩa [_methods_](https://golang.org/ref/spec#Method_declarations) trên các kiểu mới định nghĩa.

### Method là gì?

Cho đến nay chúng ta chỉ viết _functions_ nhưng cũng đã dùng một số methods. Khi gọi `t.Errorf`, chúng ta đang gọi method `Errorf` trên instance `t` \(`testing.T`\).

Method là function có receiver. Khai báo method gắn một tên với method đó và liên kết method với base type của receiver.

Method rất giống functions nhưng được gọi bằng cách invoke trên instance của một kiểu cụ thể. Trong khi bạn có thể gọi functions ở bất kỳ đâu như `Area(rectangle)`, bạn chỉ có thể gọi methods trên "đối tượng".

Hãy đổi test để gọi methods thay vì functions rồi sửa code.

```go
func TestArea(t *testing.T) {

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		got := rectangle.Area()
		want := 72.0

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		got := circle.Area()
		want := 314.1592653589793

		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	})

}
```

Nếu thử chạy test, chúng ta sẽ nhận được:

```text
./shapes_test.go:19:19: rectangle.Area undefined (type Rectangle has no field or method Area)
./shapes_test.go:29:16: circle.Area undefined (type Circle has no field or method Area)
```

> type Circle has no field or method Area

Điều quan trọng là phải dành thời gian đọc kỹ thông báo lỗi. Làm quen điều này sẽ giúp ích cho bạn lâu dài.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Hãy thêm một số methods vào các kiểu của chúng ta:

```go
type Rectangle struct {
	Width  float64
	Height float64
}

func (r Rectangle) Area() float64 {
	return 0
}

type Circle struct {
	Radius float64
}

func (c Circle) Area() float64 {
	return 0
}
```

Cú pháp khai báo methods gần giống với functions vì chúng rất giống nhau. Sự khác biệt duy nhất là cú pháp của method receiver `func (receiverName ReceiverType) MethodName(args)`.

Khi method được gọi trên biến của kiểu đó, bạn lấy tham chiếu đến dữ liệu của nó thông qua biến `receiverName`. Trong nhiều ngôn ngữ lập trình khác, điều này được thực hiện ngầm định và bạn truy cập receiver qua `this`.

Theo quy ước trong Go, biến receiver là chữ cái đầu tiên của kiểu.

```
r Rectangle
```

Nếu thử chạy lại test, chúng sẽ biên dịch được và cho một số kết quả thất bại.

## Viết đủ code để test chạy thành công

Bây giờ hãy làm cho test rectangle pass bằng cách sửa method mới của chúng ta:

```go
func (r Rectangle) Area() float64 {
	return r.Width * r.Height
}
```

Nếu chạy lại test, test rectangle sẽ pass nhưng circle vẫn fail.

Để làm cho hàm `Area` của circle pass, chúng ta sẽ dùng hằng số `Pi` từ package `math` \(nhớ import nó\).

```go
func (c Circle) Area() float64 {
	return math.Pi * c.Radius * c.Radius
}
```

## Refactor

Có một số sự trùng lặp trong các test của chúng ta.

Tất cả những gì chúng ta muốn làm là lấy một tập hợp các _shapes_, gọi method `Area()` trên chúng rồi kiểm tra kết quả.

Chúng ta muốn có thể viết hàm `checkArea` nhận cả `Rectangle` và `Circle`, nhưng thất bại biên dịch nếu chúng ta cố truyền thứ gì đó không phải là shape.

Với Go, chúng ta có thể mã hóa ý định này bằng **interfaces**.

[Interfaces](https://golang.org/ref/spec#Interface_types) là một khái niệm rất mạnh trong các ngôn ngữ kiểu tĩnh như Go vì chúng cho phép tạo các functions dùng được với nhiều kiểu khác nhau và tạo ra code decoupled cao trong khi vẫn duy trì type-safety.

Hãy giới thiệu điều này bằng cách refactor test:

```go
func TestArea(t *testing.T) {

	checkArea := func(t testing.TB, shape Shape, want float64) {
		t.Helper()
		got := shape.Area()
		if got != want {
			t.Errorf("got %g want %g", got, want)
		}
	}

	t.Run("rectangles", func(t *testing.T) {
		rectangle := Rectangle{12, 6}
		checkArea(t, rectangle, 72.0)
	})

	t.Run("circles", func(t *testing.T) {
		circle := Circle{10}
		checkArea(t, circle, 314.1592653589793)
	})

}
```

Chúng ta đang tạo hàm helper như trong các bài tập khác, nhưng lần này yêu cầu `Shape` được truyền vào. Nếu gọi với thứ gì đó không phải shape, nó sẽ không biên dịch được.

Làm thế nào để một thứ trở thành shape? Chúng ta chỉ cần nói với Go `Shape` là gì bằng khai báo interface:

```go
type Shape interface {
	Area() float64
}
```

Chúng ta đang tạo một `type` mới giống như đã làm với `Rectangle` và `Circle`, nhưng lần này là `interface` thay vì `struct`.

Sau khi thêm vào code, test sẽ pass.

### Chờ đã, sao vậy?

Điều này khá khác với interfaces trong hầu hết các ngôn ngữ lập trình khác. Thông thường bạn phải viết code để nói `My type Foo implements interface Bar`.

Nhưng trong trường hợp này:

* `Rectangle` có method tên `Area` trả về `float64` nên nó thỏa mãn interface `Shape`
* `Circle` có method tên `Area` trả về `float64` nên nó thỏa mãn interface `Shape`
* `string` không có method như vậy nên nó không thỏa mãn interface
* v.v.

Trong Go **interface resolution là implicit**. Nếu kiểu bạn truyền vào khớp với những gì interface yêu cầu, nó sẽ biên dịch được.

### Decoupling (Tách rời)

Chú ý cách helper của chúng ta không cần quan tâm đến việc shape là `Rectangle`, `Circle` hay `Triangle`. Bằng cách khai báo interface, helper được _decoupled_ khỏi các kiểu cụ thể và chỉ có method cần thiết để thực hiện công việc.

Cách tiếp cận dùng interfaces để khai báo **chỉ những gì bạn cần** rất quan trọng trong thiết kế phần mềm và sẽ được đề cập chi tiết hơn trong các phần sau.

## Refactor thêm

Giờ bạn đã hiểu về structs, hãy giới thiệu "table driven tests".

[Table driven tests](https://go.dev/wiki/TableDrivenTests) hữu ích khi bạn muốn xây dựng danh sách các test case có thể được kiểm thử theo cùng một cách.

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		shape Shape
		want  float64
	}{
		{Rectangle{12, 6}, 72.0},
		{Circle{10}, 314.1592653589793},
	}

	for _, tt := range areaTests {
		got := tt.shape.Area()
		if got != tt.want {
			t.Errorf("got %g want %g", got, tt.want)
		}
	}

}
```

Cú pháp mới duy nhất ở đây là tạo "anonymous struct", `areaTests`. Chúng ta khai báo slice of structs bằng `[]struct` với hai trường, `shape` và `want`. Sau đó điền vào slice với các test case.

Chúng ta lặp qua chúng như bất kỳ slice nào, dùng các trường struct để chạy test.

Bạn có thể thấy sẽ rất dễ dàng cho developer thêm shape mới, implement `Area` rồi thêm vào test case. Thêm vào đó, nếu bug được tìm thấy với `Area`, rất dễ thêm test case mới để tái tạo bug trước khi sửa.

Table driven tests có thể là công cụ tuyệt vời, nhưng hãy chắc rằng bạn thực sự cần chúng. Chúng rất phù hợp khi bạn muốn kiểm thử nhiều cách implement của một interface, hoặc khi data truyền vào hàm có nhiều yêu cầu khác nhau cần kiểm thử.

Hãy minh hoạ tất cả điều này bằng cách thêm shape mới và test nó: tam giác.

## Viết test trước tiên

Thêm test mới cho shape mới rất dễ. Chỉ cần thêm `{Triangle{12, 6}, 36.0},` vào danh sách.

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		shape Shape
		want  float64
	}{
		{Rectangle{12, 6}, 72.0},
		{Circle{10}, 314.1592653589793},
		{Triangle{12, 6}, 36.0},
	}

	for _, tt := range areaTests {
		got := tt.shape.Area()
		if got != tt.want {
			t.Errorf("got %g want %g", got, tt.want)
		}
	}

}
```

## Thử chạy test

Nhớ, hãy tiếp tục thử chạy test và để compiler dẫn dắt bạn đến giải pháp.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

`./shapes_test.go:25:4: undefined: Triangle`

Chúng ta chưa định nghĩa `Triangle`

```go
type Triangle struct {
	Base   float64
	Height float64
}
```

Thử lại

```text
./shapes_test.go:25:8: cannot use Triangle literal (type Triangle) as type Shape in field value:
    Triangle does not implement Shape (missing Area method)
```

Nó cho chúng ta biết không thể dùng `Triangle` như shape vì nó không có method `Area()`, vì vậy hãy thêm implementation rỗng để chạy được test:

```go
func (t Triangle) Area() float64 {
	return 0
}
```

Cuối cùng code biên dịch được và chúng ta nhận lỗi:

`shapes_test.go:31: got 0.00 want 36.00`

## Viết đủ code để test chạy thành công

```go
func (t Triangle) Area() float64 {
	return (t.Base * t.Height) * 0.5
}
```

Và test của chúng ta pass!

## Refactor

Một lần nữa, implementation ổn nhưng test của chúng ta có thể cải thiện.

Khi bạn nhìn vào:

```
{Rectangle{12, 6}, 72.0},
{Circle{10}, 314.1592653589793},
{Triangle{12, 6}, 36.0},
```

Không rõ ngay các số đại diện cho gì và bạn nên hướng tới test dễ hiểu.

Cho đến nay bạn chỉ được thấy cú pháp tạo struct `MyStruct{val1, val2}` nhưng bạn có thể đặt tên các trường một cách tùy chọn.

Hãy xem kết quả:

```
        {shape: Rectangle{Width: 12, Height: 6}, want: 72.0},
        {shape: Circle{Radius: 10}, want: 314.1592653589793},
        {shape: Triangle{Base: 12, Height: 6}, want: 36.0},
```

Trong [Test-Driven Development by Example](https://g.co/kgs/yCzDLF), Kent Beck refactor một số test đến một điểm và khẳng định:

> Test nói với chúng ta rõ ràng hơn, như thể là một khẳng định về sự thật, **không phải một chuỗi các thao tác**

\(phần nhấn mạnh trong trích dẫn là của tôi\)

Giờ test của chúng ta — đúng hơn là danh sách test case — tạo ra các khẳng định về sự thật về shapes và diện tích của chúng.

## Đảm bảo output test của bạn hữu ích

Nhớ lại khi chúng ta implement `Triangle` và có test fail? Nó in `shapes_test.go:31: got 0.00 want 36.00`.

Chúng ta biết đây liên quan đến `Triangle` vì chúng ta vừa làm việc với nó. Nhưng nếu bug xảy ra trong một trong 20 test case của bảng thì sao? Developer sẽ phải xem qua thủ công để tìm case thực sự fail.

Chúng ta có thể đổi thông báo lỗi thành `%#v got %g want %g`. Chuỗi format `%#v` sẽ in ra struct với các giá trị trong trường, giúp developer có thể khi nhìn vào thấy ngay các thuộc tính đang được test.

Để tăng tính đọc được của test case, chúng ta có thể đổi tên trường `want` thành cái gì mô tả hơn như `hasArea`.

Một mẹo cuối cùng với table driven tests là dùng `t.Run` và đặt tên cho test case.

Bằng cách bọc mỗi case trong `t.Run`, bạn sẽ có output test rõ ràng hơn khi fail vì nó sẽ in tên của case:

```text
--- FAIL: TestArea (0.00s)
    --- FAIL: TestArea/Rectangle (0.00s)
        shapes_test.go:33: main.Rectangle{Width:12, Height:6} got 72.00 want 72.10
```

Và bạn có thể chạy các test cụ thể trong bảng với `go test -run TestArea/Rectangle`.

Đây là code test cuối cùng:

```go
func TestArea(t *testing.T) {

	areaTests := []struct {
		name    string
		shape   Shape
		hasArea float64
	}{
		{name: "Rectangle", shape: Rectangle{Width: 12, Height: 6}, hasArea: 72.0},
		{name: "Circle", shape: Circle{Radius: 10}, hasArea: 314.1592653589793},
		{name: "Triangle", shape: Triangle{Base: 12, Height: 6}, hasArea: 36.0},
	}

	for _, tt := range areaTests {
		// using tt.name from the case to use it as the `t.Run` test name
		t.Run(tt.name, func(t *testing.T) {
			got := tt.shape.Area()
			if got != tt.hasArea {
				t.Errorf("%#v got %g want %g", tt.shape, got, tt.hasArea)
			}
		})

	}

}
```

## Tổng kết

Đây là nhiều thực hành TDD hơn, lặp qua các giải pháp cho các bài toán toán học cơ bản và học các tính năng ngôn ngữ mới được thúc đẩy bởi test.

* Khai báo structs để tạo kiểu dữ liệu riêng giúp nhóm dữ liệu liên quan lại và làm rõ ý định code
* Khai báo interfaces để định nghĩa functions có thể dùng với nhiều kiểu khác nhau \([parametric polymorphism](https://en.wikipedia.org/wiki/Parametric_polymorphism)\)
* Thêm methods để thêm chức năng vào kiểu dữ liệu và implement interfaces
* Table driven tests để làm assertions rõ ràng hơn và dễ mở rộng & bảo trì hơn

Đây là chương quan trọng vì chúng ta bắt đầu định nghĩa các kiểu của riêng mình. Trong các ngôn ngữ kiểu tĩnh như Go, khả năng thiết kế kiểu riêng là điều cần thiết để xây dựng phần mềm dễ hiểu, dễ ghép nối và dễ test.

Interfaces là công cụ tuyệt vời để ẩn sự phức tạp khỏi các phần khác của hệ thống. Trong trường hợp của chúng ta, code helper _test_ không cần biết chính xác shape nó đang kiểm tra, chỉ cần biết cách "hỏi" diện tích của nó.

Khi bạn quen hơn với Go, bạn sẽ thấy sức mạnh thực sự của interfaces và thư viện chuẩn. Bạn sẽ học về interfaces được định nghĩa trong thư viện chuẩn được dùng _ở khắp nơi_ và bằng cách implement chúng cho các kiểu của mình, bạn có thể nhanh chóng tái sử dụng rất nhiều chức năng tuyệt vời.
