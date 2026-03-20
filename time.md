# Time

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/time)**

Product owner muốn chúng ta mở rộng chức năng của ứng dụng command line bằng cách giúp một nhóm người chơi Texas-Holdem Poker.

## Kiến thức cơ bản về poker

Bạn không cần biết nhiều về poker, chỉ cần biết rằng sau mỗi khoảng thời gian nhất định, tất cả người chơi cần được thông báo về giá trị "blind" (mức cược mù) tăng dần.

Ứng dụng của chúng ta sẽ giúp theo dõi khi nào blind cần tăng và mức tăng là bao nhiêu.

- Khi bắt đầu, ứng dụng hỏi có bao nhiêu người chơi. Số người chơi quyết định khoảng thời gian trước khi mức cược "blind" tăng lên.
  - Thời gian cơ bản là 5 phút.
  - Mỗi người chơi thêm 1 phút.
  - Ví dụ: 6 người chơi tương đương 11 phút cho mỗi blind.
- Sau khi hết thời gian blind, trò chơi sẽ thông báo cho người chơi mức blind mới.
- Blind bắt đầu ở 100 chip, sau đó 200, 400, 600, 1000, 2000 và tiếp tục nhân đôi cho đến khi trò chơi kết thúc (chức năng "Ruth wins" trước đó vẫn kết thúc trò chơi như bình thường)

## Nhắc lại về code

Trong chương trước, chúng ta đã bắt đầu xây dựng ứng dụng command line với khả năng chấp nhận lệnh `{name} wins`. Đây là code `CLI` hiện tại, nhưng hãy nhớ tìm hiểu cả những phần code khác trước khi bắt đầu.

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
}

func NewCLI(store PlayerStore, in io.Reader) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
	}
}

func (cli *CLI) PlayPoker() {
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}

func (cli *CLI) readLine() string {
	cli.in.Scan()
	return cli.in.Text()
}
```


### `time.AfterFunc`

Chúng ta muốn lên lịch cho chương trình in ra giá trị blind tại các khoảng thời gian nhất định, phụ thuộc vào số lượng người chơi.

Để giới hạn phạm vi công việc, chúng ta sẽ tạm bỏ qua phần số người chơi và giả định có 5 người chơi. Chúng ta sẽ test rằng _cứ mỗi 10 phút, giá trị blind mới sẽ được in ra_.

Như thường lệ, thư viện chuẩn đã hỗ trợ sẵn với [`func AfterFunc(d Duration, f func()) *Timer`](https://golang.org/pkg/time/#AfterFunc)

> `AfterFunc` đợi cho đến khi khoảng thời gian trôi qua rồi gọi hàm f trong goroutine riêng của nó. Nó trả về một `Timer` có thể dùng để hủy lời gọi bằng phương thức Stop.

### [`time.Duration`](https://golang.org/pkg/time/#Duration)

> Duration đại diện cho thời gian trôi qua giữa hai thời điểm dưới dạng số nano giây int64.

Thư viện time có một số hằng số cho phép bạn nhân các nano giây đó để dễ đọc hơn cho các tình huống chúng ta sẽ gặp

```
5 * time.Second
```

Khi chúng ta gọi `PlayPoker`, chúng ta sẽ lên lịch tất cả các thông báo blind.

Tuy nhiên, việc test điều này có thể hơi khó. Chúng ta muốn xác minh rằng mỗi khoảng thời gian được lên lịch với đúng số tiền blind. Nhưng nếu bạn nhìn vào signature của `time.AfterFunc`, tham số thứ hai là hàm sẽ được chạy. Trong Go, bạn không thể so sánh các hàm với nhau, nên chúng ta không thể test hàm nào đã được truyền vào. Vì vậy, chúng ta cần viết một wrapper (lớp bọc) quanh `time.AfterFunc` nhận thời gian chạy và số tiền cần in để chúng ta có thể spy (theo dõi) nó.

## Viết test trước

Thêm một test mới vào bộ test

```go
t.Run("it schedules printing of blind values", func(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &poker.StubPlayerStore{}
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(playerStore, in, blindAlerter)
	cli.PlayPoker()

	if len(blindAlerter.alerts) != 1 {
		t.Fatal("expected a blind alert to be scheduled")
	}
})
```

Bạn sẽ thấy chúng ta đã tạo một `SpyBlindAlerter` và đang cố inject (tiêm) nó vào `CLI`. Sau đó chúng ta kiểm tra rằng sau khi gọi `PlayPoker`, một thông báo đã được lên lịch.

(Nhớ rằng chúng ta đang bắt đầu với tình huống đơn giản nhất trước, rồi sẽ mở rộng dần.)

Đây là định nghĩa của `SpyBlindAlerter`

```go
type SpyBlindAlerter struct {
	alerts []struct {
		scheduledAt time.Duration
		amount      int
	}
}

func (s *SpyBlindAlerter) ScheduleAlertAt(duration time.Duration, amount int) {
	s.alerts = append(s.alerts, struct {
		scheduledAt time.Duration
		amount      int
	}{duration, amount})
}

```


## Thử chạy test

```
./CLI_test.go:32:27: too many arguments in call to poker.NewCLI
	have (*poker.StubPlayerStore, *strings.Reader, *SpyBlindAlerter)
	want (poker.PlayerStore, io.Reader)
```

## Viết lượng code tối thiểu để test chạy được và kiểm tra output lỗi

Chúng ta đã thêm một tham số mới và trình biên dịch đang báo lỗi. _Nói chính xác_, lượng code tối thiểu là để `NewCLI` chấp nhận `*SpyBlindAlerter`. Nhưng hãy "gian lận" một chút và định nghĩa dependency (phụ thuộc) dưới dạng interface.

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}
```

