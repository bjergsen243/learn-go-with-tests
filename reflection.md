# Reflection

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/reflection)**

[Từ Twitter](https://twitter.com/peterbourgon/status/1011403901419937792?s=09)

> thử thách golang: viết một hàm `walk(x interface{}, fn func(string))` nhận vào một struct `x` và gọi `fn` cho tất cả các trường (fields) kiểu string được tìm thấy bên trong. mức độ khó: đệ quy.

Để làm điều này, chúng ta sẽ cần sử dụng _reflection_.

> Reflection trong lập trình là khả năng của một chương trình để kiểm tra cấu trúc của chính nó, đặc biệt là thông qua các kiểu dữ liệu; đó là một hình thức của metaprogramming. Nó cũng là một nguồn gây ra nhiều bối rối.

Từ [The Go Blog: Reflection](https://blog.golang.org/laws-of-reflection)

## `interface{}` là gì?

Chúng ta đã tận hưởng type-safety của Go khi làm việc với các kiểu đã biết như `string`, `int` hay `BankAccount`.

Nhờ đó, chúng ta có tài liệu sẵn và compiler sẽ báo lỗi nếu truyền sai kiểu vào hàm.

Tuy nhiên, đôi khi bạn cần viết hàm mà không biết trước kiểu dữ liệu tại compile time.

Go cho phép chúng ta vượt qua điều này với kiểu `interface{}`, bạn có thể coi nó là *bất kỳ* kiểu dữ liệu nào (thực tế, trong Go `any` là một [bí danh](https://cs.opensource.google/go/go/+/master:src/builtin/builtin.go;drc=master;l=95) cho `interface{}`).

Vì vậy, `walk(x interface{}, fn func(string))` sẽ chấp nhận bất kỳ giá trị nào cho `x`.

### Vậy tại sao không sử dụng `interface{}` cho mọi thứ để có các hàm thực sự linh hoạt?

- Người dùng hàm nhận `interface{}` mất đi type-safety. Nếu bạn định truyền `Herd.species` (kiểu `string`) nhưng lại truyền nhầm `Herd.count` (kiểu `int`)? Compiler không báo lỗi. Bạn cũng không biết mình được truyền gì vào. Biết hàm nhận `UserService` chẳng hạn sẽ rõ ràng hơn nhiều.
- Người viết hàm phải kiểm tra *bất cứ thứ gì* được truyền vào và xác định kiểu dữ liệu. Việc này dùng _reflection_ — khá rườm rà, khó đọc và kém hiệu quả hơn vì phải kiểm tra tại runtime.

Tóm lại, chỉ sử dụng reflection nếu bạn thực sự cần.

Nếu bạn muốn có các hàm đa hình (polymorphic functions), hãy cân nhắc xem liệu bạn có thể thiết kế nó dựa trên một interface (khác với `interface{}`, điều này hơi gây bối rối) để người dùng có thể sử dụng hàm của bạn với nhiều kiểu dữ liệu nếu họ triển khai các phương thức cần thiết để hàm của bạn hoạt động.

Hàm của chúng ta sẽ cần có khả năng hoạt động với nhiều thứ khác nhau. Như mọi khi, chúng ta sẽ thực hiện theo cách lặp đi lặp lại, viết các test cho từng thứ mới mà chúng ta muốn hỗ trợ và tái cấu trúc trên đường đi cho đến khi hoàn thành.

## Viết test trước tiên

Chúng ta muốn gọi hàm của mình với một struct có một trường string trong đó (`x`). Sau đó, chúng ta có thể giám sát hàm (`fn`) được truyền vào để xem nó có được gọi hay không.

```go
func TestWalk(t *testing.T) {

	expected := "Chris"
	var got []string

	x := struct {
		Name string
	}{expected}

	walk(x, func(input string) {
		got = append(got, input)
	})

	if len(got) != 1 {
		t.Errorf("wrong number of function calls, got %d want %d", len(got), 1)
	}
}
```

- Chúng ta muốn lưu trữ một slice các chuỗi (`got`) để lưu những chuỗi nào đã được truyền vào `fn` bởi `walk`. Thông thường trong các chương trước, chúng ta đã tạo các kiểu chuyên dụng cho việc này để giám sát các lời gọi hàm/phương thức nhưng trong trường hợp này, chúng ta chỉ có thể truyền vào một hàm ẩn danh cho `fn` để ghi nhận các giá trị vào `got`.
- Chúng ta sử dụng một `struct` ẩn danh với trường `Name` kiểu string để thực hiện trường hợp "thành công" đơn giản nhất.
- Cuối cùng, gọi `walk` với `x` và hàm giám sát, hiện tại chỉ kiểm tra độ dài của `got`, chúng ta sẽ đưa ra các xác nhận (assertions) cụ thể hơn sau khi đã làm cho chức năng cơ bản hoạt động.

## Thử chạy test

```
./reflection_test.go:21:2: undefined: walk
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Chúng ta cần định nghĩa hàm `walk`:

```go
func walk(x interface{}, fn func(input string)) {

}
```

Thử chạy lại test:

```
=== RUN   TestWalk
--- FAIL: TestWalk (0.00s)
    reflection_test.go:19: wrong number of function calls, got 0 want 1
FAIL
```

## Viết đủ code để test chạy thành công

Chúng ta có thể gọi hàm giám sát với bất kỳ chuỗi nào để làm test này vượt qua.

```go
func walk(x interface{}, fn func(input string)) {
	fn("I still can't believe South Korea beat Germany 2-0 to put them last in their group")
}
```

Test bây giờ sẽ vượt qua. Việc tiếp theo chúng ta cần làm là thực hiện một xác nhận cụ thể hơn về những gì `fn` đang được gọi.

## Viết test trước tiên

Thêm đoạn sau vào test hiện có để kiểm tra xem chuỗi được truyền cho `fn` có đúng không:

```go
if got[0] != expected {
	t.Errorf("got %q, want %q", got[0], expected)
}
```

## Thử chạy test

```
=== RUN   TestWalk
--- FAIL: TestWalk (0.00s)
    reflection_test.go:23: got 'I still can't believe South Korea beat Germany 2-0 to put them last in their group', want 'Chris'
FAIL
```

## Viết đủ code để test chạy thành công

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)
	field := val.Field(0)
	fn(field.String())
}
```

Đoạn mã này *rất không an toàn và rất ngây thơ*, nhưng hãy nhớ: mục tiêu của chúng ta khi đang ở trạng thái "đỏ" (các bản kiểm thử thất bại) là viết lượng mã nhỏ nhất có thể. Sau đó, chúng ta sẽ viết thêm các bản kiểm thử để giải quyết các mối lo ngại của mình.

Chúng ta cần sử dụng reflection để xem xét `x` và thử nhìn vào các thuộc tính của nó.

[Package reflect](https://pkg.go.dev/reflect) có một hàm `ValueOf` trả về cho chúng ta một `Value` của một biến cho trước. Nó có các cách để chúng ta kiểm tra một giá trị, bao gồm các trường của nó mà chúng ta sử dụng ở dòng tiếp theo.

Sau đó, chúng ta đưa ra một số giả định rất lạc quan về giá trị được truyền vào:

- Chúng ta nhìn vào trường đầu tiên và duy nhất. Tuy nhiên, có thể không có trường nào cả, điều này sẽ gây ra panic.
- Sau đó, chúng ta gọi `String()`, hàm này trả về giá trị bên dưới dưới dạng một chuỗi. Tuy nhiên, điều này sẽ sai nếu trường đó là một kiểu dữ liệu khác không phải chuỗi.

## Refactor

Mã của chúng ta đang vượt qua trường hợp đơn giản nhưng chúng ta biết rằng mã của mình còn nhiều thiếu sót.

Chúng ta sẽ viết một số bản kiểm thử nơi chúng ta truyền vào các giá trị khác nhau và kiểm tra mảng chuỗi mà `fn` được gọi.

Chúng ta nên tái cấu trúc bản kiểm thử của mình thành một table-driven test (kiểm thử dựa trên bảng) để dễ dàng tiếp tục kiểm thử các kịch bản mới.

```go
func TestWalk(t *testing.T) {

	cases := []struct {
		Name          string
		Input         interface{}
		ExpectedCalls []string
	}{
		{
			"struct with one string field",
			struct {
				Name string
			}{"Chris"},
			[]string{"Chris"},
		},
	}

	for _, test := range cases {
		t.Run(test.Name, func(t *testing.T) {
			var got []string
			walk(test.Input, func(input string) {
				got = append(got, input)
			})

			if !reflect.DeepEqual(got, test.ExpectedCalls) {
				t.Errorf("got %v, want %v", got, test.ExpectedCalls)
			}
		})
	}
}
```

Bây giờ chúng ta có thể dễ dàng thêm một kịch bản để xem điều gì xảy ra nếu chúng ta có nhiều hơn một trường string.

## Viết test trước tiên

Thêm kịch bản sau vào `cases`.

```
{
    "struct with two string fields",
    struct {
        Name string
        City string
    }{"Chris", "London"},
    []string{"Chris", "London"},
}
```

## Thử chạy test

```
=== RUN   TestWalk/struct_with_two_string_fields
    --- FAIL: TestWalk/struct_with_two_string_fields (0.00s)
        reflection_test.go:40: got [Chris], want [Chris London]
```

## Viết đủ code để test chạy thành công

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)
		fn(field.String())
	}
}
```

`val` có một phương thức `NumField` trả về số lượng trường trong giá trị đó. Điều này cho phép chúng ta lặp qua các trường và gọi `fn`, làm cho test vượt qua.

## Refactor

Có vẻ như không có bước tái cấu trúc rõ ràng nào ở đây giúp cải thiện mã nguồn, vì vậy chúng ta hãy tiếp tục.

Thiếu sót tiếp theo trong `walk` là nó giả định mọi trường đều là một `string`. Hãy viết một bản kiểm thử cho kịch bản này.

## Viết test trước tiên

Thêm trường hợp sau:

```
{
    "struct with non string field",
    struct {
        Name string
        Age  int
    }{"Chris", 33},
    []string{"Chris"},
},
```

## Thử chạy test

```
=== RUN   TestWalk/struct_with_non_string_field
    --- FAIL: TestWalk/struct_with_non_string_field (0.00s)
        reflection_test.go:46: got [Chris <int Value>], want [Chris]
```

## Viết đủ code để test chạy thành công

Chúng ta cần kiểm tra xem kiểu của trường có phải là `string` hay không.

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		if field.Kind() == reflect.String {
			fn(field.String())
		}
	}
}
```

Chúng ta có thể thực hiện điều đó bằng cách kiểm tra [`Kind`](https://pkg.go.dev/reflect#Kind) của nó.

## Refactor

Một lần nữa, có vẻ như mã nguồn đã đủ hợp lý cho thời điểm hiện tại.

Kịch bản tiếp theo là điều gì sẽ xảy ra nếu nó không phải là một `struct` "phẳng"? Nói cách khác, điều gì xảy ra nếu chúng ta có một `struct` với một số trường lồng nhau?

## Viết test trước tiên

Chúng ta đã sử dụng cú pháp struct ẩn danh để khai báo các kiểu dữ liệu cho bản kiểm thử của mình, vì vậy chúng ta có thể tiếp tục làm như vậy:

```
{
    "nested fields",
    struct {
        Name string
        Profile struct {
            Age  int
            City string
        }
    }{"Chris", struct {
        Age  int
        City string
    }{33, "London"}},
    []string{"Chris", "London"},
},
```

Nhưng chúng ta có thể thấy rằng khi bạn có các struct ẩn danh bên trong, cú pháp sẽ trở nên hơi lộn xộn. [Đã có một đề xuất để làm cho cú pháp này đẹp hơn](https://github.com/golang/go/issues/12854).

Hãy tái cấu trúc điều này bằng cách tạo một kiểu dữ liệu cụ thể cho kịch bản này và tham chiếu nó trong test. Có một chút gián tiếp ở chỗ một phần mã cho test của chúng ta nằm bên ngoài test, nhưng người đọc vẫn có thể suy ra cấu trúc của `struct` bằng cách nhìn vào phần khởi tạo.

Thêm các khai báo kiểu sau vào đâu đó trong file test của bạn:

```go
type Person struct {
	Name    string
	Profile Profile
}

type Profile struct {
	Age  int
	City string
}
```

Giờ đây chúng ta có thể thêm thông tin này vào các trường hợp kiểm thử, điều này dễ đọc hơn nhiều so với trước đây:

```
{
    "nested fields",
    Person{
        "Chris",
        Profile{33, "London"},
    },
    []string{"Chris", "London"},
},
```

## Thử chạy test

```
=== RUN   TestWalk/Nested_fields
    --- FAIL: TestWalk/nested_fields (0.00s)
        reflection_test.go:54: got [Chris], want [Chris London]
```

Vấn đề là chúng ta chỉ đang lặp qua các trường ở cấp độ đầu tiên của hệ thống phân cấp kiểu.

## Viết đủ code để test chạy thành công

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		if field.Kind() == reflect.String {
			fn(field.String())
		}

		if field.Kind() == reflect.Struct {
			walk(field.Interface(), fn)
		}
	}
}
```

Giải pháp khá đơn giản, chúng ta lại kiểm tra `Kind` của nó và nếu nó tình cờ là một `struct`, chúng ta chỉ cần gọi lại `walk` trên `struct` bên trong đó.

## Refactor

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

Khi bạn thực hiện so sánh trên cùng một giá trị nhiều hơn một lần, _nói chung_ việc tái cấu trúc thành `switch` sẽ cải thiện khả năng đọc và làm cho mã nguồn của bạn dễ mở rộng hơn.

Điều gì xảy ra nếu giá trị của struct được truyền vào là một con trỏ (pointer)?

## Viết test trước tiên

Thêm trường hợp này:

```
{
    "pointers to things",
    &Person{
        "Chris",
        Profile{33, "London"},
    },
    []string{"Chris", "London"},
},
```

## Thử chạy test

```
=== RUN   TestWalk/pointers_to_things
panic: reflect: call of reflect.Value.NumField on ptr Value [recovered]
    panic: reflect: call of reflect.Value.NumField on ptr Value
```

## Viết đủ code để test chạy thành công

```go
func walk(x interface{}, fn func(input string)) {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Pointer {
		val = val.Elem()
	}

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

Bạn không thể sử dụng `NumField` trên một `Value` của con trỏ, chúng ta cần trích xuất giá trị bên dưới trước khi có thể làm điều đó bằng cách sử dụng `Elem()`.

## Refactor

Hãy bao đóng trách nhiệm trích xuất `reflect.Value` từ một `interface{}` cho trước vào một hàm.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}

func getValue(x interface{}) reflect.Value {
	val := reflect.ValueOf(x)

	if val.Kind() == reflect.Pointer {
		val = val.Elem()
	}

	return val
}
```

Điều này thực sự làm tăng thêm một chút mã nguồn nhưng tôi cảm thấy cấp độ trừu tượng đã hợp lý hơn.

- Lấy `reflect.Value` của `x` để tôi có thể kiểm tra nó, tôi không quan tâm bằng cách nào.
- Lặp qua các trường, thực hiện bất cứ điều gì cần làm tùy thuộc vào kiểu của nó.

Tiếp theo, chúng ta cần xử lý các slice.

## Viết test trước tiên

```
{
    "slices",
    []Profile {
        {33, "London"},
        {34, "Reykjavík"},
    },
    []string{"London", "Reykjavík"},
},
```

## Thử chạy test

```
=== RUN   TestWalk/slices
panic: reflect: call of reflect.Value.NumField on slice Value [recovered]
    panic: reflect: call of reflect.Value.NumField on slice Value
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Trường hợp này tương tự như kịch bản con trỏ ở trên, chúng ta đang cố gắng gọi `NumField` trên `reflect.Value` của mình nhưng nó không có vì đó không phải là một struct.

## Viết đủ code để test chạy thành công

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	if val.Kind() == reflect.Slice {
		for i := 0; i < val.Len(); i++ {
			walk(val.Index(i).Interface(), fn)
		}
		return
	}

	for i := 0; i < val.NumField(); i++ {
		field := val.Field(i)

		switch field.Kind() {
		case reflect.String:
			fn(field.String())
		case reflect.Struct:
			walk(field.Interface(), fn)
		}
	}
}
```

## Refactor

Đoạn mã này hoạt động nhưng trông hơi tệ. Đừng lo lắng, chúng ta có mã đang hoạt động được hỗ trợ bởi các bản kiểm thử nên chúng ta có thể tự do chỉnh sửa theo ý muốn.

Nếu bạn suy nghĩ hơi trừu tượng một chút, chúng ta muốn gọi `walk` trên cả:

- Mỗi trường trong một struct
- Mỗi mục trong một slice

Hiện tại mã của chúng ta đang thực hiện điều này nhưng chưa thể hiện nó một cách rõ ràng. Chúng ta chỉ có một kiểm tra ở đầu để xem nó có phải là một slice hay không (với lệnh `return` để dừng các đoạn mã còn lại) và nếu không, chúng ta chỉ mặc nhiên coi nó là một struct.

Hãy sửa lại mã để thay vào đó chúng ta kiểm tra kiểu *trước tiên*, sau đó mới thực hiện công việc.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	switch val.Kind() {
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walk(val.Field(i).Interface(), fn)
		}
	case reflect.Slice:
		for i := 0; i < val.Len(); i++ {
			walk(val.Index(i).Interface(), fn)
		}
	case reflect.String:
		fn(val.String())
	}
}
```

Trông tốt hơn nhiều rồi đấy! Nếu đó là một struct hoặc slice, chúng ta lặp qua các giá trị của nó và gọi `walk` cho từng giá trị. Ngược lại, nếu đó là một `reflect.String`, chúng ta có thể gọi `fn`.

Tuy nhiên, đối với tôi, nó vẫn có thể tốt hơn nữa. Có sự lặp lại của hoạt động lặp qua các trường/giá trị và sau đó gọi `walk`, nhưng về mặt khái niệm chúng là giống nhau.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice:
		numberOfValues = val.Len()
		getField = val.Index
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

Nếu `value` là một `reflect.String` thì chúng ta chỉ cần gọi `fn` như bình thường.

Ngược lại, lệnh `switch` của chúng ta sẽ trích xuất ra hai thứ tùy thuộc vào kiểu dữ liệu:

- Có bao nhiêu trường (fields)
- Cách trích xuất `Value` (`Field` hoặc `Index`)

Khi đã xác định được những thứ đó, chúng ta có thể lặp qua `numberOfValues` và gọi `walk` với kết quả của hàm `getField`.

Khi chúng ta làm xong việc này, việc xử lý các mảng (arrays) sẽ trở nên rất đơn giản.

## Viết test trước tiên

Thêm vào các trường hợp kiểm thử:

```
{
    "arrays",
    [2]Profile {
        {33, "London"},
        {34, "Reykjavík"},
    },
    []string{"London", "Reykjavík"},
},
```

## Thử chạy test

```
=== RUN   TestWalk/arrays
    --- FAIL: TestWalk/arrays (0.00s)
        reflection_test.go:78: got [], want [London Reykjavík]
```

## Viết đủ code để test chạy thành công

Mảng có thể được xử lý giống như slice, vì vậy chỉ cần thêm nó vào cùng một trường hợp bằng dấu phẩy:

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice, reflect.Array:
		numberOfValues = val.Len()
		getField = val.Index
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

Kiểu dữ liệu tiếp theo chúng ta muốn xử lý là `map`.

## Viết test trước tiên

```
{
    "maps",
    map[string]string{
        "Cow": "Moo",
        "Sheep": "Baa",
    },
    []string{"Moo", "Baa"},
},
```

## Thử chạy test

```
=== RUN   TestWalk/maps
    --- FAIL: TestWalk/maps (0.00s)
        reflection_test.go:86: got [], want [Moo Baa]
```

## Viết đủ code để test chạy thành công

Một lần nữa, nếu bạn suy nghĩ hơi trừu tượng một chút, bạn sẽ thấy rằng `map` rất giống với `struct`, chỉ là các key không được biết trước tại thời điểm biên dịch.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	numberOfValues := 0
	var getField func(int) reflect.Value

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		numberOfValues = val.NumField()
		getField = val.Field
	case reflect.Slice, reflect.Array:
		numberOfValues = val.Len()
		getField = val.Index
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walk(val.MapIndex(key).Interface(), fn)
		}
	}

	for i := 0; i < numberOfValues; i++ {
		walk(getField(i).Interface(), fn)
	}
}
```

Tuy nhiên, theo thiết kế của Go, bạn không thể lấy các giá trị ra khỏi một map bằng chỉ số (index). Nó chỉ được thực hiện thông qua _key_, vì vậy điều này phá hỏng sự trừu tượng của chúng ta, thật tệ.

## Refactor

Bạn cảm thấy thế nào ngay lúc này? Có vẻ như đó là một sự trừu tượng hay vào thời điểm đó nhưng bây giờ mã nguồn cảm thấy hơi lộn xộn.

*Điều này không sao cả!* Tái cấu trúc là một hành trình và đôi khi chúng ta sẽ mắc sai lầm. Điểm quan trọng của TDD là nó cho chúng ta sự tự do để thử những điều này.

Bằng cách thực hiện các bước nhỏ được hỗ trợ bởi các bản kiểm thử, đây hoàn toàn không phải là một tình huống không thể vãn hồi. Hãy đưa nó trở lại trạng thái trước khi tái cấu trúc.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	}
}
```

