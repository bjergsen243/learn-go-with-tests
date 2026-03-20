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
 - Trong JavaSript, mở một WebSocket kết nối với server của chúng ta và xử lý sự kiện người cùng nhấn nút gửi, dữ liệu khi nhấn sẽ được gửi qua WebSocket.

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

[`html/template`](https://golang.org/pkg/html/template/) là một gói (package) của Go để tạo thẻ HTML. Trong trường hợp của chúng ta, chúng ta gọi hàm `template.ParseFiles` rồi truyền vào đấy đường dẫn file html của mình. Nếu giả sử không có lỗi nào được trả về, bạn có thể gọi `tmpl.Execute` cho file mẫu vừa định ra, cái sẽ ghi thẻ html vào kiểu `io.Writer`. Ở đây chúng ta muốn in thẻ ra màn hình trình duyệt của người dùng, nên chúng ta sẽ chuyển dữ liệu cho kiểu ghi `http.ResponseWriter`.

Vì chúng ta không viết chức năng test nào, việc có chút cẩn trọng thực hiện test server thủ công thì luôn chính đáng để đảm bảo ứng dụng có thể hoạt động đúng với kì vọng của mình. Đi tới `cmd/webserver` và chạy dòng lệnh tệp chạy `main.go`. Có thể xem chúng ở địa chỉ: `http://localhost:5000/game`.

Bạn _lẽ ra_ có thể gặp lỗi về việc không thể tìm thấy tệp game.html. Bạn có thể sửa bằng cách chuyển đường dẫn của nó thành đường dẫn tương đối, hoặc chỉ việc sao chép `game.html` vào thư mục `cmd/webserver` đang chứa file chạy. Tôi thì đã chọn cách dùng symbolic link (liên kết hệ thống `ln -s ../../game.html game.html`) trỏ file trong hệ thống tới root lúc chạy. Nhờ cách này, lúc tôi chạy thì file lúc nào cũng được cập nhật những chỉnh sửa mới nhất.

Làm cách của tôi rồi chạy lại, giờ bạn cũng có thể mường tượng được Giao diện người dùng trên web trông sẽ như nào.

Việc tiếp theo là kiểm thử trường hợp khi chúng ta nhận được một message chuỗi theo kết nối WebSocket tới máy chủ, server sẽ lấy giá trị đó để làm tên người chiến thắng của một game mới chơi.

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

Để kiểm thử những gì xảy ra từ phía trình duyệt, chúng ta phải tự mở một kết nối WebSocket và ghi dữ liệu (write) vào đó.

Các bản kiểm thử trước đây của chúng ta xung quanh máy chủ (server) của mình thì chỉ dừng lại ở các function cho server thay vì có các luồng dữ liệu liên tiếp truyền tới máy chủ. Để giả định việc này, chúng ta sử dụng `httptest.NewServer`, hàm này sẽ tạo một interface xử lý `http.Handler`, nó bao gồm việc lắng nghe các kết nối tới server.

Dùng hàm Dial `websocket.DefaultDialer.Dial` để cố gắng thử kết nối (dialing) tới server, sau đấy mới thử gửi tin nhắn tới nó với thông số `winner` đã chuẩn bị.

Cuối cùng, chúng ta khẳng định (assert) trên đối tượng player store của người chơi để kiểm tra xem liệu người chiến thắng đã thực sự được lưu lại vào CSDL hay chưa.

## Thử chạy bản kiểm thử
```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:124: could not open a ws connection on ws://127.0.0.1:55838/ws websocket: bad handshake
```

Chúng ta đã chưa thực sự cấu hình cho máy chủ của mình việc chấp nhận các kết nối giao thức qua web socket `/ws` nên có vẻ việc bắt lỗi đầu vào không thấy diễn ra.

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Thêm một route mới vào file:

```go
router.Handle("/ws", http.HandlerFunc(p.webSocket))
```

Sau đó thêm hàm mới, hàm nghe HTTP từ websocket:

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	upgrader := websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
	upgrader.Upgrade(w, r, nil)
}
```

Để cấu hình nhận WebSocket, chúng ta phải truyền hàm `Upgrade` này để nâng cấp requet request HTTP thành một web socket request, có nhiệm vụ "chắp bút" bắt tay với socket của trình duyệt người dùng. Nửa chặng đường rồi, nếu chạy kiểm thử ngay lập tức thì khả năng lỗi sẽ như thế này:

```
=== RUN   TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game
    --- FAIL: TestGame/when_we_get_a_message_over_a_websocket_it_is_a_winner_of_a_game (0.00s)
        server_test.go:132: got 0 calls to RecordWin want 1
```

Cập nhật lại phương thức đọc luồng Websocket để nhận thông điệp tới và ghi điểm lại coi nào:

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

(Vầng, chúng ta cũng phớt lờ đi khá nhiều chi tiết xử lý lỗi!)

Lệnh `conn.ReadMessage()` làm gián đoạn mọi việc khi nó phải đợi tin nhắn luồng trên trình duyệt. Lệnh tiếp theo dùng để lưu người thắng là kết quả của lệnh đó qua `RecordWin`. Khi việc này hoàn tất thì đồng nghĩa với việc kết nối sẽ dừng.

Kiểm thử báo lỗi? Khoan, làm thế quái nào được khi ta làm đúng bài!

Lỗi ở đoạn timing giữa các hành động. Có một độ trễ nhỏ giữa lúc server đọc luồn dữ liệu, lấy kết quả ghi vào file của db và lúc trả lỗi kiểm thử của chúng ta. Bạn hoàn toàn có thể chứng thực điều đó bằng việc điền hàm ngủ ảo như lệnh `time.Sleep` để trì hoãn test báo lỗi.

Cái kim trong bọc lâu chày cũng lòi ra rồi, để đó, chúng ta sẽ xem xét xem vì sao đây như là một trò đùa, và vì sao chèn những con số ảo vào trong mã kiểm thử lại là một **phương pháp rất tồi**.

```go
time.Sleep(10 * time.Millisecond)
AssertPlayerWin(t, store, winner)
```

## Tái cấu trúc

Chúng ta đã tạo ra nhiều "khối u" trong việc vừa cố để bản test làm việc trơn tru, vừa khiến cho máy chủ thực hiện đúng chức năng của mình. Nhưng trên quan điểm của quá trình hoàn thiện phần mềm, đây là phần việc khá dễ trôi.

Mớ rác này đang hoạt động theo đúng kì vọng do một cái tool báo láo! Nhiệm vụ là gì? Chúng ta sẽ tự tay dọn dẹp mớ hỗn độn này sau chứ gì nữa, thay nó bằng đoạn mã "cool ngầu" hơn chả hạn, phải chắc rằng chúng chẳng làm ai sảy chân nữa mới được.

Tới phần mã nguồn của hệ quy chiếu server, `PlayerServer`:

Bạn có thể thêm nó vào `upgrader` vì tính riêng biệt nên là ta sẽ khai báo nó chung bên ngoài package. Thay vì thế này:

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

Lời gọi hàm từ `template.ParseFiles("game.html")` bây giờ sẽ đi vào tất tần tật yêu cầu URL ở dạng `GET /game`. Cách làm này sẽ mang lại tác hại nhất định: trên mọi trang ở client trỏ vào yêu cầu URL đấy, server đều phải tự parse lại template, rõ ràng chẳng ai cần parse hai lần vào một template không hề bị chỉnh sửa (re-parse) bao giờ. Cách sửa thì là bạn parse duy nhất một lần qua file `NewPlayerServer` để có thể nhận luôn kết quả ra cho vào biến lưu, sau đó chèn vào code khi được gọi. Thử làm thế, bạn cũng cần biến hàm trả về giá trị là các mã lỗi liên quan phòng hờ hỏng lúc lấy file dữ liệu hệ thống ra để parse, hoặc file trống.

Mã nguồn mới của tệp thiết lập server `PlayerServer`:

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

Bằng cách thay đổi cấu trúc thiết kế ở package `NewPlayerServer`, mã ở các file đang xài hàm đó đang gặp vấn đề biên dịch code. Để ý xem mình có fix được các chỗ trống của mã hổng không? Tự coi code cũng là một ý hay nếu bí.

Sửa các lỗi đó ở file tệp test của chúng ta khá dễ dàng, tôi đã thêm hàm helper trả về các pointer *của server* `mustMakePlayerServer(t *testing.T, store PlayerStore) *PlayerServer` để tôi có thể làm gọn các vấn đề vặt phát sinh lúc mình truyền tham số của `PlayerStore`

```go
func mustMakePlayerServer(t *testing.T, store PlayerStore) *PlayerServer {
	server, err := NewPlayerServer(store)
	if err != nil {
		t.Fatal("problem creating player server", err)
	}
	return server
}
```

Lại một trợ thủ đắc lực nữa giúp tôi khỏi phải bịt lỗ hổng cho hàm thiết lập websocket, dùng để báo lỗi: tên helper `mustDialWS`

```go
func mustDialWS(t *testing.T, url string) *websocket.Conn {
	ws, _, err := websocket.DefaultDialer.Dial(url, nil)

	if err != nil {
		t.Fatalf("could not open a ws connection on %s %v", url, err)
	}

	return ws
}
```

Và một ông bạn cuối cùng dọn dẹp vấn đề nhắn lại message từ máy server.

```go
func writeWSMessage(t testing.TB, conn *websocket.Conn, message string) {
	t.Helper()
	if err := conn.WriteMessage(websocket.TextMessage, []byte(message)); err != nil {
		t.Fatalf("could not send message over ws connection %v", err)
	}
}
```

Chạy file để kiểm thử test pass! Server cũng hoạt động, việc khai tên những người vô tiền khoáng hậu tại `/game` hoàn toàn bình thường, có thể coi điểm trực tiếp và tên được update liên tục từ bảng điểm chung của thư mục `/league`. Ghi nhớ lấy điều này: server cắt hẳn kết nối ngay lúc xác định người thứ bao nhiêu thắng, bạn sẽ phải chịu khó mở lại kết nối URL đấy để thêm lượt khác vào.

Với biểu mẫu HTML web đơn giản để làm quen, ta đã thu hoạch được mục tiêu người thắng cuộc. Ở phiên bản tiếp theo, hãy cho tính năng tạo màn chơi, cho biết luật số lượng game chơi và báo tin nhắn về "Tiền mù" - blind qua client trong thời gian diễn ra ván đấu.

<!-- PART 2 HERE -->age over ws connection %v", err)
	}
}
```

