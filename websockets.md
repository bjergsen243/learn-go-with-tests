# WebSockets

**[Tất cả mã nguồn của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/websockets)**

Trong chương này chúng ta sẽ học cách sử dụng WebSockets để cải thiện ứng dụng của mình.

## Tóm tắt nhanh dự án

Chúng ta đang có hai ứng dụng trong codebase poker của mình:

- *Ứng dụng dòng lệnh*. Yêu cầu người dùng nhập số lượng người chơi trong một trò chơi. Từ đó thông báo cho người chơi về giá trị "blind bet" (tiền cược mù), giá trị này tăng lên theo thời gian. Bất cứ lúc nào người dùng cũng có thể nhập `"{Tên_người_chơi} wins"` để kết thúc trò chơi và ghi lại người chiến thắng vào một kho lưu trữ (store).
- *Ứng dụng web*. Cho phép người dùng ghi lại những người thắng trò chơi và hiển thị bảng xếp hạng. Ứng dụng này dùng chung kho lưu trữ với ứng dụng dòng lệnh.

## Các bước tiếp theo

Chủ sở hữu sản phẩm rất hài lòng với ứng dụng dòng lệnh nhưng muốn chúng ta có thể đưa chức năng đó lên trình duyệt. Cô ấy hình dung một trang web có một hộp văn bản cho phép người dùng nhập lượng người chơi và khi họ gửi biểu mẫu (form), trang sẽ hiển thị giá trị blind và tự động cập nhật nó khi thích hợp. Giống như ứng dụng dòng lệnh, người dùng có thể khai báo người chiến thắng và nó sẽ được lưu vào cơ sở dữ liệu.

Nói một cách bề ngoài thì có vẻ khá đơn giản nhưng như mọi khi, chúng ta phải nhấn mạnh việc thực hiện một phương pháp tiếp cận _lặp lại_ (iterative) để viết phần mềm.