Chúng ta đã đưa vào hàm `walkValue` để áp dụng nguyên tắc DRY cho các lời gọi `walk` bên trong lệnh `switch`, nhờ đó chúng chỉ cần trích xuất các `reflect.Value` từ `val`.

### Một vấn đề cuối cùng

Hãy nhớ rằng các map trong Go không đảm bảo thứ tự. Vì vậy, các test của bạn đôi khi sẽ thất bại vì chúng ta khẳng định rằng các lời gọi đến `fn` được thực hiện theo một thứ tự cụ thể.

Để khắc phục điều này, chúng ta cần chuyển phần xác nhận với map sang một bản kiểm thử mới nơi chúng ta không quan tâm đến thứ tự.

```go
t.Run("with maps", func(t *testing.T) {
	aMap := map[string]string{
		"Cow":   "Moo",
		"Sheep": "Baa",
	}

	var got []string
	walk(aMap, func(input string) {
		got = append(got, input)
	})

	assertContains(t, got, "Moo")
	assertContains(t, got, "Baa")
})
```

Đây là cách `assertContains` được định nghĩa:

```go
func assertContains(t testing.TB, haystack []string, needle string) {
	t.Helper()
	contains := false
	for _, x := range haystack {
		if x == needle {
			contains = true
		}
	}
	if !contains {
		t.Errorf("expected %v to contain %q but it didn't", haystack, needle)
	}
}
```

