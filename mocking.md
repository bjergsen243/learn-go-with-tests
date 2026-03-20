# Mocking

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/mocking)**

Bạn được yêu cầu viết một chương trình đếm ngược từ 3, in mỗi số trên một dòng mới (với độ trễ 1 giây) và khi đạt đến số 0, nó sẽ in "Go!" và thoát.

```
3
2
1
Go!
```

Chúng ta sẽ giải quyết vấn đề này bằng cách viết một hàm gọi là `Countdown`, sau đó đưa nó vào chương trình `main` để nó trông giống như thế này:

```go
package main

func main() {
	Countdown()
}
```

Mặc dù đây là một chương trình khá đơn giản, nhưng để kiểm thử nó một cách đầy đủ, chúng ta sẽ cần phải tiếp cận theo cách *lặp đi lặp lại (iterative)* và *hướng kiểm thử (test-driven)* như mọi khi.

Ý tôi là gì khi nói lặp đi lặp lại? Chúng ta đảm bảo rằng mình thực hiện các bước nhỏ nhất có thể để có được *phần mềm hữu ích*.

Chúng ta không muốn dành nhiều thời gian cho đoạn mã mà trên lý thuyết sẽ hoạt động sau một hồi "vọc vạch", vì đó thường là cách các nhà phát triển rơi vào bế tắc. **Kỹ năng quan trọng là có thể chia nhỏ các yêu cầu thành mức nhỏ nhất có thể để bạn luôn có *phần mềm hoạt động được*.**

Đây là cách chúng ta có thể chia nhỏ công việc và lặp lại trên đó:

- In ra số 3
- In ra 3, 2, 1 và Go!
- Đợi một giây giữa mỗi dòng

## Viết test trước tiên

Phần mềm của chúng ta cần in ra stdout và chúng ta đã thấy cách sử dụng Dependency Injection (DI) để hỗ trợ kiểm thử điều này trong phần DI.

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}

	Countdown(buffer)

	got := buffer.String()
	want := "3"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

Nếu bạn thấy `buffer` còn lạ lẫm, hãy đọc lại [phần trước](dependency-injection.md).

Chúng ta biết mình muốn hàm `Countdown` ghi dữ liệu vào đâu đó và `io.Writer` là cách thực tế để nắm bắt điều đó dưới dạng một interface trong Go.

- Trong `main`, chúng ta sẽ gửi đến `os.Stdout` để người dùng thấy bộ đếm ngược được in ra terminal.
- Trong test, chúng ta sẽ gửi đến `bytes.Buffer` để các bản kiểm thử có thể nắm bắt dữ liệu nào đang được tạo ra.

## Thử chạy test

`./countdown_test.go:11:2: undefined: Countdown`

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Định nghĩa hàm `Countdown`:

```go
func Countdown() {}
```

Thử lại:

```text
./countdown_test.go:11:11: too many arguments in call to Countdown
    have (*bytes.Buffer)
    want ()
```

Compiler đang cho bạn biết signature (chữ ký) hàm của bạn có thể là gì, vì vậy hãy cập nhật nó.

```go
func Countdown(out *bytes.Buffer) {}
```

`countdown_test.go:17: got '' want '3'`

Hoàn hảo!

## Viết đủ code để test chạy thành công

```go
func Countdown(out *bytes.Buffer) {
	fmt.Fprint(out, "3")
}
```

Chúng ta đang sử dụng `fmt.Fprint` nhận một `io.Writer` (như `*bytes.Buffer`) và gửi một `string` đến đó. Test sẽ vượt qua.

## Refactor

Chúng ta biết rằng mặc dù `*bytes.Buffer` hoạt động, nhưng tốt hơn là sử dụng một interface tổng quát thay thế.

```go
func Countdown(out io.Writer) {
	fmt.Fprint(out, "3")
}
```

Chạy lại các test và chúng sẽ vượt qua.

Để hoàn tất, hãy kết nối hàm của chúng ta vào `main` để chúng ta có một phần mềm thực sự hoạt động, giúp củng cố niềm tin rằng chúng ta đang tiến bộ.

```go
package main

import (
	"fmt"
	"io"
	"os"
)

func Countdown(out io.Writer) {
	fmt.Fprint(out, "3")
}

func main() {
	Countdown(os.Stdout)
}
```

Hãy thử chạy chương trình và ngạc nhiên trước công sức của bạn.

Đúng, điều này có vẻ tầm thường nhưng cách tiếp cận này là những gì tôi khuyên dùng cho bất kỳ dự án nào. **Hãy lấy một phần nhỏ chức năng và làm cho nó hoạt động từ đầu đến cuối (end-to-end), được hỗ trợ bởi các bản kiểm thử.**

Tiếp theo, chúng ta có thể làm cho nó in ra 2, 1 và sau đó là "Go!".

## Viết test trước tiên

Bằng cách đầu tư vào việc làm cho hệ thống "đường ống" tổng thể hoạt động đúng, chúng ta có thể lặp lại giải pháp của mình một cách an toàn và dễ dàng. Chúng ta sẽ không còn cần phải dừng lại và chạy lại chương trình để tin tưởng vào hoạt động của nó vì tất cả logic đã được kiểm thử.

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}

	Countdown(buffer)

	got := buffer.String()
	want := `3
2
1
Go!`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

Cú pháp dấu huyền (backtick) là một cách khác để tạo `string` nhưng cho phép bạn bao gồm những thứ như dòng mới, điều này hoàn hảo cho test của chúng ta.

## Thử chạy test

```
countdown_test.go:21: got '3' want '3
        2
        1
        Go!'
```

## Viết đủ code để test chạy thành công

```go
func Countdown(out io.Writer) {
	for i := 3; i > 0; i-- {
		fmt.Fprintln(out, i)
	}
	fmt.Fprint(out, "Go!")
}
```

Sử dụng vòng lặp `for` đếm ngược với `i--` và sử dụng `fmt.Fprintln` để in ra `out` với số của chúng ta kèm theo một ký tự dòng mới. Cuối cùng, sử dụng `fmt.Fprint` để gửi "Go!" sau đó.

## Refactor

Không có nhiều thứ để refactor ngoại trừ việc chuyển một số giá trị cụ thể (magic values) thành các hằng số có tên.

```go
const finalWord = "Go!"
const countdownStart = 3

func Countdown(out io.Writer) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
	}
	fmt.Fprint(out, finalWord)
}
```

Nếu bạn chạy chương trình ngay bây giờ, bạn sẽ nhận được kết quả mong muốn nhưng chúng ta chưa có hiệu ứng đếm ngược kịch tính với khoảng nghỉ 1 giây.

Go cho phép bạn đạt được điều này với `time.Sleep`. Hãy thử thêm nó vào mã của chúng ta.

```go
func Countdown(out io.Writer) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		time.Sleep(1 * time.Second)
	}

	fmt.Fprint(out, finalWord)
}
```

Nếu bạn chạy chương trình, nó sẽ hoạt động theo đúng ý muốn.

## Mocking

Các test vẫn vượt qua và phần mềm hoạt động như dự định nhưng chúng ta có một số vấn đề:
- Các test của chúng ta mất 3 giây để chạy.
    - Mọi bài viết có tầm nhìn xa về phát triển phần mềm đều nhấn mạnh tầm quan trọng của vòng lặp phản hồi nhanh (quick feedback loops).
    - **Các bản kiểm thử chậm chạp làm hủy hoại năng suất của nhà phát triển**.
    - Hãy tưởng tượng nếu các yêu cầu trở nên phức tạp hơn, dẫn đến cần nhiều test hơn. Liệu chúng ta có hài lòng với 3 giây cộng thêm vào lần chạy test cho mỗi test mới của `Countdown` không?
- Chúng ta chưa kiểm thử một đặc tính quan trọng của hàm.

Chúng ta có một phụ thuộc vào việc `Sleep`, chúng ta cần trích xuất nó ra để có thể kiểm soát nó trong các bản kiểm thử.

Nếu chúng ta có thể *mock* `time.Sleep`, chúng ta có thể sử dụng *dependency injection* để sử dụng nó thay vì `time.Sleep` "thực" và sau đó chúng ta có thể **spy (theo dõi) các cuộc gọi** để thực hiện các xác nhận (assertions) trên chúng.

## Viết test trước tiên

Hãy định nghĩa phụ thuộc của chúng ta dưới dạng một interface. Điều này cho phép chúng ta sử dụng một Sleeper *thực* trong `main` và một *spy sleeper* trong các bản kiểm thử của chúng ta. Bằng cách sử dụng một interface, hàm `Countdown` của chúng ta không quan tâm đến điều này và mang lại sự linh hoạt cho người gọi.

```go
type Sleeper interface {
	Sleep()
}
```

Tôi đã đưa ra quyết định thiết kế rằng hàm `Countdown` của chúng ta sẽ không chịu trách nhiệm về thời gian chờ là bao lâu. Điều này làm cho mã của chúng ta đơn giản hơn một chút (ít nhất là bây giờ) và có nghĩa là người dùng hàm của chúng ta có thể cấu hình thời gian chờ đó theo bất kỳ cách nào họ thích.

Bây giờ chúng ta cần tạo một *mock* của nó để các test sử dụng.

```go
type SpySleeper struct {
	Calls int
}

func (s *SpySleeper) Sleep() {
	s.Calls++
}
```

*Spies* (Giám sát) là một loại *mock* có thể ghi lại cách một phụ thuộc được sử dụng. Chúng có thể ghi lại các đối số được truyền vào, số lần nó được gọi, v.v. Trong trường hợp của chúng ta, chúng ta đang theo dõi số lần `Sleep()` được gọi để có thể kiểm tra nó trong test.

Cập nhật các test để tiêm (inject) một phụ thuộc vào Spy của chúng ta và xác nhận rằng việc sleep đã được gọi 3 lần.

```go
func TestCountdown(t *testing.T) {
	buffer := &bytes.Buffer{}
	spySleeper := &SpySleeper{}

	Countdown(buffer, spySleeper)

	got := buffer.String()
	want := `3
2
1
Go!`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}

	if spySleeper.Calls != 3 {
		t.Errorf("not enough calls to sleeper, want 3 got %d", spySleeper.Calls)
	}
}
```

## Thử chạy test

```
too many arguments in call to Countdown
    have (*bytes.Buffer, *SpySleeper)
    want (io.Writer)
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Chúng ta cần cập nhật `Countdown` để chấp nhận `Sleeper` của chúng ta.

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		time.Sleep(1 * time.Second)
	}

	fmt.Fprint(out, finalWord)
}
```

Nếu bạn thử lại, `main` của bạn sẽ không còn biên dịch được vì lý do tương tự:

```
./main.go:26:11: not enough arguments in call to Countdown
    have (*os.File)
    want (io.Writer, Sleeper)
```

Hãy tạo một sleeper *thực* triển khai interface chúng ta cần:

```go
type DefaultSleeper struct{}

func (d *DefaultSleeper) Sleep() {
	time.Sleep(1 * time.Second)
}
```

Sau đó chúng ta có thể sử dụng nó trong ứng dụng thực của mình như sau:

```go
func main() {
	sleeper := &DefaultSleeper{}
	Countdown(os.Stdout, sleeper)
}
```

## Viết đủ code để test chạy thành công

Test hiện đã biên dịch được nhưng không vượt qua vì chúng ta vẫn đang gọi `time.Sleep` thay vì phụ thuộc được tiêm vào. Hãy sửa điều đó.

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
		sleeper.Sleep()
	}

	fmt.Fprint(out, finalWord)
}
```

Test sẽ vượt qua và không còn mất 3 giây nữa.

### Vẫn còn một số vấn đề

Vẫn còn một đặc tính quan trọng khác mà chúng ta chưa kiểm thử.

`Countdown` nên chờ trước mỗi lần in tiếp theo, ví dụ:

- `In N`
- `Chờ`
- `In N-1`
- `Chờ`
- `In Go!`
- v.v.

Thay đổi mới nhất của chúng ta chỉ xác nhận rằng nó đã chờ 3 lần, nhưng những lần chờ đó có thể xảy ra sai thứ tự.

Khi viết các bản kiểm thử, nếu bạn không tin rằng các bản kiểm thử của mình đang mang lại sự tin cậy đầy đủ, hãy cứ làm cho nó bị lỗi! (hãy đảm bảo rằng bạn đã commit các thay đổi của mình trước đó). Thay đổi mã thành như sau:

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := countdownStart; i > 0; i-- {
		sleeper.Sleep()
	}

	for i := countdownStart; i > 0; i-- {
		fmt.Fprintln(out, i)
	}

	fmt.Fprint(out, finalWord)
}
```

Nếu bạn chạy các test, chúng vẫn sẽ vượt qua mặc dù việc triển khai là sai.

Hãy sử dụng lại kỹ thuật spying với một test mới để kiểm tra xem thứ tự các hoạt động có đúng hay không.

Chúng ta có hai phụ thuộc khác nhau và chúng ta muốn ghi lại tất cả các hoạt động của chúng vào một danh sách. Vì vậy, chúng ta sẽ tạo *một spy cho cả hai*.

```go
type SpyCountdownOperations struct {
	Calls []string
}

func (s *SpyCountdownOperations) Sleep() {
	s.Calls = append(s.Calls, sleep)
}

func (s *SpyCountdownOperations) Write(p []byte) (n int, err error) {
	s.Calls = append(s.Calls, write)
	return
}

const write = "write"
const sleep = "sleep"
```

`SpyCountdownOperations` của chúng ta triển khai cả `io.Writer` và `Sleeper`, ghi lại mọi cuộc gọi vào một slice. Trong test này, chúng ta chỉ quan tâm đến thứ tự các hoạt động, vì vậy chỉ cần ghi lại chúng dưới dạng danh sách các hoạt động có tên là đủ.

Bây giờ chúng ta có thể thêm một sub-test (test phụ) vào bộ kiểm thử của mình để xác minh rằng các lần chờ và lần in hoạt động theo thứ tự mà chúng ta mong muốn.

```go
t.Run("sleep before every print", func(t *testing.T) {
	spySleepPrinter := &SpyCountdownOperations{}
	Countdown(spySleepPrinter, spySleepPrinter)

	want := []string{
		write,
		sleep,
		write,
		sleep,
		write,
		sleep,
		write,
	}

	if !reflect.DeepEqual(want, spySleepPrinter.Calls) {
		t.Errorf("wanted calls %v got %v", want, spySleepPrinter.Calls)
	}
})
```

Test này bây giờ sẽ thất bại. Hãy hoàn tác `Countdown` về như cũ để sửa test.

Hiện tại chúng ta có hai test đang giám sát `Sleeper`, vì vậy bây giờ chúng ta có thể cấu trúc lại test của mình để một cái kiểm thử những gì đang được in ra và cái còn lại đảm bảo rằng chúng ta đang chờ giữa các lần in. Cuối cùng, chúng ta có thể xóa spy đầu tiên của mình vì nó không còn được sử dụng nữa.

```go
func TestCountdown(t *testing.T) {

	t.Run("prints 3 to Go!", func(t *testing.T) {
		buffer := &bytes.Buffer{}
		Countdown(buffer, &SpyCountdownOperations{})

		got := buffer.String()
		want := `3
2
1
Go!`

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})

	t.Run("sleep before every print", func(t *testing.T) {
		spySleepPrinter := &SpyCountdownOperations{}
		Countdown(spySleepPrinter, spySleepPrinter)

		want := []string{
			write,
			sleep,
			write,
			sleep,
			write,
			sleep,
			write,
		}

		if !reflect.DeepEqual(want, spySleepPrinter.Calls) {
			t.Errorf("wanted calls %v got %v", want, spySleepPrinter.Calls)
		}
	})
}
```

Bây giờ chúng ta có hàm của mình và 2 đặc tính quan trọng của nó đã được kiểm thử đúng cách.

## Mở rộng Sleeper để có thể cấu hình được

Một tính năng hay sẽ là `Sleeper` có thể cấu hình được. Điều này có nghĩa là chúng ta có thể điều chỉnh thời gian chờ trong chương trình chính của mình.

### Viết test trước tiên

Đầu tiên hãy tạo một kiểu mới cho `ConfigurableSleeper` chấp nhận những gì chúng ta cần cho cấu hình và kiểm thử.

```go
type ConfigurableSleeper struct {
	duration time.Duration
	sleep    func(time.Duration)
}
```

Chúng ta đang sử dụng `duration` để cấu hình thời gian chờ và `sleep` như một cách để truyền vào một hàm sleep. Signature của `sleep` giống với `time.Sleep`, cho phép chúng ta sử dụng `time.Sleep` trong quá trình triển khai thực tế và spy sau đây trong các bản kiểm thử:

```go
type SpyTime struct {
	durationSlept time.Duration
}

func (s *SpyTime) Sleep(duration time.Duration) {
	s.durationSlept = duration
}
```

Với spy của mình, chúng ta có thể tạo một test mới cho configurable sleeper.

