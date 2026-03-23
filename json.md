# JSON, routing & embedding

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/json)**

[Trong chương trước](http-server.md), chúng ta đã tạo một máy chủ web để lưu trữ số trận thắng của các người chơi.

Product owner có yêu cầu mới: tạo endpoint `/league` trả về danh sách tất cả người chơi đã lưu, dưới dạng JSON.

## Đây là code chúng ta đã có cho đến giờ

```go
// server.go
package main

import (
	"fmt"
	"net/http"
	"strings"
)

type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
}

type PlayerServer struct {
	store PlayerStore
}

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

```go
// in_memory_player_store.go
package main

func NewInMemoryPlayerStore() *InMemoryPlayerStore {
	return &InMemoryPlayerStore{map[string]int{}}
}

type InMemoryPlayerStore struct {
	store map[string]int
}

func (i *InMemoryPlayerStore) RecordWin(name string) {
	i.store[name]++
}

func (i *InMemoryPlayerStore) GetPlayerScore(name string) int {
	return i.store[name]
}
```

```go
// main.go
package main

import (
	"log"
	"net/http"
)

func main() {
	server := &PlayerServer{NewInMemoryPlayerStore()}
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Bạn có thể tìm thấy các test tương ứng trong liên kết ở đầu chương.

Chúng ta sẽ bắt đầu bằng việc tạo endpoint cho bảng xếp hạng.

## Viết test trước tiên

Chúng ta sẽ mở rộng bộ test hiện có vì chúng ta đã có một số hàm test hữu ích và một `PlayerStore` giả để sử dụng.

```go
// server_test.go
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

Trước khi lo lắng về điểm số thực tế và JSON, chúng ta sẽ cố gắng duy trì các thay đổi nhỏ với kế hoạch lặp lại dần dần hướng tới mục tiêu của mình. Bắt đầu đơn giản nhất là kiểm tra xem chúng ta có thể truy cập `/league` và nhận lại mã `OK`.

## Thử chạy test

```
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
        server_test.go:101: status code is wrong: got 404, want 200
FAIL
FAIL	playerstore	0.221s
FAIL
```

`PlayerServer` của chúng ta trả về `404 Not Found`, như thể chúng ta đang cố gắng lấy số trận thắng cho một người chơi không xác định. Nhìn vào cách `server.go` triển khai `ServeHTTP`, chúng ta nhận ra rằng nó luôn giả định được gọi với một URL trỏ đến một người chơi cụ thể:

```go
player := strings.TrimPrefix(r.URL.Path, "/players/")
```

Trong chương trước, chúng ta đã đề cập rằng đây là một cách routing khá ngây thơ. Test của chúng ta thông báo chính xác rằng chúng ta cần một khái niệm về cách xử lý các đường dẫn yêu cầu khác nhau.

## Viết đủ code để test vượt qua

Go có một routing mechanism tích hợp sẵn gọi là [`ServeMux`](https://golang.org/pkg/net/http/#ServeMux) cho phép bạn gắn các `http.Handler` vào các đường dẫn yêu cầu cụ thể.

Hãy tạm chấp nhận một số giải pháp "tội lỗi" và làm cho các test vượt qua theo cách nhanh nhất có thể, biết rằng chúng ta có thể refactor nó một cách an toàn khi biết các test đã vượt qua.

```go
// server.go
func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {

	router := http.NewServeMux()

	router.Handle("/league", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	}))

	router.Handle("/players/", http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		player := strings.TrimPrefix(r.URL.Path, "/players/")

		switch r.Method {
		case http.MethodPost:
			p.processWin(w, player)
		case http.MethodGet:
			p.showScore(w, player)
		}
	}))

	router.ServeHTTP(w, r)
}
```

- Khi yêu cầu bắt đầu, chúng ta tạo một router và sau đó chỉ định cho đường dẫn `x` sử dụng handler `y`.
- Vì vậy, đối với endpoint mới của chúng ta, chúng ta sử dụng `http.HandlerFunc` và một *anonymous function* để thực hiện `w.WriteHeader(http.StatusOK)` khi `/league` được yêu cầu để giúp test mới của chúng ta vượt qua.
- Đối với route `/players/`, chúng ta chỉ việc cắt và dán code cũ vào một `http.HandlerFunc` khác.
- Cuối cùng, chúng ta xử lý yêu cầu đến bằng cách gọi `ServeHTTP` của router mới (hãy chú ý rằng `ServeMux` cũng *là* một `http.Handler`).

Các test bây giờ sẽ vượt qua.

## Refactor

`ServeHTTP` trông khá đồ sộ, chúng ta có thể tách biệt mọi thứ bằng cách refactor các handler của mình thành các phương thức riêng biệt.

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

Thật kỳ lạ (và kém hiệu quả) khi phải thiết lập một router mỗi khi một yêu cầu đến và sau đó gọi nó. Những gì chúng ta thực sự muốn là có một loại hàm `NewPlayerServer` nhận các phụ thuộc của mình và thực hiện thiết lập router một lần duy nhất. Sau đó mỗi yêu cầu chỉ cần sử dụng phiên bản router đó.

```go
// server.go
type PlayerServer struct {
	store  PlayerStore
	router *http.ServeMux
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := &PlayerServer{
		store,
		http.NewServeMux(),
	}

	p.router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	p.router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	return p
}