Sau đó thêm nó vào constructor

```go
func NewCLI(store PlayerStore, in io.Reader, alerter BlindAlerter) *CLI
```

Các test khác của bạn sẽ fail vì chúng không truyền `BlindAlerter` vào `NewCLI`.

Spy trên BlindAlerter không liên quan đến các test khác, nên trong file test hãy thêm

```go
var dummySpyAlerter = &SpyBlindAlerter{}
```

Sau đó sử dụng nó trong các test khác để sửa lỗi biên dịch. Bằng cách đặt tên "dummy", người đọc test sẽ hiểu rõ rằng nó không quan trọng.

[> Dummy object được truyền đi nhưng không bao giờ thực sự được sử dụng. Thường chúng chỉ dùng để điền vào danh sách tham số.](https://martinfowler.com/articles/mocksArentStubs.html)

Các test bây giờ sẽ biên dịch được và test mới của chúng ta fail.

```
=== RUN   TestCLI
=== RUN   TestCLI/it_schedules_printing_of_blind_values
--- FAIL: TestCLI (0.00s)
    --- FAIL: TestCLI/it_schedules_printing_of_blind_values (0.00s)
    	CLI_test.go:38: expected a blind alert to be scheduled
```

## Viết đủ code để test pass

Chúng ta cần thêm `BlindAlerter` làm field trong `CLI` để có thể tham chiếu trong phương thức `PlayPoker`.

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
	alerter     BlindAlerter
}