```go
func TestConfigurableSleeper(t *testing.T) {
	sleepTime := 5 * time.Second

	spyTime := &SpyTime{}
	sleeper := ConfigurableSleeper{sleepTime, spyTime.Sleep}
	sleeper.Sleep()

	if spyTime.durationSlept != sleepTime {
		t.Errorf("should have slept for %v but slept for %v", sleepTime, spyTime.durationSlept)
	}
}
```

Không có gì mới trong test này và nó được thiết lập rất giống với các mock test trước đó.

### Thử chạy test
```
sleeper.Sleep undefined (type ConfigurableSleeper has no field or method Sleep, but does have sleep)

```

Bạn sẽ thấy một thông báo lỗi rất rõ ràng cho thấy chúng ta chưa tạo phương thức `Sleep` cho `ConfigurableSleeper`.

### Viết lượng mã tối thiểu để test chạy và kiểm tra kết quả lỗi
```go
func (c *ConfigurableSleeper) Sleep() {
}
```

Với hàm `Sleep` mới được triển khai, chúng ta có một test thất bại.

```
countdown_test.go:56: should have slept for 5s but slept for 0s
```

### Viết đủ code để test chạy thành công

Tất cả những gì chúng ta cần làm lúc này là triển khai hàm `Sleep` cho `ConfigurableSleeper`.

```go
func (c *ConfigurableSleeper) Sleep() {
	c.sleep(c.duration)
}
```

Với thay đổi này, tất cả các test sẽ vượt qua trở lại và bạn có thể tự hỏi tại sao lại phiền phức như vậy khi chương trình chính không thay đổi chút nào. Hy vọng nó sẽ trở nên rõ ràng sau phần sau.

### Dọn dẹp và tái cấu trúc (Cleanup and refactor)

Điều cuối cùng chúng ta cần làm là thực sự sử dụng `ConfigurableSleeper` trong hàm main.

```go
func main() {
	sleeper := &ConfigurableSleeper{1 * time.Second, time.Sleep}
	Countdown(os.Stdout, sleeper)
}
```

Nếu chúng ta chạy các test và chương trình theo cách thủ công, chúng ta có thể thấy rằng tất cả hành vi đều giữ nguyên.

