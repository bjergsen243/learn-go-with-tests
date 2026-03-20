# HTTP Server

**[Tất cả mã nguồn của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/http-server)**

Bạn đã được yêu cầu tạo một máy chủ web nơi người dùng có thể theo dõi số trận thắng của những người chơi.

-   `GET /players/{name}` nên trả về một con số cho biết tổng số trận thắng.
-   `POST /players/{name}` nên ghi lại một trận thắng cho tên đó, tăng lên cho mỗi lần `POST` tiếp theo.

Chúng ta sẽ tuân theo cách tiếp cận TDD, tạo ra phần mềm hoạt động nhanh nhất có thể và sau đó thực hiện các cải tiến lặp đi lặp lại nhỏ cho đến khi có giải pháp hoàn chỉnh. Bằng cách thực hiện phương pháp này, chúng ta:

-   Giữ cho phạm vi vấn đề nhỏ tại bất kỳ thời điểm nào.
-   Không sa đà vào những hướng đi sai lầm.
-   Nếu chúng ta bị mắc kẹt hoặc lạc lối, việc hoàn tác (revert) sẽ không làm mất quá nhiều công sức.

## Đỏ, xanh, tái cấu trúc (Red, green, refactor)

Xuyên suốt cuốn sách này, chúng ta đã nhấn mạnh quy trình TDD là viết một bản kiểm thử và xem nó thất bại (đỏ), viết lượng mã nguồn *tối thiểu* để làm cho nó hoạt động (xanh), và sau đó tái cấu trúc.

Kỷ luật viết lượng mã nguồn tối thiểu này rất quan trọng về mặt an toàn mà TDD mang lại cho bạn. Bạn nên cố gắng thoát khỏi trạng thái "đỏ" càng sớm càng tốt.

Kent Beck mô tả nó như sau:

> Làm cho bản kiểm thử hoạt động nhanh chóng, chấp nhận bất kỳ "tội lỗi" nào cần thiết trong quá trình đó.

Bạn có thể phạm những "tội lỗi" này vì sau đó bạn sẽ tái cấu trúc với sự đảm bảo an toàn từ các bản kiểm thử.

### Điều gì xảy ra nếu bạn không làm điều này?

Càng thực hiện nhiều thay đổi khi đang ở trạng thái đỏ, bạn càng có nhiều khả năng gây ra thêm các vấn đề mà không được bao phủ bởi các bản kiểm thử.

Ý tưởng là viết mã nguồn hữu ích một cách lặp đi lặp lại với các bước nhỏ, được dẫn dắt bởi các bản kiểm thử để bạn không rơi vào trạng thái bế tắc trong nhiều giờ.

### Con gà hay quả trứng

Làm thế nào chúng ta có thể xây dựng điều này một cách lặp đi lặp lại? Chúng ta không thể `GET` một người chơi mà không lưu trữ thứ gì đó, và có vẻ khó biết liệu `POST` có hoạt động hay không nếu điểm cuối (endpoint) `GET` chưa tồn tại.

Đây là lúc *mocking* tỏa sáng.

-   `GET` sẽ cần một *thứ* gọi là `PlayerStore` để lấy điểm cho một người chơi. Đây nên là một interface để khi kiểm thử, chúng ta có thể tạo một stub đơn giản nhằm kiểm thử mã nguồn của mình mà không cần phải triển khai bất kỳ mã nguồn lưu trữ thực tế nào.
-   Đối với `POST`, chúng ta có thể *spy* (giám sát) các lệnh gọi của nó tới `PlayerStore` để đảm bảo nó lưu trữ người chơi một cách chính xác. Việc triển khai lưu trữ của chúng ta sẽ không bị ràng buộc (coupled) vào việc truy xuất.
-   Để có được phần mềm hoạt động nhanh chóng, chúng ta có thể thực hiện một triển khai trong bộ nhớ (in-memory) rất đơn giản và sau đó có thể tạo một triển khai được hỗ trợ bởi bất kỳ cơ chế lưu trữ nào chúng ta muốn.

## Viết bản kiểm thử trước tiên

Chúng ta có thể viết một bản kiểm thử và làm cho nó vượt qua bằng cách trả về một giá trị được mã hóa cứng (hard-coded) để bắt đầu. Kent Beck gọi đây là "Faking it" (Giả vờ). Khi đã có một bản kiểm thử hoạt động, chúng ta có thể viết thêm các bản kiểm thử để giúp loại bỏ hằng số đó.

Bằng cách thực hiện bước rất nhỏ này, điều quan trọng là chúng ta có thể bắt đầu làm cho cấu trúc tổng thể của dự án hoạt động chính xác mà không phải lo lắng quá nhiều về logic ứng dụng.