Bây giờ các bản kiểm thử đã vượt qua, hãy thử chạy máy chủ và khai báo một số người chiến thắng trong `/game`. Bạn sẽ thấy chúng được ghi lại trong `/league`. Hãy nhớ rằng mỗi khi chúng ta nhận được người chiến thắng, chúng ta _sẽ đóng kết nối_, bạn sẽ cần làm mới trang để mở lại kết nối.

Chúng ta đã tạo ra một biểu mẫu web đơn giản cho phép người dùng ghi lại người chiến thắng của trò chơi. Hãy lặp lại quá trình đó để người dùng có thể bắt đầu trò chơi bằng cách cung cấp số lượng người chơi và máy chủ sẽ đẩy (push) tin nhắn đến máy khách (client) thông báo cho họ về giá trị blind khi thời gian trôi qua.

Đầu tiên, hãy cập nhật `game.html` để cập nhật mã phía máy khách của chúng ta cho các yêu cầu mới:

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

Các thay đổi chính là đưa vào một phần để nhập số lượng người chơi và một phần để hiển thị giá trị blind. Chúng ta có một chút logic để hiển thị/ẩn giao diện người dùng tùy thuộc vào giai đoạn của trò chơi.

Bất kỳ tin nhắn nào chúng ta nhận được qua `conn.onmessage`, chúng ta đều cho rằng đó là các cảnh báo blind, vì vậy chúng ta đặt `blindContainer.innerText` tương ứng.

Làm thế nào để chúng ta gửi các cảnh báo blind? Trong chương trước, chúng ta đã giới thiệu ý tưởng về `Game` để mã CLI của chúng ta có thể gọi một `Game` và mọi thứ khác sẽ được lo liệu bao gồm cả việc lên lịch cho các cảnh báo blind. Điều này hóa ra lại là một sự phân tách ranh giới tốt.

```go
type Game interface {
	Start(numberOfPlayers int)
	Finish(winner string)
}
```

Khi người dùng được nhắc nhập số lượng người chơi trong CLI, nó sẽ `Bắt đầu` (`Start`) trò chơi, thao tác này sẽ khởi động các cảnh báo blind và khi người dùng kết thúc với kết quả người chiến thắng, họ sẽ chọn `Kết thúc` (`Finish`). Đây là những yêu cầu giống hệt như chúng ta có lúc này, chỉ là một cách khác để nhận đầu vào; vì vậy chúng ta nên tìm cách "tái chế" (re-use) logic này.

