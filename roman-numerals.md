# Chữ số La Mã (Roman Numerals)

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/roman-numerals)**

Một số công ty sẽ yêu cầu bạn thực hiện [Roman Numeral Kata](http://codingdojo.org/kata/RomanNumerals/) như một phần của quy trình phỏng vấn. Chương này sẽ hướng dẫn cách bạn có thể giải quyết nó bằng TDD.

Chúng ta sẽ viết một hàm chuyển đổi từ [số Ả Rập](https://en.wikipedia.org/wiki/Arabic_numerals) (các số từ 0 đến 9) sang Chữ số La Mã.

Nếu bạn chưa từng nghe về [Chữ số La Mã](https://en.wikipedia.org/wiki/Roman_numerals), thì đó là cách người La Mã viết các con số.

Bạn xây dựng chúng bằng cách ghép các biểu tượng lại với nhau và các biểu tượng đó đại diện cho các con số.

Ví dụ `I` là "một". `III` là ba.

Nghe có vẻ dễ nhưng có một vài quy tắc thú vị. `V` nghĩa là năm, nhưng `IV` là 4 (không phải `IIII`).

`MCMLXXXIV` là 1984. Trông thật phức tạp và khó có thể tưởng tượng làm thế nào chúng ta có thể viết mã để giải quyết điều này ngay từ đầu.

Như cuốn sách này nhấn mạnh, một kỹ năng then chốt đối với các nhà phát triển phần mềm là cố gắng xác định các "lát cắt mỏng theo chiều dọc" (thin vertical slices) của các chức năng *có ích* và sau đó **lặp lại** (iterating). Quy trình TDD giúp tạo điều kiện cho việc phát triển lặp.

Vì vậy, thay vì 1984, hãy bắt đầu với 1.

## Viết test trước tiên

```go
func TestRomanNumerals(t *testing.T) {
	got := ConvertToRoman(1)
	want := "I"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

Nếu bạn đã đọc đến đây của cuốn sách, hy vọng bạn cảm thấy điều này thật nhàm chán và quen thuộc. Đó là một dấu hiệu tốt.

## Thử chạy test

```console
./numeral_test.go:6:9: undefined: ConvertToRoman
```

Hãy để compiler dẫn đường.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Tạo hàm của chúng ta nhưng đừng làm cho test vượt qua ngay, luôn đảm bảo rằng test thất bại theo cách bạn mong đợi.

```go
func ConvertToRoman(arabic int) string {
	return ""
}
```

Bây giờ test sẽ chạy được:

```console
=== RUN   TestRomanNumerals
--- FAIL: TestRomanNumerals (0.00s)
    numeral_test.go:10: got '', want 'I'
FAIL
```

## Viết đủ code để test chạy thành công

```go
func ConvertToRoman(arabic int) string {
	return "I"
}
```

## Refactor

Chưa có gì nhiều để tái cấu trúc.

*Tôi biết* cảm giác thật kỳ lạ khi chỉ code cứng (hard-code) kết quả, nhưng với TDD chúng ta muốn thoát khỏi trạng thái "đỏ" càng nhanh càng tốt. Có vẻ như chúng ta chưa hoàn thành được gì nhiều nhưng chúng ta đã định nghĩa được API và có một bản kiểm thử nắm bắt được một trong các quy tắc của mình; ngay cả khi mã "thực sự" trông khá ngớ ngẩn.

Bây giờ hãy sử dụng cảm giác không thoải mái đó để viết một test mới, buộc chúng ta phải viết mã bớt ngớ ngẩn hơn một chút.

## Viết test trước tiên

Chúng ta có thể sử dụng subtests để nhóm các bản kiểm thử một cách đẹp mắt.

```go
func TestRomanNumerals(t *testing.T) {
	t.Run("1 gets converted to I", func(t *testing.T) {
		got := ConvertToRoman(1)
		want := "I"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})

	t.Run("2 gets converted to II", func(t *testing.T) {
		got := ConvertToRoman(2)
		want := "II"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

## Thử chạy test

```console
=== RUN   TestRomanNumerals/2_gets_converted_to_II
    --- FAIL: TestRomanNumerals/2_gets_converted_to_II (0.00s)
        numeral_test.go:20: got 'I', want 'II'
```

Không có gì ngạc nhiên ở đây.

## Viết đủ code để test chạy thành công

```go
func ConvertToRoman(arabic int) string {
	if arabic == 2 {
		return "II"
	}
	return "I"
}
```

Đúng vậy, cảm giác vẫn như thể chúng ta chưa thực sự giải quyết được vấn đề. Vì vậy, chúng ta cần viết thêm nhiều test để thúc đẩy bản thân tiến về phía trước.

## Refactor

Chúng ta thấy có sự lặp lại trong các bản kiểm thử. Khi bạn đang kiểm thử một thứ gì đó có vẻ như là vấn đề "với đầu vào X, chúng ta mong đợi Y", có lẽ bạn nên sử dụng kiểm thử dựa trên bảng (table based tests).

```go
func TestRomanNumerals(t *testing.T) {
	cases := []struct {
		Description string
		Arabic      int
		Want        string
	}{
		{"1 gets converted to I", 1, "I"},
		{"2 gets converted to II", 2, "II"},
	}

	for _, test := range cases {
		t.Run(test.Description, func(t *testing.T) {
			got := ConvertToRoman(test.Arabic)
			if got != test.Want {
				t.Errorf("got %q, want %q", got, test.Want)
			}
		})
	}
}
```

Bây giờ chúng ta có thể dễ dàng thêm nhiều trường hợp hơn mà không cần phải viết thêm bất kỳ mã kiểm thử rườm rà nào.

Hãy thử tiếp tục với số 3.

## Viết test trước tiên

Thêm trường hợp sau vào `cases`:

```
{"3 gets converted to III", 3, "III"},
```

## Thử chạy test

```console
=== RUN   TestRomanNumerals/3_gets_converted_to_III
    --- FAIL: TestRomanNumerals/3_gets_converted_to_III (0.00s)
        numeral_test.go:20: got 'I', want 'III'
```

## Viết đủ code để test chạy thành công

```go
func ConvertToRoman(arabic int) string {
	if arabic == 3 {
		return "III"
	}
	if arabic == 2 {
		return "II"
	}
	return "I"
}
```

## Refactor

Được rồi, tôi bắt đầu thấy không thích những câu lệnh if này và nếu bạn quan sát kỹ mã nguồn, bạn có thể thấy rằng chúng ta đang xây dựng một chuỗi các ký tự `I` dựa trên giá trị của `arabic`.

Chúng ta "biết" rằng đối với các con số phức tạp hơn, chúng ta sẽ thực hiện một số loại phép tính số học và nối chuỗi.

Hãy thử tái cấu trúc với những suy nghĩ này trong đầu, nó *có thể không* phù hợp cho giải pháp cuối cùng nhưng không sao cả. Chúng ta luôn có thể vứt bỏ mã của mình và bắt đầu lại từ đầu với các bản kiểm thử mà chúng ta có để dẫn dắt.

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := 0; i < arabic; i++ {
		result.WriteString("I")
	}

	return result.String()
}
```

Có thể bạn chưa từng sử dụng [`strings.Builder`](https://golang.org/pkg/strings/#Builder) trước đây:

> Một Builder được sử dụng để xây dựng một chuỗi một cách hiệu quả bằng các phương thức Write. Nó giảm thiểu việc sao chép bộ nhớ.

Thông thường, tôi sẽ không bận tâm đến những tối ưu hóa như vậy cho đến khi gặp vấn đề thực tế về hiệu suất, nhưng lượng mã nguồn không lớn hơn nhiều so với việc nối chuỗi "thủ công", vì vậy chúng ta cũng có thể sử dụng cách tiếp cận nhanh hơn này.

Mã nguồn trông có vẻ tốt hơn và mô tả được miền kiến thức (domain) *như chúng ta biết ngay bây giờ*.

### Người La Mã cũng thích nguyên tắc DRY...

Mọi thứ bắt đầu trở nên phức tạp hơn. Người La Mã với trí tuệ của mình nghĩ rằng việc lặp lại các ký tự sẽ trở nên khó đọc và khó đếm. Vì vậy, một quy tắc với Chữ số La Mã là bạn không thể có cùng một ký tự lặp lại quá 3 lần liên tiếp.

Thay vào đó, bạn lấy biểu tượng cao hơn tiếp theo và sau đó "trừ đi" bằng cách đặt một biểu tượng ở bên trái của nó. Không phải tất cả các biểu tượng đều có thể được sử dụng làm số trừ; chỉ có I (1), X (10) và C (100).

Ví dụ số `5` trong Chữ số La Mã là `V`. Để tạo ra số 4, bạn không viết `IIII`, thay vào đó bạn viết `IV`.

## Viết test trước tiên

```
{"4 gets converted to IV (can't repeat more than 3 times)", 4, "IV"},
```

## Thử chạy test

```console
=== RUN   TestRomanNumerals/4_gets_converted_to_IV_(cant_repeat_more_than_3_times)
    --- FAIL: TestRomanNumerals/4_gets_converted_to_IV_(cant_repeat_more_than_3_times) (0.00s)
        numeral_test.go:24: got 'IIII', want 'IV'
```

## Viết đủ code để test chạy thành công

```go
func ConvertToRoman(arabic int) string {

	if arabic == 4 {
		return "IV"
	}

	var result strings.Builder

	for i := 0; i < arabic; i++ {
		result.WriteString("I")
	}

	return result.String()
}
```

## Refactor

Tôi không "thích" việc chúng ta đã phá vỡ mẫu hình xây dựng chuỗi và tôi muốn tiếp tục với nó.

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := arabic; i > 0; i-- {
		if i == 4 {
			result.WriteString("IV")
			break
		}
		result.WriteString("I")
	}

	return result.String()
}
```

Để số 4 "khớp" với suy nghĩ hiện tại, tôi đếm ngược từ số Ả Rập, thêm các biểu tượng vào chuỗi khi chúng ta tiến hành. Không chắc cách này có hoạt động lâu dài không nhưng hãy cứ xem thế nào!

Hãy làm cho số 5 hoạt động.

## Viết test trước tiên

```
{"5 gets converted to V", 5, "V"},
```

## Thử chạy test

```console
=== RUN   TestRomanNumerals/5_gets_converted_to_V
    --- FAIL: TestRomanNumerals/5_gets_converted_to_V (0.00s)
        numeral_test.go:25: got 'IIV', want 'V'
```

## Viết đủ code để test chạy thành công

Chỉ cần sao chép cách tiếp cận mà chúng ta đã làm cho số 4:

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for i := arabic; i > 0; i-- {
		if i == 5 {
			result.WriteString("V")
			break
		}
		if i == 4 {
			result.WriteString("IV")
			break
		}
		result.WriteString("I")
	}

	return result.String()
}
```

## Refactor

Sự lặp lại trong các vòng lặp như thế này thường là dấu hiệu của một sự trừu tượng đang chờ được gọi tên. Ngắt quãng các vòng lặp sớm có thể là một công cụ hiệu quả cho khả năng đọc nhưng nó cũng có thể đang nói với bạn điều gì đó khác.

Chúng ta đang lặp qua số Ả Rập và nếu gặp các biểu tượng nhất định, chúng ta gọi `break`, nhưng những gì chúng ta *thực sự* đang làm là trừ đi `i` một cách vụng về.

```go
func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for arabic > 0 {
		switch {
		case arabic > 4:
			result.WriteString("V")
			arabic -= 5
		case arabic > 3:
			result.WriteString("IV")
			arabic -= 4
		default:
			result.WriteString("I")
			arabic--
		}
	}

	return result.String()
}
```

- Dựa trên các tín hiệu tôi đọc được từ mã nguồn, được thúc đẩy từ các bản kiểm thử của chúng ta qua một số kịch bản rất cơ bản, tôi có thể thấy rằng để xây dựng một Chữ số La Mã, tôi cần trừ đi từ `arabic` khi áp dụng các biểu tượng.
- Vòng lặp `for` không còn phụ thuộc vào biến `i` và thay vào đó chúng ta sẽ tiếp tục xây dựng chuỗi cho đến khi đã trừ đủ các biểu tượng từ `arabic`.

Tôi khá chắc chắn cách tiếp cận này cũng sẽ hợp lệ cho các số 6 (VI), 7 (VII) và 8 (VIII). Tuy nhiên, hãy thêm các trường hợp vào bộ test của chúng ta và kiểm tra (tôi sẽ không đưa mã nguồn vào đây cho ngắn gọn, hãy kiểm tra github để xem các mẫu nếu bạn không chắc chắn).

Số 9 tuân theo quy tắc giống như số 4 ở chỗ chúng ta nên trừ `I` từ biểu diễn của con số tiếp theo. Số 10 được đại diện trong Chữ số La Mã bằng `X`; do đó số 9 nên là `IX`.

## Viết test trước tiên

```
{"9 gets converted to IX", 9, "IX"},
```
## Thử chạy test

```console
=== RUN   TestRomanNumerals/9_gets_converted_to_IX
    --- FAIL: TestRomanNumerals/9_gets_converted_to_IX (0.00s)
        numeral_test.go:29: got 'VIV', want 'IX'
```

## Viết đủ code để test chạy thành công

Chúng ta có thể áp dụng cùng một cách tiếp cận như trước:

```go
case arabic > 8:
    result.WriteString("IX")
    arabic -= 9
```

## Refactor

Cảm giác như mã nguồn vẫn đang mách bảo rằng có một sự tái cấu trúc nào đó ở đâu đó nhưng tôi chưa thấy rõ, vì vậy hãy tiếp tục.

Tôi cũng sẽ bỏ qua mã nguồn cho phần này, nhưng hãy thêm vào các kịch bản kiểm thử của bạn một test cho số `10` vốn nên là `X` và làm cho nó vượt qua trước khi đọc tiếp.

Dưới đây là một vài test tôi đã thêm vào vì tôi tin tưởng rằng mã nguồn của chúng ta nên hoạt động cho đến số 39:

```
{"10 gets converted to X", 10, "X"},
{"14 gets converted to XIV", 14, "XIV"},
{"18 gets converted to XVIII", 18, "XVIII"},
{"20 gets converted to XX", 20, "XX"},
{"39 gets converted to XXXIX", 39, "XXXIX"},
```

Nếu bạn đã từng lập trình hướng đối tượng (OO), bạn sẽ biết rằng bạn nên nhìn các câu lệnh `switch` với một chút nghi ngờ. Thường thì bạn đang nắm bắt một khái niệm hoặc dữ liệu trong một đoạn mã mệnh lệnh (imperative code), trong khi thực tế nó có thể được nắm bắt trong một cấu trúc lớp (class structure) thay thế.

Go không hoàn toàn là OO nhưng điều đó không có nghĩa là chúng ta bỏ qua hoàn toàn các bài học mà OO mang lại (cho dù một số người muốn nói với bạn như vậy).

Câu lệnh switch của chúng ta đang mô tả một số sự thật về Chữ số La Mã cùng với hành vi.

Chúng ta có thể tái cấu trúc điều này bằng cách tách biệt dữ liệu khỏi hành vi.

```go
type RomanNumeral struct {
	Value  int
	Symbol string
}

var allRomanNumerals = []RomanNumeral{
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}

func ConvertToRoman(arabic int) string {

	var result strings.Builder

	for _, numeral := range allRomanNumerals {
		for arabic >= numeral.Value {
			result.WriteString(numeral.Symbol)
			arabic -= numeral.Value
		}
	}

	return result.String()
}
```

Điều này cảm thấy tốt hơn nhiều. Chúng ta đã khai báo một số quy tắc xung quanh các chữ số dưới dạng dữ liệu thay vì ẩn nó trong một thuật toán và chúng ta có thể thấy cách mình làm việc qua số Ả Rập, cố gắng thêm các biểu tượng vào kết quả nếu chúng phù hợp.

Sự trừu tượng này có hoạt động cho các con số lớn hơn không? Hãy mở rộng bộ test để nó hoạt động cho chữ số La Mã của số 50 vốn là `L`.

Dưới đây là một số trường hợp kiểm thử, hãy thử làm cho chúng vượt qua.

```
{"40 gets converted to XL", 40, "XL"},
{"47 gets converted to XLVII", 47, "XLVII"},
{"49 gets converted to XLIX", 49, "XLIX"},
{"50 gets converted to L", 50, "L"},
```

Cần giúp đỡ không? Bạn có thể xem các biểu tượng cần thêm vào ở [gist này](https://gist.github.com/pamelafox/6c7b948213ba55332d86efd0f0b037de).

## Và phần còn lại!

Dưới đây là các biểu tượng còn lại:

| Số Ả Rập | La Mã |
| ------ | :---: |
| 100    |   C   |
| 500    |   D   |
| 1000   |   M   |

Hãy áp dụng cùng một cách tiếp cận cho các biểu tượng còn lại, chỉ cần thêm dữ liệu vào cả các bản kiểm thử và mảng các biểu tượng của chúng ta.

Mã nguồn của bạn có hoạt động cho số `1984`: `MCMLXXXIV` không?

Dưới đây là bộ test cuối cùng của tôi:

```go
func TestRomanNumerals(t *testing.T) {
	cases := []struct {
		Arabic int
		Roman  string
	}{
		{Arabic: 1, Roman: "I"},
		{Arabic: 2, Roman: "II"},
		{Arabic: 3, Roman: "III"},
		{Arabic: 4, Roman: "IV"},
		{Arabic: 5, Roman: "V"},
		{Arabic: 6, Roman: "VI"},
		{Arabic: 7, Roman: "VII"},
		{Arabic: 8, Roman: "VIII"},
		{Arabic: 9, Roman: "IX"},
		{Arabic: 10, Roman: "X"},
		{Arabic: 14, Roman: "XIV"},
		{Arabic: 18, Roman: "XVIII"},
		{Arabic: 20, Roman: "XX"},
		{Arabic: 39, Roman: "XXXIX"},
		{Arabic: 40, Roman: "XL"},
		{Arabic: 47, Roman: "XLVII"},
		{Arabic: 49, Roman: "XLIX"},
		{Arabic: 50, Roman: "L"},
		{Arabic: 100, Roman: "C"},
		{Arabic: 90, Roman: "XC"},
		{Arabic: 400, Roman: "CD"},
		{Arabic: 500, Roman: "D"},
		{Arabic: 900, Roman: "CM"},
		{Arabic: 1000, Roman: "M"},
		{Arabic: 1984, Roman: "MCMLXXXIV"},
		{Arabic: 3999, Roman: "MMMCMXCIX"},
		{Arabic: 2014, Roman: "MMXIV"},
		{Arabic: 1006, Roman: "MVI"},
		{Arabic: 798, Roman: "DCCXCVIII"},
	}
	for _, test := range cases {
		t.Run(fmt.Sprintf("%d gets converted to %q", test.Arabic, test.Roman), func(t *testing.T) {
			got := ConvertToRoman(test.Arabic)
			if got != test.Roman {
				t.Errorf("got %q, want %q", got, test.Roman)
			}
		})
	}
}
```

- Tôi đã xóa trường `description` vì tôi cảm thấy *dữ liệu* đã mô tả đủ thông tin.
- Tôi đã thêm một vài trường hợp biên khác mà tôi tìm thấy để có thêm sự tự tin. Với table based tests, việc này thực hiện rất nhanh chóng.

Tôi không thay đổi thuật toán, tất cả những gì tôi phải làm là cập nhật mảng `allRomanNumerals`.

```go
var allRomanNumerals = []RomanNumeral{
	{1000, "M"},
	{900, "CM"},
	{500, "D"},
	{400, "CD"},
	{100, "C"},
	{90, "XC"},
	{50, "L"},
	{40, "XL"},
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}
```

## Phân tích (Parsing) Chữ số La Mã

Chúng ta vẫn chưa xong. Tiếp theo, chúng ta sẽ viết một hàm chuyển đổi *từ* một Chữ số La Mã sang một số `int`.

## Viết test trước tiên

Chúng ta có thể sử dụng lại các kịch bản kiểm thử của mình ở đây với một chút tái cấu trúc.

Di chuyển biến `cases` ra ngoài test làm biến package trong một khối `var`.

```go
func TestConvertingToArabic(t *testing.T) {
	for _, test := range cases[:1] {
		t.Run(fmt.Sprintf("%q gets converted to %d", test.Roman, test.Arabic), func(t *testing.T) {
			got := ConvertToArabic(test.Roman)
			if got != test.Arabic {
				t.Errorf("got %d, want %d", got, test.Arabic)
			}
		})
	}
}
```

Lưu ý rằng tôi đang sử dụng chức năng slice để chỉ chạy một trong các test ngay lúc này (`cases[:1]`) vì cố gắng làm cho tất cả các test đó vượt qua cùng một lúc là bước tiến quá lớn.

## Thử chạy test

```console
./numeral_test.go:60:11: undefined: ConvertToArabic
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Thêm định nghĩa hàm mới:

```go
func ConvertToArabic(roman string) int {
	return 0
}
```

Bản kiểm thử bây giờ sẽ chạy được và thất bại:

```console
--- FAIL: TestConvertingToArabic (0.00s)
    --- FAIL: TestConvertingToArabic/'I'_gets_converted_to_1 (0.00s)
        numeral_test.go:62: got 0, want 1
```

## Viết đủ code để test chạy thành công

Bạn biết phải làm gì rồi:

```go
func ConvertToArabic(roman string) int {
	return 1
}
```

Tiếp theo, hãy thay đổi chỉ số slice trong test của chúng ta để chuyển sang kịch bản tiếp theo (ví dụ: `cases[:2]`). Hãy tự mình làm cho nó vượt qua bằng mã nguồn ngớ ngẩn nhất mà bạn có thể nghĩ ra, tiếp tục viết mã ngớ ngẩn (cuốn sách hay nhất từ trước đến nay phải không nào?) cho cả trường hợp thứ ba. Đây là mã ngớ ngẩn của tôi:

```go
func ConvertToArabic(roman string) int {
	if roman == "III" {
		return 3
	}
	if roman == "II" {
		return 2
	}
	return 1
}
```

Thông qua sự ngớ ngẩn của *mã nguồn thực sự hoạt động*, chúng ta bắt đầu thấy một mẫu hình giống như trước đây. Chúng ta cần lặp qua đầu vào và xây dựng một thứ gì đó, trong trường hợp này là một tổng giá trị.

```go
func ConvertToArabic(roman string) int {
	total := 0
	for range roman {
		total++
	}
	return total
}
```

## Viết test trước tiên

Tiếp theo chúng ta chuyển sang `cases[:4]` (`IV`), hiện tại test sẽ thất bại vì nó nhận lại kết quả là 2 vốn là độ dài của chuỗi.

## Viết đủ code để test chạy thành công

```go
// đoạn trước..
var allRomanNumerals = RomanNumerals{
	{1000, "M"},
	{900, "CM"},
	{500, "D"},
	{400, "CD"},
	{100, "C"},
	{90, "XC"},
	{50, "L"},
	{40, "XL"},
	{10, "X"},
	{9, "IX"},
	{5, "V"},
	{4, "IV"},
	{1, "I"},
}

// đoạn sau..
func ConvertToArabic(roman string) int {
	var arabic = 0

	for _, numeral := range allRomanNumerals {
		for strings.HasPrefix(roman, numeral.Symbol) {
			arabic += numeral.Value
			roman = strings.TrimPrefix(roman, numeral.Symbol)
		}
	}

	return arabic
}
```

Về cơ bản đây là thuật toán của `ConvertToRoman(int)` được thực hiện theo chiều ngược lại. Tại đây, chúng ta lặp qua chuỗi chữ số La Mã được cung cấp:
- Chúng ta tìm các biểu tượng chữ số La Mã lấy từ `allRomanNumerals`, từ cao đến thấp, ở đầu chuỗi.
- Nếu tìm thấy tiền tố (prefix) phù hợp, chúng ta thêm giá trị của nó vào `arabic` và cắt bỏ phần tiền tố đó.

Cuối cùng, chúng ta trả về tổng dưới dạng số Ả Rập.

Hàm `HasPrefix(s, prefix)` kiểm tra xem chuỗi `s` có bắt đầu bằng `prefix` hay không và `TrimPrefix(s, prefix)` gỡ bỏ `prefix` khỏi `s`, để chúng ta có thể tiến hành với các biểu tượng chữ số La Mã còn lại. Nó hoạt động với `IV` và tất cả các trường hợp kiểm thử khác.

Bạn có thể triển khai điều này như một hàm đệ quy, cái mà sẽ thanh thoát hơn (theo cảm nhận của tôi) nhưng có thể chậm hơn. Tôi sẽ để việc này lại cho bạn cùng một số test `Benchmark...`.

Bây giờ chúng ta đã có các hàm để chuyển đổi số Ả Rập sang chữ số La Mã và ngược lại, chúng ta có thể đưa các test đi xa hơn một chút:

## Giới thiệu về property based tests (kiểm thử dựa trên thuộc tính)

Đã có một vài quy tắc trong miền kiến thức về Chữ số La Mã mà chúng ta đã làm việc trong chương này:

- Không thể có quá 3 biểu tượng liên tiếp
- Chỉ có I (1), X (10) và C (100) có thể là "số trừ"
- Lấy kết quả của `ConvertToRoman(N)` và truyền nó vào `ConvertToArabic` sẽ nhận lại kết quả là `N`

Các bản kiểm thử chúng ta đã viết cho đến nay có thể được mô tả là các bản kiểm thử dựa trên "ví dụ" (example based tests), nơi chúng ta cung cấp *ví dụ* cho bộ công cụ để xác minh.

Điều gì sẽ xảy ra nếu chúng ta có thể lấy những quy tắc mà mình biết về lĩnh vực này và bằng cách nào đó thử nghiệm chúng dựa trên mã nguồn của mình?

Kiểm thử dựa trên thuộc tính giúp bạn thực hiện điều này bằng cách đưa dữ liệu ngẫu nhiên vào mã nguồn và xác minh xem các quy tắc bạn mô tả có luôn đúng hay không. Nhiều người nghĩ rằng kiểm thử dựa trên thuộc tính chủ yếu về dữ liệu ngẫu nhiên nhưng họ đã lầm. Thử thách thực sự đối với kiểm thử dựa trên thuộc tính là phải có một hiểu biết *tốt* về miền kiến thức để bạn có thể viết các thuộc tính này.

Nói đủ rồi, hãy xem một số mã nguồn:

```go
func TestPropertiesOfConversion(t *testing.T) {
	assertion := func(arabic int) bool {
		roman := ConvertToRoman(arabic)
		fromRoman := ConvertToArabic(roman)
		return fromRoman == arabic
	}

	if err := quick.Check(assertion, nil); err != nil {
		t.Error("failed checks", err)
	}
}
```

### Lý do của thuộc tính

Bản kiểm thử đầu tiên của chúng ta sẽ kiểm tra xem nếu chúng ta chuyển đổi một số thành chữ số La Mã, khi chúng ta sử dụng hàm khác để chuyển đổi nó trở lại thành một con số thì chúng ta sẽ nhận được kết quả ban đầu của mình.

- Với một số ngẫu nhiên (ví dụ `4`).
- Gọi `ConvertToRoman` với số ngẫu nhiên đó (nên trả về `IV` nếu là `4`).
- Lấy kết quả trên và truyền nó vào `ConvertToArabic`.
- Kết quả trên sẽ mang lại cho chúng ta đầu vào ban đầu (`4`).

Đây cảm giác như là một test tốt để xây dựng niềm tin cho chúng ta vì nó sẽ bị lỗi nếu có bug ở một trong hai hàm. Cách duy nhất để nó vượt qua là cả hai cùng có một kiểu bug giống nhau; điều này không phải là không thể nhưng cảm giác khó xảy ra.

### Giải thích kỹ thuật

Chúng ta đang sử dụng package [testing/quick](https://golang.org/pkg/testing/quick/) từ thư viện chuẩn.

Đọc từ dưới lên, chúng ta cung cấp cho `quick.Check` một hàm mà nó sẽ chạy dựa trên một số đầu vào ngẫu nhiên, nếu hàm trả về `false` thì nó sẽ được xem là kiểm tra thất bại.

Hàm `assertion` của chúng ta ở trên lấy một số ngẫu nhiên và chạy các hàm của chúng ta để kiểm tra thuộc tính.

### Chạy test

Thử chạy nó; máy tính của bạn có thể bị treo một lúc, vì vậy hãy dừng nó khi bạn thấy chán :)

Chuyện gì đang xảy ra? Hãy thử thêm đoạn sau vào mã nguồn xác nhận:

```go
assertion := func(arabic int) bool {
	if arabic < 0 || arabic > 3999 {
		log.Println(arabic)
		return true
	}
	roman := ConvertToRoman(arabic)
	fromRoman := ConvertToArabic(roman)
	return fromRoman == arabic
}
```

Bạn sẽ thấy một thứ gì đó như thế này:

```console
=== RUN   TestPropertiesOfConversion
2019/07/09 14:41:27 6849766357708982977
2019/07/09 14:41:27 -7028152357875163913
2019/07/09 14:41:27 -6752532134903680693
2019/07/09 14:41:27 4051793897228170080
2019/07/09 14:41:27 -1111868396280600429
2019/07/09 14:41:27 8851967058300421387
2019/07/09 14:41:27 562755830018219185
```

Chỉ cần chạy thuộc tính rất đơn giản này đã làm bộc lộ một lỗ hổng trong quá trình thực hiện của chúng ta. Chúng ta đã sử dụng `int` làm đầu vào nhưng:
- Bạn không thể sử dụng số âm với Chữ số La Mã.
- Theo quy tắc tối đa 3 biểu tượng liên tiếp, chúng ta không thể biểu diễn một giá trị lớn hơn 3999 ([à, thực ra là có thể](https://www.quora.com/Which-is-the-maximum-number-in-Roman-numerals)) và `int` có giá trị tối đa cao hơn nhiều so với 3999.

Điều này thật tuyệt vời! Chúng ta đã bị buộc phải suy nghĩ sâu sắc hơn về miền kiến thức của mình, đây là một sức mạnh thực sự của kiểm thử dựa trên thuộc tính.

Rõ ràng `int` không phải là một kiểu dữ liệu tốt. Điều gì sẽ xảy ra nếu chúng ta thử một thứ gì đó phù hợp hơn một chút?

### [`uint16`](https://golang.org/pkg/builtin/#uint16)

Go có các kiểu dữ liệu cho *số nguyên không dấu* (unsigned integers), nghĩa là chúng không thể âm; điều đó giúp loại bỏ một loại bug trong mã nguồn của chúng ta ngay lập tức. Bằng cách thêm số 16, nó có nghĩa đó là một số nguyên 16 bit có thể lưu trữ tối đa `65535`, vẫn còn quá lớn nhưng giúp chúng ta tiến gần hơn đến những gì mình cần.

Thử cập nhật mã nguồn để sử dụng `uint16` thay vì `int`. Tôi đã cập nhật `assertion` trong test để hiển thị rõ ràng hơn một chút.

```go
assertion := func(arabic uint16) bool {
	if arabic > 3999 {
		return true
	}
	t.Log("testing", arabic)
	roman := ConvertToRoman(arabic)
	fromRoman := ConvertToArabic(roman)
	return fromRoman == arabic
}
```
Lưu ý rằng bây giờ chúng ta đang log đầu vào bằng phương thức `log` từ testing framework. Hãy đảm bảo bạn chạy lệnh `go test` với flag `-v` để in ra các output bổ sung (`go test -v`).

Nếu bạn chạy test, bây giờ chúng thực sự chạy và bạn có thể thấy những gì đang được kiểm thử. Bạn có thể chạy nhiều lần để thấy mã nguồn của chúng ta đứng vững trước các giá trị khác nhau! Điều này cho tôi rất nhiều sự tự tin rằng mã của mình đang hoạt động như mong muốn.

Số lần chạy mặc định mà `quick.Check` thực hiện là 100 nhưng bạn có thể thay đổi điều đó bằng một cấu hình.

```go
if err := quick.Check(assertion, &quick.Config{
	MaxCount: 1000,
}); err != nil {
	t.Error("failed checks", err)
}
```

### Các bước tiếp theo

- Bạn có thể viết các bản kiểm thử thuộc tính để kiểm tra các thuộc tính khác mà chúng ta đã mô tả không?
- Bạn có thể nghĩ ra cách nào để khiến ai đó không thể gọi mã của chúng ta với một con số lớn hơn 3999 không?
    - Bạn có thể trả về một lỗi.
    - Hoặc tạo một kiểu dữ liệu mới không thể thể hiện giá trị > 3999.
        - Bạn nghĩ cách nào là tốt nhất?

## Tổng kết

### Luyện tập thêm TDD với phát triển lặp

Suy nghĩ về việc viết mã nguồn chuyển đổi số 1984 thành MCMLXXXIV có khiến bạn cảm thấy e sợ ngay từ đầu không? Với tôi thì có, và tôi đã viết phần mềm khá lâu rồi.

Bí quyết, như mọi khi, là **bắt đầu với một thứ gì đó đơn giản** và thực hiện các **bước nhỏ**.

Tại bất kỳ thời điểm nào trong quy trình này, chúng ta không có những bước nhảy vọt lớn, không thực hiện bất kỳ đợt tái cấu trúc khổng lồ nào, hay rơi vào một mớ hỗn độn.

Tôi có thể nghe thấy ai đó nói một cách mỉa mai rằng "đây chỉ là một bài kata thôi mà". Tôi không phủ nhận điều đó, nhưng tôi vẫn áp dụng phương pháp tương tự này cho mọi dự án tôi thực hiện. Tôi không bao giờ xuất bản một hệ thống phân tán lớn ngay trong bước đầu tiên, tôi tìm thứ đơn giản nhất mà nhóm có thể xuất bản (thường là một trang web "Hello world") và sau đó lặp lại trên các mẩu chức năng nhỏ theo từng phần có thể quản lý được, giống như cách chúng ta đã làm ở đây.

Kỹ năng nằm ở việc biết *cách* phân chia công việc, và điều đó đi kèm với sự luyện tập và một chút TDD thú vị để giúp bạn trên con đường của mình.

### Kiểm thử dựa trên thuộc tính (Property based tests)

- Được tích hợp sẵn trong thư viện chuẩn.
- Nếu bạn có thể nghĩ ra cách mô tả các quy tắc miền kiến thức trong mã nguồn, chúng là một công cụ tuyệt vời mang lại cho bạn nhiều sự tự tin hơn.
- Buộc bạn phải suy nghĩ về miền kiến thức của mình một cách sâu sắc.
- Tiềm năng là một sự bổ sung tốt cho bộ test của bạn.

## Lời kết (Postscript)

Cuốn sách này phụ thuộc vào những phản hồi quý báu từ cộng đồng.
[Dave](http://github.com/gypsydave5) là một sự trợ giúp to lớn trong thực tế ở mọi chương. Nhưng anh ấy đã thực sự gay gắt về việc tôi sử dụng cụm từ 'Arabic numerals' (số Ả Rập) trong chương này, vì vậy, để minh bạch hoàn toàn, dưới đây là những gì anh ấy nói.

> Tôi sẽ viết ra lý do tại sao một giá trị của kiểu `int` không thực sự là một 'số Ả Rập'. Điều này có thể là do tôi quá chi tiết nên tôi hoàn toàn hiểu nếu ông bảo tôi biến đi.
>
> Một *chữ số* (digit) là một ký tự được dùng trong biểu diễn các con số - bắt nguồn từ tiếng Latin để chỉ 'ngón tay', vì chúng ta thường có mười ngón tay. Trong hệ thống số Ả Rập (còn được gọi là Hindu-Ả Rập), có mười chữ số. Những chữ số Ả Rập này là:
>
> ```console
>   0 1 2 3 4 5 6 7 8 9
> ```
>
> Một *số* (numeral) là sự đại diện của một số (number) sử dụng một tập hợp các chữ số. Một số Ả Rập là một số được đại diện bởi các chữ số Ả Rập trong hệ cơ số 10. Chúng ta gọi là 'vị trí' (positional) vì mỗi chữ số có một giá trị khác nhau dựa trên vị trí của nó trong số đó. Vì vậy:
>
> ```console
>   1337
> ```
>
> Số `1` có giá trị là một nghìn vì nó là chữ số đầu tiên trong một con số có bốn chữ số.
>
> Chữ số La Mã được xây dựng bằng cách sử dụng một số lượng chữ số ít hơn (`I`, `V` v.v...) chủ yếu là các giá trị để tạo ra con số đó. Có một chút về vị trí nhưng phần lớn `I` luôn đại diện cho 'một'.
>
> Vậy, với điều này, liệu `int` có phải là một 'số Ả Rập'? Khái niệm về một con số không hề gắn liền với sự đại diện của nó - chúng ta có thể thấy điều này nếu tự hỏi bản thân xem đâu là sự đại diện đúng cho con số này:
>
> ```console
> 255
> 11111111
> hai trăm năm mươi lăm
> FF
> 377
> ```
>
> Đúng, đây là một câu hỏi mẹo. Tất cả đều đúng. Chúng đại diện cho cùng một con số lần lượt trong các hệ thống thập phân, nhị phân, tiếng Anh, thập lục phân và bát phân.
>
> Sự đại diện của một số dưới dạng chữ số là *độc lập* với các đặc tính của nó dưới dạng một con số - và chúng ta có thể thấy điều này khi nhìn vào các integer literals trong Go:
>
> ```go
> 	0xFF == 255 // true
> ```
>
> Và cách chúng ta in các số nguyên trong một chuỗi định dạng:
>
> ```go
> n := 255
> fmt.Printf("%b %c %d %o %q %x %X %U", n, n, n, n, n, n, n, n)
> // 11111111 ÿ 255 377 'ÿ' ff FF U+00FF
> ```
>
> Chúng ta có thể viết cùng một số nguyên cả dưới dạng thập lục phân và dưới dạng chữ số Ả Rập (thập phân).
>
> Vì vậy, khi signature của hàm trông như `ConvertToRoman(arabic int) string`, nó đang đưa ra một chút giả định về cách nó được gọi. Bởi vì đôi khi `arabic` sẽ được viết dưới dạng một literal số nguyên thập phân:
>
> ```go
> 	ConvertToRoman(255)
> ```
>
> Nhưng nó cũng có thể được viết là:
>
> ```go
> 	ConvertToRoman(0xFF)
> ```
>
> Thực sự, chúng ta hoàn toàn không 'chuyển đổi' từ một số Ả Rập, chúng ta đang 'in ra' - đại diện - một số `int` dưới dạng chữ số La Mã - và `int` không phải là chữ số, dù là Ả Rập hay kiểu khác; chúng chỉ là các con số. Hàm `ConvertToRoman` giống như `strconv.Itoa` ở chỗ nó chuyển một số `int` thành một `string`.
>
> Nhưng mọi phiên bản khác của bài kata đều không quan tâm đến sự phân biệt này nên :shrug:
