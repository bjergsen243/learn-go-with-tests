# Giao diện dòng lệnh và cấu trúc dự án

**[Tất cả mã nguồn của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/command-line)**

Chủ sở hữu sản phẩm của chúng ta bây giờ muốn *thay đổi hướng đi* bằng cách giới thiệu một ứng dụng thứ hai - một ứng dụng dòng lệnh.

Hiện tại, nó sẽ chỉ cần có khả năng ghi lại trận thắng của người chơi khi người dùng nhập `Ruth wins`. Mục đích là cuối cùng nó sẽ trở thành một công cụ giúp người dùng chơi poker.

Chủ sở hữu sản phẩm muốn cơ sở dữ liệu được chia sẻ giữa hai ứng dụng để bảng xếp hạng cập nhật theo các trận thắng được ghi lại trong ứng dụng mới.

## Nhắc lại về mã nguồn

Chúng ta có một ứng dụng với tệp `main.go` khởi chạy một máy chủ HTTP. Máy chủ HTTP sẽ không thú vị đối với chúng ta trong bài tập này nhưng sự trừu tượng hóa mà nó sử dụng thì có. Nó phụ thuộc vào một `PlayerStore`.

```go
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() League
}
```

Trong chương trước, chúng ta đã tạo một `FileSystemPlayerStore` thực hiện interface đó. Chúng ta sẽ có thể tái sử dụng một phần của nó cho ứng dụng mới của mình.

## Tái cấu trúc dự án trước tiên

Dự án của chúng ta bây giờ cần tạo ra hai bản thực thi (binaries), máy chủ web hiện có và ứng dụng dòng lệnh (CLI).

Trước khi bắt tay vào công việc mới, chúng ta nên cấu trúc lại dự án của mình để phù hợp với việc này.

Cho đến nay, tất cả mã nguồn đều nằm trong một thư mục, trong một đường dẫn trông như thế này:

`$GOPATH/src/github.com/your-name/my-app`

Để bạn có thể tạo một ứng dụng trong Go, bạn cần một hàm `main` bên trong một `package main`. Cho đến nay, tất cả mã nguồn "tên miền" (domain) của chúng ta đều nằm bên trong `package main` và `func main` của chúng ta có thể tham chiếu đến mọi thứ.

Điều này vẫn ổn cho đến nay và là một phương pháp hay khi không quá sa đà vào cấu trúc gói (package structure). Nếu bạn dành thời gian xem qua thư viện tiêu chuẩn, bạn sẽ thấy rất ít các thư mục và cấu trúc phức tạp.

Rất may là việc thêm cấu trúc khi bạn cần nó khá đơn giản.

Bên trong dự án hiện có, hãy tạo một thư mục `cmd` với một thư mục `webserver` bên trong đó (ví dụ: `mkdir -p cmd/webserver`).

Di chuyển tệp `main.go` vào trong đó.

Nếu bạn đã cài đặt `tree`, bạn nên chạy nó và cấu trúc của bạn sẽ trông như thế này:

```
.
|-- file_system_store.go
|-- file_system_store_test.go
|-- cmd
|   |-- webserver
|       |-- main.go
|-- league.go
|-- server.go
|-- server_integration_test.go
|-- server_test.go
|-- tape.go
|-- tape_test.go
```

Hiện tại chúng ta đã thực hiện việc tách biệt hiệu quả giữa ứng dụng và mã nguồn thư viện nhưng bây giờ chúng ta cần thay đổi một số tên gói. Hãy nhớ rằng khi bạn xây dựng một ứng dụng Go, gói của nó *phải* là `main`.

Thay đổi tất cả các mã nguồn khác để có một gói tên là `poker`.

Cuối cùng, chúng ta cần nhập (import) gói này vào `main.go` để có thể sử dụng nó nhằm tạo máy chủ web của mình. Sau đó, chúng ta có thể sử dụng mã nguồn thư viện của mình bằng cách sử dụng `poker.TenHam`.

Các đường dẫn sẽ khác nhau trên máy tính của bạn, nhưng nó sẽ tương tự như thế này:

```go
// cmd/webserver/main.go
package main

import (
	"github.com/quii/learn-go-with-tests/command-line/v1"
	"log"
	"net/http"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	server := poker.NewPlayerServer(store)

	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Đường dẫn đầy đủ có vẻ hơi rắc rối, nhưng đây là cách bạn có thể nhập *bất kỳ* thư viện công khai nào vào mã nguồn của mình.

Bằng cách tách biệt mã nguồn tên miền của chúng ta thành một gói riêng biệt và đẩy nó lên một kho lưu trữ công khai như GitHub, bất kỳ nhà phát triển Go nào cũng có thể viết mã nguồn của riêng họ để nhập gói đó và sử dụng các tính năng mà chúng ta đã viết. Lần đầu tiên bạn thử chạy nó, nó sẽ phàn nàn rằng tệp không tồn tại nhưng tất cả những gì bạn cần làm là chạy `go get`.

Ngoài ra, người dùng có thể xem [tài liệu tại pkg.go.dev](https://pkg.go.dev/github.com/quii/learn-go-with-tests/command-line/v1).

### Kiểm tra cuối cùng

- Bên trong thư mục gốc, hãy chạy `go test` và kiểm tra xem chúng vẫn vượt qua
- Đi vào bên trong `cmd/webserver` của chúng ta và thực hiện `go run main.go`
  - Truy cập `http://localhost:5000/league` và bạn sẽ thấy nó vẫn hoạt động

### Khung xương di động (Walking skeleton)

Trước khi bắt tay vào viết các bản kiểm thử, hãy thêm một ứng dụng mới mà dự án của chúng ta sẽ xây dựng. Tạo một thư mục khác bên trong `cmd` gọi là `cli` (giao diện dòng lệnh) và thêm một `main.go` với nội dung sau:

```go
// cmd/cli/main.go
package main

import "fmt"

func main() {
	fmt.Println("Hãy chơi poker")
}
```

Yêu cầu đầu tiên chúng ta sẽ giải quyết là ghi lại một trận thắng khi người dùng nhập `{TenNguoiChoi} wins`.

## Viết bản kiểm thử trước tiên

Chúng ta biết mình cần tạo ra một thứ gọi là `CLI` cho phép chúng ta `Chơi` (Play) poker. Nó sẽ cần đọc đầu vào của người dùng và sau đó ghi lại các trận thắng vào một `PlayerStore`.

Tuy nhiên, trước khi đi quá xa, hãy viết một bản kiểm thử để kiểm tra xem nó có tích hợp với `PlayerStore` như chúng ta mong muốn hay không.

Bên trong `CLI_test.go` (trong thư mục gốc của dự án, không phải bên trong `cmd`):

```go
// CLI_test.go
package poker

import "testing"

func TestCLI(t *testing.T) {
	playerStore := &StubPlayerStore{}
	cli := &CLI{playerStore}
	cli.PlayPoker()

	if len(playerStore.winCalls) != 1 {
		t.Fatal("expected a win call but didn't get any")
	}
}
```

- Chúng ta có thể sử dụng `StubPlayerStore` từ các bản kiểm thử khác
- Chúng ta truyền sự phụ thuộc của mình vào kiểu `CLI` chưa tồn tại
- Kích hoạt trò chơi bằng một phương thức `PlayPoker` chưa được viết
- Kiểm tra xem một trận thắng đã được ghi lại hay chưa

## Thử chạy bản kiểm thử

```
# github.com/quii/learn-go-with-tests/command-line/v2
./cli_test.go:25:10: undefined: CLI
```

## Viết lượng mã nguồn tối thiểu để bản kiểm thử chạy và kiểm tra kết quả lỗi

Tại thời điểm này, bạn nên cảm thấy đủ thoải mái để tạo cấu trúc `CLI` mới của chúng ta với trường tương ứng cho sự phụ thuộc của chúng ta và thêm một phương thức.

Bạn sẽ kết thúc với mã nguồn như thế này:

```go
// CLI.go
package poker

type CLI struct {
	playerStore PlayerStore
}

func (cli *CLI) PlayPoker() {}
```

Hãy nhớ rằng chúng ta chỉ đang cố gắng làm cho bản kiểm thử chạy để có thể kiểm tra xem bản kiểm thử có thất bại như chúng ta mong đợi hay không:

```
--- FAIL: TestCLI (0.00s)
    cli_test.go:30: expected a win call but didn't get any
FAIL
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

```go
// CLI.go
func (cli *CLI) PlayPoker() {
	cli.playerStore.RecordWin("Cleo")
}
```

Điều đó sẽ làm cho nó vượt qua.

Tiếp theo, chúng ta cần mô phỏng việc đọc từ `Stdin` (đầu vào từ người dùng) để chúng ta có thể ghi lại các trận thắng cho những người chơi cụ thể.

Hãy mở rộng bản kiểm thử của chúng ta để thực hiện việc này.

## Viết bản kiểm thử trước tiên

```go
// CLI_test.go
func TestCLI(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &StubPlayerStore{}

	cli := &CLI{playerStore, in}
	cli.PlayPoker()

	if len(playerStore.winCalls) != 1 {
		t.Fatal("expected a win call but didn't get any")
	}

	got := playerStore.winCalls[0]
	want := "Chris"

	if got != want {
		t.Errorf("didn't record correct winner, got %q, want %q", got, want)
	}
}
```

`os.Stdin` là những gì chúng ta sẽ sử dụng trong `main` để thu thập đầu vào của người dùng. Nó là một `*File` bên dưới, có nghĩa là nó triển khai `io.Reader`, mà như chúng ta đã biết cho đến nay, là một cách thuận tiện để thu thập văn bản.

Chúng ta tạo một `io.Reader` trong bản kiểm thử của mình bằng cách sử dụng `strings.NewReader` tiện dụng, làm đầy nó bằng những gì chúng ta mong đợi người dùng sẽ nhập.

## Thử chạy bản kiểm thử

`./CLI_test.go:12:32: too many values in struct initializer`

## Viết lượng mã nguồn tối thiểu để bản kiểm thử chạy và kiểm tra kết quả lỗi

Chúng ta cần thêm sự phụ thuộc mới của mình vào `CLI`.

```go
// CLI.go
package poker

import "io"

type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}
```

```
--- FAIL: TestCLI (0.00s)
    CLI_test.go:23: didn't record the correct winner, got 'Cleo', want 'Chris'
FAIL
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Hãy nhớ làm điều đơn giản nhất trước tiên:

```go
func (cli *CLI) PlayPoker() {
	cli.playerStore.RecordWin("Chris")
}
```

Bản kiểm thử vượt qua. Tiếp theo chúng ta sẽ thêm một bản kiểm thử khác để buộc mình phải viết một số mã nguồn thực sự, nhưng trước tiên, hãy tái cấu trúc.

## Tái cấu trúc

Trong `server_test`, trước đó chúng ta đã thực hiện các kiểm tra để xem liệu các trận thắng có được ghi lại như chúng ta có ở đây hay không. Hãy làm cho khẳng định đó gọn gàng hơn (DRY) bằng một helper:

```go
// server_test.go
func assertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
	t.Helper()

	if len(store.winCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
	}

	if store.winCalls[0] != winner {
		t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], winner)
	}
}
```

Bây giờ hãy thay thế các khẳng định trong cả `server_test.go` và `CLI_test.go`.

Bản kiểm thử bây giờ nên trông như sau:

```go
// CLI_test.go
func TestCLI(t *testing.T) {
	in := strings.NewReader("Chris wins\n")
	playerStore := &StubPlayerStore{}

	cli := &CLI{playerStore, in}
	cli.PlayPoker()

	assertPlayerWin(t, playerStore, "Chris")
}
```

Bây giờ hãy viết một bản kiểm thử *khác* với đầu vào người dùng khác để buộc chúng ta thực sự phải đọc nó.

## Viết bản kiểm thử trước tiên