Sự triển khai "thực" của `Game` là hàm xử lý ván đấu `TexasHoldem`

```go
type TexasHoldem struct {
	alerter BlindAlerter
	store   PlayerStore
}
```

Bằng cách gửi vào một `BlindAlerter`, `TexasHoldem` có thể lập lịch đẩy thông báo tièn mù (blind) đến _bất cứ nơi nào_

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int)
}
```

Nhắc một chút nha, đây là giao diện định nghĩa hàm blind đã sử dụng trong giao diện CLI `BlindAlerter`. 

```go
func StdOutAlerter(duration time.Duration, amount int) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(os.Stdout, "Blind is now %d\n", amount)
	})
}
```

Nó hoạt động trong CLI vì chúng ta _luôn muốn gửi các cảnh báo đến `os.Stdout`_ nhưng điều này sẽ không hoạt động cho máy chủ web của chúng ta. Đối với mỗi yêu cầu, chúng ta nhận được một `http.ResponseWriter` mới và sau đó nâng cấp thành `*websocket.Conn`. Vì vậy, chúng ta không thể biết khi cấu trúc các phụ thuộc của mình thì các cảnh báo của chúng ta cần đi đâu.

Vì lý do đó, chúng ta cần thay đổi `BlindAlerter.ScheduleAlertAt` để nó nhận một đích đến cho các cảnh báo, nhằm mục đích tái sử dụng trong máy chủ web của mình.

Mở `blind_alerter.go` và thêm tham số `io.Writer`:

```go
type BlindAlerter interface {
	ScheduleAlertAt(duration time.Duration, amount int, to io.Writer)
}

type BlindAlerterFunc func(duration time.Duration, amount int, to io.Writer)

func (a BlindAlerterFunc) ScheduleAlertAt(duration time.Duration, amount int, to io.Writer) {
	a(duration, amount, to)
}
```

Ý tưởng về một `StdoutAlerter` không còn phù hợp với mô hình mới của chúng ta nữa, vì vậy hãy đổi tên nó thành `Alerter`:

```go
func Alerter(duration time.Duration, amount int, to io.Writer) {
	time.AfterFunc(duration, func() {
		fmt.Fprintf(to, "Blind is now %d\n", amount)
	})
}
```

Nếu bạn thử biên dịch, nó sẽ báo lỗi trong `TexasHoldem` vì nó đang gọi `ScheduleAlertAt` mà không có đích đến (tham số `to`), để mọi thứ biên dịch lại được _ngay bây giờ_ ,tốt nhất cứ hãy gán cứng giá trị vào (hard-code) kiểu `os.Stdout` cái đã.

Thử chạy các bài kiểm thử và chúng sẽ thất bại vì `SpyBlindAlerter` không còn triển khai `BlindAlerter` nữa, sửa lỗi này bằng cách cập nhật header của hàm (chữ ký hàm) `ScheduleAlertAt`, khởi chạy các bài test và hệ thống khả năng cao vẫn sẽ xanh lên đèn.

Chẳng có ý đồ gì cho việc `TexasHoldem` biết được nơi gửi đi tin nhắn báo lỗi tiền blind (tiền mù) cả. Cập nhật struct `Game` để khi trò chơi chạy, nó xác định rõ xem _ở đâu_ tin nhắn nên được chuyển tới.

```go
type Game interface {
	Start(numberOfPlayers int, alertsDestination io.Writer)
	Finish(winner string)
}
```

Biên dịch sẽ chỉ cho bạn có lỗi nào và đoạn sửa nó nằm ở đâu (không tốn nhiều sức lắm):

- Làm lại hàm `TexasHoldem` nó sẽ được sử dụng thích hợp cho `Game`.
- Ở phía `CLI`, ngay khi ta chơi game, gán thêm thông số `out` cho object này (`cli.game.Start(numberOfPlayers, cli.out)`)
- Ở đoạn code test cho tệp `TexasHoldem` tôi sử dụng đoạn mã như sau `game.Start(5, io.Discard)`  để sửa các lỗi gây ra khi build và cấu hình output cho thông báo được bỏ qua đi 

Mọi thứ chạy không lỗi lầm gì? Thế chúng ta sẽ dùng hàm `Game` này ngay và luôn ở tệp `Server`.

## Viết bản kiểm thử trước tiên

Ràng buộc về môi trường của `CLI` cũng như `Server` cũng chỉ có ngần ấy! Khác mỗi cách thông báo được gửi đi kiểu chi.

Lật qua lật lại các mã test của `CLI` để khơi nguồn cảm hứng nhắm mắt viết code

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

Chúng ta hoàn toàn khả dĩ dựa vào test này mà viết mã tương tự cho việc áp dụng chức năng đó vào đối tượng `GameSpy` đấy. Sửa đoạn test websocket cũ đinh ninh lại bằng nội dung bên dưới:

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

- Việc chúng ta tạo một gián điệp tên là spy `Game` để được chuyển dữ liệu sang `mustMakePlayerServer` đã được thảo luận ở trên (nhớ không quên cập nhập hàm helper dùng được chức năng này).
- Sau đó chúng ta có thể gửi tin nhắn qua Websocket đến game.
- Chúng ta xác nhận game được khởi tạo đúng theo kì vọng tại phần code assert ở cuối.

## Chạy thử mã test

Bạn sẽ gặp các lỗi lúc biên dịch qua ở phần `mustMakePlayerServer` trong vài tests khác. Khai một biến ẩn kiểu gán `dummyGame` vào xem xét gán cho nó vào các test đang treo không dịch lại được.

```go
var (
	dummyGame = &GameSpy{}
)
```

Rồi, sẽ đến lỗi lúc ta tọng cục giá trị của `Game` vào constructor tên là `NewPlayerServer` tuy nhiên nó không được support cái parameter ấy.

```
./server_test.go:21:38: too many arguments in call to "github.com/quii/learn-go-with-tests/WebSockets/v2".NewPlayerServer
	have ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore, "github.com/quii/learn-go-with-tests/WebSockets/v2".Game)
	want ("github.com/quii/learn-go-with-tests/WebSockets/v2".PlayerStore)
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Add luôn argument vào chỗ này đi để qua test lỗi thứ n.