func NewCLI(store PlayerStore, in io.Reader, alerter BlindAlerter) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
		alerter:     alerter,
	}
}
```

Để test pass, chúng ta có thể gọi `BlindAlerter` với bất kỳ giá trị nào

```go
func (cli *CLI) PlayPoker() {
	cli.alerter.ScheduleAlertAt(5*time.Second, 100)
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

Tiếp theo, chúng ta muốn kiểm tra rằng nó lên lịch tất cả các thông báo mà chúng ta mong đợi cho 5 người chơi.

## Viết test trước

```go
	t.Run("it schedules printing of blind values", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}
		blindAlerter := &SpyBlindAlerter{}

		cli := poker.NewCLI(playerStore, in, blindAlerter)
		cli.PlayPoker()

		cases := []struct {
			expectedScheduleTime time.Duration
			expectedAmount       int
		}{
			{0 * time.Second, 100},
			{10 * time.Minute, 200},
			{20 * time.Minute, 300},
			{30 * time.Minute, 400},
			{40 * time.Minute, 500},
			{50 * time.Minute, 600},
			{60 * time.Minute, 800},
			{70 * time.Minute, 1000},
			{80 * time.Minute, 2000},
			{90 * time.Minute, 4000},
			{100 * time.Minute, 8000},
		}

		for i, c := range cases {
			t.Run(fmt.Sprintf("%d scheduled for %v", c.expectedAmount, c.expectedScheduleTime), func(t *testing.T) {

				if len(blindAlerter.alerts) <= i {
					t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
				}

				alert := blindAlerter.alerts[i]

				amountGot := alert.amount
				if amountGot != c.expectedAmount {
					t.Errorf("got amount %d, want %d", amountGot, c.expectedAmount)
				}

				gotScheduledTime := alert.scheduledAt
				if gotScheduledTime != c.expectedScheduleTime {
					t.Errorf("got scheduled time of %v, want %v", gotScheduledTime, c.expectedScheduleTime)
				}
			})
		}
	})
```

Table-based test (test dựa trên bảng) hoạt động rất tốt ở đây và minh họa rõ ràng yêu cầu của chúng ta. Chúng ta duyệt qua bảng và kiểm tra `SpyBlindAlerter` để xem thông báo đã được lên lịch với đúng giá trị chưa.

## Thử chạy test

Bạn sẽ thấy nhiều lỗi như thế này

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_schedules_printing_of_blind_values
    --- FAIL: TestCLI/it_schedules_printing_of_blind_values (0.00s)
=== RUN   TestCLI/it_schedules_printing_of_blind_values/100_scheduled_for_0s
        --- FAIL: TestCLI/it_schedules_printing_of_blind_values/100_scheduled_for_0s (0.00s)
        	CLI_test.go:71: got scheduled time of 5s, want 0s
=== RUN   TestCLI/it_schedules_printing_of_blind_values/200_scheduled_for_10m0s
        --- FAIL: TestCLI/it_schedules_printing_of_blind_values/200_scheduled_for_10m0s (0.00s)
        	CLI_test.go:59: alert 1 was not scheduled [{5000000000 100}]
```

## Viết đủ code để test pass

```go
func (cli *CLI) PlayPoker() {

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + 10*time.Minute
	}

	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

Nó không phức tạp hơn nhiều so với trước. Chúng ta chỉ đang duyệt qua mảng `blinds` và gọi scheduler với `blindTime` tăng dần.

## Refactor

Chúng ta có thể đóng gói các thông báo blind đã lên lịch vào một phương thức để `PlayPoker` dễ đọc hơn.

```go
func (cli *CLI) PlayPoker() {
	cli.scheduleBlindAlerts()
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func (cli *CLI) scheduleBlindAlerts() {
	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + 10*time.Minute
	}
}
```

Cuối cùng, các test của chúng ta trông hơi cồng kềnh. Chúng ta có hai anonymous struct đại diện cho cùng một thứ, một `ScheduledAlert`. Hãy refactor nó thành type mới và tạo một số helper để so sánh chúng.

```go
type scheduledAlert struct {
	at     time.Duration
	amount int
}

func (s scheduledAlert) String() string {
	return fmt.Sprintf("%d chips at %v", s.amount, s.at)
}

type SpyBlindAlerter struct {
	alerts []scheduledAlert
}

func (s *SpyBlindAlerter) ScheduleAlertAt(at time.Duration, amount int) {
	s.alerts = append(s.alerts, scheduledAlert{at, amount})
}
```

Chúng ta đã thêm phương thức `String()` cho type để nó in đẹp hơn khi test fail.

Cập nhật test để sử dụng type mới

```go
t.Run("it schedules printing of blind values", func(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &poker.StubPlayerStore{}
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(playerStore, in, blindAlerter)
	cli.PlayPoker()

	cases := []scheduledAlert{
		{0 * time.Second, 100},
		{10 * time.Minute, 200},
		{20 * time.Minute, 300},
		{30 * time.Minute, 400},
		{40 * time.Minute, 500},
		{50 * time.Minute, 600},
		{60 * time.Minute, 800},
		{70 * time.Minute, 1000},
		{80 * time.Minute, 2000},
		{90 * time.Minute, 4000},
		{100 * time.Minute, 8000},
	}

	for i, want := range cases {
		t.Run(fmt.Sprint(want), func(t *testing.T) {

			if len(blindAlerter.alerts) <= i {
				t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
			}

			got := blindAlerter.alerts[i]
			assertScheduledAlert(t, got, want)
		})
	}
})
```

Hãy tự triển khai `assertScheduledAlert`.

Chúng ta đã dành khá nhiều thời gian ở đây để viết test và đã hơi "hư" khi chưa tích hợp với ứng dụng. Hãy giải quyết điều đó trước khi thêm bất kỳ yêu cầu nào nữa.

Thử chạy ứng dụng và nó sẽ không biên dịch được, báo lỗi không đủ tham số cho `NewCLI`.

Hãy tạo một triển khai của `BlindAlerter` để dùng trong ứng dụng.

Tạo file `blind_alerter.go` và di chuyển interface `BlindAlerter` của chúng ta vào đó, thêm các phần mới bên dưới

```go
package poker

import (
	"fmt"
	"os"
	"time"
)

type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

type BlindAlerterFunc func(duration time.Duration, amount int)

func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int) {
	a(duration, amount)
}

func StdOutAlerter(duration time.Duration, amount int) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(os.Stdout, "Blind is now %d\n", amount)
	})
}
```

Hãy nhớ rằng bất kỳ _type_ nào cũng có thể triển khai interface, không chỉ `struct`. Nếu bạn đang tạo một thư viện có interface với một hàm duy nhất, một quy ước phổ biến là cũng cung cấp type `MyInterfaceFunc`.

Type này sẽ là một `func` cũng triển khai interface của bạn. Nhờ đó, người dùng interface có thể triển khai nó chỉ với một hàm, thay vì phải tạo một `struct` rỗng.

Sau đó chúng ta tạo hàm `StdOutAlerter` có cùng signature với hàm đó và sử dụng `time.AfterFunc` để lên lịch in ra `os.Stdout`.

Cập nhật `main` nơi chúng ta tạo `NewCLI` để xem nó hoạt động

```go
poker.NewCLI(store, os.Stdin, poker.BlindAlerterFunc(poker.StdOutAlerter)).PlayPoker()
```

Trước khi chạy, bạn có thể muốn thay đổi khoảng tăng `blindTime` trong `CLI` thành 10 giây thay vì 10 phút để có thể thấy nó hoạt động.

Bạn sẽ thấy nó in ra các giá trị blind như mong đợi mỗi 10 giây. Lưu ý rằng bạn vẫn có thể gõ `Shaun wins` vào CLI và nó sẽ dừng chương trình như mong đợi.

Trò chơi không phải lúc nào cũng chơi với 5 người, nên chúng ta cần hỏi người dùng nhập số lượng người chơi trước khi trò chơi bắt đầu.

## Viết test trước

Để kiểm tra rằng chúng ta đang hỏi số lượng người chơi, chúng ta sẽ muốn ghi lại những gì được viết ra StdOut. Chúng ta đã làm điều này vài lần rồi. Chúng ta biết rằng `os.Stdout` là một `io.Writer`, nên có thể kiểm tra những gì được viết ra nếu dùng dependency injection để truyền vào `bytes.Buffer` trong test.

Chúng ta không quan tâm đến các collaborator khác trong test này nên đã tạo một số dummy trong file test.

Chúng ta nên hơi cẩn thận rằng `CLI` giờ đã có 4 dependency, có vẻ như nó đang bắt đầu có quá nhiều trách nhiệm. Hãy tạm chấp nhận và xem liệu cơ hội refactor có xuất hiện khi chúng ta thêm chức năng mới.

```go
var dummyBlindAlerter = &SpyBlindAlerter{}
var dummyPlayerStore = &poker.StubPlayerStore{}
var dummyStdIn = &bytes.Buffer{}
var dummyStdOut = &bytes.Buffer{}
```

Đây là test mới

```go
t.Run("it prompts the user to enter the number of players", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	cli := poker.NewCLI(dummyPlayerStore, dummyStdIn, stdout, dummyBlindAlerter)
	cli.PlayPoker()

	got := stdout.String()
	want := "Please enter the number of players: "

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
})
```

Chúng ta truyền vào thứ sẽ là `os.Stdout` trong `main` và xem những gì được viết ra.

## Thử chạy test

```
./CLI_test.go:38:27: too many arguments in call to poker.NewCLI
	have (*poker.StubPlayerStore, *bytes.Buffer, *bytes.Buffer, *SpyBlindAlerter)
	want (poker.PlayerStore, io.Reader, poker.BlindAlerter)
```

## Viết lượng code tối thiểu để test chạy được và kiểm tra output lỗi

Chúng ta có dependency mới nên cần cập nhật `NewCLI`

```go
func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI
```

Bây giờ các test _khác_ sẽ không biên dịch được vì chúng không truyền `io.Writer` vào `NewCLI`.

Thêm `dummyStdout` cho các test khác.

Test mới sẽ fail như sau

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players
    --- FAIL: TestCLI/it_prompts_the_user_to_enter_the_number_of_players (0.00s)
    	CLI_test.go:46: got '', want 'Please enter the number of players: '
FAIL
```

## Viết đủ code để test pass

Chúng ta cần thêm dependency mới vào `CLI` để có thể tham chiếu trong `PlayPoker`

```go
type CLI struct {
	playerStore PlayerStore
	in          *bufio.Scanner
	out         io.Writer
	alerter     BlindAlerter
}

func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI {
	return &CLI{
		playerStore: store,
		in:          bufio.NewScanner(in),
		out:         out,
		alerter:     alerter,
	}
}
```

Cuối cùng chúng ta có thể viết lời nhắc ở đầu trò chơi

```go
func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, "Please enter the number of players: ")
	cli.scheduleBlindAlerts()
	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}
