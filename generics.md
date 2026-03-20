# Generics

**[Tất cả mã nguồn của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/generics)**

Chương này sẽ cung cấp cho bạn phần giới thiệu về generics, xua tan những e ngại mà bạn có thể có về chúng, và cho bạn ý tưởng về cách đơn giản hóa một số mã nguồn của mình trong tương lai. Sau khi đọc chương này, bạn sẽ biết cách viết:

- Một hàm nhận các đối số generic (tham số hóa kiểu)
- Một cấu trúc dữ liệu generic


## Các hàm hỗ trợ kiểm thử của riêng chúng ta (`AssertEqual`, `AssertNotEqual`)

Để khám phá generics, chúng ta sẽ viết một số hàm hỗ trợ kiểm thử.

### Kiểm tra (Assert) trên các số nguyên

Hãy bắt đầu với một cái gì đó cơ bản và lặp lại theo mục tiêu của chúng ta

```go
import "testing"

func TestAssertFunctions(t *testing.T) {
	t.Run("asserting on integers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})
}

func AssertEqual(t *testing.T, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("got %d, want %d", got, want)
	}
}

func AssertNotEqual(t *testing.T, got, want int) {
	t.Helper()
	if got == want {
		t.Errorf("didn't want %d", got)
	}
}
```


### Kiểm tra trên các chuỗi

Có thể kiểm tra sự bằng nhau của các số nguyên là điều tuyệt vời nhưng nếu chúng ta muốn kiểm tra trên `string thì sao?

```go
t.Run("asserting on strings", func(t *testing.T) {
	AssertEqual(t, "hello", "hello")
	AssertNotEqual(t, "hello", "Grace")
})
```

Bạn sẽ nhận được một lỗi

```
# github.com/quii/learn-go-with-tests/generics [github.com/quii/learn-go-with-tests/generics.test]
./generics_test.go:12:18: cannot use "hello" (untyped string constant) as int value in argument to AssertEqual
./generics_test.go:13:21: cannot use "hello" (untyped string constant) as int value in argument to AssertNotEqual
./generics_test.go:13:30: cannot use "Grace" (untyped string constant) as int value in argument to AssertNotEqual
```

Nếu bạn dành thời gian để đọc lỗi, bạn sẽ thấy trình biên dịch đang phàn nàn rằng chúng ta đang cố gắng truyền một `string` vào một hàm mong đợi một số nguyên (`integer`).

#### Nhắc lại về tính an toàn kiểu (type-safety)

Nếu bạn đã đọc các chương trước của cuốn sách này, hoặc có kinh nghiệm với các ngôn ngữ định kiểu tĩnh, điều này sẽ không làm bạn ngạc nhiên. Trình biên dịch Go mong đợi bạn viết các hàm, struct, v.v. bằng cách mô tả các kiểu dữ liệu bạn muốn làm việc cùng.

Bạn không thể truyền một `string` vào một hàm mong đợi một số nguyên.

Mặc dù điều này có vẻ như là sự rườm rà, nhưng nó có thể cực kỳ hữu ích. Bằng cách mô tả các ràng buộc này, bạn:

- Làm cho việc triển khai hàm trở nên đơn giản hơn. Bằng cách mô tả cho trình biên dịch những kiểu dữ liệu bạn làm việc cùng, bạn **ràng buộc số lượng các triển khai hợp lệ có thể có**. Bạn không thể "cộng" một `Person` và một `BankAccount`. Bạn không thể viết hoa một số nguyên. Trong phần mềm, các ràng buộc thường cực kỳ hữu ích.
- Ngăn chặn việc vô tình truyền dữ liệu vào một hàm mà bạn không có ý định.

Go cung cấp cho bạn một cách để trừu tượng hóa hơn với các kiểu dữ liệu của mình bằng [interface](./structs-methods-and-interfaces.md), để bạn có thể thiết kế các hàm không nhận các kiểu cụ thể (concrete types) mà thay vào đó là các kiểu cung cấp hành vi bạn cần. Điều này mang lại cho bạn sự linh hoạt trong khi vẫn duy trì tính an toàn kiểu.

### Một hàm nhận một chuỗi hoặc một số nguyên? (hoặc thực sự là những thứ khác)

Một tùy chọn khác mà Go có để làm cho các hàm của bạn linh hoạt hơn là khai báo kiểu của đối số là `interface{}`, có nghĩa là "bất cứ thứ gì".

Hãy thử thay đổi các chữ ký hàm để sử dụng kiểu này thay thế.

```go
func AssertEqual(got, want interface{})