```go
// CLI_test.go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &StubPlayerStore{}

		cli := &CLI{playerStore, in}
		cli.PlayPoker()

		assertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &StubPlayerStore{}

		cli := &CLI{playerStore, in}
		cli.PlayPoker()

		assertPlayerWin(t, playerStore, "Cleo")
	})

}
```

## Thử chạy bản kiểm thử

```
=== RUN   TestCLI
--- FAIL: TestCLI (0.00s)
=== RUN   TestCLI/record_chris_win_from_user_input
    --- PASS: TestCLI/record_chris_win_from_user_input (0.00s)
=== RUN   TestCLI/record_cleo_win_from_user_input
    --- FAIL: TestCLI/record_cleo_win_from_user_input (0.00s)
        CLI_test.go:27: did not store correct winner got 'Chris' want 'Cleo'
FAIL
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Chúng ta sẽ sử dụng một [`bufio.Scanner`](https://golang.org/pkg/bufio/) để đọc đầu vào từ `io.Reader`.

> Gói bufio triển khai I/O có bộ đệm. Nó bao bọc một đối tượng io.Reader hoặc io.Writer, tạo ra một đối tượng khác (Reader hoặc Writer) cũng thực hiện interface đó nhưng cung cấp bộ đệm và một số trợ giúp cho I/O dạng văn bản.

Cập nhật mã nguồn thành như sau:

```go
// CLI.go
package poker

import (
	"bufio"
	"io"
	"strings"
)

type CLI struct {
	playerStore PlayerStore
	in          io.Reader
}

func (cli *CLI) PlayPoker() {
	reader := bufio.NewScanner(cli.in)
	reader.Scan()
	cli.playerStore.RecordWin(extractWinner(reader.Text()))
}

func extractWinner(userInput string) string {
	return strings.Replace(userInput, " wins", "", 1)
}
```

Các bản kiểm thử bây giờ sẽ vượt qua.

- `Scanner.Scan()` sẽ đọc cho đến khi gặp dòng mới.
- Sau đó, chúng ta sử dụng `Scanner.Text()` để trả về chuỗi (`string`) mà scanner đã đọc được.

Bây giờ khi chúng ta đã có một số bản kiểm thử vượt qua, chúng ta nên kết nối mọi thứ vào `main`. Hãy nhớ rằng chúng ta nên luôn cố gắng có phần mềm hoạt động được tích hợp đầy đủ càng nhanh càng tốt.

Trong `main.go`, hãy thêm đoạn mã nguồn sau và chạy nó. (bạn có thể phải điều chỉnh đường dẫn của sự phụ thuộc thứ hai để khớp với đường dẫn trên máy tính của bạn)

```go
package main