```

## Refactor

Chúng ta có chuỗi trùng lặp cho lời nhắc, nên trích xuất thành hằng số

```go
const PlayerPrompt = "Please enter the number of players: "
```

Sử dụng hằng số này trong cả test code và `CLI`.

Bây giờ chúng ta cần gửi một số và trích xuất nó ra. Cách duy nhất để biết nó có tác dụng mong muốn là xem những thông báo blind nào đã được lên lịch.

## Viết test trước

```go
t.Run("it prompts the user to enter the number of players", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	in := strings.NewReader("7\n")
	blindAlerter := &SpyBlindAlerter{}

	cli := poker.NewCLI(dummyPlayerStore, in, stdout, blindAlerter)
	cli.PlayPoker()

	got := stdout.String()
	want := poker.PlayerPrompt

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}

	cases := []scheduledAlert{
		{0 * time.Second, 100},
		{12 * time.Minute, 200},
		{24 * time.Minute, 300},
		{36 * time.Minute, 400},
	}

	for i, want := range cases {
		t.Run(fmt.Sprint(want), func(t *testing.T) {

			if len(blindAlerter.alerts) <= i {
				t.Fatalf("alert %d was not scheduled %v", i, blindAlerter.alerts)
			}

			got := blindAlerter.alerts[i]
			assertScheduledAlert(t, got, want)
		})
	}
})
```

Khá nhiều thay đổi!

- Chúng ta bỏ dummy cho StdIn và thay vào đó gửi phiên bản mock đại diện cho người dùng nhập 7
- Chúng ta cũng bỏ dummy trên blind alerter để xem số người chơi đã ảnh hưởng đến việc lên lịch như thế nào
- Chúng ta test những thông báo nào được lên lịch

## Thử chạy test

Test vẫn biên dịch được và fail, báo rằng thời gian lên lịch sai vì chúng ta đã hard-code trò chơi cho 5 người chơi

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players
    --- FAIL: TestCLI/it_prompts_the_user_to_enter_the_number_of_players (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players/100_chips_at_0s
        --- PASS: TestCLI/it_prompts_the_user_to_enter_the_number_of_players/100_chips_at_0s (0.00s)
=== RUN   TestCLI/it_prompts_the_user_to_enter_the_number_of_players/200_chips_at_12m0s
```

## Viết đủ code để test pass

Nhớ rằng, chúng ta tự do "phạm tội" bao nhiêu tùy thích để code hoạt động. Khi đã có phần mềm chạy được, chúng ta có thể refactor mớ hỗn độn mà chúng ta sắp tạo ra!

```go
func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, PlayerPrompt)

	numberOfPlayers, _ := strconv.Atoi(cli.readLine())

	cli.scheduleBlindAlerts(numberOfPlayers)

	userInput := cli.readLine()
	cli.playerStore.RecordWin(extractWinner(userInput))
}

func (cli *CLI) scheduleBlindAlerts(numberOfPlayers int) {
	blindIncrement := time.Duration(5+numberOfPlayers) * time.Minute

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		cli.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + blindIncrement
	}
}
```

- Chúng ta đọc `numberOfPlayersInput` vào chuỗi
- Chúng ta dùng `cli.readLine()` để nhận input từ người dùng rồi gọi `Atoi` để chuyển thành số nguyên - tạm bỏ qua các tình huống lỗi. Chúng ta sẽ cần viết test cho tình huống đó sau.
- Từ đây, chúng ta thay đổi `scheduleBlindAlerts` để nhận số người chơi. Chúng ta tính `blindIncrement` để cộng vào `blindTime` khi duyệt qua các mức blind

Trong khi test mới đã được sửa, nhiều test khác fail vì hệ thống giờ chỉ hoạt động nếu trò chơi bắt đầu với người dùng nhập một số. Bạn cần sửa các test bằng cách thay đổi input người dùng sao cho một số theo sau bởi dấu xuống dòng được thêm vào (điều này làm lộ thêm các khuyết điểm trong cách tiếp cận hiện tại).

## Refactor

Tất cả cảm giác khá tệ phải không? Hãy **lắng nghe test của chúng ta**.