Vì chúng ta đã trích xuất các map vào một bản kiểm thử mới, chúng ta không thấy thông báo lỗi. Hãy cố tình làm hỏng bản kiểm thử `with maps` ở đây để bạn có thể kiểm tra thông báo lỗi, sau đó sửa lại để tất cả các test vượt qua.

Kiểu dữ liệu tiếp theo chúng ta muốn xử lý là `chan`.

## Viết test trước tiên

```go
t.Run("with channels", func(t *testing.T) {
	aChannel := make(chan Profile)

	go func() {
		aChannel <- Profile{33, "Berlin"}
		aChannel <- Profile{34, "Katowice"}
		close(aChannel)
	}()

	var got []string
	want := []string{"Berlin", "Katowice"}

	walk(aChannel, func(input string) {
		got = append(got, input)
	})

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

## Thử chạy test

```
--- FAIL: TestWalk (0.00s)
    --- FAIL: TestWalk/with_channels (0.00s)
        reflection_test.go:115: got [], want [Berlin Katowice]
```

## Viết đủ code để test chạy thành công

Chúng ta có thể lặp qua tất cả các giá trị được gửi qua channel cho đến khi nó được đóng bằng cách sử dụng `Recv()`.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	case reflect.Chan:
		for {
			if v, ok := val.Recv(); ok {
				walkValue(v)
			} else {
				break
			}
		}
	}
}
```