Để tạo một máy chủ web trong Go, thông thường bạn sẽ gọi [ListenAndServe](https://golang.org/pkg/net/http/#ListenAndServe).

```go
func ListenAndServe(addr string, handler Handler) error
```

Điều này sẽ bắt đầu một máy chủ web lắng nghe trên một cổng, tạo một goroutine cho mỗi yêu cầu và chạy nó thông qua một [`Handler`](https://golang.org/pkg/net/http/#Handler).

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

Một kiểu thực thi interface Handler bằng cách triển khai phương thức `ServeHTTP`, mong đợi hai đối số: đối số thứ nhất là nơi chúng ta *viết phản hồi* (response) và đối số thứ hai là yêu cầu HTTP (request) được gửi đến máy chủ.

Hãy tạo một tệp tên là `server_test.go` và viết một bản kiểm thử cho một hàm `PlayerServer` nhận hai đối số đó. Yêu cầu được gửi đến sẽ là lấy điểm của một người chơi, mà chúng ta mong đợi là `"20"`.

```go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		got := response.Body.String()
		want := "20"

		if got != want {
			t.Errorf("got %q, want %q", got, want)
		}
	})
}
```

Để kiểm thử máy chủ của mình, chúng ta sẽ cần một `Request` để gửi vào và chúng ta sẽ muốn *spy* những gì handler của chúng ta viết vào `ResponseWriter`.

-   Chúng ta sử dụng `http.NewRequest` để tạo một yêu cầu. Đối số đầu tiên là phương thức của yêu cầu và đối số thứ hai là đường dẫn của yêu cầu. Đối số `nil` đề cập đến thân (body) của yêu cầu, mà chúng ta không cần thiết lập trong trường hợp này.
-   `net/http/httptest` có một spy đã được tạo sẵn cho chúng ta gọi là `ResponseRecorder`, vì vậy chúng ta có thể sử dụng nó. Nó có nhiều phương thức hữu ích để kiểm tra những gì đã được viết dưới dạng phản hồi.

## Thử chạy bản kiểm thử

`./server_test.go:13:2: undefined: PlayerServer`

## Viết lượng mã nguồn tối thiểu để bản kiểm thử chạy và kiểm tra kết quả lỗi

Trình biên dịch ở đây để giúp đỡ, hãy lắng nghe nó.

Tạo một tệp tên là `server.go` và định nghĩa `PlayerServer`.

```go
func PlayerServer() {}
```

Thử lại lần nữa:

```
./server_test.go:13:14: too many arguments in call to PlayerServer
    have (*httptest.ResponseRecorder, *http.Request)
    want ()
```

Thêm các đối số vào hàm của chúng ta:

```go
import "net/http"

func PlayerServer(w http.ResponseWriter, r *http.Request) {

}
```

Mã nguồn bây giờ đã biên dịch và bản kiểm thử thất bại:

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- FAIL: TestGETPlayers/returns_Pepper's_score (0.00s)
        server_test.go:20: got '', want '20'
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Từ chương DI, chúng ta đã chạm đến các máy chủ HTTP với hàm `Greet`. Chúng ta đã biết rằng `ResponseWriter` của net/http cũng triển khai interface `Writer` của io, vì vậy chúng ta có thể sử dụng `fmt.Fprint` để gửi các chuỗi dưới dạng phản hồi HTTP.

```go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "20")
}
```

Bản kiểm thử bây giờ sẽ vượt qua.

## Hoàn thiện khung ứng dụng (Scaffolding)

Chúng ta muốn kết nối điều này vào một ứng dụng. Điều này quan trọng vì:

-   Chúng ta sẽ có *phần mềm thực tế đang hoạt động*, chúng ta không muốn viết các bản kiểm thử chỉ để cho có, thật tốt khi thấy mã nguồn hoạt động.
-   Khi chúng ta tái cấu trúc mã nguồn, có khả năng chúng ta sẽ thay đổi cấu trúc của chương trình. Chúng ta muốn đảm bảo điều này cũng được phản ánh trong ứng dụng như một phần của cách tiếp cận lặp đi lặp lại.

Tạo một tệp `main.go` mới cho ứng dụng của chúng ta và đưa mã nguồn này vào:

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(PlayerServer)
	log.Fatal(http.ListenAndServe(":5000", handler))
}
```

Cho đến nay, tất cả mã nguồn ứng dụng của chúng ta đều nằm trong một tệp, tuy nhiên, đây không phải là phương pháp tốt nhất cho các dự án lớn hơn, nơi bạn sẽ muốn tách biệt mọi thứ thành các tệp khác nhau.

Để chạy ứng dụng này, hãy thực hiện lệnh `go build`, lệnh này sẽ lấy tất cả các tệp `.go` trong thư mục và xây dựng cho bạn một chương trình. Sau đó, bạn có thể thực thi nó bằng lệnh `./myprogram`.

### `http.HandlerFunc`

Trước đó, chúng ta đã khám phá rằng interface `Handler` là những gì chúng ta cần triển khai để tạo một máy chủ. *Thông thường*, chúng ta làm điều đó bằng cách tạo một `struct` và làm cho nó triển khai interface bằng cách thực hiện phương thức ServeHTTP của chính nó. Tuy nhiên, trường hợp sử dụng của struct là để giữ dữ liệu nhưng *hiện tại* chúng ta không có trạng thái, vì vậy việc tạo một cái cảm thấy không đúng lắm.

[HandlerFunc](https://golang.org/pkg/net/http/#HandlerFunc) giúp chúng ta tránh được điều này.

> Kiểu HandlerFunc là một bộ điều hợp (adapter) cho phép sử dụng các hàm thông thường làm các HTTP handler. Nếu f là một hàm có chữ ký thích hợp, HandlerFunc(f) là một Handler gọi f.

```go
type HandlerFunc func(ResponseWriter, *Request)
```

Từ tài liệu, chúng ta thấy rằng kiểu `HandlerFunc` đã triển khai phương thức `ServeHTTP`. Bằng cách ép kiểu hàm `PlayerServer` của chúng ta với nó, chúng ta đã triển khai `Handler` theo yêu cầu.

### `http.ListenAndServe(":5000"...)`

`ListenAndServe` nhận một cổng để lắng nghe và một `Handler`. Nếu có vấn đề, máy chủ web sẽ trả về một lỗi; ví dụ cho điều đó có thể là cổng đã được sử dụng rồi. Vì lý do đó, chúng ta bọc lời gọi trong `log.Fatal` để ghi lại lỗi cho người dùng.

Những gì chúng ta sẽ làm bây giờ là viết một bản kiểm thử *khác* để buộc chúng ta thực hiện một thay đổi tích cực nhằm cố gắng thoát khỏi giá trị được mã hóa cứng.

## Viết bản kiểm thử trước tiên

Chúng ta sẽ thêm một bản kiểm thử phụ (subtest) khác vào bộ kiểm thử của mình nhằm cố gắng lấy điểm của một người chơi khác, điều này sẽ làm hỏng cách tiếp cận mã hóa cứng của chúng ta.

```go
t.Run("returns Floyd's score", func(t *testing.T) {
	request, _ := http.NewRequest(http.MethodGet, "/players/Floyd", nil)
	response := httptest.NewRecorder()

	PlayerServer(response, request)

	got := response.Body.String()
	want := "10"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
})
```

Bạn có thể đã nghĩ:

> Chắc chắn chúng ta cần một loại khái niệm về lưu trữ để kiểm soát người chơi nào nhận được số điểm nào. Thật kỳ lạ khi các giá trị có vẻ tùy ý trong các bản kiểm thử của chúng ta.

Hãy nhớ rằng chúng ta chỉ đang cố gắng thực hiện các bước nhỏ nhất có thể, vì vậy hiện tại chúng ta chỉ đang cố gắng phá bỏ hằng số.

## Thử chạy bản kiểm thử

```
=== RUN   TestGETPlayers/returns_Pepper's_score
    --- PASS: TestGETPlayers/returns_Pepper's_score (0.00s)
=== RUN   TestGETPlayers/returns_Floyd's_score
    --- FAIL: TestGETPlayers/returns_Floyd's_score (0.00s)
        server_test.go:34: got '20', want '10'
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

```go
// server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	if player == "Pepper" {
		fmt.Fprint(w, "20")
		return
	}

	if player == "Floyd" {
		fmt.Fprint(w, "10")
		return
	}
}
```

Bản kiểm thử này đã buộc chúng ta thực sự phải nhìn vào URL của yêu cầu và đưa ra quyết định. Vì vậy, trong khi trong đầu chúng ta có thể đã lo lắng về player store và interfaces, bước hợp lý tiếp theo thực sự có vẻ là về *định tuyến* (routing).

Nếu chúng ta bắt đầu với mã nguồn của store thì lượng thay đổi chúng ta phải thực hiện sẽ rất lớn so với điều này. **Đây là một bước nhỏ hơn hướng tới mục tiêu cuối cùng của chúng ta và được dẫn dắt bởi các bản kiểm thử**.

Chúng ta đang chống lại sự cám dỗ sử dụng bất kỳ thư viện định tuyến nào ngay bây giờ, chỉ là bước nhỏ nhất để bản kiểm thử của chúng ta vượt qua.

`r.URL.Path` trả về đường dẫn của yêu cầu mà sau đó chúng ta có thể sử dụng [`strings.TrimPrefix`](https://golang.org/pkg/strings/#TrimPrefix) để cắt bỏ `/players/` nhằm lấy người chơi được yêu cầu. Nó không quá mạnh mẽ nhưng sẽ giải quyết được vấn đề hiện tại.

## Tái cấu trúc (Refactor)

Chúng ta có thể đơn giản hóa `PlayerServer` bằng cách tách việc truy xuất điểm thành một hàm:

```go
// server.go
func PlayerServer(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	fmt.Fprint(w, GetPlayerScore(player))
}

func GetPlayerScore(name string) string {
	if name == "Pepper" {
		return "20"
	}

	if name == "Floyd" {
		return "10"
	}

	return ""
}
```

Và chúng ta có thể làm cho mã nguồn trong các bản kiểm thử gọn gàng hơn bằng cách tạo ra một số helper:

```go
// server_test.go
func TestGETPlayers(t *testing.T) {
	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		PlayerServer(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

Tuy nhiên, chúng ta vẫn chưa nên hài lòng. Việc máy chủ của chúng ta biết các số điểm cảm thấy không đúng.

Việc tái cấu trúc đã làm cho hướng đi tiếp theo trở nên khá rõ ràng.

Chúng ta đã chuyển việc tính toán điểm ra khỏi thân chính của handler vào một hàm `GetPlayerScore`. Đây có vẻ là nơi thích hợp để tách biệt các mối quan tâm bằng cách sử dụng interface.

Hãy chuyển hàm mà chúng ta đã tái cấu trúc thành một interface:

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
}
```

Để `PlayerServer` của chúng ta có thể sử dụng `PlayerStore`, nó sẽ cần một tham chiếu đến một cái. Bây giờ có vẻ là thời điểm thích hợp để thay đổi kiến trúc sao cho `PlayerServer` giờ đây là một `struct`.

```go
type PlayerServer struct {
	store PlayerStore
}
```

Cuối cùng, bây giờ chúng ta sẽ triển khai interface `Handler` bằng cách thêm một phương thức vào struct mới của mình và chuyển mã nguồn handler hiện tại vào đó.

```go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

Thay đổi duy nhất khác là giờ đây chúng ta gọi `p.store.GetPlayerScore` để lấy điểm, thay vì hàm cục bộ mà chúng ta đã định nghĩa (mà bây giờ chúng ta có thể xóa).

Dưới đây là danh sách đầy đủ mã nguồn cho máy chủ của chúng ta:

```go
// server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
}

type PlayerServer struct {
	store PlayerStore
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

### Khắc phục các vấn đề

Đây là một vài thay đổi lớn và chúng ta biết rằng các bản kiểm thử và ứng dụng của chúng ta sẽ không còn biên dịch được nữa, nhưng hãy bình tĩnh và để trình biên dịch giải quyết vấn đề đó.

`./main.go:9:58: type PlayerServer is not an expression`

Chúng ta cần thay đổi các bản kiểm thử để thay vào đó là tạo một phiên bản (instance) mới của `PlayerServer` và sau đó gọi phương thức `ServeHTTP` của nó.

```go
// server_test.go
func TestGETPlayers(t *testing.T) {
	server := &PlayerServer{}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

Hãy chú ý rằng chúng ta vẫn chưa lo lắng về việc tạo các store *ngay bây giờ*, chúng ta chỉ muốn trình biên dịch thông qua nhanh nhất có thể.

Bạn nên hình thành thói quen ưu tiên có mã nguồn được biên dịch và sau đó là mã nguồn vượt qua các bản kiểm thử.

Bằng cách thêm nhiều chức năng hơn (như stub store) trong khi mã nguồn vẫn chưa biên dịch được, chúng ta đang mở ra khả năng gặp phải *nhiều hơn* các vấn đề về biên dịch.

Bây giờ `main.go` cũng sẽ không biên dịch được vì lý do tương tự.

```go
func main() {
	server := &PlayerServer{}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Cuối cùng, mọi thứ đã được biên dịch nhưng các bản kiểm thử lại thất bại:

```
=== RUN   TestGETPlayers/returns_the_Pepper's_score
panic: runtime error: invalid memory address or nil pointer dereference [recovered]
    panic: runtime error: invalid memory address or nil pointer dereference
```

Điều này là do chúng ta chưa truyền vào một `PlayerStore` trong các bản kiểm thử của mình. Chúng ta sẽ cần tạo một stub cho nó.

```go
// server_test.go
type StubPlayerStore struct {
	scores map[string]int
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}
```

Một `map` là cách nhanh chóng và dễ dàng để tạo một kho lưu trữ key/value stub cho các bản kiểm thử của chúng ta. Bây giờ hãy tạo một trong những store này cho các bản kiểm thử của mình và gửi nó vào `PlayerServer`.

```go
// server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertResponseBody(t, response.Body.String(), "10")
	})
}
```

Các bản kiểm thử của chúng ta hiện đã vượt qua và trông ổn hơn. *Ý định* đằng sau mã nguồn của chúng ta bây giờ rõ ràng hơn nhờ việc giới thiệu store. Chúng ta đang nói với người đọc rằng vì chúng ta có *dữ liệu này trong một `PlayerStore`* nên khi bạn sử dụng nó với một `PlayerServer`, bạn sẽ nhận được các phản hồi sau đây.

### Chạy ứng dụng

Bây giờ các bản kiểm thử đã vượt qua, điều cuối cùng chúng ta cần làm để hoàn thành việc tái cấu trúc này là kiểm tra xem ứng dụng có hoạt động hay không. Chương trình sẽ khởi động nhưng bạn sẽ nhận được một phản hồi rất tệ nếu thử truy cập máy chủ tại `http://localhost:5000/players/Pepper`.

Lý do cho việc này là chúng ta chưa truyền vào một `PlayerStore`.

Chúng ta sẽ cần thực hiện một sự triển khai (implementation) của nó, nhưng điều đó khó thực hiện ngay bây giờ vì chúng ta chưa lưu trữ bất kỳ dữ liệu có ý nghĩa nào, vì vậy nó sẽ phải được mã hóa cứng tạm thời.

```go
// main.go
type InMemoryPlayerStore struct{}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return 123
}

func main() {
	server := &PlayerServer{&InMemoryPlayerStore{}}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Nếu bạn chạy `go build` lại và truy cập cùng một URL đó, bạn sẽ nhận được `"123"`. Không mấy tuyệt vời, nhưng cho đến khi chúng ta lưu trữ dữ liệu thực tế thì đó là điều tốt nhất chúng ta có thể làm. Điều đó cũng gợi ý rằng không ổn lắm khi ứng dụng chính của chúng ta khởi động nhưng lại không thực sự hoạt động. Chúng ta đã phải kiểm thử thủ công để thấy vấn đề.

Chúng ta có một vài lựa chọn cho việc sẽ làm gì tiếp theo:

-   Xử lý kịch bản khi người chơi không tồn tại.
-   Xử lý kịch bản `POST /players/{name}`.

Trong khi kịch bản `POST` đưa chúng ta đến gần hơn với "luồng hoạt động trơn tru" (happy path), tôi cảm thấy sẽ dễ dàng hơn khi giải quyết kịch bản thiếu người chơi trước vì chúng ta đã đang ở trong ngữ cảnh đó rồi. Chúng ta sẽ thực hiện những phần còn lại sau.

## Viết bản kiểm thử trước tiên

Thêm kịch bản thiếu người chơi vào bộ kiểm thử hiện có của chúng ta:

```go
// server_test.go
t.Run("returns 404 on missing players", func(t *testing.T) {
	request := newGetScoreRequest("Apollo")
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := response.Code
	want := http.StatusNotFound

	if got != want {
		t.Errorf("got status %d want %d", got, want)
	}
})
```

## Thử chạy bản kiểm thử

```
=== RUN   TestGETPlayers/returns_404_on_missing_players
    --- FAIL: TestGETPlayers/returns_404_on_missing_players (0.00s)
        server_test.go:56: got status 200 want 404
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

```go
// server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	w.WriteHeader(http.StatusNotFound)

	fmt.Fprint(w, p.store.GetPlayerScore(player))
}
```

Đôi khi tôi thực sự cảm thấy chán nản khi những người ủng hộ TDD nói rằng "hãy đảm bảo bạn chỉ viết lượng mã nguồn tối thiểu để bản kiểm thử vượt qua" vì nó có vẻ rất giáo điều.

Nhưng kịch bản này minh họa khá tốt cho ví dụ đó. Tôi đã thực hiện những điều tối thiểu (biết rằng nó không đúng), đó là viết `StatusNotFound` cho **tất cả các phản hồi** nhưng tất cả các bản kiểm thử của chúng ta đều vượt qua!

**Bằng cách thực hiện những điều tối thiểu để làm cho các bản kiểm thử vượt qua, nó có thể làm lộ ra các lỗ hổng trong bản kiểm thử của bạn**. Trong trường hợp của chúng ta, chúng ta chưa khẳng định rằng mình nên nhận được một `StatusOK` khi người chơi *có* tồn tại trong store.

Cập nhật hai bản kiểm thử khác để khẳng định trạng thái và sửa mã nguồn.

Dưới đây là các bản kiểm thử mới:

```go
// server_test.go
func TestGETPlayers(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{
			"Pepper": 20,
			"Floyd":  10,
		},
	}
	server := &PlayerServer{&store}

	t.Run("returns Pepper's score", func(t *testing.T) {
		request := newGetScoreRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "20")
	})

	t.Run("returns Floyd's score", func(t *testing.T) {
		request := newGetScoreRequest("Floyd")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
		assertResponseBody(t, response.Body.String(), "10")
	})

	t.Run("returns 404 on missing players", func(t *testing.T) {
		request := newGetScoreRequest("Apollo")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusNotFound)
	})
}

func assertStatus(t testing.TB, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("did not get correct status, got %d, want %d", got, want)
	}
}

func newGetScoreRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodGet, fmt.Sprintf("/players/%s", name), nil)
	return req
}

func assertResponseBody(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("response body is wrong, got %q want %q", got, want)
	}
}
```

Chúng ta đang kiểm tra trạng thái trong tất cả các bản kiểm thử của mình ngay bây giờ, vì vậy tôi đã tạo một helper `assertStatus` để tạo điều kiện thuận lợi cho việc đó.

Bây giờ hai bản kiểm thử đầu tiên của chúng ta thất bại vì mã lỗi 404 thay vì 200, vì vậy chúng ta có thể sửa `PlayerServer` để chỉ trả về "không tìm thấy" nếu số điểm bằng 0.

```go
// server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

### Lưu trữ số điểm

Bây giờ chúng ta đã có thể truy xuất số điểm từ store, điều hợp lý tiếp theo là có thể lưu trữ số điểm mới.

## Viết bản kiểm thử trước tiên

```go
// server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it returns accepted on POST", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodPost, "/players/Pepper", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)
	})
}
```

Đầu tiên chúng ta hãy kiểm tra xem mình có nhận được mã trạng thái chính xác hay không nếu chúng ta truy cập vào route cụ thể với POST. Điều này cho phép chúng ta dẫn dắt chức năng chấp nhận một loại yêu cầu khác và xử lý nó khác với `GET /players/{name}`. Một khi điều này hoạt động, chúng ta có thể bắt đầu khẳng định sự tương tác của handler với store.

## Thử chạy bản kiểm thử

```
=== RUN   TestStoreWins/it_returns_accepted_on_POST
    --- FAIL: TestStoreWins/it_returns_accepted_on_POST (0.00s)
        server_test.go:70: did not get correct status, got 404, want 202
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Hãy nhớ rằng chúng ta đang cố tình phạm "tội lỗi", vì vậy một câu lệnh `if` dựa trên phương thức của yêu cầu sẽ giải quyết được vấn đề.