- Để test rằng chúng ta lên lịch một số thông báo, chúng ta phải thiết lập 4 dependency khác nhau. Khi nào bạn có nhiều dependency cho một _thứ_ trong hệ thống, điều đó ngụ ý nó đang làm quá nhiều việc. Trực quan chúng ta có thể thấy test rất lộn xộn; **lắng nghe test là rất quan trọng**.
- Theo tôi, có vẻ như **chúng ta cần tạo một abstraction (trừu tượng hóa) rõ ràng hơn giữa việc đọc input người dùng và logic nghiệp vụ**.
- Một test tốt hơn sẽ là _với input người dùng này, chúng ta có gọi type `Game` mới với đúng số người chơi không_.
- Sau đó chúng ta sẽ tách việc test lên lịch vào các test cho `Game` mới.

Chúng ta có thể refactor hướng tới `Game` trước và test vẫn nên pass. Khi đã thực hiện xong các thay đổi cấu trúc, chúng ta có thể nghĩ về cách refactor test để phản ánh việc phân tách trách nhiệm mới.

Nhớ rằng khi thực hiện thay đổi trong refactoring, hãy giữ chúng nhỏ nhất có thể và chạy lại test sau mỗi thay đổi.

Hãy thử tự làm trước. Nghĩ về ranh giới của `Game` sẽ cung cấp gì và `CLI` nên làm gì.

Hiện tại **đừng** thay đổi interface bên ngoài của `NewCLI` vì chúng ta không muốn thay đổi test code và client code cùng lúc, vì đó là quá nhiều thứ cần xử lý và có thể gây lỗi.

Đây là kết quả của tôi:

```go
// game.go
type Game struct {
	alerter BlindAlerter
	store   PlayerStore
}

func (p *Game) Start(numberOfPlayers int) {
	blindIncrement := time.Duration(5+numberOfPlayers) * time.Minute

	blinds := []int{100, 200, 300, 400, 500, 600, 800, 1000, 2000, 4000, 8000}
	blindTime := 0 * time.Second
	for _, blind := range blinds {
		p.alerter.ScheduleAlertAt(blindTime, blind)
		blindTime = blindTime + blindIncrement
	}
}

func (p *Game) Finish(winner string) {
	p.store.RecordWin(winner)
}

// cli.go
type CLI struct {
	in   *bufio.Scanner
	out  io.Writer
	game *Game
}

func NewCLI(store PlayerStore, in io.Reader, out io.Writer, alerter BlindAlerter) *CLI {
	return &CLI{
		in:  bufio.NewScanner(in),
		out: out,
		game: &Game{
			alerter: alerter,
			store:   store,
		},
	}
}

const PlayerPrompt = "Please enter the number of players: "

func (cli *CLI) PlayPoker() {
	fmt.Fprint(cli.out, PlayerPrompt)

	numberOfPlayersInput := cli.readLine()
	numberOfPlayers, _ := strconv.Atoi(strings.Trim(numberOfPlayersInput, "\n"))

	cli.game.Start(numberOfPlayers)

	winnerInput := cli.readLine()
	winner := extractWinner(winnerInput)

	cli.game.Finish(winner)
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins\n", "", 1)
}

func (cli *CLI) readLine() string {
	cli.in.Scan()
	return cli.in.Text()
}
```

Từ góc nhìn "domain" (miền nghiệp vụ):
- Chúng ta muốn `Start` (bắt đầu) một `Game`, chỉ ra có bao nhiêu người đang chơi
- Chúng ta muốn `Finish` (kết thúc) một `Game`, tuyên bố người thắng

Type `Game` mới đóng gói điều này cho chúng ta.

Với thay đổi này, chúng ta đã truyền `BlindAlerter` và `PlayerStore` cho `Game` vì giờ nó chịu trách nhiệm thông báo và lưu kết quả.

`CLI` của chúng ta giờ chỉ quan tâm đến:

- Tạo `Game` với các dependency hiện có (sẽ refactor tiếp)
- Diễn giải input người dùng thành các lời gọi phương thức cho `Game`

Chúng ta muốn tránh thực hiện refactoring "lớn" khiến test fail trong thời gian dài, vì điều đó tăng khả năng mắc lỗi. (Nếu bạn làm việc trong nhóm lớn/phân tán, điều này càng quan trọng hơn)

Điều đầu tiên chúng ta sẽ làm là refactor `Game` để inject nó vào `CLI`. Chúng ta sẽ thực hiện thay đổi nhỏ nhất trong test để hỗ trợ điều đó, rồi xem cách chia test thành các chủ đề: phân tích input người dùng và quản lý trò chơi.

Tất cả những gì cần làm bây giờ là thay đổi `NewCLI`

```go
func NewCLI(in io.Reader, out io.Writer, game *Game) *CLI {
	return &CLI{
		in:   bufio.NewScanner(in),
		out:  out,
		game: game,
	}
}
```

Điều này cảm giác đã cải thiện rồi. Chúng ta có ít dependency hơn và _danh sách dependency phản ánh mục tiêu thiết kế tổng thể_ của CLI: quan tâm đến input/output và ủy thác các hành động liên quan đến trò chơi cho `Game`.

Nếu bạn thử biên dịch sẽ có lỗi. Bạn nên tự sửa được. Đừng lo tạo mock cho `Game` lúc này, chỉ cần khởi tạo `Game` _thật_ để mọi thứ biên dịch và test xanh.

Để làm điều này, bạn cần tạo constructor

```go
func NewGame(alerter BlindAlerter, store PlayerStore) *Game {
	return &Game{
		alerter: alerter,
		store:   store,
	}
}
```

Đây là ví dụ về cách sửa setup cho một trong các test

```go
stdout := &bytes.Buffer{}
in := strings.NewReader("7\n")
blindAlerter := &SpyBlindAlerter{}
game := poker.NewGame(blindAlerter, dummyPlayerStore)

cli := poker.NewCLI(in, stdout, game)
cli.PlayPoker()
```

