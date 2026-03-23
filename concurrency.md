# Concurrency

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/concurrency)**

Bối cảnh như sau: một đồng nghiệp đã viết một hàm, `CheckWebsites`, để kiểm tra trạng thái của một danh sách các URL.

```go
package concurrency

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		results[url] = wc(url)
	}

	return results
}
```

Nó trả về một map của mỗi URL được kiểm tra với một giá trị boolean: `true` cho phản hồi tốt; `false` cho phản hồi xấu.

Bạn cũng phải truyền vào một `WebsiteChecker`, nó nhận một URL duy nhất và trả về một boolean. Điều này được hàm sử dụng để kiểm tra tất cả các trang web.

Nhờ [dependency injection][DI], họ có thể test hàm mà không cần gọi HTTP thực sự — nhanh và đáng tin cậy hơn.

Đây là test họ đã viết:

```go
package concurrency

import (
	"reflect"
	"testing"
)

func mockWebsiteChecker(url string) bool {
	return url != "waat://furhurterwe.geds"
}

func TestCheckWebsites(t *testing.T) {
	websites := []string{
		"http://google.com",
		"http://blog.gypsydave5.com",
		"waat://furhurterwe.geds",
	}

	want := map[string]bool{
		"http://google.com":          true,
		"http://blog.gypsydave5.com": true,
		"waat://furhurterwe.geds":    false,
	}

	got := CheckWebsites(mockWebsiteChecker, websites)

	if !reflect.DeepEqual(want, got) {
		t.Fatalf("wanted %v, got %v", want, got)
	}
}
```

Hàm này đang được đưa vào triển khai thực tế và được sử dụng để kiểm tra hàng trăm trang web. Nhưng đồng nghiệp của bạn đã bắt đầu nhận được khiếu nại rằng nó chậm, vì vậy họ yêu cầu bạn giúp tăng tốc nó.

## Viết một test

Hãy sử dụng benchmark để kiểm tra tốc độ của `CheckWebsites` để chúng ta có thể thấy tác động của các thay đổi của mình.

```go
package concurrency

import (
	"testing"
	"time"
)

func slowStubWebsiteChecker(_ string) bool {
	time.Sleep(20 * time.Millisecond)
	return true
}

func BenchmarkCheckWebsites(b *testing.B) {
	urls := make([]string, 100)
	for i := 0; i < len(urls); i++ {
		urls[i] = "a url"
	}
	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		CheckWebsites(slowStubWebsiteChecker, urls)
	}
}
```

Benchmark kiểm thử `CheckWebsites` bằng cách sử dụng một slice gồm một trăm url và sử dụng một implementation giả mới của `WebsiteChecker`. `slowStubWebsiteChecker` cố tình chạy chậm. Nó sử dụng `time.Sleep` để đợi chính xác hai mươi mili giây và sau đó nó trả về true. Chúng ta sử dụng `b.ResetTimer()` trong test này để đặt lại thời gian của test trước khi nó thực sự chạy.

Khi chúng ta chạy benchmark bằng `go test -bench=.` (hoặc nếu bạn đang ở Windows Powershell thì dùng `go test -bench="."`):

```sh
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v0
BenchmarkCheckWebsites-4               1        2249228637 ns/op
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v0        2.268s
```

`CheckWebsites` đã được benchmark ở mức 2.249.228.637 nano giây - khoảng hai giây và một phần tư.

Hãy cố gắng làm cho nó nhanh hơn.

### Viết đủ code để test chạy thành công

Cuối cùng chúng ta có thể nói về concurrency (lập trình đồng thời), mà đối với mục đích của phần sau, có nghĩa là "có nhiều hơn một việc đang được thực hiện". Đây là điều mà chúng ta thực hiện một cách tự nhiên hàng ngày.

Chẳng hạn, sáng nay tôi đã pha một tách trà. Tôi bật ấm đun nước điện và sau đó, trong khi chờ nó sôi, tôi lấy sữa ra khỏi tủ lạnh, lấy trà ra khỏi tủ chén, tìm chiếc cốc yêu thích của mình, cho túi trà vào cốc và sau đó, khi ấm đun nước đã sôi, tôi đổ nước vào cốc.

Điều tôi *không* làm là bật ấm đun nước lên và sau đó đứng đó ngây người nhìn ấm cho đến khi nó sôi, rồi mới làm mọi việc khác sau khi ấm đã sôi.

Nếu bạn có thể hiểu tại sao cách pha trà thứ nhất nhanh hơn, thì bạn có thể hiểu cách chúng ta sẽ làm cho `CheckWebsites` nhanh hơn. Thay vì đợi một trang web phản hồi trước khi gửi yêu cầu đến trang web tiếp theo, chúng ta sẽ bảo máy tính thực hiện yêu cầu tiếp theo trong khi nó đang chờ đợi.

Thông thường trong Go khi chúng ta gọi một hàm `doSomething()`, chúng ta đợi nó quay trở lại (ngay cả khi nó không có giá trị trả về, chúng ta vẫn đợi nó hoàn thành). Chúng ta nói rằng hoạt động này là *blocking* (chặn) - nó khiến chúng ta phải đợi nó kết thúc. Một hoạt động không chặn trong Go sẽ chạy trong một *process* (quy trình) riêng biệt được gọi là một *goroutine*. Hãy tưởng tượng một quy trình giống như việc đọc mã Go từ trên xuống dưới, đi 'vào bên trong' mỗi hàm khi nó được gọi để đọc những gì nó làm. Khi một quy trình riêng biệt bắt đầu, nó giống như một người đọc khác bắt đầu đọc bên trong hàm, để mặc người đọc ban đầu tiếp tục đi xuống trang giấy.

Để bảo Go bắt đầu một goroutine mới, chúng ta chuyển một lời gọi hàm thành một câu lệnh `go` bằng cách đặt từ khóa `go` phía trước nó: `go doSomething()`.

```go
package concurrency

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		go func() {
			results[url] = wc(url)
		}()
	}

	return results
}
```

Bởi vì cách duy nhất để bắt đầu một goroutine là đặt `go` trước một lời gọi hàm, chúng ta thường sử dụng *anonymous functions* (hàm ẩn danh) khi muốn bắt đầu một goroutine. Một hàm ẩn danh trông giống hệt như một khai báo hàm bình thường, nhưng không có tên (không ngạc nhiên lắm). Bạn có thể thấy một ví dụ ở trên trong thân vòng lặp `for`.

Các hàm ẩn danh có một số tính năng làm cho chúng hữu ích, hai tính năng trong số đó mà chúng ta đang sử dụng ở trên. Thứ nhất, chúng có thể được thực thi ngay tại thời điểm chúng được khai báo - đây là những gì cặp ngoặc `()` ở cuối hàm ẩn danh đang làm. Thứ hai, chúng duy trì quyền truy cập vào phạm vi từ vựng (lexical scope) mà chúng được định nghĩa - tất cả các biến có sẵn tại thời điểm bạn khai báo hàm ẩn danh cũng có sẵn trong thân hàm.

Thân của hàm ẩn danh ở trên giống hệt như thân vòng lặp trước đó. Sự khác biệt duy nhất là mỗi lần lặp của vòng lặp sẽ bắt đầu một goroutine mới, đồng thời với quy trình hiện tại (hàm `WebsiteChecker`). Mỗi goroutine sẽ thêm kết quả của nó vào map results.

Nhưng khi chúng ta chạy `go test`:

```sh
--- FAIL: TestCheckWebsites (0.00s)
        CheckWebsites_test.go:31: Wanted map[http://google.com:true http://blog.gypsydave5.com:true waat://furhurterwe.geds:false], got map[]
FAIL
exit status 1
FAIL    github.com/gypsydave5/learn-go-with-tests/concurrency/v1        0.010s
```

### Một cái nhìn thoáng qua vào vũ trụ của concurrency...