import (
	"fmt"
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	fmt.Println("Hãy chơi poker")
	fmt.Println("Nhập {Ten} wins để ghi lại một trận thắng")

	db, err := os.OpenFile(dbFileName, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		log.Fatalf("problem opening %s %v", dbFileName, err)
	}

	store, err := poker.NewFileSystemPlayerStore(db)

	if err != nil {
		log.Fatalf("problem creating file system player store, %v ", err)
	}

	game := poker.CLI{store, os.Stdin}
	game.PlayPoker()
}
```

Bạn sẽ nhận được một lỗi:

```
command-line/v3/cmd/cli/main.go:32:25: implicit assignment of unexported field 'playerStore' in poker.CLI literal
command-line/v3/cmd/cli/main.go:32:34: implicit assignment of unexported field 'in' in poker.CLI literal
```

Những gì đang xảy ra ở đây là do chúng ta đang cố gắng gán cho các trường `playerStore` và `in` trong `CLI`. Đây là các trường chưa được xuất khẩu (riêng tư - unexported). Chúng ta *có thể* làm điều này trong mã nguồn kiểm thử của mình vì bản kiểm thử của chúng ta nằm trong cùng một gói với `CLI` (`poker`). Nhưng `main` của chúng ta nằm trong gói `main` nên nó không có quyền truy cập.

Điều này nhấn mạnh tầm quan trọng của việc *tích hợp công việc của bạn*. Chúng ta đã biến các sự phụ thuộc của `CLI` thành riêng tư một cách đúng đắn (vì chúng ta không muốn chúng bị lộ cho người dùng của `CLI`) nhưng chưa tạo ra cách để người dùng khởi tạo nó.

Có cách nào để phát hiện vấn đề này sớm hơn không?

### `package mypackage_test`

Trong tất cả các ví dụ khác cho đến nay, khi chúng ta tạo một tệp kiểm thử, chúng ta khai báo nó nằm trong cùng một gói với gói mà chúng ta đang kiểm thử.

Điều này vẫn ổn và nó có nghĩa là trong những dịp hiếm hoi khi chúng ta muốn kiểm thử thứ gì đó bên trong gói, chúng ta có quyền truy cập vào các kiểu chưa được xuất khẩu.

Nhưng vì chúng ta đã ủng hộ việc *không* kiểm thử các thứ bên trong *nói chung*, liệu Go có thể giúp thực thi điều đó không? Điều gì sẽ xảy ra nếu chúng ta có thể kiểm thử mã nguồn của mình ở nơi mà chúng ta chỉ có quyền truy cập vào các kiểu đã được xuất khẩu (giống như `main` của chúng ta đã làm)?

Khi bạn viết một dự án với nhiều gói, tôi khuyên bạn nên đặt tên gói kiểm thử có hậu tố `_test`. Khi bạn làm điều này, bạn sẽ chỉ có quyền truy cập vào các kiểu công khai trong gói của mình. Điều này sẽ giúp ích trong trường hợp cụ thể này nhưng cũng giúp thực thi kỷ luật chỉ kiểm thử các API công khai. Nếu bạn vẫn muốn kiểm thử các nội dung bên trong, bạn có thể tạo một bản kiểm thử riêng biệt với gói bạn muốn kiểm thử.

Có một câu nói với TDD là nếu bạn không thể kiểm thử mã nguồn của mình thì có lẽ người dùng mã nguồn của bạn cũng khó tích hợp với nó. Sử dụng `package foo_test` sẽ giúp giải quyết vấn đề này bằng cách buộc bạn phải kiểm thử mã nguồn của mình như thể bạn đang nhập nó giống như người dùng gói của bạn sẽ làm.

Trước khi sửa lỗi `main`, hãy đổi gói của bản kiểm thử bên trong `CLI_test.go` thành `poker_test`.

Nếu bạn có một IDE được cấu hình tốt, bạn sẽ đột nhiên thấy rất nhiều màu đỏ! Nếu bạn chạy trình biên dịch, bạn sẽ nhận được các lỗi sau:

```
./CLI_test.go:12:19: undefined: StubPlayerStore
./CLI_test.go:17:3: undefined: assertPlayerWin
./CLI_test.go:22:19: undefined: StubPlayerStore
./CLI_test.go:27:3: undefined: assertPlayerWin
```

Bây giờ chúng ta đã vấp phải nhiều câu hỏi hơn về thiết kế gói. Để kiểm thử phần mềm của mình, chúng ta đã tạo ra các stub chưa xuất khẩu và các hàm helper mà hiện tại không còn khả dụng cho chúng ta sử dụng trong `CLI_test` vì các helper được định nghĩa trong các tệp `_test.go` trong gói `poker`.

#### Chúng ta có muốn các stub và helper của mình là 'công khai' không?

Đây là một cuộc thảo luận mang tính chủ quan. Một số người có thể lập luận rằng bạn không muốn làm ô nhiễm API của gói mình bằng các mã nguồn để phục vụ các bản kiểm thử.

Trong bài trình bày ["Advanced Testing with Go"](https://speakerdeck.com/mitchellh/advanced-testing-with-go?slide=53) của Mitchell Hashimoto, có mô tả cách tại HashiCorp, họ ủng hộ việc làm điều này để người dùng gói có thể viết các bản kiểm thử mà không cần phải phát minh lại các stub. Trong trường hợp của chúng ta, điều này có nghĩa là bất kỳ ai sử dụng gói `poker` của chúng ta sẽ không phải tự tạo ra stub `PlayerStore` của riêng họ nếu họ muốn làm việc với mã nguồn của chúng ta.

Theo kinh nghiệm cá nhân, tôi đã sử dụng kỹ thuật này trong các gói dùng chung khác và nó đã tỏ ra cực kỳ hữu ích về mặt tiết kiệm thời gian cho người dùng khi tích hợp với các gói của chúng ta.

Vì vậy, hãy tạo một tệp gọi là `testing.go` và thêm stub của chúng ta cùng các helper.

```go
// testing.go
package poker