func AssertNotEqual(got, want interface{})

```

Các bản kiểm thử bây giờ sẽ biên dịch và vượt qua. Nếu bạn thử làm cho chúng thất bại, bạn sẽ thấy đầu ra hơi kém vì chúng ta đang sử dụng chuỗi định dạng số nguyên `%d` để in các thông báo của mình, vì vậy hãy đổi chúng thành định dạng chung `%+v` để có đầu ra tốt hơn cho bất kỳ loại giá trị nào.

### Vấn đề với `interface{}`

Các hàm `AssertX` của chúng ta khá ngây thơ nhưng về mặt khái niệm thì không quá khác biệt so với cách các [thư viện phổ biến khác cung cấp chức năng này](https://github.com/matryer/is/blob/master/is.go#L150)

```go
func (is *I) Equal(a, b interface{})
```

Vậy vấn đề là gì?

Bằng cách sử dụng `interface{}`, trình biên dịch không thể giúp chúng ta khi viết mã nguồn, vì chúng ta không nói cho nó bất cứ điều gì hữu ích về kiểu của những thứ được truyền vào hàm. Hãy thử so sánh hai kiểu khác nhau.

```go
AssertEqual(1, "1")
```

Trong trường hợp này, chúng ta thoát được; bản kiểm thử biên dịch, và nó thất bại như chúng ta mong đợi, mặc dù thông báo lỗi `got 1, want 1` không rõ ràng; nhưng chúng ta có thực sự muốn có thể so sánh chuỗi với số nguyên không? Thế còn việc so sánh một `Person` với một `Airport`?

Viết các hàm nhận `interface{}` có thể cực kỳ thách thức và dễ xảy ra lỗi vì chúng ta đã *mất đi* các ràng buộc của mình, và chúng ta không có thông tin tại thời điểm biên dịch về loại dữ liệu mà chúng ta đang xử lý.

Điều này có nghĩa là **trình biên dịch không thể giúp chúng ta** và thay vào đó chúng ta có nhiều khả năng gặp phải các **lỗi runtime** có thể ảnh hưởng đến người dùng, gây ra sự cố, hoặc tệ hơn.

Thường thì các nhà phát triển phải sử dụng reflection để triển khai các hàm *hừm* generic này, việc này có thể trở nên phức tạp để đọc và viết, và có thể làm giảm hiệu năng của chương trình.

## Các hàm hỗ trợ kiểm thử của riêng chúng ta với generics

Lý tưởng nhất là chúng ta không muốn phải tạo các hàm `AssertX` cụ thể cho mọi kiểu dữ liệu mà chúng ta từng xử lý. Chúng ta muốn có thể có *một* hàm `AssertEqual` hoạt động với *bất kỳ* kiểu dữ liệu nào nhưng không cho phép bạn so sánh [râu ông nọ chắp cằm bà kia (apples and oranges)](https://en.wikipedia.org/wiki/Apples_and_oranges).

Generics cung cấp cho chúng ta một cách để tạo ra các trừu tượng (như interface) bằng cách để chúng ta **mô tả các ràng buộc của mình**. Chúng cho phép chúng ta viết các hàm có mức độ linh hoạt tương tự như `interface{}` nhưng vẫn giữ được tính an toàn kiểu và mang lại trải nghiệm phát triển tốt hơn cho người gọi.

```go
func TestAssertFunctions(t *testing.T) {
	t.Run("asserting on integers", func(t *testing.T) {
		AssertEqual(t, 1, 1)
		AssertNotEqual(t, 1, 2)
	})

	t.Run("asserting on strings", func(t *testing.T) {
		AssertEqual(t, "hello", "hello")
		AssertNotEqual(t, "hello", "Grace")
	})

	// AssertEqual(t, 1, "1") // bỏ chú thích để thấy lỗi
}

func AssertEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
}

func AssertNotEqual[T comparable](t *testing.T, got, want T) {
	t.Helper()
	if got == want {
		t.Errorf("didn't want %v", got)
	}
}
```

Để viết các hàm generic trong Go, bạn cần cung cấp các "tham số kiểu" (type parameters), đây chỉ là một cách nói mỹ miều của việc "mô tả kiểu generic của bạn và đặt cho nó một cái nhãn".

Trong trường hợp của chúng ta, kiểu của tham số kiểu là `comparable` và chúng ta đã đặt cho nó cái nhãn là `T`. Cái nhãn này sau đó cho phép chúng ta mô tả các kiểu cho các đối số của hàm (`got, want T`).

Chúng ta đang sử dụng `comparable` vì chúng ta muốn mô tả cho trình biên dịch rằng chúng ta muốn sử dụng các toán tử `==` và `!=` trên các thứ thuộc kiểu `T` trong hàm của mình, chúng ta muốn so sánh! Nếu bạn thử thay đổi kiểu thành `any`,

```go
func AssertNotEqual[T any](got, want T)
```

Bạn sẽ nhận được lỗi sau:

```
prog.go2:15:5: cannot compare got != want (operator != not defined for T)
```

Điều này rất có ý nghĩa, vì bạn không thể sử dụng các toán tử đó trên mọi (hoặc `any`) kiểu dữ liệu.

### Có phải một hàm generic với [`T any`](https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md#the-constraint) cũng giống như `interface{}`?

Hãy xem xét hai hàm

```go
func GenericFoo[T any](x, y T)
```

```go
func InterfaceyFoo(x, y interface{})
```

Điểm khác biệt của generics ở đây là gì? Chẳng phải `any` mô tả... bất cứ thứ gì sao?

Về mặt ràng buộc, `any` có nghĩa là "bất cứ thứ gì" và `interface{}` cũng vậy. Thực tế, `any` đã được thêm vào Go 1.18 và nó *chỉ là một bí danh cho `interface{}`*.

Sự khác biệt với phiên bản generic là *bạn vẫn đang mô tả một kiểu cụ thể* và điều đó có nghĩa là chúng ta vẫn ràng buộc hàm này chỉ hoạt động với *một* kiểu dữ liệu duy nhất.

Điều này có nghĩa là bạn có thể gọi `InterfaceyFoo` với bất kỳ sự kết hợp kiểu nào (ví dụ: `InterfaceyFoo(apple, orange)`). Tuy nhiên, `GenericFoo` vẫn cung cấp một số ràng buộc vì chúng ta đã nói rằng nó chỉ hoạt động với *một* kiểu dữ liệu, `T`.

Hợp lệ:

- `GenericFoo(apple1, apple2)`
- `GenericFoo(orange1, orange2)`
- `GenericFoo(1, 2)`
- `GenericFoo("one", "two")`

Không hợp lệ (lỗi khi biên dịch):

- `GenericFoo(apple1, orange1)`
- `GenericFoo("1", 1)`

Nếu hàm của bạn trả về kiểu generic, người gọi cũng có thể sử dụng kiểu đó như vốn có, thay vì phải thực hiện khẳng định kiểu (type assertion) vì khi một hàm trả về `interface{}`, trình biên dịch không thể đảm bảo bất cứ điều gì về kiểu dữ liệu đó.

## Tiếp theo: Các kiểu dữ liệu Generic

Chúng ta sẽ tạo một kiểu dữ liệu [stack](https://en.wikipedia.org/wiki/Stack_(abstract_data_type)). Stack nên khá dễ hiểu từ quan điểm yêu cầu. Chúng là một tập hợp các mục mà bạn có thể `Push` (đẩy) các mục vào "đỉnh" và để lấy lại các mục, bạn `Pop` (lấy) các mục từ đỉnh (LIFO - vào sau, ra trước).

Vì mục đích ngắn gọn, tôi đã lược bỏ quy trình TDD dẫn tôi đến mã nguồn sau cho một stack chứa các số nguyên (`int`), và một stack chứa các chuỗi (`string`).

```go
type StackOfInts struct {
	values []int
}