Đầu tiên, chúng ta sẽ cần phục vụ (serve) HTML. Cho đến nay, tất cả các endpoint HTTP của chúng ta đều trả về dữ liệu văn bản thuần túy hoặc JSON. Chúng ta _có thể_ sử dụng các kỹ thuật tương tự mà chúng ta đã biết (vì chúng cuối cùng đều là chuỗi) nhưng chúng ta cũng có thể sử dụng gói (package) [html/template](https://golang.org/pkg/html/template/) cho một giải pháp sạch sẽ hơn.

Chúng ta cũng cần có khả năng gửi tin nhắn một cách bất đồng bộ đến người dùng thông báo `The blind is now *y*` (Mức cược blind bây giờ là *y*) mà không cần phải tải lại trình duyệt. Chúng ta có thể sử dụng [WebSockets](https://en.wikipedia.org/wiki/WebSocket) để tạo điều kiện cho việc này.

> WebSocket là một giao thức truyền thông máy tính, cung cấp các kênh truyền thông song công toàn phần (full-duplex) qua một kết nối TCP duy nhất

Bởi vì chúng ta đang thực hiện một số kỹ thuật nên điều càng quan trọng hơn là chúng ta sẽ làm lượng công việc hữu ích nhỏ nhất có thể trước tiên và sau đó lặp lại một cách tăng dần (iterate).

Vì lý do đó, điều đầu tiên chúng ta sẽ làm là tạo ra một trang web có một biểu mẫu để người dùng ghi lại người chiến thắng. Thay vì sử dụng một biểu mẫu kiểu cũ, chúng ta sẽ sử dụng WebSockets để gửi dữ liệu đó đến máy chủ của chúng ta để nó ghi lại.

Sau đó, chúng ta sẽ xem xét giải quyết phần thông báo blind, đến lúc đó chúng ta sẽ có một chút mã nguồn cơ sở hạ tầng đã được thiết lập.

### Còn các bản kiểm thử cho JavaScript thì sao?

Sẽ có một số mã JavaScript được viết để làm việc này nhưng tôi sẽ không đi sâu vào việc viết bản kiểm thử cho nó.

Dĩ nhiên là có thể làm được nhưng vì sự ngắn gọn, tôi sẽ không bao gồm bất kỳ lời giải thích nào cho nó.

Xin lỗi các bạn. Hãy vận động O'Reilly trả tiền cho tôi để tôi viết cuốn "Learn JavaScript with tests".

## Viết bản kiểm thử trước tiên

Điều đầu tiên chúng ta cần làm là phục vụ một số HTML cho người dùng khi họ truy cập `/game`.

Đây là một lời nhắc về mã nguồn thích hợp trong máy chủ web của chúng ta:

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

const jsonContentType = "application/json"

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

Điều _dễ nhất_ chúng ta có thể làm hiện tại là kiểm tra xem khi chúng ta `GET /game`, chúng ta có nhận được phản hồi `200` hay không.

```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request, _ := http.NewRequest(http.MethodGet, "/game", nil)
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response.Code, http.StatusOK)
	})
}
```

## Thử chạy bản kiểm thử
```
--- FAIL: TestGame (0.00s)
=== RUN   TestGame/GET_/game_returns_200
    --- FAIL: TestGame/GET_/game_returns_200 (0.00s)
    	server_test.go:109: did not get correct status, got 404, want 200
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Máy chủ của chúng ta đã được thiết lập bộ định tuyến (router) nên nó tương đối dễ sửa.

Thêm vào router của chúng ta:

```go
router.Handle("/game", http.HandlerFunc(p.game))
```

Và sau đó viết phương thức `game`:

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
}
```

## Tái cấu trúc

Mã nguồn của máy chủ trông đã ổn do chúng ta chỉ việc đưa thêm mã nguồn vào khối mã sẵn có và được cấu trúc tốt một cách rất dễ dàng.

Chúng ta có thể dọn dẹp bản kiểm thử một chút bằng cách thêm một hàm hỗ trợ kiểm thử `newGameRequest` để thực hiện yêu cầu tới `/game`. Thử tự viết nó nhé.

```go
func TestGame(t *testing.T) {
	t.Run("GET /game returns 200", func(t *testing.T) {
		server := NewPlayerServer(&StubPlayerStore{})

		request := newGameRequest()
		response := httptest.NewRecorder()

		server.ServeHTTP(response, request)

		assertStatus(t, response, http.StatusOK)
	})
}
```

Bạn cũng sẽ nhận thấy tôi đã thay đổi `assertStatus` để nó chấp nhận `response` thay vì `response.Code` vì tôi cảm thấy nó dễ đọc hơn.

Bây giờ chúng ta cần làm cho endpoint này trả về một ít mã HTML, đây là nó:

```html

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Let's play poker</title>
</head>
<body>
<section id="game">
    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>
</section>
</body>
<script type="application/javascript">

    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    if (window['WebSocket']) {
        const conn = new WebSocket('ws://' + document.location.host + '/ws')

        submitWinnerButton.onclick = event => {
            conn.send(winnerInput.value)
        }
    }
</script>
</html>
```

Chúng ta có một trang web rất đơn giản:

 - Một vùng nhập văn bản để người dùng nhập tên người chiến thắng vào.
 - Một nút bấm để họ báo cáo người chiến thắng đó.
 - Trong JavaScript, mở một kết nối WebSocket đến máy chủ của chúng ta và xử lý sự kiện khi người dùng nhấn nút gửi, dữ liệu sẽ được gửi qua kết nối WebSocket đó.

`WebSocket` được tích hợp sẵn trong hầu hết các trình duyệt hiện đại nên chúng ta không cần lo lắng về việc mang thêm bất kỳ thư viện nào. Trang web sẽ không hoạt động trên các trình duyệt cũ hơn, nhưng chúng ta tạm ổn với điều đó cho kịch bản này.

### Làm thế nào để chúng ta kiểm thử xem mình trả về đúng cấu trúc (markup)?

Có một vài cách. Như đã được nhấn mạnh trong suốt cuốn sách, điều quan trọng là các bản kiểm thử mà bạn viết phải có đủ giá trị để biện minh cho chi phí công sức bỏ ra.

1. Viết một bản kiểm thử dựa trên trình duyệt, sử dụng công cụ như Selenium. Các bản kiểm thử này "chân thực" nhất trong tất cả các cách tiếp cận vì chúng sẽ thực sự mở lên một trình duyệt web và mô phỏng một người dùng tương tác trên nó. Các bản kiểm thử này có thể mang lại cho bạn rất nhiều tự tin rằng hệ thống của bạn hoạt động, thế nhưng nó khó viết hơn các unit test và chạy chậm hơn nhiều. Đối với với mục đích của sản phẩm của chúng ta thì đó lại là overkill (làm quá mức cần thiết).
2. So khớp chuỗi chính xác (exact string match). Điều này _có thể_ ổn nhưng loại bản kiểm thử này thường rất dễ bị hỏng gãy (brittle). Ngay khi ai đó thay đổi điều gì đó trong cấu trúc markup, một test sẽ chạy thất bại, mặc dù trên khía cạnh thực tế thì không có trang nào bị lỗi cả.
3. Kiểm tra xem chúng ta có gọi đúng template không. Chúng ta sẽ sử dụng thư viện template từ thư viện chuẩn (sẽ được thảo luận ở phần sau) để phục vụ (serve) file HTML, và chúng ta có thể truyền (inject) cái _thứ_ tạo ra HTML vào và "do thám" (spy) cuộc gọi của nó để kiểm tra. Thay đổi này thì sẽ ảnh hưởng đôi chút đến thiết kế mã nguồn của trò chơi nhưng nó không thực sự kiểm thử được nhiều; ngoại trừ việc chúng ta có đang gọi đúng tệp của template hay chưa. Với việc hiện tại chúng ta chỉ có duy nhất một template trong project, khả năng xảy ra lỗi ở đây có vẻ rất thấp.

Vậy nên, lần đầu tiên trong tựa sách "Learn Go with Tests", chúng ta sẽ không viết test nào cả.

Hãy lưu các đoạn mã markup trên vào một tệp tên là `game.html`

Tiếp theo hãy đổi nội dung trong phương thức `game` vừa nãy bằng đoạn bên dưới:

```go
func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	tmpl, err := template.ParseFiles("game.html")

	if err != nil {
		http.Error(w, fmt.Sprintf("problem loading template %s", err.Error()), http.StatusInternalServerError)
		return
	}

	tmpl.Execute(w, nil)
}
```

[`html/template`](https://golang.org/pkg/html/template/) là một gói (package) của Go để tạo HTML. Trong trường hợp của chúng ta, chúng ta gọi hàm `template.ParseFiles` và truyền vào đường dẫn tệp HTML. Nếu không có lỗi nào được trả về, bạn có thể gọi `Execute` trên template đó, hàm này sẽ ghi HTML ra một `io.Writer`. Trong trường hợp này, chúng ta muốn ghi ra trình duyệt của người dùng nên chúng ta truyền vào `http.ResponseWriter`.

Vì chúng ta không viết bản kiểm thử nào, nên việc kiểm tra thủ công trên máy chủ là hoàn toàn hợp lý để đảm bảo mọi thứ hoạt động đúng như mong đợi. Hãy vào thư mục `cmd/webserver` và chạy tệp `main.go`. Sau đó truy cập `http://localhost:5000/game`.

Bạn _có thể_ sẽ gặp lỗi về việc không thể tìm thấy tệp `game.html`. Bạn có thể sửa bằng cách thay đổi đường dẫn thành đường dẫn tuyệt đối, hoặc đơn giản là sao chép tệp `game.html` vào thư mục `cmd/webserver`. Cá nhân tôi đã chọn cách tạo một liên kết tượng trưng (symbolic link) bằng lệnh `ln -s ../../game.html game.html` để tệp luôn được cập nhật với phiên bản mới nhất.

Sau khi hoàn tất, chạy lại chương trình và bạn sẽ thấy giao diện người dùng trên web trông như thế nào.

Bước tiếp theo là kiểm thử rằng khi chúng ta nhận được một tin nhắn dạng chuỗi qua kết nối WebSocket, máy chủ sẽ ghi nhận giá trị đó làm tên người chiến thắng của trò chơi.

## Viết bản kiểm thử trước tiên

Đây là lần đầu tiên chúng ta sẽ sử dụng thư viện bên ngoài để có thể làm việc với WebSockets.

Chạy lệnh `go get github.com/gorilla/websocket`

Lệnh này sẽ tải về mã nguồn của thư viện tuyệt vời [Gorilla WebSocket](https://github.com/gorilla/websocket). Bây giờ chúng ta có thể cập nhật các bản kiểm thử của mình cho yêu cầu mới.

```go
t.Run("when we get a message over a websocket it is a winner of a game", func(t *testing.T) {
	store := &StubPlayerStore{}
	winner := "Ruth"
	server := httptest.NewServer(NewPlayerServer(store))
	defer server.Close()

	wsURL := "ws" + strings.TrimPrefix(server.URL, "http") + "/ws"

	ws, _, err := websocket.DefaultDialer.Dial(wsURL, nil)
	if err != nil {
		t.Fatalf("could not open a ws connection on %s %v", wsURL, err)
	}
	defer ws.Close()

	if err := ws.WriteMessage(websocket.TextMessage, []byte(winner)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}

	AssertPlayerWin(t, store, winner)
})
```

Hãy đảm bảo rằng bạn đã thêm import thư viện `websocket` vào tệp của mình. IDE của tôi tự động làm điều đó cho tôi, mong là của bạn cũng vậy.

Để kiểm thử những gì xảy ra từ phía trình duyệt, chúng ta phải tự mở một kết nối WebSocket và gửi dữ liệu (write) vào đó.

Các bản kiểm thử trước đây của chúng ta đối với máy chủ chỉ gọi các phương thức đơn lẻ, không có các luồng kết nối liên tục. Để kiểm thử WebSockets, chúng ta cần sử dụng `httptest.NewServer`, hàm này nhận một `http.Handler` và khởi động một máy chủ lắng nghe các kết nối đến.

Sử dụng `websocket.DefaultDialer.Dial` để thử kết nối đến máy chủ, sau đó gửi một tin nhắn chứa tên người chiến thắng.

Cuối cùng, chúng ta kiểm tra (assert) trên đối tượng player store để xác nhận rằng người chiến thắng đã được ghi lại.

## Thử chạy bản kiểm thử
```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:124: could not open a ws connection on ws://127.0.0.1:55838/ws websocket: bad handshake
```

Chúng ta chưa cấu hình cho máy chủ chấp nhận kết nối WebSocket tại đường dẫn `/ws`, vì vậy quá trình bắt tay (handshake) thất bại là điều dễ hiểu.

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Thêm một route mới vào bộ định tuyến:

```go
router.Handle("/ws", http.HandlerFunc(p.webSocket))
```

Sau đó thêm hàm xử lý WebSocket:

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	upgrader.Upgrade(w, r, nil)
}
```

Để chấp nhận kết nối WebSocket, chúng ta cần sử dụng `Upgrade` để nâng cấp yêu cầu HTTP thành kết nối WebSocket. Quá trình này thực hiện bắt tay (handshake) với trình duyệt của người dùng. Nếu bạn chạy bản kiểm thử ngay lúc này, lỗi sẽ như sau:

```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:132: got 0 calls to RecordWin want 1
```

Bây giờ hãy cập nhật hàm xử lý WebSocket để đọc tin nhắn và ghi lại người chiến thắng:

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	conn, _ := upgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

(Đúng vậy, chúng ta đang bỏ qua khá nhiều lỗi ở đây!)

`conn.ReadMessage()` sẽ chặn (block) cho đến khi nhận được một tin nhắn qua kết nối. Sau khi nhận được, chúng ta gọi `RecordWin` để ghi lại người chiến thắng. Khi hoàn tất, kết nối sẽ được đóng.

Nếu bạn chạy bản kiểm thử, bạn có thể thấy nó vẫn thất bại.

Vấn đề nằm ở thời gian (timing). Có một độ trễ nhỏ giữa lúc kết nối WebSocket đọc tin nhắn và ghi lại kết quả, với lúc bản kiểm thử thực hiện phép kiểm tra. Bạn có thể xác minh điều này bằng cách thêm một khoảng nghỉ ngắn `time.Sleep` trước phép kiểm tra.

Hãy tạm ghi nhận vấn đề này. Chúng ta sẽ quay lại xem xét tại sao đây là một vấn đề, và tại sao việc chèn các giá trị sleep tùy ý vào mã kiểm thử là một **thực hành không tốt**.

```go
time.Sleep(10 * time.Millisecond)
AssertPlayerWin(t, store, winner)
```

## Tái cấu trúc

Chúng ta đã phải cắt vài góc để bản kiểm thử chạy được và máy chủ hoạt động đúng. Nhưng đây là một nền tảng tốt để chúng ta tiếp tục cải thiện.

Mã nguồn hiện tại hoạt động đúng theo mong đợi nhờ vào bộ công cụ kiểm thử. Nhiệm vụ bây giờ là dọn dẹp mã nguồn, thay thế bằng các đoạn mã sạch hơn, và đảm bảo rằng chúng không gây ra lỗi trong tương lai.

Đầu tiên, hãy xem xét mã nguồn của `PlayerServer`.

Đối tượng `upgrader` được sử dụng riêng cho phương thức WebSocket nên chúng ta có thể khai báo nó ở cấp package (cấp gói). Mã nguồn sẽ như sau:

```go
var wsUpgrader = websocket.Upgrader{
	ReadBufferSize:  1024,
	WriteBufferSize: 1024,
}

func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	conn, _ := wsUpgrader.Upgrade(w, r, nil)
	_, winnerMsg, _ := conn.ReadMessage()
	p.store.RecordWin(string(winnerMsg))
}
```

Lời gọi `template.ParseFiles("game.html")` hiện đang được thực thi mỗi khi có yêu cầu `GET /game`. Điều này không cần thiết vì template không thay đổi giữa các yêu cầu, và việc phân tích cú pháp (parse) lại template mỗi lần là lãng phí tài nguyên. Giải pháp là phân tích template một lần duy nhất trong hàm `NewPlayerServer` và lưu kết quả vào một trường (field) của struct. Khi thực hiện thay đổi này, hàm khởi tạo cũng cần trả về lỗi trong trường hợp không thể đọc hoặc phân tích tệp template.

Mã nguồn mới của `PlayerServer`:

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
	template *template.Template
}

const htmlTemplatePath = "game.html"

func NewPlayerServer(store PlayerStore) (*PlayerServer, error) {
	p := new(PlayerServer)

	tmpl, err := template.ParseFiles(htmlTemplatePath)

	if err != nil {
		return nil, fmt.Errorf("problem opening %s %v", htmlTemplatePath, err)
	}

	p.template = tmpl
	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))
	router.Handle("/game", http.HandlerFunc(p.game))
	router.Handle("/ws", http.HandlerFunc(p.webSocket))

	p.Handler = router

	return p, nil
}

func (p *PlayerServer) game(w http.ResponseWriter, r *http.Request) {
	p.template.Execute(w, nil)
}
```

Vì chúng ta đã thay đổi chữ ký (signature) của `NewPlayerServer`, mã nguồn ở các tệp gọi hàm này sẽ gặp lỗi biên dịch. Hãy thử tự sửa các lỗi đó. Nếu gặp khó khăn, bạn có thể tham khảo mã nguồn gốc.

Việc sửa lỗi trong tệp kiểm thử khá đơn giản. Tôi đã tạo một hàm hỗ trợ (helper) `mustMakePlayerServer` để xử lý gọn việc tạo server và kiểm tra lỗi:

```go
func mustMakePlayerServer(t *testing.T, store PlayerStore) *PlayerServer {
	server, err := NewPlayerServer(store)
	if err != nil {
		t.Fatal("problem creating player server", err)
	}
	return server
}
```

Tương tự, tôi cũng tạo một hàm hỗ trợ `mustDialWS` để đóng gói việc thiết lập kết nối WebSocket và xử lý lỗi:

```go
func mustDialWS(t *testing.T, url string) *websocket.Conn {
	ws, _, err := websocket.DefaultDialer.Dial(url, nil)

	if err != nil {
		t.Fatalf("could not open a ws connection on %s %v", url, err)
	}

	return ws
}
```

Và cuối cùng, một hàm hỗ trợ để gửi tin nhắn qua kết nối WebSocket:

```go
func writeWSMessage(t testing.TB, conn *websocket.Conn, message string) {
	t.Helper()
	if err := conn.WriteMessage(websocket.TextMessage, []byte(message)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}
}
```

Bây giờ các bản kiểm thử đã vượt qua. Hãy thử chạy máy chủ và khai báo một số người chiến thắng tại `/game`. Bạn sẽ thấy chúng được ghi lại trong `/league`. Hãy nhớ rằng mỗi khi chúng ta nhận được người chiến thắng, chúng ta _sẽ đóng kết nối_, vì vậy bạn sẽ cần tải lại trang để mở lại kết nối.

Chúng ta đã tạo ra một biểu mẫu web đơn giản cho phép người dùng ghi lại người chiến thắng của trò chơi. Hãy tiếp tục lặp lại quá trình này để người dùng có thể bắt đầu trò chơi bằng cách cung cấp số lượng người chơi, và máy chủ sẽ đẩy (push) thông báo về giá trị blind đến trình duyệt khi thời gian trôi qua.

Đầu tiên, hãy cập nhật `game.html` để phản ánh các yêu cầu mới phía trình duyệt:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Lets play poker</title>
</head>
<body>
<section id="game">
    <div id="game-start">
        <label for="player-count">Number of players</label>
        <input type="number" id="player-count"/>
        <button id="start-game">Start</button>
    </div>

    <div id="declare-winner">
        <label for="winner">Winner</label>
        <input type="text" id="winner"/>
        <button id="winner-button">Declare winner</button>
    </div>

    <div id="blind-value"/>
</section>

<section id="game-end">
    <h1>Another great game of poker everyone!</h1>
    <p><a href="/league">Go check the league table</a></p>
</section>

</body>
<script type="application/javascript">
    const startGame = document.getElementById('game-start')

    const declareWinner = document.getElementById('declare-winner')
    const submitWinnerButton = document.getElementById('winner-button')
    const winnerInput = document.getElementById('winner')

    const blindContainer = document.getElementById('blind-value')

    const gameContainer = document.getElementById('game')
    const gameEndContainer = document.getElementById('game-end')

    declareWinner.hidden = true
    gameEndContainer.hidden = true

    document.getElementById('start-game').addEventListener('click', event => {
        startGame.hidden = true
        declareWinner.hidden = false

        const numberOfPlayers = document.getElementById('player-count').value

        if (window['WebSocket']) {
            const conn = new WebSocket('ws://' + document.location.host + '/ws')

            submitWinnerButton.onclick = event => {
                conn.send(winnerInput.value)
                gameEndContainer.hidden = false
                gameContainer.hidden = true
            }

            conn.onclose = evt => {
                blindContainer.innerText = 'Connection closed'
            }

            conn.onmessage = evt => {
                blindContainer.innerText = evt.data
            }

            conn.onopen = function () {
                conn.send(numberOfPlayers)
            }
        }
    })
</script>
</html>
```

Các thay đổi chính là thêm một phần để nhập số lượng người chơi và một phần để hiển thị giá trị blind. Chúng ta cũng thêm một chút logic để hiển thị hoặc ẩn các phần của giao diện tùy thuộc vào giai đoạn của trò chơi.

Bất kỳ tin nhắn nào chúng ta nhận được qua `conn.onmessage`, chúng ta đều coi đó là thông báo blind, vì vậy chúng ta gán nội dung cho `blindContainer.innerText`.

Làm thế nào để chúng ta gửi các thông báo blind? Trong chương trước, chúng ta đã giới thiệu khái niệm `Game` để mã CLI có thể gọi một `Game` và mọi thứ khác sẽ được xử lý, bao gồm cả việc lên lịch cho các thông báo blind. Đây hóa ra là một cách phân tách trách nhiệm (separation of concerns) rất tốt.

```go
type Game interface {
	Start(numberOfPlayers int)
	Finish(winner string)
}
```

Khi người dùng được nhắc nhập số lượng người chơi trong CLI, ứng dụng sẽ gọi `Start` để bắt đầu trò chơi, kích hoạt các thông báo blind. Khi người dùng khai báo người chiến thắng, ứng dụng sẽ gọi `Finish`. Đây chính xác là những yêu cầu mà chúng ta có bây giờ, chỉ khác cách nhận đầu vào. Vì vậy, chúng ta nên tái sử dụng (re-use) logic này.

Triển khai thực tế của `Game` là `TexasHoldem`:

```go
type TexasHoldem struct {
	alerter BlindAlerter
	store   PlayerStore
}
```

Bằng cách truyền vào một `BlindAlerter`, `TexasHoldem` có thể lên lịch gửi thông báo blind đến _bất cứ đâu_.

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}
```

Để nhắc lại, đây là triển khai `BlindAlerter` mà chúng ta đang sử dụng trong CLI:

```go
func StdOutAlerter(duration time.Duration, amount int) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(os.Stdout, "Blind is now %d\n", amount)
	})
}
```

Cách này hoạt động tốt trong CLI vì chúng ta _luôn muốn gửi thông báo đến `os.Stdout`_. Tuy nhiên, nó sẽ không phù hợp với máy chủ web. Đối với mỗi yêu cầu, chúng ta nhận được một `http.ResponseWriter` mới và sau đó nâng cấp thành `*websocket.Conn`. Vì vậy, tại thời điểm khởi tạo các phụ thuộc, chúng ta không thể biết trước đích đến của các thông báo.

Vì lý do đó, chúng ta cần thay đổi `BlindAlerter.ScheduleAlertAt` để nó nhận thêm một đích đến (destination), nhằm có thể tái sử dụng trong máy chủ web.

Mở tệp `blind_alerter.go` và thêm tham số `io.Writer`:

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int, to io.Writer)
}