```go
// server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	if r.Method == http.MethodPost {
		w.WriteHeader(http.StatusAccepted)
		return
	}

	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}
```

## Tái cấu trúc

Handler bây giờ trông có vẻ hơi lộn xộn. Hãy tách mã nguồn ra để dễ theo dõi hơn và cô lập các chức năng khác nhau vào các hàm mới.

```go
// server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	switch r.Method {
	case http.MethodPost:
		p.processWin(w)
	case http.MethodGet:
		p.showScore(w, r)
	}

}

func (p *PlayerServer) showScore(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter) {
	w.WriteHeader(http.StatusAccepted)
}
```

Điều này làm cho khía cạnh định tuyến của `ServeHTTP` rõ ràng hơn một chút và có nghĩa là các lần lặp tiếp theo của chúng ta về việc lưu trữ có thể chỉ nằm trong `processWin`.

Tiếp theo, chúng ta muốn kiểm tra xem khi thực hiện `POST /players/{name}` thì `PlayerStore` của chúng ta có được yêu cầu ghi lại trận thắng hay không.

## Viết bản kiểm thử trước tiên

Chúng ta có thể hoàn thành việc này bằng cách mở rộng `StubPlayerStore` của mình với một phương thức `RecordWin` mới và sau đó theo dõi các lần gọi của nó.