func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	p.router.ServeHTTP(w, r)
}
```

- `PlayerServer` bây giờ cần lưu trữ một router.
- Chúng ta đã di chuyển việc tạo router ra khỏi `ServeHTTP` và đưa vào `NewPlayerServer` để việc này chỉ phải thực hiện một lần, thay vì cho mỗi yêu cầu.
- Bạn sẽ cần cập nhật tất cả các code test và code thực tế ở những nơi chúng ta đã sử dụng `PlayerServer{&store}` thành `NewPlayerServer(&store)`.

### Một bước refactor cuối cùng

Hãy thử thay đổi code thành như sau:

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}
```

Sau đó thay thế `server := &PlayerServer{&store}` thành `server := NewPlayerServer(&store)` trong `server_test.go`, `server_integration_test.go`, và `main.go`.

Cuối cùng hãy đảm bảo bạn **xóa** `func (p *PlayerServer) ServeHTTP(w http.ResponseWriter, r *http.Request)` vì nó không còn cần thiết nữa!

## Embedding

Chúng ta đã thay đổi thuộc tính thứ hai của `PlayerServer`, loại bỏ thuộc tính có tên `router http.ServeMux` và thay thế nó bằng `http.Handler`; điều này được gọi là *embedding*.

> Go không cung cấp khái niệm subclass điển hình dựa trên kiểu, nhưng nó có khả năng "mượn" các phần của một triển khai bằng cách embed các kiểu vào trong một struct hoặc interface.