func (s *StackOfInts) Push(value int) {
	s.values = append(s.values, value)
}

func (s *StackOfInts) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfInts) Pop() (int, bool) {
	if s.IsEmpty() {
		return 0, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}

type StackOfStrings struct {
	values []string
}

func (s *StackOfStrings) Push(value string) {
	s.values = append(s.values, value)
}

func (s *StackOfStrings) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *StackOfStrings) Pop() (string, bool) {
	if s.IsEmpty() {
		return "", false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

Tôi đã tạo một vài hàm kiểm tra khác để hỗ trợ

```go
func AssertTrue(t *testing.T, got bool) {
	t.Helper()
	if !got {
		t.Errorf("got %v, want true", got)
	}
}

func AssertFalse(t *testing.T, got bool) {
	t.Helper()
	if got {
		t.Errorf("got %v, want false", got)
	}
}
```

Và đây là các bản kiểm thử

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := new(StackOfInts)

		// kiểm tra stack đang trống
		AssertTrue(t, myStackOfInts.IsEmpty())

		// thêm một thứ, sau đó kiểm tra nó không trống
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// thêm một thứ khác, lấy nó ra lại
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())
	})

	t.Run("string stack", func(t *testing.T) {
		myStackOfStrings := new(StackOfStrings)

		// kiểm tra stack đang trống
		AssertTrue(t, myStackOfStrings.IsEmpty())

		// thêm một thứ, sau đó kiểm tra nó không trống
		myStackOfStrings.Push("123")
		AssertFalse(t, myStackOfStrings.IsEmpty())

		// thêm một thứ khác, lấy nó ra lại
		myStackOfStrings.Push("456")
		value, _ := myStackOfStrings.Pop()
		AssertEqual(t, value, "456")
		value, _ = myStackOfStrings.Pop()
		AssertEqual(t, value, "123")
		AssertTrue(t, myStackOfStrings.IsEmpty())
	})
}
```

### Vấn đề

- Mã nguồn cho cả `StackOfStrings` và `StackOfInts` gần như giống hệt nhau. Mặc dù sự trùng lặp không phải lúc nào cũng là dấu chấm hết cho thế giới, nhưng nó có nghĩa là có nhiều mã nguồn hơn để đọc, viết và bảo trì.
- Vì chúng ta đang trùng lặp logic trên hai kiểu dữ liệu, chúng ta cũng phải trùng lặp cả các bản kiểm thử.

Chúng ta thực sự muốn tóm gọn *ý tưởng* về một stack trong một kiểu dữ liệu duy nhất, và có một bộ kiểm thử duy nhất cho chúng. Chúng ta nên đội chiếc mũ tái cấu trúc ngay bây giờ, điều đó có nghĩa là chúng ta không nên thay đổi các bản kiểm thử vì chúng ta muốn duy trì cùng một hành vi.

Nếu không có generics, đây là những gì chúng ta *có thể* làm:

```go
type StackOfInts = Stack
type StackOfStrings = Stack

type Stack struct {
	values []interface{}
}

func (s *Stack) Push(value interface{}) {
	s.values = append(s.values, value)
}