Việc sửa test và quay lại trạng thái xanh không nên tốn nhiều công sức (đó là mục đích!). Nhưng hãy nhớ sửa cả `main.go` trước giai đoạn tiếp theo.

```go
// main.go
game := poker.NewGame(poker.BlindAlerterFunc(poker.StdOutAlerter), store)
cli := poker.NewCLI(os.Stdin, os.Stdout, game)
cli.PlayPoker()
```

Bây giờ khi đã tách `Game` ra, chúng ta nên chuyển các assertion liên quan đến game vào test riêng, tách biệt khỏi CLI.

Đây chỉ là bài tập sao chép test CLI nhưng với ít dependency hơn

```go
func TestGame_Start(t *testing.T) {
	t.Run("schedules alerts on game start for 5 players", func(t *testing.T) {
		blindAlerter := &poker.SpyBlindAlerter{}
		game := poker.NewGame(blindAlerter, dummyPlayerStore)

		game.Start(5)

		cases := []poker.ScheduledAlert{
			{At: 0 * time.Second, Amount: 100},
			{At: 10 * time.Minute, Amount: 200},
			{At: 20 * time.Minute, Amount: 300},
			{At: 30 * time.Minute, Amount: 400},
			{At: 40 * time.Minute, Amount: 500},
			{At: 50 * time.Minute, Amount: 600},
			{At: 60 * time.Minute, Amount: 800},
			{At: 70 * time.Minute, Amount: 1000},
			{At: 80 * time.Minute, Amount: 2000},
			{At: 90 * time.Minute, Amount: 4000},
			{At: 100 * time.Minute, Amount: 8000},
		}

		checkSchedulingCases(cases, t, blindAlerter)
	})

	t.Run("schedules alerts on game start for 7 players", func(t *testing.T) {
		blindAlerter := &poker.SpyBlindAlerter{}
		game := poker.NewGame(blindAlerter, dummyPlayerStore)

		game.Start(7)

		cases := []poker.ScheduledAlert{
			{At: 0 * time.Second, Amount: 100},
			{At: 12 * time.Minute, Amount: 200},
			{At: 24 * time.Minute, Amount: 300},
			{At: 36 * time.Minute, Amount: 400},
		}

		checkSchedulingCases(cases, t, blindAlerter)
	})

}

func TestGame_Finish(t *testing.T) {
	store := &poker.StubPlayerStore{}
	game := poker.NewGame(dummyBlindAlerter, store)
	winner := "Ruth"

	game.Finish(winner)
	poker.AssertPlayerWin(t, store, winner)
}
```

Ý định đằng sau những gì xảy ra khi bắt đầu trò chơi poker giờ rõ ràng hơn nhiều.

Hãy nhớ chuyển cả test cho khi trò chơi kết thúc.

Khi hài lòng rằng đã chuyển xong test cho logic game, chúng ta có thể đơn giản hóa test CLI để phản ánh rõ hơn trách nhiệm dự kiến:

- Xử lý input người dùng và gọi các phương thức của `Game` khi phù hợp
- Gửi output
- Quan trọng là nó không biết về cách hoạt động thực tế của trò chơi

Để làm điều này, chúng ta cần làm cho `CLI` không còn phụ thuộc vào type `Game` cụ thể mà thay vào đó chấp nhận interface với `Start(numberOfPlayers)` và `Finish(winner)`. Sau đó chúng ta có thể tạo spy của type đó và xác minh các lời gọi đúng được thực hiện.

Ở đây chúng ta nhận ra rằng đặt tên đôi khi rất khó. Đổi tên `Game` thành `TexasHoldem` (vì đó là _loại_ trò chơi chúng ta đang chơi) và interface mới sẽ được gọi là `Game`. Điều này giữ đúng với ý tưởng rằng CLI không biết trò chơi thực tế chúng ta đang chơi và điều gì xảy ra khi bạn `Start` và `Finish`.

```go
type Game interface {
	Start(numberOfPlayers int)
	Finish(winner string)
}
```

Thay thế tất cả tham chiếu đến `*Game` trong `CLI` bằng `Game` (interface mới). Như mọi khi, hãy chạy lại test để kiểm tra mọi thứ vẫn xanh trong khi refactoring.

Bây giờ khi đã tách `CLI` khỏi `TexasHoldem`, chúng ta có thể dùng spy để kiểm tra rằng `Start` và `Finish` được gọi khi mong đợi, với đúng tham số.

Tạo spy triển khai `Game`

```go
type GameSpy struct {
	StartedWith  int
	FinishedWith string
}

func (g *GameSpy) Start(numberOfPlayers int) {
	g.StartedWith = numberOfPlayers
}

func (g *GameSpy) Finish(winner string) {
	g.FinishedWith = winner
}
```

Thay thế các test CLI đang test logic game bằng kiểm tra cách `GameSpy` được gọi. Điều này sẽ phản ánh rõ ràng trách nhiệm của CLI trong test.

Đây là ví dụ sửa một test; hãy thử tự sửa phần còn lại và kiểm tra mã nguồn nếu bị kẹt.

```go
	t.Run("it prompts the user to enter the number of players and starts the game", func(t *testing.T) {
		stdout := &bytes.Buffer{}
		in := strings.NewReader("7\n")
		game := &GameSpy{}

		cli := poker.NewCLI(in, stdout, game)
		cli.PlayPoker()

		gotPrompt := stdout.String()
		wantPrompt := poker.PlayerPrompt

		if gotPrompt != wantPrompt {
			t.Errorf("got %q, want %q", gotPrompt, wantPrompt)
		}

		if game.StartedWith != 7 {
			t.Errorf("wanted Start called with 7 but got %d", game.StartedWith)
		}
	})
```