```go
// server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}

func (s *StubPlayerStore) RecordWin(name string) {
	s.winCalls = append(s.winCalls, name)
}
```

Bây giờ hãy mở rộng bản kiểm thử của chúng ta để kiểm tra số lần gọi:

```go
// server_test.go
func TestStoreWins(t *testing.T) {
	store := StubPlayerStore{
		map[string]int{},
	}
	server := &PlayerServer{&store}

	t.Run("it records wins when POST", func(t *testing.T) {
		request := newPostWinRequest("Pepper")
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusAccepted)

		if len(store.winCalls) != 1 {
			t.Errorf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
		}
	})
}

func newPostWinRequest(name string) *http.Request {
	req, _ := http.NewRequest(http.MethodPost, fmt.Sprintf("/players/%s", name), nil)
	return req
}
```

## Thử chạy bản kiểm thử

```
./server_test.go:26:20: too few values in struct initializer
./server_test.go:65:20: too few values in struct initializer
```

## Viết lượng mã nguồn tối thiểu để bản kiểm thử chạy và kiểm tra kết quả lỗi

Chúng ta cần cập nhật mã nguồn của mình ở những nơi chúng ta tạo một `StubPlayerStore` vì chúng ta đã thêm một trường mới:

```go
// server_test.go
store := StubPlayerStore{
	map[string]int{},
	nil,
}
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

```go
// server.go
func (p *PlayerServer) processWin(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

Chúng ta cần cập nhật `processWin` để nhận `*http.Request` để có thể trích xuất tên người chơi.

```go
// server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, r)
	case http.MethodGet:
		p.showScore(w, r)
	}
}
```

## Tái cấu trúc

Bây giờ mã nguồn của chúng ta đã vượt qua các bản kiểm thử, chúng ta nên hít một hơi thật sâu và xem xét lại mã nguồn.

Hàm `ServeHTTP` của chúng ta đang thực hiện khá nhiều việc: nó chịu trách nhiệm điều phối các yêu cầu (routing) và đồng thời cũng xử lý logic phản hồi. Điều này sẽ trở nên khó khăn hơn khi chúng ta thêm nhiều endpoint hơn.

Go cung cấp một giải pháp tích hợp sẵn rất mạnh mẽ gọi là `ServeMux`.

> ServeMux là một bộ dồn kênh (multiplexer) yêu cầu HTTP. Nó so khớp URL của mỗi yêu cầu được nhận với một danh sách các mẫu đã đăng ký và gọi handler cho mẫu khớp gần nhất với URL.

Hãy sử dụng nó để tái cấu trúc mã nguồn của chúng ta.

```go
// server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	router := http.NewServeMux()
	router.Handle("/players/", http.HandlerFunc(p.showScore))
	router.Handle("/players/", http.HandlerFunc(p.processWin))

	router.ServeHTTP(w, r)
}
```

Vấn đề ở đây là chúng ta không thể đăng ký cùng một đường dẫn cho các phương thức HTTP khác nhau với `ServeMux` cơ bản. Chúng ta cần lồng chúng lại hoặc sử dụng một trình định tuyến khác.

Hãy tiếp tục sử dụng `switch` cho bây giờ vì nó đơn giản, nhưng hãy di chuyển logic lấy tên người chơi vào một phương thức helper.

```go
// server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}

func (p *PlayerServer) showScore(w http.ResponseWriter, player string) {
	score := p.store.GetPlayerScore(player)

	if score == 0 {
		w.WriteHeader(http.StatusNotFound)
	}

	fmt.Fprint(w, score)
}

func (p *PlayerServer) processWin(w http.ResponseWriter, player string) {
	p.store.RecordWin(player)
	w.WriteHeader(http.StatusAccepted)
}
```

Điều này trông sạch sẽ hơn rất nhiều. Logic lấy tên người chơi chỉ được thực hiện ở một nơi.

### Triển khai store thực tế

Bước tiếp theo của chúng ta là thực hiện một triển khai `PlayerStore` thực tế thay vì stub. Chúng ta sẽ bắt đầu với phiên bản trong bộ nhớ (`InMemoryPlayerStore`).

## Viết bản kiểm thử trước tiên

Hãy tạo một file mới `in_memory_player_store_test.go` (hoặc tích hợp vào file hiện có).

```go
func TestInMemoryPlayerStore(t *testing.T) {
	t.Run("it stores wins for players", func(t *testing.T) {
		store := InMemoryPlayerStore{map[string]int{}}

		player := "Pepper"
		store.RecordWin(player)
		store.RecordWin(player)
		store.RecordWin(player)

		got := store.GetPlayerScore(player)
		want := 3

		if got != want {
			t.Errorf("got %d want %d", got, want)
		}
	})
}
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

```go
// in_memory_player_store.go
type InMemoryPlayerStore struct {
	store map[string]int
}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return i.store[name]
}