[Go hiệu quả - Nhúng](https://golang.org/doc/effective_go.html#embedding)

Điều này có nghĩa là `PlayerServer` của chúng ta bây giờ có tất cả các phương thức mà `http.Handler` có, trong trường hợp này chỉ là `ServeHTTP`.

Để "lấp đầy" `http.Handler`, chúng ta gán nó cho `router` mà chúng ta tạo trong `NewPlayerServer`. Chúng ta có thể làm điều này vì `http.ServeMux` có phương thức `ServeHTTP`.

Điều này cho phép chúng ta xóa phương thức `ServeHTTP` của chính mình, vì chúng ta đã cung cấp một phương thức thông qua kiểu được embed.

Nhúng là một tính năng ngôn ngữ rất thú vị. Bạn có thể sử dụng nó với các interface để hợp thành các interface mới.

```go
type Animal interface {
	Eater
	Sleeper
}
```

Và bạn cũng có thể sử dụng nó với các concrete type, không chỉ với các interface. Đúng như mong đợi, nếu bạn embed một concrete type, bạn sẽ có quyền truy cập vào tất cả các phương thức và trường public của nó.

### Có nhược điểm nào không?

Bạn phải cẩn thận với việc embed các kiểu vì bạn sẽ để lộ tất cả các phương thức và trường công khai của kiểu mà bạn embed. Trong trường hợp của chúng ta, điều đó ổn vì chúng ta chỉ embed *interface* mà chúng ta muốn để lộ (`http.Handler`).

Nếu chúng ta lười biếng và embed `http.ServeMux` thay thế (kiểu cụ thể), nó vẫn hoạt động *nhưng* người dùng `PlayerServer` sẽ có thể thêm các route mới vào máy chủ của chúng ta vì `Handle(path, handler)` sẽ ở chế độ công khai.

**Khi embed các kiểu, hãy thực sự nghĩ về tác động của nó đối với API công khai của bạn.**

Một sai lầm *rất* phổ biến là lạm dụng việc embed và cuối cùng làm hỏng các API của bạn cũng như làm lộ các chi tiết nội bộ bên trong kiểu của bạn.

Bây giờ chúng ta đã cấu trúc lại ứng dụng của mình, chúng ta có thể dễ dàng thêm các route mới và bắt đầu với endpoint `/league`. Bây giờ chúng ta cần làm cho nó trả về một số thông tin hữu ích.

Chúng ta nên trả về một số dữ liệu JSON trông giống như thế này:

```json
[
   {
      "Name":"Bill",
      "Wins":10
   },
   {
      "Name":"Alice",
      "Wins":15
   }
]
```

## Viết test trước tiên

Chúng ta sẽ bắt đầu bằng việc cố gắng phân tích phản hồi thành thứ gì đó có ý nghĩa.

```go
// server_test.go
func TestLeague(t *testing.T) {
	store := StubPlayerStore{}
	server := NewPlayerServer(&store)

	t.Run("it returns 200 on /league", func(t *testing.T) {
		request, _ := http.NewRequest(http.MethodGet, "/league", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		var got []Player

		err := json.NewDecoder(response.Body).Decode(&got)

		if err != nil {
			t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", response.Body, err)
		}

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

### Tại sao không kiểm tra chuỗi JSON?

Bạn có thể tranh luận rằng bước ban đầu đơn giản hơn sẽ chỉ là khẳng định rằng thân phản hồi có một chuỗi JSON cụ thể.

Theo kinh nghiệm của tôi, các test khẳng định với các chuỗi JSON có các vấn đề sau:

- *Brittleness*. Nếu bạn thay đổi mô hình dữ liệu, các test của bạn sẽ thất bại.
- *Hard to debug*. Có thể khó hiểu vấn đề thực sự là gì khi so sánh hai chuỗi JSON.
- *Poor intention*. Mặc dù đầu ra phải là JSON, điều thực sự quan trọng là chính xác dữ liệu đó là gì, thay vì cách nó được encode.
- *Test lại thư viện tiêu chuẩn*. Không cần thiết phải test cách thư viện tiêu chuẩn xuất ra JSON, nó đã được test rồi. Đừng test code của người khác.

Thay vào đó, chúng ta nên cân nhắc việc phân tích JSON thành các cấu trúc dữ liệu phù hợp để chúng ta test.

### Data modelling

Dựa trên mô hình dữ liệu JSON, có vẻ như chúng ta cần một mảng gồm các `Player` với một số trường, vì vậy chúng ta đã tạo một kiểu mới để nắm bắt điều này.

```go
// server.go
type Player struct {
	Name string
	Wins int
}
```

### JSON decoding

```go
// server_test.go
var got []Player
err := json.NewDecoder(response.Body).Decode(&got)
```

Để phân tích JSON thành mô hình dữ liệu của mình, chúng ta tạo một `Decoder` từ gói `encoding/json` và sau đó gọi phương thức `Decode` của nó. Để tạo một `Decoder`, nó cần một `io.Reader` để đọc từ đó, trong trường hợp của chúng ta là trường `Body` của response spy.

`Decode` nhận địa chỉ của thứ mà chúng ta đang cố gắng giải mã vào, đó là lý do tại sao chúng ta khai báo một slice trống của `Player` ở dòng trước đó.

Việc phân tích JSON có thể thất bại nên `Decode` có thể trả về một `error`. Không có lý do gì để tiếp tục test nếu việc đó thất bại, vì vậy chúng ta kiểm tra lỗi và dừng test bằng `t.Fatalf` nếu nó xảy ra. Lưu ý rằng chúng ta in thân phản hồi cùng với lỗi vì điều quan trọng là người chạy test phải thấy chuỗi nào không thể phân tích được.

## Thử chạy test

```
=== RUN   TestLeague/it_returns_200_on_/league
    --- FAIL: TestLeague/it_returns_200_on_/league (0.00s)
        server_test.go:107: Unable to parse response from server '' into slice of Player, 'unexpected end of JSON input'
```

Điểm cuối của chúng ta hiện không trả về thân phản hồi nên nó không thể được phân tích thành JSON.

## Viết đủ code để test vượt qua

```go
// server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	leagueTable := []Player{
		{"Chris", 20},
	}

	json.NewEncoder(w).Encode(leagueTable)

	w.WriteHeader(http.StatusOK)
}
```

Các test bây giờ đã vượt qua.

### Encoding và Decoding

Hãy chú ý sự đối xứng đáng yêu trong thư viện tiêu chuẩn.

- Để tạo một `Encoder`, bạn cần một `io.Writer`, chính là những gì `http.ResponseWriter` triển khai.
- Để tạo một `Decoder`, bạn cần một `io.Reader`, chính là những gì trường `Body` của response spy triển khai.

Xuyên suốt cuốn sách này, chúng ta đã sử dụng `io.Writer` và đây là một minh chứng khác cho sự phổ biến của nó trong thư viện tiêu chuẩn và cách nhiều thư viện dễ dàng làm việc với nó.

## Refactor

Sẽ rất tốt nếu giới thiệu một separation of concern giữa handler của chúng ta và việc lấy `leagueTable`, vì chúng ta biết rằng chúng ta sẽ không hardcode việc đó lâu nữa đâu.

```go
// server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.getLeagueTable())
	w.WriteHeader(http.StatusOK)
}