Bây giờ khi đã phân tách trách nhiệm rõ ràng, việc kiểm tra các trường hợp đặc biệt liên quan đến IO trong `CLI` sẽ dễ dàng hơn.

Chúng ta cần xử lý tình huống người dùng nhập giá trị không phải số khi được hỏi số lượng người chơi:

Code không nên bắt đầu trò chơi. Nó nên in thông báo lỗi hữu ích cho người dùng rồi thoát.

## Viết test trước

Chúng ta sẽ bắt đầu bằng cách đảm bảo trò chơi không bắt đầu

```go
t.Run("it prints an error when a non numeric value is entered and does not start the game", func(t *testing.T) {
	stdout := &bytes.Buffer{}
	in := strings.NewReader("Pies\n")
	game := &GameSpy{}

	cli := poker.NewCLI(in, stdout, game)
	cli.PlayPoker()

	if game.StartCalled {
		t.Errorf("game should not have started")
	}
})
```

Bạn cần thêm field `StartCalled` vào `GameSpy`, field này chỉ được set khi `Start` được gọi.

## Thử chạy test
```
=== RUN   TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game
    --- FAIL: TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game (0.00s)
        CLI_test.go:62: game should not have started
```

## Viết đủ code để test pass

Ở chỗ gọi `Atoi`, chúng ta chỉ cần kiểm tra lỗi

```go
numberOfPlayers, err := strconv.Atoi(cli.readLine())

if err != nil {
	return
}
```

Tiếp theo chúng ta cần thông báo cho người dùng biết họ đã làm gì sai, nên sẽ assert trên những gì được in ra `stdout`.

## Viết test trước

Chúng ta đã assert trên những gì in ra `stdout` trước đó nên có thể sao chép code đó

```go
gotPrompt := stdout.String()

wantPrompt := poker.PlayerPrompt + "you're so silly"

if gotPrompt != wantPrompt {
	t.Errorf("got %q, want %q", gotPrompt, wantPrompt)
}
```

Chúng ta đang lưu _mọi thứ_ được viết vào stdout nên vẫn mong đợi `poker.PlayerPrompt`. Sau đó chỉ kiểm tra thêm một thứ nữa được in ra. Chúng ta chưa quan tâm lắm đến từ ngữ chính xác, sẽ xử lý khi refactor.

## Thử chạy test

```
=== RUN   TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game
    --- FAIL: TestCLI/it_prints_an_error_when_a_non_numeric_value_is_entered_and_does_not_start_the_game (0.00s)
        CLI_test.go:70: got 'Please enter the number of players: ', want 'Please enter the number of players: you're so silly'
```

## Viết đủ code để test pass

Thay đổi code xử lý lỗi

```go
if err != nil {
	fmt.Fprint(cli.out, "you're so silly")
	return
}
```

## Refactor

Giờ hãy refactor thông báo thành hằng số giống `PlayerPrompt`

```go
wantPrompt := poker.PlayerPrompt + poker.BadPlayerInputErrMsg
```

và đặt thông báo phù hợp hơn

```go
const BadPlayerInputErrMsg = "Bad value received for number of players, please try again with a number"
```

Cuối cùng, việc test những gì gửi đến `stdout` khá dài dòng. Hãy viết hàm assert để gọn hơn.

```go
func assertMessagesSentToUser(t testing.TB, stdout *bytes.Buffer, messages ...string) {
	t.Helper()
	want := strings.Join(messages, "")
	got := stdout.String()
	if got != want {
		t.Errorf("got %q sent to stdout but expected %+v", got, messages)
	}
}
```

Sử dụng cú pháp vararg (`...string`) rất tiện ở đây vì chúng ta cần assert trên số lượng thông báo khác nhau.

Sử dụng helper này trong cả hai test nơi chúng ta assert trên thông báo gửi đến người dùng.

Có nhiều test có thể cải thiện với các hàm `assertX`, nên hãy luyện tập refactoring bằng cách dọn dẹp test để chúng dễ đọc hơn.

Dành thời gian suy nghĩ về giá trị của một số test chúng ta đã viết. Nhớ rằng chúng ta không muốn nhiều test hơn mức cần thiết. Bạn có thể refactor/bỏ một số test _mà vẫn tự tin mọi thứ hoạt động_ không?

Đây là kết quả của tôi

```go
func TestCLI(t *testing.T) {

	t.Run("start game with 3 players and finish game with 'Chris' as winner", func(t *testing.T) {
		game := &GameSpy{}
		stdout := &bytes.Buffer{}

		in := userSends("3", "Chris wins")
		cli := poker.NewCLI(in, stdout, game)

		cli.PlayPoker()

		assertMessagesSentToUser(t, stdout, poker.PlayerPrompt)
		assertGameStartedWith(t, game, 3)
		assertFinishCalledWith(t, game, "Chris")
	})

	t.Run("start game with 8 players and record 'Cleo' as winner", func(t *testing.T) {
		game := &GameSpy{}

		in := userSends("8", "Cleo wins")
		cli := poker.NewCLI(in, dummyStdOut, game)

		cli.PlayPoker()

		assertGameStartedWith(t, game, 8)
		assertFinishCalledWith(t, game, "Cleo")
	})

	t.Run("it prints an error when a non numeric value is entered and does not start the game", func(t *testing.T) {
		game := &GameSpy{}

		stdout := &bytes.Buffer{}
		in := userSends("pies")

		cli := poker.NewCLI(in, stdout, game)
		cli.PlayPoker()

		assertGameNotStarted(t, game)
		assertMessagesSentToUser(t, stdout, poker.PlayerPrompt, poker.BadPlayerInputErrMsg)
	})
}
```