func (i *InMemoryPlayerStore) RecordWin(name string) {
	i.store[name]++
}
```

Đừng quên khởi tạo map trong `main.go`.

```go
// main.go
func main() {
	server := &PlayerServer{&InMemoryPlayerStore{map[string]int{}}}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Bây giờ chúng ta đã có một ứng dụng chạy được thực sự! Bạn có thể `POST` để tăng điểm và `GET` để xem điểm.

### JSON và Định dạng Phản hồi

Endpoint tiếp theo mà chúng ta muốn có là một bảng xếp hạng (league table). Nó nên trả về danh sách những người chơi và số trận thắng của họ dưới dạng JSON.

- `GET /league` nên trả về `[{ "Name": "Chris", "Wins": 20 }]`

## Viết bản kiểm thử trước tiên

```go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := &PlayerServer{&store}

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Bây giờ chúng ta thực sự cần một trình định tuyến (router) vì chúng ta có các đường dẫn khác nhau.

```go
// server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	router.ServeHTTP(w, r)
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) playersHandler(w http.ResponseWriter, r *http.Request) {
	player := strings.TrimPrefix(r.URL.Path, "/players/")

	switch r.Method {
	case http.MethodPost:
		p.processWin(w, player)
	case http.MethodGet:
		p.showScore(w, player)
	}
}
```

Vấn đề là chúng ta đang tạo một trình định tuyến mới cho *mỗi* yêu cầu. Điều này không hiệu quả. Hãy sử dụng một kỹ thuật gọi là "constructor" để tạo trình định tuyến một lần khi server được tạo.

```go
// server.go
type PlayerServer struct {
	store  PlayerStore
	router *http.ServeMux
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := &PlayerServer{
		store:  store,
		router: http.NewServeMux(),
	}

	p.router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	p.router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	return p
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	p.router.ServeHTTP(w, r)
}
```

Đừng quên cập nhật tất cả các lần khởi tạo `&PlayerServer{...}` thành `NewPlayerServer(...)`.

## Tái cấu trúc - Trả về JSON

Bây giờ hãy làm cho `leagueHandler` trả về một số dữ liệu JSON thực tế.

```go
type Player struct {
	Name string
	Wins int
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	league := []Player{
		{"Chris", 20},
	}

	w.Header().Set("content-type", "application/json")
	json.NewEncoder(w).Encode(league)
}
```

Chúng ta nên cập nhật bản kiểm thử để kiểm tra xem JSON có đúng không.

```go
func TestLeague(t *testing.T) {
	// ... khởi tạo ...
	t.Run("it returns the league table as JSON", func(t *testing.T) {
		wantedLeague := []Player{
			{"Cleo", 32},
			{"Chris", 20},
			{"Tiest", 14},
		}

		store.league = wantedLeague
		server := NewPlayerServer(&store)

		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player
		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)

		if !reflect.DeepEqual(got, wantedLeague) {
			t.Errorf("got %v want %v", got, wantedLeague)
		}
	})
}
```

Chúng ta cần cập nhật interface `PlayerStore` để bao gồm phương thức `GetLeague()`.

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() []Player
}
```

Và cập nhật handler:

```go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", "application/json")
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

## Tổng kết

Trong chương này, chúng ta đã học cách xây dựng một máy chủ HTTP trong Go từng bước một thông qua TDD.

-   Chúng ta đã bắt đầu với một hàm đơn giản và sau đó chuyển sang một `struct` triển khai interface `http.Handler`.
-   Chúng ta đã sử dụng `httptest.ResponseRecorder` để kiểm thử các phản hồi HTTP mà không cần khởi động máy chủ thực.
-   Chúng ta đã thấy cách sử dụng interfaces để tách biệt máy chủ khỏi cơ chế lưu trữ dữ liệu (`PlayerStore`).
-   Chúng ta đã sử dụng `http.NewServeMux` để định tuyến các yêu cầu đến các handler khác nhau.
-   Chúng ta đã học cách trả về dữ liệu JSON bằng `json.NewEncoder`.
-   Chúng ta đã triển khai một phiên bản lưu trữ đơn giản bằng bộ nhớ (`InMemoryPlayerStore`).

Việc duy trì các bước nhỏ trong TDD giúp chúng ta kiểm soát được độ phức tạp và luôn có mã nguồn sẵn sàng để chạy.