func (p *PlayerServer) getLeagueTable() []Player {
	return []Player{
		{"Chris", 20},
	}
}
```

Tiếp theo, chúng ta muốn mở rộng test của mình để có thể kiểm soát chính xác dữ liệu nào chúng ta muốn nhận lại.

## Viết test trước tiên

Chúng ta có thể cập nhật test để khẳng định rằng bảng xếp hạng chứa một số người chơi mà chúng ta sẽ tạo stub trong store của mình.

Cập nhật `StubPlayerStore` để cho phép nó lưu trữ một bảng xếp hạng, vốn chỉ là một slice của các `Player`. Chúng ta sẽ lưu trữ dữ liệu mong đợi của mình ở đó.

```go
// server_test.go
type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
	league   []Player
}
```

Tiếp theo, cập nhật test hiện tại của chúng ta bằng cách đưa một số người chơi vào thuộc tính league của stub và khẳng định họ được trả về từ máy chủ của chúng ta.

```go
// server_test.go
func TestLeague(t *testing.T) {

	t.Run("it returns the league table as JSON", func(t *testing.T) {
		wantedLeague := []Player{
			{"Cleo", 32},
			{"Chris", 20},
			{"Tiest", 14},
		}

		store := StubPlayerStore{nil, nil, wantedLeague}
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

## Thử chạy test

```
./server_test.go:33:3: too few values in struct initializer
./server_test.go:70:3: too few values in struct initializer
```

## Viết lượng code tối thiểu để test chạy và kiểm tra kết quả lỗi

Bạn sẽ cần cập nhật các test khác vì chúng ta có một trường mới trong `StubPlayerStore`; hãy đặt nó thành `nil` cho các test khác.

Thử chạy lại các test và bạn sẽ nhận được:

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: got [{Chris 20}] want [{Cleo 32} {Chris 20} {Tiest 14}]
```

## Viết đủ code để test vượt qua

Chúng ta biết dữ liệu nằm trong `StubPlayerStore` và chúng ta đã trừu tượng hóa nó vào một interface `PlayerStore`. Chúng ta cần cập nhật cái này để bất kỳ ai truyền cho chúng ta một `PlayerStore` đều có thể cung cấp cho chúng ta dữ liệu cho các bảng xếp hạng.

```go
// server.go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() []Player
}
```

Bây giờ chúng ta có thể cập nhật code handler của mình để gọi phương thức đó thay vì trả về một danh sách được hardcode. Xóa phương thức `getLeagueTable()` của chúng ta và sau đó cập nhật `leagueHandler` để gọi `GetLeague()`.

```go
// server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	json.NewEncoder(w).Encode(p.store.GetLeague())
	w.WriteHeader(http.StatusOK)
}
```

Thử và chạy các test.

```
# github.com/quii/learn-go-with-tests/json-and-io/v4
./main.go:9:50: cannot use NewInMemoryPlayerStore() (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_integration_test.go:11:27: cannot use store (type *InMemoryPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *InMemoryPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:36:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:74:28: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
./server_test.go:106:29: cannot use &store (type *StubPlayerStore) as type PlayerStore in argument to NewPlayerServer:
    *StubPlayerStore does not implement PlayerStore (missing GetLeague method)
```

Trình biên dịch đang phàn nàn vì `InMemoryPlayerStore` và `StubPlayerStore` không có phương thức mới mà chúng ta đã thêm vào interface của mình.

Đối với `StubPlayerStore`, nó khá dễ dàng, chỉ cần trả về trường `league` mà chúng ta đã thêm trước đó.

```go
// server_test.go
func (s *StubPlayerStore) GetLeague() []Player {
	return s.league
}
```

Dưới đây là lời nhắc về cách `InMemoryStore` được triển khai:

```go
// in_memory_player_store.go
type InMemoryPlayerStore struct {
	store map[string]int
}
```

Mặc dù sẽ khá đơn giản để thực hiện `GetLeague` một cách "đúng đắn" bằng cách lặp qua map, hãy nhớ rằng chúng ta chỉ đang cố gắng *viết lượng code tối thiểu để làm cho các test vượt qua*.

Vì vậy, hãy tạm thời làm cho trình biên dịch hài lòng và chấp nhận cảm giác khó chịu về một implementation chưa hoàn chỉnh trong `InMemoryStore` của chúng ta.

```go
// in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	return nil
}
```

Điều mà điều này thực sự đang nói với chúng ta là *sau này* chúng ta sẽ muốn test cái này nhưng hãy tạm gác nó lại.

Thử chạy các test, trình biên dịch sẽ thông qua và các test cũng sẽ vượt qua!

## Refactor

Code test không truyền đạt tốt lắm ý định của chúng ta và có rất nhiều boilerplate mà chúng ta có thể refactor.

```go
// server_test.go
t.Run("it returns the league table as JSON", func(t *testing.T) {
	wantedLeague := []Player{
		{"Cleo", 32},
		{"Chris", 20},
		{"Tiest", 14},
	}

	store := StubPlayerStore{nil, nil, wantedLeague}
	server := NewPlayerServer(&store)

	request := newLeagueRequest()
	response := httptest.NewRecorder()

	server.ServeHTTP(response, request)

	got := getLeagueFromResponse(t, response.Body)
	assertStatus(t, response.Code, http.StatusOK)
	assertLeague(t, got, wantedLeague)
})
```

Dưới đây là các helper mới:

```go
// server_test.go
func getLeagueFromResponse(t testing.TB, body io.Reader) (league []Player) {
	t.Helper()
	err := json.NewDecoder(body).Decode(&league)

	if err != nil {
		t.Fatalf("Unable to parse response from server %q into slice of Player, '%v'", body, err)
	}

	return
}

func assertLeague(t testing.TB, got, want []Player) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %v want %v", got, want)
	}
}