func (s *Stack) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack) Pop() (interface{}, bool) {
	if s.IsEmpty() {
		var zero interface{}
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

- Chúng ta đang đặt bí danh (alias) cho các triển khai trước đó của `StackOfInts` và `StackOfStrings` thành một kiểu thống nhất mới là `Stack`.
- Chúng ta đã loại bỏ tính an toàn kiểu khỏi `Stack` bằng cách làm cho `values` trở thành một [slice](https://github.com/quii/learn-go-with-tests/blob/main/arrays-and-slices.md) thuộc kiểu `interface{}`.

Để thử mã nguồn này, bạn sẽ phải loại bỏ các ràng buộc kiểu khỏi các hàm assert của chúng ta:

```go
func AssertEqual(t *testing.T, got, want interface{})
```

Nếu bạn làm điều này, các bản kiểm thử của chúng ta vẫn vượt qua. Ai cần generics chứ?

### Vấn đề của việc vứt bỏ tính an toàn kiểu

Vấn đề đầu tiên tương tự như những gì chúng ta đã thấy với `AssertEquals` - chúng ta đã mất đi tính an toàn kiểu. Bây giờ tôi có thể `Push` những quả táo lên một stack đựng cam.

Thậm chí nếu chúng ta có kỷ luật để không làm điều đó, mã nguồn vẫn không thú vị để làm việc cùng vì các phương thức **trả về `interface{}` rất khủng khiếp khi sử dụng**.

Hãy thêm bản kiểm thử sau,

```go
t.Run("interface stack DX is horrid", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()
	AssertEqual(t, firstNum+secondNum, 3)
})
```

Bạn nhận được một lỗi trình biên dịch, cho thấy điểm yếu của việc mất tính an toàn kiểu:

```
invalid operation: operator + not defined on firstNum (variable of type interface{})
```

Khi `Pop` trả về `interface{}`, điều đó có nghĩa là trình biên dịch không có thông tin về dữ liệu đó là gì và do đó hạn chế nghiêm trọng những gì chúng ta có thể làm. Nó không thể biết rằng đó nên là một số nguyên, vì vậy nó không cho phép chúng ta sử dụng toán tử `+`.

Để lách qua điều này, người gọi phải thực hiện một [khẳng định kiểu (type assertion)](https://golang.org/ref/spec#Type_assertions) cho mỗi giá trị.

```go
t.Run("interface stack dx is horrid", func(t *testing.T) {
	myStackOfInts := new(StackOfInts)

	myStackOfInts.Push(1)
	myStackOfInts.Push(2)
	firstNum, _ := myStackOfInts.Pop()
	secondNum, _ := myStackOfInts.Pop()

	// lấy các số nguyên từ interface{}
	reallyFirstNum, ok := firstNum.(int)
	AssertTrue(t, ok) // cần kiểm tra chắc chắn rằng chúng ta nhận được một số nguyên từ interface{}

	reallySecondNum, ok := secondNum.(int)
	AssertTrue(t, ok) // và một lần nữa!

	AssertEqual(t, reallyFirstNum+reallySecondNum, 3)
})
```

Sự khó chịu tỏa ra từ bản kiểm thử này sẽ bị lặp lại cho mọi người dùng tiềm năng của triển khai `Stack` của chúng ta, thật tệ.

### Kiểu dữ liệu Generic là cứu cánh

Giống như việc bạn có thể định nghĩa các đối số generic cho hàm, bạn có thể định nghĩa các cấu trúc dữ liệu generic.

Đây là triển khai `Stack` mới của chúng ta, sử dụng một kiểu dữ liệu generic.

```go
type Stack[T any] struct {
	values []T
}

func (s *Stack[T]) Push(value T) {
	s.values = append(s.values, value)
}

func (s *Stack[T]) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack[T]) Pop() (T, bool) {
	if s.IsEmpty() {
		var zero T
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

Dưới đây là các bản kiểm thử, cho thấy chúng hoạt động theo cách chúng ta mong muốn, với sự an toàn kiểu hoàn toàn.

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := new(Stack[int])

		// kiểm tra stack đang trống
		AssertTrue(t, myStackOfInts.IsEmpty())

		// thêm một thứ, sau đó kiểm tra nó không trống
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// thêm một thứ khác, lấy nó ra lại
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())

		// có thể lấy các con số chúng ta đưa vào dưới dạng số, không phải interface{}
		myStackOfInts.Push(1)
		myStackOfInts.Push(2)
		firstNum, _ := myStackOfInts.Pop()
		secondNum, _ := myStackOfInts.Pop()
		AssertEqual(t, firstNum+secondNum, 3)
	})
}
```

Bạn sẽ nhận thấy cú pháp để định nghĩa cấu trúc dữ liệu generic nhất quán với việc định nghĩa các đối số generic cho hàm.

```go
type Stack[T any] struct {
	values []T
}
```

Nó *gần như* giống như trước, chỉ là những gì chúng ta đang nói là kiểu của **stack ràng buộc những kiểu giá trị nào bạn có thể làm việc cùng**.

Một khi bạn tạo một `Stack[Orange]` hoặc một `Stack[Apple]`, các phương thức được định nghĩa trên stack của chúng ta sẽ chỉ cho phép bạn truyền vào và sẽ chỉ trả về kiểu dữ liệu cụ thể của stack mà bạn đang làm việc:

```go
func (s *Stack[T]) Pop() (T, bool)
```

Bạn có thể tưởng tượng các triển khai kiểu dữ liệu được tạo ra cho bạn theo cách nào đó, tùy thuộc vào loại stack bạn tạo:

```go
func (s *Stack[Orange]) Pop() (Orange, bool)
```

```go
func (s *Stack[Apple]) Pop() (Apple, bool)
```

Bây giờ chúng ta đã thực hiện việc tái cấu trúc này, chúng ta có thể an tâm xóa bản kiểm thử string stack vì chúng ta không cần chứng minh cùng một logic lặp đi lặp lại.

Lưu ý rằng cho đến nay trong các ví dụ về việc gọi hàm generic, chúng ta không cần phải chỉ định các kiểu generic. Ví dụ, để gọi `AssertEqual[T]`, chúng ta không cần chỉ định kiểu `T` là gì vì nó có thể được suy luận từ các đối số. Trong trường hợp các kiểu generic không thể được suy luận, bạn cần chỉ định các kiểu khi gọi hàm. Cú pháp giống như khi định nghĩa hàm, tức là bạn chỉ định các kiểu bên trong ngoặc vuông trước các đối số.

Để lấy một ví dụ cụ thể, hãy xem xét việc tạo một constructor cho `Stack[T]`.

```go
func NewStack[T any]() *Stack[T] {
	return new(Stack[T])
}
```

Để sử dụng constructor này tạo một stack chứa số nguyên và một stack chứa chuỗi chẳng hạn, bạn gọi nó như sau:

```go
myStackOfInts := NewStack[int]()
myStackOfStrings := NewStack[string]()
```

Đây là triển khai `Stack` và các bản kiểm thử sau khi thêm constructor.

```go
type Stack[T any] struct {
	values []T
}

func NewStack[T any]() *Stack[T] {
	return new(Stack[T])
}

func (s *Stack[T]) Push(value T) {
	s.values = append(s.values, value)
}

func (s *Stack[T]) IsEmpty() bool {
	return len(s.values) == 0
}

func (s *Stack[T]) Pop() (T, bool) {
	if s.IsEmpty() {
		var zero T
		return zero, false
	}

	index := len(s.values) - 1
	el := s.values[index]
	s.values = s.values[:index]
	return el, true
}
```

```go
func TestStack(t *testing.T) {
	t.Run("integer stack", func(t *testing.T) {
		myStackOfInts := NewStack[int]()

		// kiểm tra stack đang trống
		AssertTrue(t, myStackOfInts.IsEmpty())

		// thêm một thứ, sau đó kiểm tra nó không trống
		myStackOfInts.Push(123)
		AssertFalse(t, myStackOfInts.IsEmpty())

		// thêm một thứ khác, lấy nó ra lại
		myStackOfInts.Push(456)
		value, _ := myStackOfInts.Pop()
		AssertEqual(t, value, 456)
		value, _ = myStackOfInts.Pop()
		AssertEqual(t, value, 123)
		AssertTrue(t, myStackOfInts.IsEmpty())

		// có thể lấy các con số chúng ta đưa vào dưới dạng số, không phải interface{}
		myStackOfInts.Push(1)
		myStackOfInts.Push(2)
		firstNum, _ := myStackOfInts.Pop()
		secondNum, _ := myStackOfInts.Pop()
		AssertEqual(t, firstNum+secondNum, 3)
	})
}
```

Sử dụng kiểu dữ liệu generic, chúng ta đã:

- Giảm bớt sự trùng lặp của logic quan trọng.
- Làm cho `Pop` trả về `T` để nếu chúng ta tạo một `Stack[int]`, thực tế chúng ta nhận lại `int` từ `Pop`; bây giờ chúng ta có thể sử dụng `+` mà không cần các màn nhào lộn khẳng định kiểu.
- Ngăn chặn việc lạm dụng tại thời điểm biên dịch. Bạn không thể `Push` những quả cam vào một apple stack.


## Tổng kết

Chương này lẽ ra đã cho bạn cảm nhận về cú pháp generics, và một số ý tưởng tại sao generics lại có ích. Chúng ta đã viết các hàm `Assert` của riêng mình mà chúng ta có thể tái sử dụng một cách an toàn để thử nghiệm các ý tưởng khác xung quanh generics, và chúng ta đã triển khai một cấu trúc dữ liệu đơn giản để lưu trữ bất kỳ loại dữ liệu nào chúng ta muốn, theo cách an toàn kiểu.

### Generics đơn giản hơn nhiều so với việc sử dụng `interface{}` trong hầu hết các trường hợp

Nếu bạn chưa có kinh nghiệm với các ngôn ngữ định kiểu tĩnh, ưu điểm của generics có thể không quá rõ ràng ngay lập tức, nhưng tôi hy vọng các ví dụ trong chương này đã minh họa cho bạn thấy ngôn ngữ Go đôi khi không biểu cảm như chúng ta mong muốn. Cụ thể, việc sử dụng `interface{}` làm cho mã nguồn của bạn:

- Kém an toàn hơn (trộn lẫn cam và táo), yêu cầu xử lý lỗi nhiều hơn
- Kém biểu cảm hơn, `interface{}` không cho bạn biết bất cứ điều gì về dữ liệu
- Có nhiều khả năng phải dựa vào [reflection](https://github.com/quii/learn-go-with-tests/blob/main/reflection.md), khẳng định kiểu, v.v., làm cho mã nguồn của bạn khó làm việc hơn và dễ xảy ra lỗi hơn vì nó đẩy việc kiểm tra từ thời điểm biên dịch sang thời điểm chạy (runtime)

Sử dụng ngôn ngữ định kiểu tĩnh là một hành động mô tả các ràng buộc. Nếu bạn làm tốt, bạn tạo ra mã nguồn không chỉ an toàn và đơn giản để sử dụng mà còn đơn giản hơn để viết vì không gian giải pháp khả thi là nhỏ hơn.

Generics cung cấp cho chúng ta một cách mới để thể hiện các ràng buộc trong mã nguồn của mình, và như đã chứng minh, nó sẽ cho phép chúng ta củng cố và đơn giản hóa mã nguồn mà trước đây không thể thực hiện được cho đến phiên bản Go 1.18.

### Generics sẽ biến Go thành Java?

- Không.

Có rất nhiều [FUD (nỗi sợ hãi, sự không chắc chắn và nghi ngờ)](https://en.wikipedia.org/wiki/Fear,_uncertainty,_and_doubt) trong cộng đồng Go về việc generics dẫn đến các trừu tượng hóa ác mộng và các cơ sở mã nguồn gây bối rối. Điều này thường đi kèm với cảnh báo "chúng phải được sử dụng cẩn thận".

Mặc dù điều này đúng, nhưng nó không phải là lời khuyên đặc biệt hữu ích vì điều này đúng với bất kỳ tính năng ngôn ngữ nào.

Không có nhiều người phàn nàn về khả năng định nghĩa interface của chúng ta, vốn cũng giống như generics là một cách mô tả các ràng buộc trong mã nguồn của chúng ta. Khi bạn mô tả một interface, bạn đang đưa ra một lựa chọn thiết kế *có thể kém*, generics không phải là thứ duy nhất có khả năng tạo ra mã nguồn gây nhầm lẫn và khó chịu khi sử dụng.

### Bạn vốn dĩ đã đang sử dụng generics

Khi bạn xem xét rằng nếu bạn đã từng sử dụng array, slice hoặc map; bạn *vốn dĩ đã là một người tiêu dùng mã nguồn generic*.

```go
var myApples []Apple
// Bạn không thể làm điều này!
append(myApples, Orange{})
```

### Trừu tượng hóa không phải là một từ xấu

Thật dễ dàng để châm chọc [AbstractSingletonProxyFactoryBean](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/aop/framework/AbstractSingletonProxyFactoryBean.html) nhưng đừng giả vờ rằng một cơ sở mã nguồn không có sự trừu tượng hóa nào cả lại không tệ. Công việc của bạn là *tập hợp* các khái niệm liên quan khi thích hợp, để hệ thống của bạn dễ hiểu và dễ thay đổi hơn; thay vì là một tập hợp các hàm và kiểu dữ liệu rời rạc, thiếu rõ ràng.

### [Làm cho nó chạy, làm cho nó đúng, làm cho nó nhanh](https://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast#:~:text=%22Make%20it%20work%2C%20make%20it,to%20DesignForPerformance%20ahead%20of%20time.)

Mọi người gặp vấn đề với generics khi họ trừu tượng hóa quá nhanh mà không có đủ thông tin để đưa ra các quyết định thiết kế tốt.

Chu kỳ TDD đỏ, xanh, tái cấu trúc có nghĩa là bạn có nhiều sự hướng dẫn hơn về việc bạn *thực sự cần* mã nguồn nào để phân phối hành vi của mình, **thay vì tưởng tượng ra các trừu tượng ngay từ đầu**; nhưng bạn vẫn cần cẩn thận.

Không có quy tắc cứng nhắc nào ở đây nhưng hãy cưỡng lại việc làm cho mọi thứ trở nên generic cho đến khi bạn có thể thấy rằng mình có một sự khái quát hóa hữu ích. Khi chúng ta tạo ra các triển khai `Stack` khác nhau, chúng ta đã bắt đầu một cách quan trọng với các hành vi *cụ thể* như `StackOfStrings` và `StackOfInts` được hỗ trợ bởi các bản kiểm thử. Từ mã nguồn *thực tế* của mình, chúng ta có thể bắt đầu thấy các mẫu thực sự, và được hỗ trợ bởi các bản kiểm thử, chúng ta có thể khám phá việc tái cấu trúc theo hướng một giải pháp đa năng hơn.

Mọi người thường khuyên bạn chỉ nên khái quát hóa khi bạn thấy cùng một đoạn mã nguồn xuất hiện ba lần, điều này có vẻ như là một quy tắc bắt đầu tốt.

Một con đường phổ biến mà tôi đã đi trong các ngôn ngữ lập trình khác là:

- Một chu kỳ TDD để dẫn dắt một số hành vi
- Một chu kỳ TDD khác để thực hiện các kịch bản liên quan khác

> Chà, những thứ này trông giống nhau - nhưng một chút trùng lặp thì tốt hơn là gắn kết (coupling) vào một sự trừu tượng tồi

- Hãy ngủ một giấc
- Một chu kỳ TDD khác

> OK, tôi muốn thử xem mình liệu có thể khái quát hóa thứ này không. Tạ ơn trời đất vì tôi thật thông minh và đẹp trai vì tôi sử dụng TDD, vì vậy tôi có thể tái cấu trúc bất cứ khi nào tôi muốn, và quy trình này đã giúp tôi hiểu những hành vi nào tôi thực sự cần trước khi thiết kế quá nhiều.

- Sự trừu tượng này cảm thấy thật tuyệt! Các bản kiểm thử vẫn đang vượt qua, và mã nguồn đơn giản hơn
- Bây giờ tôi có thể xóa một số bản kiểm thử, tôi đã nắm bắt được *bản chất* của hành vi và loại bỏ những chi tiết không cần thiết