Vì chúng ta đang sử dụng `ConfigurableSleeper`, bây giờ chúng ta có thể an tâm xóa implementation của `DefaultSleeper`. Tổng kết lại chương trình của chúng ta và có một Sleeper [tổng quát (generic)](https://stackoverflow.com/questions/19291776/whats-the-difference-between-abstraction-and-generalization) hơn với thời gian đếm ngược tùy ý.

## Nhưng chẳng phải Mocking là "quỷ dữ" sao?

Bạn có thể đã nghe nói rằng mocking là quỷ dữ. Giống như bất kỳ điều gì trong phát triển phần mềm, nó có thể được sử dụng sai mục đích, giống như [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself).

Mọi người thường rơi vào tình trạng tồi tệ khi họ không *lắng nghe các bản kiểm thử* và *không tôn trọng giai đoạn tái cấu trúc (refactoring stage)*.

Nếu mã mocking của bạn đang trở nên phức tạp hoặc bạn phải mock rất nhiều thứ để kiểm thử một điều gì đó, bạn nên *lắng nghe* cảm giác không ổn đó và suy nghĩ về mã của mình. Thông thường đó là dấu hiệu của:

- Thứ bạn đang kiểm thử phải làm quá nhiều việc (vì nó có quá nhiều phụ thuộc cần mock).
  - Hãy chia nhỏ module ra để nó làm ít việc hơn.
- Các phụ thuộc của nó quá phân tán (too fine-grained).
  - Hãy nghĩ về cách bạn có thể hợp nhất một số phụ thuộc này thành một module có ý nghĩa.
- Test của bạn quá quan tâm đến các chi tiết thực hiện (implementation details).
  - Hãy ưu tiên kiểm thử hành vi mong đợi thay vì cách thức thực hiện.

Thông thường việc mock quá nhiều chỉ ra sự *trừu tượng tồi (bad abstraction)* trong mã của bạn.

**Những gì mọi người thấy ở đây là một điểm yếu trong TDD nhưng thực chất nó là một điểm mạnh**, thông thường mã kiểm thử kém là kết quả của thiết kế tồi, hay nói cách khác, mã được thiết kế tốt thì dễ kiểm thử.

### Nhưng mock và test vẫn đang làm khó tôi!

Bạn đã bao giờ rơi vào tình huống này chưa?

- Bạn muốn tái cấu trúc một chút mã.
- Để làm điều này, bạn cuối cùng phải thay đổi rất nhiều test.
- Bạn nghi ngờ TDD và viết một bài trên Medium có tiêu đề "Mocking considered harmful".

Đây thường là dấu hiệu cho thấy bạn đang kiểm thử quá nhiều *chi tiết thực hiện*. Hãy cố gắng làm cho các test của bạn kiểm thử *hành vi hữu ích* trừ khi việc thực hiện là thực sự quan trọng đối với cách hệ thống vận hành.

Đôi khi thật khó để biết *mức độ nào* để kiểm thử chính xác nhưng đây là một số quá trình suy nghĩ và quy tắc tôi cố gắng tuân theo:

- **Định nghĩa của việc tái cấu trúc (refactoring) là mã thay đổi nhưng hành vi vẫn giữ nguyên**. Nếu bạn đã quyết định thực hiện một số hoạt động tái cấu trúc, trên lý thuyết bạn nên có thể tạo commit mà không cần thay đổi bất kỳ bản kiểm thử nào. Vì vậy khi viết một test, hãy tự hỏi:
  - Tôi đang kiểm thử hành vi mình muốn, hay các chi tiết thực hiện?
  - Nếu tôi tái cấu trúc mã này, tôi có phải thay đổi nhiều ở các bản kiểm thử không?
- Mặc dù Go cho phép bạn kiểm thử các hàm private, tôi khuyên bạn nên tránh điều đó vì các hàm private là chi tiết thực hiện để hỗ trợ hành vi public. Hãy kiểm thử hành vi public. Sandi Metz mô tả các hàm private là "kém ổn định hơn" và bạn không muốn liên kết các bản kiểm thử của mình với chúng.
- Tôi cảm thấy rằng nếu một test làm việc với **nhiều hơn 3 mock thì đó là một dấu hiệu cảnh báo (red flag)** - đã đến lúc phải suy nghĩ lại về thiết kế.
- Sử dụng spies cẩn thận. Spies cho phép bạn nhìn thấy bên trong thuật toán bạn đang viết, điều này có thể rất hữu ích nhưng đồng nghĩa với việc liên kết chặt chẽ hơn giữa mã kiểm thử và mã triển khai. **Hãy chắc chắn rằng bạn thực sự quan tâm đến những chi tiết này nếu bạn định spy lên chúng.**

#### Tôi có thể sử dụng một framework mocking không?

Mocking không đòi hỏi phép thuật gì và tương đối đơn giản; việc sử dụng một framework có thể làm cho mocking có vẻ phức tạp hơn thực tế. Chúng ta không sử dụng automocking trong chương này để chúng ta:

- hiểu rõ hơn về cách mock.
- thực hành triển khai các interface.

Trong các dự án hợp tác, việc tự động tạo mock có giá trị riêng. Trong một nhóm, một công cụ tạo mock sẽ thống nhất tính nhất quán xung quanh các "test doubles". Điều này sẽ tránh việc các test doubles được viết không nhất quán, có thể dẫn đến các bản kiểm thử không nhất quán.

Bạn chỉ nên sử dụng một trình tạo mock dựa trên một interface. Bất kỳ công cụ nào đưa ra quá nhiều quy định về cách viết test, hoặc sử dụng nhiều "phép thuật", đều không đáng dùng.

## Tổng kết

### Tìm hiểu thêm về cách tiếp cận TDD

- Khi đối mặt với các ví dụ ít tầm thường hơn, hãy chia nhỏ vấn đề thành các "lát cắt dọc mỏng" (thin vertical slices). Hãy cố gắng đạt đến điểm mà bạn có *phần mềm hoạt động được hỗ trợ bởi các bản kiểm thử* càng sớm càng tốt, để tránh rơi vào bế tắc và áp dụng cách tiếp cận "vụ nổ lớn" (big bang).
- Khi bạn đã có một số phần mềm hoạt động được, việc *lặp lại với các bước nhỏ* sẽ dễ dàng hơn cho đến khi bạn đạt được phần mềm mình cần.

> "Khi nào nên sử dụng phát triển lặp đi lặp lại? Bạn chỉ nên sử dụng phát triển lặp đi lặp lại trong các dự án mà bạn muốn thành công."
>
> Martin Fowler.

### Mocking

- **Nếu không có mocking, các phần quan trọng trong mã của bạn sẽ không được kiểm thử**. Trong trường hợp của chúng ta, chúng ta sẽ không thể kiểm thử rằng mã của mình đã tạm dừng giữa mỗi lần in, và còn vô số ví dụ khác. Gọi một dịch vụ *có thể* thất bại? Muốn kiểm thử hệ thống của bạn ở một trạng thái cụ thể? Rất khó để kiểm thử các kịch bản này mà không có mocking.
- Nếu không có mock, bạn có thể phải thiết đặt cơ sở dữ liệu và những thứ của bên thứ ba khác chỉ để kiểm thử các quy tắc nghiệp vụ đơn giản. Bạn có thể sẽ gặp các bản kiểm thử chậm chạp, dẫn đến **vòng lặp phản hồi chậm**.
- Bằng việc phải khởi động một cơ sở dữ liệu hoặc một dịch vụ web để kiểm thử điều gì đó, bạn có khả năng gặp phải **các bản kiểm thử dễ gãy (fragile tests)** do sự không đáng tin cậy của các dịch vụ đó.

Một khi nhà phát triển tìm hiểu về mocking, họ rất dễ sa đà vào việc kiểm thử quá mức từng khía cạnh nhỏ của một hệ thống về *cách thức nó hoạt động* thay vì *những gì nó làm*. Luôn lưu tâm về **giá trị của các bản kiểm thử của bạn** và tác động của chúng đối với việc tái cấu trúc trong tương lai.

Trong bài viết này về mocking, chúng ta mới chỉ đề cập đến **Spies**, một loại của mock. Mocks là một loại của "test double."

> [Test Double là một thuật ngữ chung cho bất kỳ trường hợp nào bạn thay thế một đối tượng thực (production object) cho mục đích kiểm thử.](https://martinfowler.com/bliki/TestDouble.html)

Dưới test doubles, có nhiều loại khác nhau như stubs, spies và thực sự là mocks! Hãy xem bài viết của [Martin Fowler](https://martinfowler.com/bliki/TestDouble.html) để biết thêm chi tiết.

## Bonus - Ví dụ về iterators từ Go 1.23

Trong Go 1.23 [iterators đã được giới thiệu](https://tip.golang.org/doc/go1.23). Chúng ta có thể sử dụng iterators theo nhiều cách khác nhau, trong trường hợp này chúng ta có thể tạo một iterator `countdownFrom`, nó sẽ trả về các số để đếm ngược theo thứ tự đảo ngược.

Trước khi đi sâu vào cách viết các iterator tùy chỉnh, hãy xem cách sử dụng chúng. Thay vì viết một vòng lặp trông khá khô khan để đếm ngược từ một số, chúng ta có thể làm cho đoạn mã này trở nên ý nghĩa hơn bằng cách dùng `range` trên iterator `countdownFrom` tùy chỉnh của chúng ta.

```go
func Countdown(out io.Writer, sleeper Sleeper) {
	for i := range countDownFrom(3) {
		fmt.Fprintln(out, i)
		sleeper.Sleep()
	}

	fmt.Fprint(out, finalWord)
}
```

Để viết một iterator như `countDownFrom`, bạn cần viết một hàm theo một cách cụ thể. Từ tài liệu:

    Mệnh đề "range" trong vòng lặp "for-range" giờ đây chấp nhận các hàm iterator thuộc các kiểu sau:
        func(func() bool)
        func(func(K) bool)
        func(func(K, V) bool)

(Phần `K` và `V` lần lượt đại diện cho các kiểu key và value.)

Trong trường hợp của chúng ta, chúng ta không có key, chỉ có các giá trị. Go cũng cung cấp một kiểu tiện lợi `iter.Seq[T]`, là một bí danh kiểu cho `func(func(T) bool)`.

```go
func countDownFrom(from int) iter.Seq[int] {
	return func(yield func(int) bool) {
		for i := from; i > 0; i-- {
			if !yield(i) {
				return
			}
		}
	}
}
```

Đây là một iterator đơn giản, nó sẽ cung cấp các số theo thứ tự đảo ngược, bắt đầu từ `from` - hoàn hảo cho trường hợp sử dụng của chúng ta.