func newLeagueRequest() *http.Request {
	req, _ := http.NewRequest(http.MethodGet, "/league", nil)
	return req
}
```

Một điều cuối cùng chúng ta cần làm để máy chủ hoạt động là đảm bảo chúng ta trả về một header `content-type` trong phản hồi để các máy móc có thể nhận biết chúng ta đang trả về `JSON`.

## Viết test trước tiên

Thêm khẳng định này vào test hiện có của bạn:

```go
// server_test.go
if response.Result().Header.Get("content-type") != "application/json" {
	t.Errorf("response did not have content-type of application/json, got %v", response.Result().Header)
}
```

## Thử chạy test

```
=== RUN   TestLeague/it_returns_the_league_table_as_JSON
    --- FAIL: TestLeague/it_returns_the_league_table_as_JSON (0.00s)
        server_test.go:124: response did not have content-type of application/json, got map[Content-Type:[text/plain; charset=utf-8]]
```

## Viết đủ code để test vượt qua

Cập nhật `leagueHandler`:

```go
// server.go
func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", "application/json")
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

Test bây giờ sẽ vượt qua.

## Refactor

Tạo một hằng số cho "application/json" và sử dụng nó trong `leagueHandler`.

```go
// server.go
const jsonContentType = "application/json"

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", jsonContentType)
	json.NewEncoder(w).Encode(p.store.GetLeague())
}
```

Sau đó thêm một helper cho `assertContentType`.

```go
// server_test.go
func assertContentType(t testing.TB, response *httptest.ResponseRecorder, want string) {
	t.Helper()
	if response.Result().Header.Get("content-type") != want {
		t.Errorf("response did not have content-type of %s, got %v", want, response.Result().Header)
	}
}
```

Sử dụng nó trong test:

```go
// server_test.go
assertContentType(t, response, jsonContentType)
```