type BlindAlerterFunc func(duration time.Duration, amount int, to io.Writer)

func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int, to io.Writer) {
	a(duration, amount, to)
}
```

Khái niệm `StdoutAlerter` không còn phù hợp với mô hình mới, vì vậy hãy đổi tên nó thành `Alerter`:

```go
func Alerter(duration time.Duration, amount int, to io.Writer) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(to, "Blind is now %d\n", amount)
	})
}
```

Nếu bạn thử biên dịch, mã trong `TexasHoldem` sẽ báo lỗi vì đang gọi `ScheduleAlertAt` mà thiếu tham số `to`. Để mã biên dịch được _ngay bây giờ_, hãy tạm thời gán cứng (hard-code) giá trị `os.Stdout` vào.

Tiếp theo, chạy các bản kiểm thử. Chúng sẽ thất bại vì `SpyBlindAlerter` không còn triển khai đúng giao diện `BlindAlerter` nữa. Hãy cập nhật chữ ký hàm `ScheduleAlertAt` của nó. Sau khi sửa, các bản kiểm thử sẽ vượt qua.

Không có lý do gì để `TexasHoldem` phải biết đích đến của thông báo blind. Hãy cập nhật giao diện `Game` để khi trò chơi bắt đầu, nó nhận thêm thông tin về _nơi_ mà thông báo cần được gửi đến:

```go
type Game interface {
	Start(numberOfPlayers int, alertsDestination io.Writer)
	Finish(winner string)
}
```

Trình biên dịch sẽ chỉ ra các vị trí cần sửa. Các thay đổi không quá phức tạp:

- Cập nhật `TexasHoldem` để triển khai đúng giao diện `Game` mới.
- Trong `CLI`, khi bắt đầu trò chơi, truyền thêm đích xuất `out`: `cli.game.Start(numberOfPlayers, cli.out)`.
- Trong các bản kiểm thử của `TexasHoldem`, sử dụng `game.Start(5, io.Discard)` để bỏ qua đầu ra thông báo.

Nếu mọi thứ biên dịch thành công và các bản kiểm thử vượt qua, chúng ta đã sẵn sàng sử dụng `Game` trong `Server`.

## Viết bản kiểm thử trước tiên

Các yêu cầu của `CLI` và `Server` rất giống nhau. Điểm khác biệt duy nhất là cách thông báo được gửi đến người dùng.

Hãy tham khảo các bản kiểm thử của `CLI` để lấy cảm hứng:

```go
t.Run("start game with 3 players and finish game with 'Chris' as winner", func(t *testing.T) {
	game := &GameSpy{}

	out := &bytes.Buffer{}
	in := userSends("3", "Chris wins")

	poker.NewCLI(in, out, game).PlayPoker()

	assertMessagesSentToUser(t, out, poker.PlayerPrompt)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, "Chris")
})
```

Chúng ta hoàn toàn có thể viết mã tương tự cho `Server` bằng cách sử dụng `GameSpy`. Hãy cập nhật bản kiểm thử WebSocket như sau:

```go
t.Run("start a game with 3 players and declare Ruth the winner", func(t *testing.T) {
	game := &poker.GameSpy{}
	winner := "Ruth"
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(10 * time.Millisecond)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)
})
```

- Chúng ta tạo một `GameSpy` và truyền nó vào `mustMakePlayerServer` (đừng quên cập nhật hàm hỗ trợ này để nhận thêm tham số `Game`).
- Sau đó chúng ta gửi tin nhắn qua kết nối WebSocket cho trò chơi.
- Cuối cùng, chúng ta kiểm tra rằng trò chơi đã được khởi tạo và kết thúc đúng như mong đợi.

## Thử chạy bản kiểm thử

Bạn sẽ gặp các lỗi biên dịch tại `mustMakePlayerServer` trong một số bản kiểm thử khác. Hãy khai báo một biến `dummyGame` và truyền vào các bản kiểm thử đang bị lỗi:

```go
var (
	dummyGame = &GameSpy{}
)
```

Tiếp theo, bạn sẽ gặp lỗi biên dịch vì `NewPlayerServer` chưa nhận tham số `Game`:

```
./server_test.go:21:38: too many arguments in call to "github.com/quii/learn-go-with-tests/WebSockets/v2".NewPlayerServer
	have ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore, "github.com/quii/learn-go-with-tests/WebSockets/v2".Game)
	want ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore)