```go
func NewPlayerServer(store PlayerStore, game Game) (*PlayerServer, error)
```

Chạy

```
=== RUN   TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.01s)
    --- FAIL: TestGame/start_a_game_with_3_players_and_declare_Ruth_the_winner (0.01s)
    	server_test.go:146: wanted Start called with 3 but got 0
    	server_test.go:147: expected finish called with 'Ruth' but got ''
FAIL
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Vậy là cần bổ sung đối tượng `Game` này vào trường (field) của gói thuộc về struct mang tên `PlayerServer` giúp code vận hành theo những lệnh truy cập (request).

```go
type PlayerServer struct {
	store PlayerStore
	http.Handler
	template *template.Template
	game     Game
}
```

(Tiện thì rename nốt methods hiện đang trùng tên game thành `playGame` đi cho đỡ hoa mắt nhé)

Sau đó cập nhật giá trị khởi tạo trong phương thức định nghĩa đối tượng của nó

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

Đã đến lúc dùng `Game` cùng `webSocket` rồi.

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

Oki la! Những phương thức test không còn mắng tôi nữa.

Vẫn chưa thể đẩy ngay cái tin nhắn alert được, khoan hãng làm vậy, suy xét một tẹo này. Khi gọi hàm `game.Start`, thực sự là mình truyền vào cho nó hàm xóa dấu vết dữ liệu in ra thông qua `io.Discard`, dọn đường cho bất kỳ thông điệp nào tới đều tàng hình.

Cứ khởi động cái web server lên nghe ngóng tình hình, sau đó thì nên để cả file `main.go`. Nạp `Game` này vào để xài qua `PlayerServer` mới

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

Tạm bỏ qua chuyện bạn chưa thấy nó báo cáo điểm mù blind tới giờ này. App _tỏ vẻ_ là vẫn tốt! Thành quả là nhồi đủ thứ để `Game` hoạt động nhịp nhàng ăn ý với `PlayerServer` cùng đống code backend tốn cơm khác, khi hiểu được gửi cái cục đó của nợ thế nào mà vẫn thấy vui vui vì những đống đổ rác không bị báo tràn bộ nhớ, code nó vẫn _work_ ổn.

Xử trước mớ bòng bong không đáng có kia sau này đã.

## Tái cấu trúc

Cách mà chúng ta đang triển khai WebSockets khá là cơ bản và cách báo kết quả mã code cũng khá trẻ con và ngu ngốc, để tránh lỗi vặt từ server nên ta gán thuộc tính đóng gói mảng thuộc tính như thế, sửa thì lúc khác bớt ngu cũng được. Tạm dẹp cái đống tủa tủa này và đi tiếp.

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

Mã nguồn trên máy chủ nay đã gọn gàng hẳn

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)
	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, io.Discard) //todo: Đừng loại bỏ thông báo tiền cược mù (blind alerts)!

	winner := ws.WaitForMsg()
	p.game.Finish(winner)
}
```