Bây giờ chúng ta đã tạm thời xử lý xong `PlayerServer`, chúng ta có thể chuyển sự chú ý sang `InMemoryPlayerStore` vì hiện tại nếu chúng ta thử demo cái này cho product owner thì `/league` sẽ không hoạt động.

Cách nhanh nhất để chúng ta có sự tin cậy là thêm một integration test, chúng ta có thể truy cập endpoint mới và kiểm tra xem chúng ta có nhận được phản hồi chính xác từ `/league` hay không.

## Viết test trước tiên

Chúng ta có thể sử dụng `t.Run` để phân nhỏ test này ra một chút và chúng ta có thể sử dụng lại các helper từ các test server của mình - một lần nữa cho thấy tầm quan trọng của việc refactor các test.

```go
// server_integration_test.go
func TestRecordingWinsAndRetrievingThem(t *testing.T) {
	store := NewInMemoryPlayerStore()
	server := NewPlayerServer(store)
	player := "Pepper"

	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))
	server.ServeHTTP(httptest.NewRecorder(), newPostWinRequest(player))

	t.Run("get score", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newGetScoreRequest(player))
		assertStatus(t, response.Code, http.StatusOK)

		assertResponseBody(t, response.Body.String(), "3")
	})

	t.Run("get league", func(t *testing.T) {
		response := httptest.NewRecorder()
		server.ServeHTTP(response, newLeagueRequest())
		assertStatus(t, response.Code, http.StatusOK)

		got := getLeagueFromResponse(t, response.Body)
		want := []Player{
			{"Pepper", 3},
		}
		assertLeague(t, got, want)
	})
}
```

## Thử chạy test

```
=== RUN   TestRecordingWinsAndRetrievingThem/get_league
    --- FAIL: TestRecordingWinsAndRetrievingThem/get_league (0.00s)
        server_integration_test.go:35: got [] want [{Pepper 3}]
```

## Viết đủ code để test vượt qua

`InMemoryPlayerStore` đang trả về `nil` khi bạn gọi `GetLeague()` nên chúng ta cần phải sửa chữa nó.

```go
// in_memory_player_store.go
func (i *InMemoryPlayerStore) GetLeague() []Player {
	var league []Player
	for name, wins := range i.store {
		league = append(league, Player{name, wins})
	}
	return league
}
```

Tất cả những gì chúng ta cần làm là lặp qua map và chuyển đổi từng key/value thành một `Player`.

Test bây giờ sẽ vượt qua.

## Tổng kết

Chúng ta đã tiếp tục lặp lại chương trình của mình một cách an toàn bằng TDD, làm cho nó hỗ trợ các endpoint mới theo cách dễ bảo trì với một router và giờ đây nó có thể trả về JSON cho những người dùng của mình. Trong chương tiếp theo, chúng ta sẽ đề cập đến việc persist dữ liệu và sắp xếp bảng xếp hạng của mình.

Những gì chúng ta đã đề cập:

- **Routing**. Thư viện tiêu chuẩn cung cấp cho bạn một kiểu dễ sử dụng để routing. Nó hoàn toàn đi theo interface `http.Handler` trong đó bạn gán các route cho các `Handler` và chính router cũng là một `Handler`. Tuy nhiên, nó không có một số tính năng mà bạn có thể mong đợi như các path variable (ví dụ `/users/{id}`). Bạn có thể dễ dàng tự phân tích thông tin này nhưng bạn có thể muốn cân nhắc việc xem qua các thư viện routing khác nếu việc đó trở thành gánh nặng. Hầu hết các thư viện phổ biến đều tuân theo triết lý của thư viện tiêu chuẩn là cũng thực hiện `http.Handler`.
- **Type embedding**. Chúng ta đã chạm một chút đến kỹ thuật này nhưng bạn có thể [tìm hiểu thêm về nó từ Go hiệu quả](https://golang.org/doc/effective_go.html#embedding). Nếu có một điều bạn nên rút ra từ việc này thì đó là nó có thể cực kỳ hữu ích nhưng *luôn phải suy nghĩ về API công khai của bạn, chỉ để lộ những gì phù hợp*.
- **JSON encoding/decoding**. Thư viện tiêu chuẩn làm cho việc serialize và deserialize dữ liệu của bạn trở nên rất dễ dàng. Nó cũng cho phép cấu hình và bạn có thể tùy chỉnh cách các chuyển đổi dữ liệu này hoạt động nếu cần thiết.