Kiểu dữ liệu tiếp theo chúng ta muốn xử lý là `func` (hàm).

## Viết test trước tiên

```go
t.Run("with function", func(t *testing.T) {
	aFunction := func() (Profile, Profile) {
		return Profile{33, "Berlin"}, Profile{34, "Katowice"}
	}

	var got []string
	want := []string{"Berlin", "Katowice"}

	walk(aFunction, func(input string) {
		got = append(got, input)
	})

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

## Thử chạy test

```
--- FAIL: TestWalk (0.00s)
    --- FAIL: TestWalk/with_function (0.00s)
        reflection_test.go:132: got [], want [Berlin Katowice]
```

## Viết đủ code để test chạy thành công

Các hàm không có đối số dường như không có nhiều ý nghĩa trong kịch bản này. Tuy nhiên, chúng ta nên cho phép có các giá trị trả về tùy ý.

```go
func walk(x interface{}, fn func(input string)) {
	val := getValue(x)

	walkValue := func(value reflect.Value) {
		walk(value.Interface(), fn)
	}

	switch val.Kind() {
	case reflect.String:
		fn(val.String())
	case reflect.Struct:
		for i := 0; i < val.NumField(); i++ {
			walkValue(val.Field(i))
		}
	case reflect.Slice, reflect.Array:
		for i := 0; i < val.Len(); i++ {
			walkValue(val.Index(i))
		}
	case reflect.Map:
		for _, key := range val.MapKeys() {
			walkValue(val.MapIndex(key))
		}
	case reflect.Chan:
		for v, ok := val.Recv(); ok; v, ok = val.Recv() {
			walkValue(v)
		}
	case reflect.Func:
		valFnResult := val.Call(nil)
		for _, res := range valFnResult {
			walkValue(res)
		}
	}
}
```

## Tổng kết

- Đã giới thiệu một số khái niệm từ package `reflect`.
- Đã sử dụng đệ quy (recursion) để duyệt qua các cấu trúc dữ liệu tùy ý.
- Đã thực hiện một bước tái cấu trúc mà khi nhìn lại thấy không ổn nhưng không quá bối rối vì điều đó. Bằng cách làm việc lặp đi lặp lại với các bản kiểm thử, đó không phải là vấn đề quá lớn.
- Chương này chỉ đề cập đến một khía cạnh nhỏ của reflection. [The Go blog có một bài viết tuyệt vời đề cập đến nhiều chi tiết hơn](https://blog.golang.org/laws-of-reflection).
- Bây giờ bạn đã biết về reflection, hãy cố gắng hết sức để tránh sử dụng nó.