import "testing"

type StubPlayerStore struct {
	scores   map[string]int
	winCalls []string
	league   []Player
}

func (s *StubPlayerStore) GetPlayerScore(name string) int {
	score := s.scores[name]
	return score
}

func (s *StubPlayerStore) RecordWin(name string) {
	s.winCalls = append(s.winCalls, name)
}

func (s *StubPlayerStore) GetLeague() League {
	return s.league
}

func AssertPlayerWin(t testing.TB, store *StubPlayerStore, winner string) {
	t.Helper()

	if len(store.winCalls) != 1 {
		t.Fatalf("got %d calls to RecordWin want %d", len(store.winCalls), 1)
	}

	if store.winCalls[0] != winner {
		t.Errorf("did not store correct winner got %q want %q", store.winCalls[0], winner)
	}
}

// bài tập cho bạn - các helper còn lại
```

Bạn sẽ cần làm cho các helper trở thành công khai (hãy nhớ rằng việc xuất khẩu được thực hiện bằng một chữ cái viết hoa ở đầu) nếu bạn muốn chúng được lộ ra cho những người nhập gói của chúng ta.

Trong bản kiểm thử `CLI`, bạn sẽ cần gọi mã nguồn như thể bạn đang sử dụng nó trong một gói khác.

```go
// CLI_test.go
func TestCLI(t *testing.T) {

	t.Run("record chris win from user input", func(t *testing.T) {
		in := strings.NewReader("Chris wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := &poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Chris")
	})

	t.Run("record cleo win from user input", func(t *testing.T) {
		in := strings.NewReader("Cleo wins\n")
		playerStore := &poker.StubPlayerStore{}

		cli := &poker.CLI{playerStore, in}
		cli.PlayPoker()

		poker.AssertPlayerWin(t, playerStore, "Cleo")
	})

}
```

Bây giờ bạn sẽ thấy chúng ta gặp các vấn đề tương tự như chúng ta đã gặp trong `main`:

```
./CLI_test.go:15:26: implicit assignment of unexported field 'playerStore' in poker.CLI literal
./CLI_test.go:15:39: implicit assignment of unexported field 'in' in poker.CLI literal
./CLI_test.go:25:26: implicit assignment of unexported field 'playerStore' in poker.CLI literal
./CLI_test.go:25:39: implicit assignment of unexported field 'in' in poker.CLI literal
```

Cách dễ nhất để vượt qua điều này là tạo một constructor như chúng ta đã làm cho các kiểu khác. Chúng ta cũng sẽ thay đổi `CLI` để nó lưu trữ một `bufio.Scanner` thay vì reader vì hiện tại nó được bọc tự động vào thời điểm khởi tạo.

```go
// CLI.go
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
```

Bằng cách làm này, chúng ta có thể đơn giản hóa và tái cấu trúc mã nguồn đọc:

```go
// CLI.go
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

Thay đổi bản kiểm thử để sử dụng constructor thay thế và chúng ta sẽ quay lại tình trạng các bản kiểm thử vượt qua.

Cuối cùng, chúng ta có thể quay lại `main.go` mới của mình và sử dụng constructor mà chúng ta vừa tạo:

```go
// cmd/cli/main.go
game := poker.NewCLI(store, os.Stdin)
```

Thử và chạy nó, nhập "Bob wins".

### Tái cấu trúc