Bạn có lẽ sẽ không nhận được kết quả này. Bạn có thể nhận được một thông báo panic mà chúng ta sẽ nói đến sau đây một chút. Đừng lo lắng nếu bạn nhận được kết quả đó, chỉ cần tiếp tục chạy test cho đến khi bạn *thực sự* nhận được kết quả ở trên. Hoặc giả vờ rằng bạn đã nhận được. Tùy bạn thôi. Chào mừng bạn đến với concurrency: khi nó không được xử lý chính xác, thật khó để đoán trước điều gì sẽ xảy ra. Đừng lo lắng - đó là lý do tại sao chúng ta viết các test, để giúp chúng ta biết khi nào mình đang xử lý concurrency một cách có thể đoán trước được.

### ... và chúng ta đã quay trở lại.

Chúng ta đã bị vướng bởi test ban đầu `CheckWebsites`, giờ đây nó đang trả về một map trống. Có chuyện gì đã xảy ra?

Không có goroutine nào mà vòng lặp `for` của chúng ta khởi động có đủ thời gian để thêm kết quả của chúng vào map `results`; hàm `WebsiteChecker` quá nhanh đối với chúng, và nó trả về map vẫn còn trống.

Để khắc phục điều này, chúng ta có thể chỉ cần đợi trong khi tất cả các goroutine thực hiện công việc của họ, rồi mới trả về. Hai giây chắc là đủ, phải không?

```go
package concurrency

import "time"

type WebsiteChecker func(string) bool

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)

	for _, url := range urls {
		go func() {
			results[url] = wc(url)
		}()
	}

	time.Sleep(2 * time.Second)

	return results
}
```

Bây giờ nếu bạn may mắn, bạn sẽ nhận được:

```sh
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v1        2.012s
```

Nhưng nếu bạn không may mắn (điều này dễ xảy ra hơn nếu bạn chạy chúng với benchmark vì bạn sẽ có nhiều lần thử hơn):

```sh
fatal error: concurrent map writes

goroutine 8 [running]:
runtime.throw(0x12c5895, 0x15)
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/panic.go:605 +0x95 fp=0xc420037700 sp=0xc4200376e0 pc=0x102d395
runtime.mapassign_faststr(0x1271d80, 0xc42007acf0, 0x12c6634, 0x17, 0x0)
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:783 +0x4f5 fp=0xc420037780 sp=0xc420037700 pc=0x100eb65
github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1(0xc42007acf0, 0x12d3938, 0x12c6634, 0x17)
        /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x71 fp=0xc4200377c0 sp=0xc420037780 pc=0x12308f1
runtime.goexit()
        /usr/local/Cellar/go/1.9.3/libexec/src/runtime/asm_amd64.s:2337 +0x1 fp=0xc4200377c8 sp=0xc4200377c0 pc=0x105cf01
created by github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker
        /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xa1

        ... còn nhiều dòng chữ đáng sợ khác nữa ...
```

Điều này dài và đáng sợ, nhưng tất cả những gì chúng ta cần làm là hít một hơi thật sâu và đọc stacktrace (dấu vết ngăn xếp): `fatal error: concurrent map writes`. Đôi khi, khi chúng ta chạy các bản kiểm thử, hai trong số các goroutine ghi vào map results cùng một lúc. Maps trong Go không thích việc có nhiều hơn một thứ cố gắng ghi vào chúng cùng lúc, và do đó xảy ra `fatal error`.

Đây là một *race condition* (điều kiện tranh đua), một lỗi xảy ra khi đầu ra của phần mềm phụ thuộc vào thời gian và trình tự của các sự kiện mà chúng ta không có quyền kiểm soát. Bởi vì chúng ta không thể kiểm soát chính xác thời điểm mỗi goroutine ghi vào map results, chúng ta dễ bị tổn thương khi hai goroutine ghi vào đó cùng một lúc.

Go có thể giúp chúng ta phát hiện các race condition bằng trình phát hiện tranh đua built-in của nó ([*race detector*][godoc_race_detector]). Để bật tính năng này, hãy chạy các test với cờ `race`: `go test -race`.

Bạn sẽ nhận được một số kết quả trông như thế này:

```sh
==================
WARNING: DATA RACE
Write at 0x00c420084d20 by goroutine 8:
  runtime.mapassign_faststr()
      /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:774 +0x0
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x82

Previous write at 0x00c420084d20 by goroutine 7:
  runtime.mapassign_faststr()
      /usr/local/Cellar/go/1.9.3/libexec/src/runtime/hashmap_fast.go:774 +0x0
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker.func1()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12 +0x82

Goroutine 8 (running) created at:
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xc4
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.TestWebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker_test.go:27 +0xad
  testing.tRunner()
      /usr/local/Cellar/go/1.9.3/libexec/src/testing/testing.go:746 +0x16c

Goroutine 7 (finished) created at:
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.WebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11 +0xc4
  github.com/gypsydave5/learn-go-with-tests/concurrency/v3.TestWebsiteChecker()
      /Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker_test.go:27 +0xad
  testing.tRunner()
      /usr/local/Cellar/go/1.9.3/libexec/src/testing/testing.go:746 +0x16c
==================
```

Các chi tiết, một lần nữa, lại khó đọc - nhưng `WARNING: DATA RACE` là khá rõ ràng. Đọc vào nội dung của lỗi, chúng ta có thể thấy hai goroutine khác nhau đang thực hiện ghi trên một map:

`Write at 0x00c420084d20 by goroutine 8:`

đang viết vào cùng một khối bộ nhớ với

`Previous write at 0x00c420084d20 by goroutine 7:`

Trên hết, chúng ta có thể thấy dòng mã nơi việc ghi đang diễn ra:

`/Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:12`

và dòng mã nơi goroutine 7 và 8 được khởi động:

`/Users/gypsydave5/go/src/github.com/gypsydave5/learn-go-with-tests/concurrency/v3/websiteChecker.go:11`

Mọi thứ bạn cần biết đều được in ra terminal của bạn - tất cả những gì bạn phải làm là đủ kiên nhẫn để đọc nó.

### Channels

Chúng ta có thể giải quyết race condition này bằng cách điều phối các goroutine của mình bằng các *channels*. Channels là một cấu trúc dữ liệu của Go có thể vừa nhận vừa gửi các giá trị. Các hoạt động này, cùng với các chi tiết của chúng, cho phép giao tiếp giữa các quy trình khác nhau.

Trong trường hợp này, chúng ta muốn nghĩ về sự giao tiếp giữa quy trình cha và mỗi goroutine mà nó tạo ra để thực hiện công việc chạy hàm `WebsiteChecker` với url.

```go
package concurrency

type WebsiteChecker func(string) bool
type result struct {
	string
	bool
}

func CheckWebsites(wc WebsiteChecker, urls []string) map[string]bool {
	results := make(map[string]bool)
	resultChannel := make(chan result)

	for _, url := range urls {
		go func() {
			resultChannel <- result{url, wc(url)}
		}()
	}

	for i := 0; i < len(urls); i++ {
		r := <-resultChannel
		results[r.string] = r.bool
	}

	return results
}
```

Bên cạnh map `results` hiện nay chúng ta có thêm một `resultChannel`, mà chúng ta `make` theo cùng một cách. `chan result` là kiểu của channel - một channel chứa các `result`. Kiểu mới, `result`, đã được tạo ra để liên kết giá trị trả về của `WebsiteChecker` với url đang được kiểm tra - đó là một struct gồm `string` và `bool`. Vì chúng ta không cần đặt tên cho cả hai giá trị, mỗi giá trị trong số đó là ẩn danh trong struct; điều này có thể hữu ích khi khó biết đặt tên cho một giá trị là gì.

Bây giờ khi chúng ta lặp qua các url, thay vì viết trực tiếp vào `map`, chúng ta đang gửi một struct `result` cho mỗi lần gọi `wc` đến `resultChannel` bằng *send statement* (câu lệnh gửi). Nó sử dụng toán tử `<-`, lấy một channel ở bên trái và một giá trị ở bên phải:

```go
// Send statement
resultChannel <- result{u, wc(u)}
```

Vòng lặp `for` tiếp theo lặp một lần cho mỗi url. Bên trong nó, chúng ta đang sử dụng *receive expression* (biểu thức nhận), biểu thức này gán một giá trị nhận được từ một channel cho một biến. Biểu thức này cũng sử dụng toán tử `<-`, nhưng với hai toán hạng bây giờ được đảo ngược: channel hiện ở bên phải và biến mà chúng ta đang gán cho ở bên trái:

```go
// Receive expression
r := <-resultChannel
```

Sau đó chúng ta sử dụng `result` nhận được để cập nhật map.

Bằng cách gửi các kết quả vào một channel, chúng ta có thể kiểm soát thời gian của mỗi lần ghi vào map kết quả, đảm bảo rằng nó diễn ra từng cái một. Mặc dù mỗi lần gọi `wc`, và mỗi lần gửi đến channel kết quả, đang diễn ra đồng thời trong quy trình riêng của nó, mỗi kết quả đang được xử lý từng cái một khi chúng ta lấy các giá trị ra khỏi channel kết quả bằng receive expression.

Chúng ta đã sử dụng concurrency cho phần mã mà chúng ta muốn làm cho nhanh hơn, đồng thời vẫn đảm bảo rằng phần không thể diễn ra đồng thời vẫn diễn ra một cách tuyến tính. Và chúng ta đã giao tiếp qua nhiều quy trình liên quan bằng cách sử dụng các channel.

Khi chúng ta chạy benchmark:

```sh
pkg: github.com/gypsydave5/learn-go-with-tests/concurrency/v2
BenchmarkCheckWebsites-8             100          23406615 ns/op
PASS
ok      github.com/gypsydave5/learn-go-with-tests/concurrency/v2        2.377s
```

23.406.615 nano giây - 0,023 giây, nhanh gấp khoảng một trăm lần so với hàm ban đầu. Một thành công rực rỡ.

## Tổng kết

Bài tập này ít sử dụng TDD hơn bình thường một chút. Theo một cách nào đó, chúng ta đã tham gia vào một quá trình tái cấu trúc dài cho hàm `CheckWebsites`; đầu vào và đầu ra không bao giờ thay đổi, nó chỉ nhanh hơn thôi. Nhưng các test chúng ta đã có sẵn, cũng như benchmark chúng ta đã viết, đã cho phép chúng ta tái cấu trúc `CheckWebsites` theo cách duy trì sự tin cậy rằng phần mềm vẫn hoạt động tốt, đồng thời chứng minh rằng nó thực sự đã trở nên nhanh hơn.

Để làm cho nó nhanh hơn, chúng ta đã tìm hiểu về:

- *goroutines*, đơn vị cơ bản trong lập trình đồng thời của Go, cho phép chúng ta quản lý nhiều hơn một yêu cầu kiểm tra trang web.
- *anonymous functions* (hàm ẩn danh), mà chúng ta đã sử dụng để bắt đầu từng quy trình đồng thời kiểm tra các trang web.
- *channels*, để giúp tổ chức và kiểm soát giao tiếp giữa các quy trình khác nhau, cho phép chúng ta tránh lỗi *race condition*.
- *trình phát hiện tranh đua (race detector)* đã giúp chúng ta gỡ lỗi cho các mã đồng thời.

### Hãy làm cho nó nhanh (Make it fast)

Một công thức phát triển phần mềm theo phương thức Agile, thường bị gán nhầm cho Kent Beck, là:

> [Hãy làm nó hoạt động, làm nó đúng, rồi làm nó nhanh][wrf]

Trong đó 'hoạt động' (work) là làm cho các test vượt qua, 'đúng' (right) là tái cấu trúc mã, và 'nhanh' (fast) là tối ưu hóa mã để mã chạy nhanh hơn chẳng hạn. Chúng ta chỉ có thể 'làm cho nó nhanh' khi chúng ta đã làm cho nó hoạt động và làm cho nó đúng. Chúng ta đã may mắn khi mã được giao cho đã được chứng minh là hoạt động tốt, và không cần phải tái cấu trúc. Chúng ta không bao giờ nên cố gắng 'làm cho nó nhanh' trước khi hai bước kia được thực hiện vì:

> [Tối ưu hóa sớm là nguồn gốc của mọi quỷ dữ][popt]
> -- Donald Knuth

[DI]: dependency-injection.md
[wrf]: http://wiki.c2.com/?MakeItWorkMakeItRightMakeItFast
[godoc_race_detector]: https://blog.golang.org/race-detector
[popt]: http://wiki.c2.com/?PrematureOptimization