```

## Viết lượng mã nguồn tối thiểu để chạy bản kiểm thử và kiểm tra kết quả lỗi

Thêm tham số `Game` vào hàm `NewPlayerServer`:

```go
func NewPlayerServer(store PlayerStore, game Game) (*PlayerServer, error)
```

Chạy bản kiểm thử:

```
=== RUN   TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.01s)
    --- FAIL: TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner (0.01s)
    	server_test.go:146: wanted Start called with 3 but got 0
    	server_test.go:147: expected finish called with 'Ruth' but got ''
FAIL
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Chúng ta cần thêm trường `Game` vào struct `PlayerServer` để có thể sử dụng nó khi xử lý các yêu cầu:

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
	template *template.Template
	game     Game
}
```

(Nhân tiện, hãy đổi tên phương thức `game` hiện tại thành `playGame` để tránh trùng tên với trường mới.)

Sau đó cập nhật hàm khởi tạo:

```go
func NewPlayerServer(store PlayerStore, game Game) (*PlayerServer, error) {
	p := new(PlayerServer)

	tmpl, err := template.ParseFiles(htmlTemplatePath)

	if err != nil {
		return nil, fmt.Errorf("problem opening %s %v", htmlTemplatePath, err)
	}

	p.game = game

	// v.v...
}
```

Bây giờ chúng ta sẽ sử dụng `Game` trong hàm `webSocket`:

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	conn, _ := wsUpgrader.Upgrade(w, r, nil)

	_, numberOfPlayersMsg, _ := conn.ReadMessage()
	numberOfPlayers, _ := strconv.Atoi(string(numberOfPlayersMsg))
	p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!

	_, winner, _ := conn.ReadMessage()
	p.game.Finish(string(winner))
}
```