Các test giờ phản ánh khả năng chính của CLI: nó có thể đọc input người dùng về số người chơi và ai thắng, đồng thời xử lý khi nhập giá trị sai cho số người chơi. Nhờ vậy, người đọc hiểu rõ `CLI` làm gì, và cũng hiểu rõ những gì nó _không_ làm.

Điều gì xảy ra nếu thay vì nhập `Ruth wins`, người dùng nhập `Lloyd is a killer`?

Hãy kết thúc chương này bằng cách viết test cho tình huống đó và làm cho nó pass.

## Tổng kết

### Tóm tắt nhanh về dự án

Trong 5 chương vừa qua, chúng ta đã từ từ TDD một lượng code đáng kể

- Chúng ta có hai ứng dụng: một ứng dụng command line và một web server.
- Cả hai ứng dụng đều dựa vào `PlayerStore` để ghi nhận người thắng.
- Web server cũng có thể hiển thị bảng xếp hạng ai thắng nhiều nhất.
- Ứng dụng command line giúp người chơi chơi poker bằng cách theo dõi giá trị blind hiện tại.

### time.Afterfunc

Một cách rất tiện lợi để lên lịch gọi hàm sau một khoảng thời gian cụ thể. Rất đáng để đầu tư thời gian [xem tài liệu của `time`](https://golang.org/pkg/time/) vì nó có nhiều hàm và phương thức hữu ích.

Một số mục yêu thích của tôi là

- `time.After(duration)` trả về `chan Time` khi khoảng thời gian đã hết. Nếu bạn muốn thực hiện gì đó _sau_ một thời gian cụ thể, hàm này có thể giúp ích.
- `time.NewTicker(duration)` trả về `Ticker` tương tự như trên, nó trả về channel nhưng channel này "tích tắc" sau mỗi khoảng duration, thay vì chỉ một lần. Rất hữu ích nếu bạn muốn thực thi code mỗi `N duration`.

### Thêm ví dụ về phân tách trách nhiệm tốt

_Nói chung_, việc tách biệt trách nhiệm xử lý input/response người dùng khỏi code domain (miền nghiệp vụ) là thực hành tốt. Bạn thấy điều này ở đây trong ứng dụng command line và cả web server.

Test của chúng ta trở nên lộn xộn. Chúng ta có quá nhiều assertion (kiểm tra input này, lên lịch các thông báo kia, v.v.) và quá nhiều dependency. Chúng ta có thể nhìn thấy trực quan sự lộn xộn; **lắng nghe test là rất quan trọng**.

- Nếu test trông lộn xộn, hãy thử refactor chúng.
- Nếu đã refactor mà vẫn lộn xộn, rất có thể nó đang chỉ ra khuyết điểm trong thiết kế.
- Đây là một trong những sức mạnh thực sự của test.

Mặc dù test và code sản phẩm hơi lộn xộn, chúng ta có thể tự do refactor với sự hỗ trợ của test.

Nhớ rằng khi gặp những tình huống này, luôn thực hiện từng bước nhỏ và chạy lại test sau mỗi thay đổi.

Sẽ rất nguy hiểm nếu refactor cả test code _và_ code sản phẩm cùng lúc. Vì vậy, chúng ta refactor code sản phẩm trước (ở trạng thái hiện tại không thể cải thiện test nhiều) mà không thay đổi interface để có thể dựa vào test nhiều nhất có thể khi thay đổi. _Sau đó_ chúng ta mới refactor test khi thiết kế đã được cải thiện.

Sau khi refactor, danh sách dependency phản ánh mục tiêu thiết kế. Đây là một lợi ích khác của DI (Dependency Injection - tiêm phụ thuộc): nó thường ghi lại ý định. Khi bạn dựa vào biến toàn cục, trách nhiệm trở nên rất không rõ ràng.

## Ví dụ về hàm triển khai interface

Khi bạn định nghĩa interface với một phương thức duy nhất, bạn có thể cân nhắc định nghĩa type `MyInterfaceFunc` đi kèm để người dùng có thể triển khai interface chỉ với một hàm.

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}

// BlindAlerterFunc allows you to implement BlindAlerter with a function
type BlindAlerterFunc func(duration time.Duration, amount int)

// ScheduleAlertAt is BlindAlerterFunc implementation of BlindAlerter
func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int) {
	a(duration, amount)
}
```

Bằng cách này, người dùng thư viện có thể triển khai interface chỉ với một hàm. Họ có thể sử dụng [Type Conversion](https://go.dev/tour/basics/13) (chuyển đổi kiểu) để chuyển hàm thành `BlindAlerterFunc` và dùng nó như `BlindAlerter` (vì `BlindAlerterFunc` triển khai `BlindAlerter`).

```go
game := poker.NewTexasHoldem(poker.BlindAlerterFunc(poker.StdOutAlerter), store)
```

Điểm quan trọng hơn ở đây là, trong Go bạn có thể thêm phương thức vào _type_, không chỉ struct. Đây là tính năng rất mạnh mẽ, và bạn có thể dùng nó để triển khai interface theo những cách tiện lợi hơn.

Hãy cân nhắc rằng bạn không chỉ có thể định nghĩa type cho hàm, mà còn có thể định nghĩa type bao quanh các type khác để thêm phương thức cho chúng.

```go
type Blog map[string]string

func (b Blog) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, b[r.URL.Path])
}
```

Ở đây chúng ta đã tạo HTTP handler triển khai một "blog" rất đơn giản, sử dụng đường dẫn URL làm key cho các bài viết được lưu trong map.