Một khi chúng ta tìm ra cách để không loại bỏ các thông báo tiền mù, công việc của chúng ta sẽ hoàn thành.

### _Đừng_ viết bản kiểm thử nhé!

Đôi khi, khi chúng ta không chắc chắn phải làm gì đó, cách tốt nhất là cứ thử nghiệm và vọc vạch một chút! Đảm bảo rằng công việc của bạn đã được commit trước, bởi vì khi chúng ta đã tìm ra một cách hợp lý thì chúng ta mới nên thực hiện qua bản kiểm thử theo đúng hướng (drive it through a test).

Dòng code đang có vấn đề của chúng ta là:

```go
p.game.Start(numberOfPlayers, io.Discard) //todo: Don't discard the blinds messages!
```

Chúng ta cần truyền vào một `io.Writer` để trò chơi có thể ghi thông báo cảnh báo mù (blind alerts).

Chẳng phải sẽ rất tốt nếu chúng ta có thể truyền vào biến `playerServerWS` mà chúng ta viết lúc nãy sao? Nó được bọc để che đi WebSocket của chúng ta nên có _cảm giác_ như chúng ta có thể truyền nó cho đối tượng `Game` để gửi tin nhắn đến.

Hãy thử xem:

```go
func (p *PlayerServer) webSocket(w http.ResponseWriter, r *http.Request) {
	ws := newPlayerServerWS(w, r)

	numberOfPlayersMsg := ws.WaitForMsg()
	numberOfPlayers, _ := strconv.Atoi(numberOfPlayersMsg)
	p.game.Start(numberOfPlayers, ws)
	//v.v...
}
```

Trình biên dịch phàn nàn:

```
./server.go:71:14: cannot use ws (type *playerServerWS) as type io.Writer in argument to p.game.Start:
	*playerServerWS does not implement io.Writer (missing Write method)
```

Điều hiển nhiên có vẻ cần làm là khiến đối tượng `playerServerWS` có thể triển khai `io.Writer`. Để làm vậy, chúng ta sử dụng con trỏ trỏ tới `*websocket.Conn` bên dưới cùng hàm `WriteMessage` để gửi tin nhắn qua websocket:

```go
func (w *playerServerWS) Write(p []byte) (n int, err error) {
	err = w.WriteMessage(websocket.TextMessage, p)

	if err != nil {
		return 0, err
	}

	return len(p), nil
}
```

Trông có vẻ mọi thứ quá dễ dàng! Chạy thử ứng dụng xem sao.

Trước tiên hãy sửa lại `TexasHoldem` để khoảng thời gian gia tăng tiền mù ngắn hơn nhằm dễ theo dõi hành động:

```go
blindIncrement := time.Duration(5+numberOfPlayers) * time.Second // (thay vì làm cả phút)
```

Đáng lẽ bạn phải thấy nó hoạt động mới đúng! Số tiền cược tự động tăng lên trên trình duyệt như thể có phép thuật vậy.

Bây giờ hãy đảo ngược phiên code lại (revert code) và nghĩ cách để test cái này xem nào. Để nhằm _thực thi_ tất cả mọi điều mà chúng ta đã làm - truyền dữ liệu thẳng tới `StartGame` như đối tượng `playerServerWS` thay cho kiểu biến mất dạng `io.Discard`, việc này có thể khiến bạn nghĩ rằng chúng ta phải do thám lệnh gọi trên chỉ để kiểm chứng nó có chạy được không.

Do thám một lời gọi là rất tốt để hỗ trợ ta đối soát các khâu bên trong bản triển khai của hàm nhưng ta nên thiên về việc xây dựn test bằng các luồng hoạt động _chính thống_ của ứng dụng lúc sử dụng thực tế vì khi bạn nhúng tay vào việc Refactor cấu trúc dự án thì mấy bài test kiểu spy calls này rất hay fail ngớ ngẩn do chúng thường check quá sâu các chi tiết trong luồng thực thi mà đáng ra bạn đang cần thay đổi.