Bây giờ các bản kiểm thử sẽ vượt qua.

Lưu ý rằng chúng ta vẫn chưa gửi thông báo blind đến người dùng. Khi gọi `game.Start`, chúng ta đang truyền `io.Discard`, nghĩa là mọi thông báo đều bị bỏ qua. Chúng ta sẽ xử lý vấn đề này sau.

Trước tiên, hãy đảm bảo mọi thứ hoạt động từ đầu đến cuối. Cập nhật tệp `main.go` để truyền `Game` vào `PlayerServer`:

```go
func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	game := poker.NewTexasHoldem(poker.BlindAlerterFunc(poker.Alerter), store)

	server, err := poker.NewPlayerServer(store, game)

	if err != nil {
		log.Fatalf("problem creating player server %v", err)
	}

	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Tạm thời bạn sẽ chưa thấy thông báo blind trên trình duyệt, nhưng ứng dụng vẫn hoạt động tốt. Thành quả chúng ta đạt được là tích hợp `Game` vào `PlayerServer` cùng với toàn bộ mã nguồn phía backend. Mã nguồn vẫn biên dịch thành công và các bản kiểm thử vẫn vượt qua.

Bây giờ hãy xử lý vấn đề còn lại.

## Tái cấu trúc

Cách chúng ta đang sử dụng WebSocket hiện tại khá cơ bản và việc xử lý lỗi chưa đầy đủ. Hãy tạo một kiểu dữ liệu mới để đóng gói kết nối WebSocket, giúp mã nguồn máy chủ gọn gàng hơn:

```go
type playerServerWS struct {
	*websocket.Conn
}