Chúng ta có một số sự lặp lại trong các ứng dụng tương ứng của mình, nơi chúng ta đang mở một tệp và tạo một `file_system_store` từ nội dung của nó. Đây có cảm giác như là một điểm yếu nhẹ trong thiết kế gói của chúng ta, vì vậy chúng ta nên tạo một hàm trong đó để đóng gói việc mở tệp từ một đường dẫn và trả về cho bạn `PlayerStore`.

```go
// file_system_store.go
func FileSystemPlayerStoreFromFile(path string) (*FileSystemPlayerStore, func(), error) {
	db, err := os.OpenFile(path, os.O_RDWR|os.O_CREATE, 0666)

	if err != nil {
		return nil, nil, fmt.Errorf("problem opening %s %v", path, err)
	}

	closeFunc := func() {
		db.Close()
	}

	store, err := NewFileSystemPlayerStore(db)

	if err != nil {
		return nil, nil, fmt.Errorf("problem creating file system player store, %v ", err)
	}

	return store, closeFunc, nil
}
```

Bây giờ hãy tái cấu trúc cả hai ứng dụng của chúng ta để sử dụng hàm này nhằm tạo kho lưu trữ.

#### Mã nguồn ứng dụng CLI

```go
// cmd/cli/main.go
package main

import (
	"fmt"
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"os"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	fmt.Println("Hãy chơi poker")
	fmt.Println("Nhập {Ten} wins để ghi lại một trận thắng")
	poker.NewCLI(store, os.Stdin).PlayPoker()
}
```

#### Mã nguồn ứng dụng máy chủ web

```go
// cmd/webserver/main.go
package main

import (
	"github.com/quii/learn-go-with-tests/command-line/v3"
	"log"
	"net/http"
)

const dbFileName = "game.db.json"

func main() {
	store, close, err := poker.FileSystemPlayerStoreFromFile(dbFileName)

	if err != nil {
		log.Fatal(err)
	}
	defer close()

	server := poker.NewPlayerServer(store)

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

Hãy chú ý sự đối xứng: mặc dù là các giao diện người dùng khác nhau nhưng việc thiết lập gần như giống hệt nhau. Điều này có cảm giác như một sự xác nhận tốt cho thiết kế của chúng ta cho đến nay.
Và cũng lưu ý rằng `FileSystemPlayerStoreFromFile` trả về một hàm đóng, vì vậy chúng ta có thể đóng tệp bên dưới sau khi sử dụng xong Store.

## Tổng kết

### Cấu trúc gói (Package structure)

Chương này có nghĩa là chúng ta muốn tạo hai ứng dụng, tái sử dụng mã nguồn tên miền mà chúng ta đã viết cho đến nay. Để làm được điều này, chúng ta cần cập nhật cấu trúc gói của mình để có các thư mục riêng biệt cho các hàm `main` tương ứng.

Bằng cách này, chúng ta đã gặp phải các vấn đề tích hợp do các giá trị chưa xuất khẩu, vì vậy điều này chứng minh thêm giá trị của việc làm việc theo từng "lát" nhỏ và tích hợp thường xuyên.

Chúng ta đã học cách `mypackage_test` giúp chúng ta tạo một môi trường kiểm thử giống như trải nghiệm đối với các gói khác khi tích hợp với mã nguồn của bạn, nhằm giúp bạn nắm bắt các vấn đề tích hợp và xem mã nguồn của bạn dễ hoạt động như thế nào (hoặc không!).

### Đọc dữ liệu người dùng nhập

Chúng ta đã thấy việc đọc từ `os.Stdin` rất dễ dàng đối với chúng ta vì nó thực hiện `io.Reader`. Chúng ta đã sử dụng `bufio.Scanner` để dễ dàng đọc dữ liệu người dùng nhập theo từng dòng.

### Sự trừu tượng hóa đơn giản dẫn đến việc tái sử dụng mã nguồn đơn giản hơn

Hầu như không mất công sức để tích hợp `PlayerStore` vào ứng dụng mới của chúng ta (sau khi chúng ta đã thực hiện các điều chỉnh về gói) và việc kiểm thử sau đó cũng rất dễ dàng vì chúng ta đã quyết định để lộ phiên bản stub của mình.