Bộ kiểm thử hiện tại của chúng ta thực hiện điều đó thông qua việc mở kết nối socket trò chơi đến ứng dụng đang chạy ở local và gửi một mảng tin nhắn yêu cầu ứng dụng xử lý những việc nhất định. Rõ ràng là ta cũng có khả năng test những bản tin trả về từ hệ thống server dựa theo đường đó.

## Viết bản kiểm thử trước tiên

Ta sẽ điều chỉnh lại bộ thử nghiệm hiện đang có.

Hiện tại test của hàm `GameSpy` lúc gọi hàm `Start` không in băm ra tý tin nhắn hay output nào. Chúng ta nên cập nhật lại một tí thành hệ thống gửi đi một mẫu thư rác cài sẵn, để lúc server trả lại cho socket xem ta có check đc message ấy chuẩn chưa. Nó là thứ sẽ giúp chúng minh mình cấu hình chuẩn mọi thứ đồng thời cho ra output đúng với bản chất ứng dụng.

```go
type GameSpy struct {
	StartCalled     bool
	StartCalledWith int
	BlindAlert      []byte

	FinishedCalled   bool
	FinishCalledWith string
}
```

Hãy chèn trường dữ liệu kiểu byte là `BlindAlert` vô đây.

Bổ sung lại logic trả hàm `Start` ở đối tượng `GameSpy` báo tin nhắn đóng hộp vào dữ liệu output `out`.

```go
func (g *GameSpy) Start(numberOfPlayers int, out io.Writer) {
	g.StartCalled = true
	g.StartCalledWith = numberOfPlayers
	out.Write(g.BlindAlert)
}
```

Ý đồ của chuyện này nằm ở lúc chạy kiểm tra qua gói `PlayerServer`, quá trình gọi `Bắt đầu` màn chơi `Start` từ đầu cuối máy chủ nó sẽ ném tin nhắn xuyên suốt giao tiếp kết nối vào đường truyền của websocket theo kỳ vọng đặt sẵn.

Bây giờ là đoạn chúng ta vá mã cho function về test:

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

- Một thuộc tính `wantedBlindAlert` vừa mới vào sân, cùng cấu hình nhồi tin `GameSpy` trả cái này ra stream `out` khi hàm khởi động `Start` có hiệu lực.
- Để chờ đón tin vui nhắn qua cầu nối websocket, ta phải dùng lệnh bốc dỡ qua đối tượng socket read stream ngóng biến trả về là `ws.ReadMessage()` rồi lại mang ra đọ xem cái message từ ws đấy có đúng bản auth hay không (chứ đừng báo "Cảnh mù" linh tinh).

## Thử chạy bản kiểm thử

Bạn sẽ thấy test cứ bị treo. Sở dĩ có việc này là do cái lệnh `ws.ReadMessage()` có đặc điểm làm đóng băng code chờ một message trả lại mà message ấy lại éo bao giờ được tung ra cả. 


## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Không được để test chạy kiểu đứng hình, vì thế hãy đem vào đây một kỹ thuật bẫy xử lý với kiểu code mà nó gây vượt thời gian ngóng chờ - time outs.

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

Cách mã `within` thiết kế ra: nhận một hàm `assert` truyền từ đối số gọi vào nó, sau đấy mới bốc hàm này xử lý trong một khối goroutine. Khi/Nếu quy trình này thực thi xong thì việc gởi tiếp hiệu lệnh kết thúc (struct không mang data) nhét tạm vào block chứa `done` ở dạng channel - tức luồng.

Song song trong quy trình đợi ở đó chúng ta sẽ gọi tới khái niệm `select` điều đó giúp nghe ngóng và đợi ở các hướng kênh - channel channel để lấy message ra. Nơi này là sân chơi thi gan đọ độ chây ì của hai lệnh gồm `assert` (từ goroutines) hay lệnh cảnh báo là `time.After` cái thứ có thể sút hiệu lệnh khi hạn timeout gõ chuông cảnh báo.

Sau chót, việc bọc assertion vào thành một tiểu hàm - function giúp cấu trúc đoạn test trong ngắn gọn súc tích hơn đấy.