func newPlayerServerWS(w http.ResponseWriter, r *http.Request) *playerServerWS {
	conn, err := wsUpgrader.Upgrade(w, r, nil)

	if err != nil {
		log.Printf("problem upgrading connection to WebSockets %v\n", err)
	}

	return &playerServerWS{conn}
}

func (w *playerServerWS) WaitForMsg() string {
	_, msg, err := w.ReadMessage()
	if err != nil {
		log.Printf("error reading from websocket %v\n", err)
	}
	return string(msg)
}
```

Giờ đây mã nguồn xử lý WebSocket trên máy chủ trở nên gọn gàng hơn nhiều:

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)
	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!

	winner := ws.WaitForMsg()
	p.game.Finish(winner)
}
```

Khi chúng ta tìm ra cách để không loại bỏ các thông báo blind, công việc của chúng ta sẽ hoàn thành.

### _Đừng_ viết bản kiểm thử!

Đôi khi, khi chúng ta chưa chắc chắn cách triển khai, cách tốt nhất là thử nghiệm trước. Hãy đảm bảo rằng công việc của bạn đã được commit, vì khi tìm ra cách tiếp cận hợp lý, chúng ta sẽ quay lại và triển khai đúng cách thông qua bản kiểm thử (drive it through a test).

Dòng mã đang cần xử lý là:

```go
p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!
```

Chúng ta cần truyền vào một `io.Writer` để trò chơi có thể ghi thông báo blind vào đó.

Sẽ rất tốt nếu chúng ta có thể truyền `playerServerWS` vào đây. Kiểu dữ liệu này đóng gói kết nối WebSocket, vì vậy chúng ta _có cảm giác_ có thể truyền nó cho `Game` để gửi tin nhắn đến trình duyệt.

Hãy thử:

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, ws)
	//v.v...
}
```

Trình biên dịch sẽ báo lỗi:

```
./server.go:71:14: cannot use ws (type *playerServerWS) as type io.Writer in argument to p.game.Start:
	*playerServerWS does not implement io.Writer (missing Write method)
```

Giải pháp hiển nhiên là triển khai phương thức `Write` cho `playerServerWS`. Chúng ta sẽ sử dụng `*websocket.Conn` bên trong cùng hàm `WriteMessage` để gửi tin nhắn qua WebSocket:

```go
func (w *playerServerWS) Write(p []byte) (n int, err error) {
	err = w.WriteMessage(websocket.TextMessage, p)

	if err != nil {
		return 0, err
	}

	return len(p), nil
}
```

Mọi thứ có vẻ rất đơn giản. Hãy chạy thử ứng dụng.

Trước tiên, hãy điều chỉnh `TexasHoldem` để khoảng thời gian tăng blind ngắn hơn, nhằm dễ quan sát:

```go
blindIncrement := time.Duration(5+numberOfPlayers) * time.Second // (thay vì tính bằng phút)
```

Bạn sẽ thấy nó hoạt động. Giá trị blind tự động tăng lên trên trình duyệt.

Bây giờ hãy hoàn tác (revert) thay đổi thử nghiệm vừa rồi và tìm cách kiểm thử đúng cách. Để _kiểm thử_ rằng chúng ta truyền `playerServerWS` vào `Start` thay vì `io.Discard`, bạn có thể nghĩ đến việc do thám (spy) lời gọi đó. Tuy nhiên, chúng ta nên ưu tiên kiểm thử dựa trên _hành vi thực tế_ thay vì chi tiết triển khai. Lý do là khi bạn tái cấu trúc (refactor) mã nguồn, các bản kiểm thử dựa trên chi tiết triển khai rất dễ bị hỏng dù hệ thống vẫn hoạt động đúng.

Bộ kiểm thử hiện tại của chúng ta đã kiểm thử theo hành vi: mở kết nối WebSocket đến ứng dụng đang chạy, gửi tin nhắn và kiểm tra kết quả. Chúng ta hoàn toàn có thể kiểm thử các tin nhắn trả về theo cách tương tự.

## Viết bản kiểm thử trước tiên

Chúng ta sẽ chỉnh sửa bộ kiểm thử hiện có.

Hiện tại `GameSpy` khi được gọi `Start` không gửi bất kỳ thông báo nào. Chúng ta cần cập nhật để nó ghi một tin nhắn mẫu vào đầu ra, sau đó kiểm tra xem tin nhắn đó có được nhận qua kết nối WebSocket hay không. Điều này chứng minh rằng chúng ta đã cấu hình đúng mọi thứ và đầu ra được gửi đến đúng nơi.

```go
type GameSpy struct {
	StartCalled     bool
	StartCalledWith int
	BlindAlert      []byte

	FinishedCalled   bool
	FinishCalledWith string
}
```

Chúng ta thêm trường `BlindAlert` kiểu `[]byte`.

Cập nhật phương thức `Start` của `GameSpy` để ghi tin nhắn đóng gói sẵn vào đầu ra:

```go
func (g *GameSpy) Start(numberOfPlayers int, out io.Writer) {
	g.StartCalled = true
	g.StartCalledWith = numberOfPlayers
	out.Write(g.BlindAlert)
}
```

Ý tưởng ở đây là: khi `PlayerServer` gọi `Start`, tin nhắn sẽ được ghi vào kết nối WebSocket. Bản kiểm thử sẽ đọc tin nhắn từ kết nối WebSocket và xác minh rằng nó đúng như mong đợi.

Bây giờ hãy cập nhật bản kiểm thử:

```go
t.Run("start a game with 3 players, send some blind alerts down WS and declare Ruth the winner", func(t *testing.T) {
	wantedBlindAlert := "Blind is 100"
	winner := "Ruth"

	game := &GameSpy{BlindAlert: []byte(wantedBlindAlert)}
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(10 * time.Millisecond)
	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)

	_, gotBlindAlert, _ := ws.ReadMessage()

	if string(gotBlindAlert) != wantedBlindAlert {
		t.Errorf("got blind alert %q, want %q", string(gotBlindAlert), wantedBlindAlert)
	}
})
```

- Chúng ta khai báo `wantedBlindAlert` và cấu hình `GameSpy` để ghi giá trị này vào luồng đầu ra khi `Start` được gọi.
- Sau khi gửi tin nhắn qua WebSocket, chúng ta gọi `ws.ReadMessage()` để đọc tin nhắn trả về từ máy chủ và kiểm tra xem nội dung có khớp với thông báo blind mong đợi hay không.

## Thử chạy bản kiểm thử

Bạn sẽ thấy bản kiểm thử bị treo (hang). Nguyên nhân là `ws.ReadMessage()` sẽ chặn (block) chờ một tin nhắn, nhưng tin nhắn đó không bao giờ được gửi đến vì chúng ta vẫn đang truyền `io.Discard` thay vì kết nối WebSocket thực.

## Viết lượng mã nguồn tối thiểu để chạy bản kiểm thử và kiểm tra kết quả lỗi

Chúng ta không nên để bản kiểm thử bị treo mãi. Hãy tạo một hàm hỗ trợ xử lý thời gian chờ (timeout):

```go
func within(t testing.TB, d time.Duration, assert func()) {
	t.Helper()

	done := make(chan struct{}, 1)

	go func() {
		assert()
		done <- struct{}{}
	}()

	select {
	case <-time.After(d):
		t.Error("timed out")
	case <-done:
	}
}
```

Hàm `within` hoạt động như sau: nó nhận một hàm `assert` và chạy hàm đó trong một goroutine riêng. Khi hàm `assert` hoàn tất, nó sẽ gửi tín hiệu vào channel `done`.

Trong khi đó, chúng ta sử dụng `select` để chờ hai sự kiện: hoặc hàm `assert` hoàn tất (nhận tín hiệu từ channel `done`), hoặc thời gian chờ hết hạn (nhận tín hiệu từ `time.After`). Sự kiện nào xảy ra trước sẽ được xử lý.

Tiếp theo, hãy đóng gói phần kiểm tra tin nhắn WebSocket vào một hàm hỗ trợ để bản kiểm thử gọn gàng hơn:

```go
func assertWebsocketGotMsg(t *testing.T, ws *websocket.Conn, want string) {
	_, msg, _ := ws.ReadMessage()
	if string(msg) != want {
		t.Errorf(`got "%s", want "%s"`, string(msg), want)
	}
}
```

Bản kiểm thử cuối cùng sẽ như sau:

```go
t.Run("start a game with 3 players, send some blind alerts down WS and declare Ruth the winner", func(t *testing.T) {
	wantedBlindAlert := "Blind is 100"
	winner := "Ruth"

	game := &GameSpy{BlindAlert: []byte(wantedBlindAlert)}
	server := httptest.NewServer(mustMakePlayerServer(t, dummyPlayerStore, game))
	ws := mustDialWS(t, "ws"+strings.TrimPrefix(server.URL, "http")+"/ws")

	defer server.Close()
	defer ws.Close()

	writeWSMessage(t, ws, "3")
	writeWSMessage(t, ws, winner)

	time.Sleep(tenMS)

	assertGameStartedWith(t, game, 3)
	assertFinishCalledWith(t, game, winner)
	within(t, tenMS, func() { assertWebsocketGotMsg(t, ws, wantedBlindAlert) })
})
```

Chạy bản kiểm thử:

```
=== RUN   TestGame
=== RUN   TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.02s)
    --- FAIL: TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner (0.02s)
    	server_test.go:143: timed out
    	server_test.go:150: got "", want "Blind is 100"
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Cuối cùng, chúng ta có thể cập nhật mã nguồn máy chủ để truyền kết nối WebSocket vào `Start` thay vì `io.Discard`:

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, ws)

	winner := ws.WaitForMsg()
	p.game.Finish(winner)
}
```

## Tái cấu trúc

Mã nguồn máy chủ chỉ thay đổi rất ít, nên không có nhiều việc cần làm ở đây. Tuy nhiên, phần mã kiểm thử vẫn đang sử dụng `time.Sleep` để chờ các tác vụ bất đồng bộ hoàn tất, điều này không lý tưởng.

Chúng ta có thể tái cấu trúc các hàm `assertGameStartedWith` và `assertFinishCalledWith` để chúng tự động thử lại (retry) phép kiểm tra trong một khoảng thời gian ngắn, thay vì phụ thuộc vào `time.Sleep`.

Ví dụ với `assertFinishCalledWith`:

```go
func assertFinishCalledWith(t testing.TB, game *GameSpy, winner string) {
	t.Helper()

	passed := retryUntil(500*time.Millisecond, func() bool {
		return game.FinishCalledWith == winner
	})

	if !passed {
		t.Errorf("expected finish called with %q but got %q", winner, game.FinishCalledWith)
	}
}
```

Hàm `retryUntil` được định nghĩa như sau:

```go
func retryUntil(d time.Duration, f func() bool) bool {
	deadline := time.Now().Add(d)
	for time.Now().Before(deadline) {
		if f() {
			return true
		}
	}
	return false
}
```

## Tổng kết

Ứng dụng của chúng ta giờ đây đã hoàn thiện hơn. Người dùng có thể chơi poker qua trình duyệt web, và trình duyệt sẽ nhận được các thông báo blind thông qua WebSockets. Khi trò chơi kết thúc, kết quả người chiến thắng được lưu trữ (persisted) nhờ mã nguồn mà chúng ta đã xây dựng từ các chương trước. Người chơi cũng có thể xem bảng xếp hạng tại endpoint `/league`.

Nhờ kiên trì áp dụng quy trình TDD, chúng ta luôn có thể tiến về phía trước một cách tự tin dù gặp nhiều thách thức. Việc lặp đi lặp lại (iterating) và thử nghiệm (experimenting) giúp chúng ta tìm ra giải pháp tốt.

Chương này kết thúc với một vài điểm tổng kết về các quyết định thiết kế và những gì chúng ta đã học được.

### WebSockets

- Đây là cách đơn giản và hiệu quả để thiết lập giao tiếp hai chiều giữa máy khách (client) và máy chủ (server). Máy khách không cần liên tục gửi yêu cầu đến máy chủ để kiểm tra cập nhật mới. Cách triển khai ở cả hai phía đều khá gọn nhẹ.
- Việc kiểm thử WebSockets không quá phức tạp, tuy nhiên cần đặc biệt chú ý đến tính bất đồng bộ (asynchronous) của mã nguồn.

### Xử lý mã nguồn bị treo hoặc chờ vô hạn trong bản kiểm thử

- Tạo các hàm hỗ trợ (helper) cho bản kiểm thử với cơ chế thử lại (retry) và thời gian chờ (timeout).
- Sử dụng goroutine để chạy phần kiểm tra bất đồng bộ, tránh làm chặn luồng chính của bản kiểm thử. Sau đó sử dụng channel để nhận tín hiệu hoàn tất.
- Go cung cấp sẵn gói `time` với các hàm hẹn giờ hữu ích, cho phép gửi tín hiệu qua channel khi hết thời gian chờ (timeout).