```go
func assertWebsocketGotMsg(t *testing.T, ws *websocket.Conn, want string) {
	_, msg, _ := ws.ReadMessage()
	if string(msg) != want {
		t.Errorf(`got "%s", want "%s"`, string(msg), want)
	}
}
```

Trông bài kiểm tra rốt cuộc sẽ như thế nào nào:

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

Và giờ hãy chích mồi lệnh run code:

```
=== RUN   TestGame
=== RUN   TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner
--- FAIL: TestGame (0.02s)
    --- FAIL: TestGame/start_a_game_with_3_players,_send_some_blind_alerts_down_WS_and_declare_Ruth_the_winner (0.02s)
    	server_test.go:143: timed out
    	server_test.go:150: got "", want "Blind is 100"
```

## Viết đủ code để test chạy thành công

Sau bao bộn bề thì cuối cùng cũng có thể can thiệp lại vào mã nguồn chạy hệ thống server của chúng ta để gửi thẳng cái `Kết nối WebSocket` của ứng dụng qua kênh server ở lúc game thiết lập:

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

Code trên máy chủ chỉ được thay đổi chút ét nên có lặn ở đây cũng không có nhiều việc phải khuân vác, ngoại trừ phần khai báo cho code test đang chình ình cái phương thức `time.Sleep` vì chúng ta bắt ép mọi công cụ đi ngủ để đội đợi cái hàm bất đồng bộ phía server kịp nhả xong kết quả.

Hàm sửa lỗi có thể refactor lại 2 phương thức là `assertGameStartedWith` and `assertFinishCalledWith` với việc hỗ trợ chúng việc retry liên tục hành động assertion trong thời lượng rất ngắn để tránh bị đánh fail khi chậm có output.

Làm điều này với `assertFinishCalledWith` kiểu này nhé:

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

Đoạn định nghĩa khối `retryUntil` :

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

Ứng dụng của chúng ta hiện nay là một dự án "sống" rồi đấy. Trò chơi có thể đi vào hành trình bài bạc poker qua web duyệt trên trình duyệt được luôn, và client được nhận các thông số gửi cọc mù khi qua giao thức gọi là the WebSockets. Càng mừng nữa là hồi cuối ván mọi record người nào "đớp" có thể lưu cứng (persisted) lấy nguồn từ code ta khai sinh cả mấy chương trước. Những cao thủ muốn so tài (hay do may nốt) giờ có luôn một endpoint tra bảng xếp hạng `/league` danh dá thế kia.

Kể từ những đầu tư công sức vào TDD flow, dù làm vấp lắm đi nữa song chúng ta luôn tìm thấy đích tới thay vì lạc một phương trời phần mềm lởm khởm chết xó đâu đấy. Tha hồ "Iterating", tha hồ "Experimenting"!

Chương kết thúc khép lại chặng được theo đuổi TDD bằng buổi review về sự lựa chọn về đường theo thiết kế code và gói nó thật ngăn nắp nhé.

Có một vài việc ta có được tại chương này như sau:

### WebSockets

- Đây là phương án đơn giản nhưng trúng đích vô việc truyền và nhận các luồng chuỗi dữ liệu nhét giữa clients (nhiều máy) với (một) servers để hệ thống khỏi rơi vào hoàn cảnh client lại liên tục làm gián đoạn việc hỏi "server ơi gửi gì chửa?". Bản cài phía hai bên nghe chừng rất tối giản.
- Khá bình dân với mảng testing, tuy nhiên phải hết sức dè chừng vào cái thói quen test mảng code chạy không hẹn (asynchronous/ bất đồng bộ).

### Kỹ nghệ giữ lấy đoạn code trong bài kiểm tra dẫu code lấp lửng hay chờ vĩnh viễn

- Bịa ngay hàm chắp vá (helper) cho lệnh kiểm tra để cấu kiện tính retry cho code test lúc bị timeout.
- Có thẻ móc `go routines` của anh bạn để xử việc bất đồng bộ của assert khỏi lo code bơi tắc đường với hệ thống và rồi gọng lẩy tin báo là tao xong hay fail ra mẻ rổ của Go là cái list `channels` 
- Mấy thanh niên Go build trong package một bộ hẹn thời gian tuyệt phẩm tên `time` giúp nả đạn vào channels báo tín hiệu có thay đổi hay hết thời gian (timeout).
