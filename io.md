# IO và sắp xếp (sorting)

**[Tất cả mã nguồn của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/io)**

[Trong chương trước](json.md), chúng ta đã tiếp tục lặp lại ứng dụng của mình bằng cách thêm một điểm cuối (endpoint) mới `/league`. Trong quá trình đó, chúng ta đã tìm hiểu về cách xử lý JSON, nhúng các kiểu (embedding types) và định tuyến (routing).

Chủ sở hữu sản phẩm của chúng ta hơi lo lắng vì phần mềm bị mất điểm số khi máy chủ được khởi động lại. Điều này là do việc triển khai kho lưu trữ của chúng ta đang ở trong bộ nhớ (in-memory). Cô ấy cũng không hài lòng khi chúng ta không hiểu rằng endpoint `/league` nên trả về những người chơi được sắp xếp theo số trận thắng!

## Mã nguồn cho đến nay

```go
// server.go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"
	"strings"
)

// PlayerStore lưu trữ thông tin về điểm số của các người chơi
type PlayerStore interface {
	GetPlayerScore(name string) int
	RecordWin(name string)
	GetLeague() []Player
}

// Player lưu trữ tên cùng với số trận thắng
type Player struct {
	Name string
	Wins int
}

// PlayerServer là một interface HTTP cho thông tin người chơi
type PlayerServer struct {
	store PlayerStore
	http.Handler
}

const jsonContentType = "application/json"

// NewPlayerServer tạo một PlayerServer với định tuyến đã được cấu hình
func NewPlayerServer(store PlayerStore) *PlayerServer {
	p := new(PlayerServer)

	p.store = store

	router := http.NewServeMux()
	router.Handle("/league", http.HandlerFunc(p.leagueHandler))
	router.Handle("/players/", http.HandlerFunc(p.playersHandler))

	p.Handler = router

	return p
}

func (p *PlayerServer) leagueHandler(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("content-type", jsonContentType)
	json.NewEncoder(w).Encode(p.store.GetLeague())
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

func (i *InMemoryPlayerStore) GetLeague() []Player {
	var league []Player
	for name, wins := range i.store {
		league = append(league, Player{name, wins})
	}
	return league
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
	server := NewPlayerServer(NewInMemoryPlayerStore())
	log.Fatal(http.ListenAndServe(":5000", server))
}
```

Bạn có thể tìm thấy các bản kiểm thử tương ứng trong liên kết ở đầu chương.

## Lưu trữ dữ liệu

Có hàng tá cơ sở dữ liệu mà chúng ta có thể sử dụng cho việc này nhưng chúng ta sẽ chọn một cách tiếp cận rất đơn giản. Chúng ta sẽ lưu trữ dữ liệu cho ứng dụng này trong một tệp dưới dạng JSON.

Điều này giúp dữ liệu rất dễ di chuyển và tương đối đơn giản để triển khai.

Nó sẽ không mở rộng đặc biệt tốt nhưng vì đây là một bản mẫu (prototype) nên nó sẽ ổn cho lúc này. Nếu hoàn cảnh của chúng ta thay đổi và nó không còn phù hợp nữa, việc đổi nó sang một thứ khác sẽ đơn giản nhờ vào sự trừu tượng hóa `PlayerStore` mà chúng ta đã sử dụng.

Chúng ta sẽ giữ lại `InMemoryPlayerStore` trong lúc này để các bản kiểm thử tích hợp tiếp tục vượt qua khi chúng ta phát triển kho lưu trữ mới của mình. Một khi chúng ta tự tin rằng triển khai mới của mình đủ để làm cho bản kiểm thử tích hợp vượt qua, chúng ta sẽ thay thế nó và sau đó xóa `InMemoryPlayerStore`.

## Viết bản kiểm thử trước tiên

Đến bây giờ bạn chắc hẳn đã quen thuộc với các interface xung quanh thư viện tiêu chuẩn để đọc dữ liệu (`io.Reader`), ghi dữ liệu (`io.Writer`) và cách chúng ta có thể sử dụng thư viện tiêu chuẩn để kiểm thử các hàm này mà không cần sử dụng các tệp thật.

Để công việc này hoàn tất, chúng ta sẽ cần triển khai `PlayerStore`, vì vậy chúng ta sẽ viết các bản kiểm thử cho kho lưu trữ của mình bằng cách gọi các phương thức cần triển khai. Chúng ta sẽ bắt đầu với `GetLeague`.

```go
// file_system_store_test.go
func TestFileSystemStore(t *testing.T) {

	t.Run("league from a reader", func(t *testing.T) {
		database := strings.NewReader(`[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)

		store := FileSystemPlayerStore{database}

		got := store.GetLeague()

		want := []Player{
			{"Cleo", 10},
			{"Chris", 33},
		}

		assertLeague(t, got, want)
	})
}
```

Chúng ta đang sử dụng `strings.NewReader` cái mà sẽ trả về cho chúng ta một `Reader`, vốn là thứ mà `FileSystemPlayerStore` của chúng ta sẽ sử dụng để đọc dữ liệu. Trong hàm `main`, chúng ta sẽ mở một tệp, cái mà cũng là một `Reader`.

## Thử chạy bản kiểm thử

```
# github.com/quii/learn-go-with-tests/io/v1
./file_system_store_test.go:15:12: undefined: FileSystemPlayerStore
```

## Viết lượng mã nguồn tối thiểu để bản kiểm thử chạy và kiểm tra kết quả lỗi

Hãy định nghĩa `FileSystemPlayerStore` trong một tệp mới:

```go
// file_system_store.go
type FileSystemPlayerStore struct{}
```

Thử lại lần nữa:

```
# github.com/quii/learn-go-with-tests/io/v1
./file_system_store_test.go:15:28: too many values in struct initializer
./file_system_store_test.go:17:15: store.GetLeague undefined (type FileSystemPlayerStore has no field or method GetLeague)
```

Nó đang phàn nàn vì chúng ta đang truyền vào một `Reader` nhưng không mong đợi một cái và nó chưa có phương thức `GetLeague` được định nghĩa.

```go
// file_system_store.go
type FileSystemPlayerStore struct {
	database io.Reader
}

func (f *FileSystemPlayerStore) GetLeague() []Player {
	return nil
}
```

Thử thêm một lần nữa...

```
=== RUN   TestFileSystemStore//league_from_a_reader
    --- FAIL: TestFileSystemStore//league_from_a_reader (0.00s)
        file_system_store_test.go:24: got [] want [{Cleo 10} {Chris 33}]
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Chúng ta đã đọc JSON từ một reader trước đó rồi:

```go
// file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() []Player {
	var league []Player
	json.NewDecoder(f.database).Decode(&league)
	return league
}
```

Bản kiểm thử bây giờ sẽ vượt qua.

## Tái cấu trúc (Refactor)

Chúng ta *đã* làm việc này trước đây! Mã nguồn kiểm thử cho máy chủ của chúng ta đã phải giải mã JSON từ phản hồi.

Hãy thử làm cho mã nguồn gọn gàng hơn (DRY) bằng một hàm.

Tạo một tệp mới tên là `league.go` và đưa mã nguồn này vào:

```go
// league.go
func NewLeague(rdr io.Reader) ([]Player, error) {
	var league []Player
	err := json.NewDecoder(rdr).Decode(&league)
	if err != nil {
		err = fmt.Errorf("problem parsing league, %v", err)
	}

	return league, err
}
```

Gọi hàm này trong bản triển khai của chúng ta và trong helper kiểm thử `getLeagueFromResponse` trong `server_test.go`.

```go
// file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() []Player {
	league, _ := NewLeague(f.database)
	return league
}
```

Chúng ta vẫn chưa có chiến lược để xử lý các lỗi phân tích cú pháp nhưng hãy tiếp tục.

### Vấn đề về việc tìm kiếm (Seeking)

Có một lỗ hổng trong bản triển khai của chúng ta. Đầu tiên, hãy nhắc nhở bản thân cách `io.Reader` được định nghĩa:

```go
type Reader interface {
	Read(p []byte) (n int, err error)
}
```

Với tệp của mình, bạn có thể tưởng tượng nó đọc qua từng byte cho đến khi kết thúc. Điều gì xảy ra nếu bạn cố gắng `Read` lần thứ hai?

Thêm đoạn mã nguồn sau vào cuối bản kiểm thử hiện tại của chúng ta:

```go
// file_system_store_test.go

// đọc lại lần nữa
got = store.GetLeague()
assertLeague(t, got, want)
```

Chúng ta muốn phần này vượt qua, nhưng nếu bạn chạy bản kiểm thử thì nó không vượt qua.

Vấn đề là `Reader` của chúng ta đã đi đến điểm kết thúc nên không còn gì để đọc nữa. Chúng ta cần một cách để bảo nó quay lại điểm bắt đầu.

[`ReadSeeker`](https://golang.org/pkg/io/#ReadSeeker) là một interface khác trong thư viện tiêu chuẩn có thể giúp ích.

```go
type ReadSeeker interface {
	Reader
	Seeker
}
```

Bạn còn nhớ về nhúng (embedding) chứ? Đây là một interface bao gồm `Reader` và [`Seeker`](https://golang.org/pkg/io/#Seeker).

```go
type Seeker interface {
	Seek(offset int64, whence int) (int64, error)
}
```

Điều này nghe có vẻ tốt, liệu chúng ta có thể thay đổi `FileSystemPlayerStore` để nhận interface này thay thế không?

```go
// file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadSeeker
}

func (f *FileSystemPlayerStore) GetLeague() []Player {
	f.database.Seek(0, io.SeekStart)
	league, _ := NewLeague(f.database)
	return league
}
```

Thử chạy bản kiểm thử, nó hiện đã vượt qua! Thật may cho chúng ta là `strings.NewReader` mà chúng ta đã sử dụng trong bản kiểm thử của mình cũng triển khai `ReadSeeker` nên chúng ta không phải thực hiện bất kỳ thay đổi nào khác.

Tiếp theo chúng ta sẽ triển khai `GetPlayerScore`.

## Viết bản kiểm thử trước tiên

```go
// file_system_store_test.go
t.Run("get player score", func(t *testing.T) {
	database := strings.NewReader(`[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)

	store := FileSystemPlayerStore{database}

	got := store.GetPlayerScore("Chris")

	want := 33

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
})
```

## Thử chạy bản kiểm thử

```
./file_system_store_test.go:38:15: store.GetPlayerScore undefined (type FileSystemPlayerStore has no field or method GetPlayerScore)
```

## Viết lượng mã nguồn tối thiểu để bản kiểm thử chạy và kiểm tra kết quả lỗi

Chúng ta cần thêm phương thức vào kiểu mới của mình để bản kiểm thử có thể biên dịch.

```go
// file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {
	return 0
}
```

Bây giờ nó đã biên dịch và bản kiểm thử thất bại:

```
=== RUN   TestFileSystemStore/get_player_score
    --- FAIL: TestFileSystemStore//get_player_score (0.00s)
        file_system_store_test.go:43: got 0 want 33
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Chúng ta có thể lặp qua bảng xếp hạng để tìm người chơi và trả về số điểm của họ.

```go
// file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	var wins int

	for _, player := range f.GetLeague() {
		if player.Name == name {
			wins = player.Wins
			break
		}
	}

	return wins
}
```

## Tái cấu trúc

Bạn đã thấy hàng tá các bản tái cấu trúc helper kiểm thử rồi, vì vậy tôi sẽ để việc làm cho nó hoạt động lại cho bạn:

```go
// file_system_store_test.go
t.Run("get player score", func(t *testing.T) {
	database := strings.NewReader(`[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)

	store := FileSystemPlayerStore{database}

	got := store.GetPlayerScore("Chris")
	want := 33
	assertScoreEquals(t, got, want)
})
```

Cuối cùng, chúng ta cần bắt đầu ghi lại số điểm với `RecordWin`.

## Viết bản kiểm thử trước tiên

Cách tiếp cận của chúng ta khá là thiển cận đối với việc ghi dữ liệu. Chúng ta không thể (một cách dễ dàng) chỉ cập nhật một "hàng" JSON trong một tệp. Chúng ta sẽ cần lưu trữ *toàn bộ* biểu diễn mới của cơ sở dữ liệu sau mỗi lần ghi.

Làm thế nào để chúng ta ghi? Thông thường chúng ta sẽ sử dụng một `Writer` nhưng chúng ta đã có `ReadSeeker` của mình. Tiềm năng là chúng ta có thể có hai sự phụ thuộc nhưng thư viện tiêu chuẩn đã có một interface cho chúng ta là `ReadWriteSeeker`, cho phép chúng ta thực hiện tất cả những việc cần làm với một tệp.

Hãy cập nhật kiểu của chúng ta:

```go
// file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadWriteSeeker
}
```

Kiểm tra xem nó có biên dịch không:

```
./file_system_store_test.go:15:34: cannot use database (type *strings.Reader) as type io.ReadWriteSeeker in field value:
    *strings.Reader does not implement io.ReadWriteSeeker (missing Write method)
./file_system_store_test.go:36:34: cannot use database (type *strings.Reader) as type io.ReadWriteSeeker in field value:
    *strings.Reader does not implement io.ReadWriteSeeker (missing Write method)
```

Không quá ngạc nhiên khi `strings.Reader` không triển khai `ReadWriteSeeker`, vậy chúng ta phải làm gì?

Chúng ta có hai lựa chọn:

- Tạo một tệp tạm thời cho mỗi bản kiểm thử. `*os.File` triển khai `ReadWriteSeeker`. Ưu điểm của việc này là nó trở thành một bản kiểm thử tích hợp nhiều hơn, chúng ta thực sự đang đọc và ghi từ hệ thống tệp nên nó sẽ mang lại cho chúng ta mức độ tin cậy rất cao. Nhược điểm là chúng ta thích các bản kiểm thử đơn vị hơn vì chúng nhanh hơn và thường đơn giản hơn. Chúng ta cũng sẽ cần phải làm việc nhiều hơn xung quanh việc tạo các tệp tạm thời và sau đó đảm bảo chúng được xóa sau bản kiểm thử.
- Chúng ta có thể sử dụng thư viện của bên thứ ba. [Mattetti](https://github.com/mattetti) đã viết một thư viện [filebuffer](https://github.com/mattetti/filebuffer) triển khai interface chúng ta cần và không chạm vào hệ thống tệp.

Tôi không nghĩ có một câu trả lời nào là đặc biệt sai ở đây, nhưng bằng cách chọn sử dụng một thư viện bên thứ ba, tôi sẽ phải giải thích về quản lý phụ thuộc (dependency management)! Vì vậy, chúng ta sẽ sử dụng các tệp thay thế.

Trước khi thêm bản kiểm thử của mình, chúng ta cần làm cho các bản kiểm thử khác biên dịch bằng cách thay thế `strings.Reader` bằng một `os.File`.

Hãy tạo một số hàm helper sẽ tạo một tệp tạm thời với một số dữ liệu bên trong và trừu tượng hóa các bản kiểm thử điểm số của chúng ta:

```go
// file_system_store_test.go
func createTempFile(t testing.TB, initialData string) (io.ReadWriteSeeker, func()) {
	t.Helper()

	tmpfile, err := os.CreateTemp("", "db")

	if err != nil {
		t.Fatalf("could not create temp file %v", err)
	}

	tmpfile.Write([]byte(initialData))

	removeFile := func() {
		tmpfile.Close()
		os.Remove(tmpfile.Name())
	}

	return tmpfile, removeFile
}

func assertScoreEquals(t testing.TB, got, want int) {
	t.Helper()
	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

[`CreateTemp`](https://pkg.go.dev/os#CreateTemp) tạo một tệp tạm thời cho chúng ta sử dụng. Giá trị `"db"` mà chúng ta đã truyền vào là một tiền tố được đặt trên một tên tệp ngẫu nhiên mà nó sẽ tạo ra. Điều này là để đảm bảo nó sẽ không trùng lặp với các tệp khác một cách vô tình.

Bạn sẽ nhận thấy chúng ta không chỉ trả về `ReadWriteSeeker` (tệp) mà còn trả về một hàm. Chúng ta cần đảm bảo rằng tệp được xóa sau khi bản kiểm thử kết thúc. Chúng ta không muốn để lộ chi tiết của các tệp vào bản kiểm thử vì nó dễ dẫn đến lỗi và không thú vị cho người đọc. Bằng cách trả về một hàm `removeFile`, chúng ta có thể lo liệu các chi tiết trong helper của mình và tất cả những gì người gọi phải làm là chạy `defer cleanDatabase()`.

```go
// file_system_store_test.go
func TestFileSystemStore(t *testing.T) {

	t.Run("league from a reader", func(t *testing.T) {
		database, cleanDatabase := createTempFile(t, `[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)
		defer cleanDatabase()

		store := FileSystemPlayerStore{database}

		got := store.GetLeague()

		want := []Player{
			{"Cleo", 10},
			{"Chris", 33},
		}

		assertLeague(t, got, want)

		// đọc lại lần nữa
		got = store.GetLeague()
		assertLeague(t, got, want)
	})

	t.Run("get player score", func(t *testing.T) {
		database, cleanDatabase := createTempFile(t, `[
			{"Name": "Cleo", "Wins": 10},
			{"Name": "Chris", "Wins": 33}]`)
		defer cleanDatabase()

		store := FileSystemPlayerStore{database}

		got := store.GetPlayerScore("Chris")
		want := 33
		assertScoreEquals(t, got, want)
	})
}
```

Chạy các bản kiểm thử và chúng sẽ vượt qua! Có một lượng kha khá các thay đổi nhưng hiện tại có cảm giác như chúng ta đã hoàn thành định nghĩa interface của mình và việc thêm các bản kiểm thử mới từ giờ sẽ rất dễ dàng.

Hãy chuyển sang lần lặp đầu tiên của việc ghi lại một trận thắng cho một người chơi đã tồn tại:

```go
// file_system_store_test.go
t.Run("store wins for existing players", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer cleanDatabase()

	store := FileSystemPlayerStore{database}

	store.RecordWin("Chris")

	got := store.GetPlayerScore("Chris")
	want := 34
	assertScoreEquals(t, got, want)
})
```

## Thử chạy bản kiểm thử

`./file_system_store_test.go:67:8: store.RecordWin undefined (type FileSystemPlayerStore has no field or method RecordWin)`

## Viết lượng mã nguồn tối thiểu để bản kiểm thử chạy và kiểm tra kết quả lỗi

Thêm phương thức mới:

```go
// file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {

}
```

```
=== RUN   TestFileSystemStore/store_wins_for_existing_players
    --- FAIL: TestFileSystemStore/store_wins_for_existing_players (0.00s)
        file_system_store_test.go:71: got 33 want 34
```

Bản triển khai của chúng ta trống rỗng nên số điểm cũ vẫn được trả về.

## Viết đủ mã nguồn để bản kiểm thử vượt qua

```go
// file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()

	for i, player := range league {
		if player.Name == name {
			league[i].Wins++
		}
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

Bạn có thể tự hỏi tại sao tôi lại sử dụng `league[i].Wins++` thay vì `player.Wins++`.

Khi bạn `range` qua một lát cắt, bạn sẽ nhận được chỉ số hiện tại của vòng lặp (trong trường hợp của chúng ta là `i`) và một *bản sao* của phần tử tại chỉ số đó. Việc thay đổi giá trị `Wins` của một bản sao sẽ không có bất kỳ tác động nào đến lát cắt `league` mà chúng ta lặp qua. Vì lý do đó, chúng ta cần lấy tham chiếu đến giá trị thực tế bằng cách thực hiện `league[i]` và sau đó thay đổi giá trị đó.

Nếu bạn chạy các bản kiểm thử, chúng sẽ vượt qua.

## Tái cấu trúc (Refactor)

Trong `GetPlayerScore` và `RecordWin`, chúng ta đang lặp qua biểu thức `[]Player` để tìm một người chơi theo tên.

Chúng ta có thể tái cấu trúc mã nguồn chung này trong các chi tiết nội bộ của `FileSystemStore`, nhưng đối với tôi, có vẻ như đây là mã nguồn có khả năng hữu ích mà chúng ta có thể đưa vào một kiểu mới. Làm việc với một "League" cho đến nay luôn là với `[]Player` nhưng chúng ta có thể tạo một kiểu mới gọi là `League`. Điều này sẽ dễ dàng hơn cho các nhà phát triển khác hiểu và sau đó chúng ta có thể gắn các phương thức hữu ích vào kiểu đó để sử dụng.

Bên trong `league.go`, hãy thêm đoạn mã nguồn sau:

```go
// league.go
type League []Player

func (l League) Find(name string) *Player {
	for i, p := range l {
		if p.Name == name {
			return &l[i]
		}
	}
	return nil
}
```

Bây giờ nếu ai đó có một `League`, họ có thể dễ dàng tìm thấy một người chơi nhất định.

Thay đổi interface `PlayerStore` của chúng ta để trả về `League` thay vì `[]Player`. Thử chạy lại các bản kiểm thử, bạn sẽ gặp vấn đề biên dịch vì chúng ta đã thay đổi interface nhưng nó rất dễ khắc phục; chỉ cần thay đổi kiểu trả về từ `[]Player` thành `League`.

Điều này cho phép chúng ta đơn giản hóa các phương thức trong `file_system_store`.

```go
// file_system_store.go
func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	player := f.GetLeague().Find(name)

	if player != nil {
		return player.Wins
	}

	return 0
}

func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()
	player := league.Find(name)

	if player != nil {
		player.Wins++
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

Điều này trông tốt hơn nhiều và chúng ta có thể thấy cách chúng ta có thể tìm thấy các chức năng hữu ích khác xung quanh `League` để tái cấu trúc.

Bây giờ chúng ta cần xử lý kịch bản ghi lại các trận thắng của những người chơi mới.

## Viết bản kiểm thử trước tiên

```go
// file_system_store_test.go
t.Run("store wins for new players", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer cleanDatabase()

	store := FileSystemPlayerStore{database}

	store.RecordWin("Pepper")

	got := store.GetPlayerScore("Pepper")
	want := 1
	assertScoreEquals(t, got, want)
})
```

## Thử chạy bản kiểm thử

```
=== RUN   TestFileSystemStore/store_wins_for_new_players#01
    --- FAIL: TestFileSystemStore/store_wins_for_new_players#01 (0.00s)
        file_system_store_test.go:86: got 0 want 1
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Chúng ta chỉ cần xử lý kịch bản khi `Find` trả về `nil` vì nó không thể tìm thấy người chơi.

```go
// file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	league := f.GetLeague()
	player := league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		league = append(league, Player{name, 1})
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(league)
}
```

Luồng hoạt động trơn tru (happy path) trông khá ổn vì vậy chúng ta có thể thử sử dụng `Store` mới của mình trong bản kiểm thử tích hợp. Điều này sẽ mang lại cho chúng ta sự tin cậy hơn rằng phần mềm hoạt động và sau đó chúng ta có thể xóa `InMemoryPlayerStore` dư thừa.

Trong `TestRecordingWinsAndRetrievingThem`, hãy thay thế kho lưu trữ cũ:

```go
// server_integration_test.go
database, cleanDatabase := createTempFile(t, "")
defer cleanDatabase()
store := &FileSystemPlayerStore{database}
```

Nếu bạn chạy bản kiểm thử, nó sẽ vượt qua và giờ chúng ta có thể xóa `InMemoryPlayerStore`. `main.go` bây giờ sẽ có vấn đề biên dịch, điều này sẽ thúc đẩy chúng ta sử dụng kho lưu trữ mới trong mã nguồn "thực".

```go
// main.go
package main

import (
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

	store := &FileSystemPlayerStore{db}
	server := NewPlayerServer(store)

	if err := http.ListenAndServe(":5000", server); err != nil {
		log.Fatalf("could not listen on port 5000 %v", err)
	}
}
```

- Chúng ta tạo một tệp cho cơ sở dữ liệu của mình.
- Đối số thứ hai của `os.OpenFile` cho phép bạn xác định các quyền để mở tệp, trong trường hợp của chúng ta `O_RDWR` có nghĩa là chúng ta muốn đọc và ghi *và* `os.O_CREATE` có nghĩa là tạo tệp nếu nó không tồn tại.
- Đối số thứ ba xác định các quyền cho tệp, trong trường hợp của chúng ta, tất cả người dùng có thể đọc và ghi tệp. [(Xem superuser.com để biết lời giải thích chi tiết hơn)](https://superuser.com/questions/295591/what-is-the-meaning-of-chmod-666).

Chạy chương trình bây giờ sẽ lưu trữ dữ liệu bền vững trong một tệp giữa các lần khởi động lại, hoan hô!

## Tái cấu trúc thêm và các mối quan tâm về hiệu suất (performance)

Mỗi khi có ai đó gọi `GetLeague()` hoặc `GetPlayerScore()`, chúng ta lại đọc toàn bộ tệp và phân tích nó thành JSON. Chúng ta không nhất thiết phải làm điều đó vì `FileSystemStore` hoàn toàn chịu trách nhiệm về trạng thái của bảng xếp hạng; nó chỉ cần đọc tệp khi chương trình khởi động và chỉ cần cập nhật tệp khi dữ liệu thay đổi.

Chúng ta có thể tạo một constructor cái mà có thể thực hiện một số việc khởi tạo này cho chúng ta và lưu trữ bảng xếp hạng như một giá trị trong `FileSystemStore` để sử dụng cho việc đọc sau này.

```go
// file_system_store.go
type FileSystemPlayerStore struct {
	database io.ReadWriteSeeker
	league   League
}

func NewFileSystemPlayerStore(database io.ReadWriteSeeker) *FileSystemPlayerStore {
	database.Seek(0, io.SeekStart)
	league, _ := NewLeague(database)
	return &FileSystemPlayerStore{
		database: database,
		league:   league,
	}
}
```

Theo cách này, chúng ta chỉ phải đọc từ đĩa một lần. Giờ đây chúng ta có thể thay thế tất cả các lần gọi lấy bảng xếp hạng từ đĩa trước đó và chỉ cần sử dụng `f.league` để thay thế.

```go
// file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() League {
	return f.league
}

func (f *FileSystemPlayerStore) GetPlayerScore(name string) int {

	player := f.league.Find(name)

	if player != nil {
		return player.Wins
	}

	return 0
}

func (f *FileSystemPlayerStore) RecordWin(name string) {
	player := f.league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		f.league = append(f.league, Player{name, 1})
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(f.league)
}
```

Tuy nhiên, có một vấn đề tiềm ẩn ở đây. `json.NewEncoder` sẽ ghi dữ liệu vào tệp, nhưng điều gì xảy ra nếu số lượng byte mới ghi vào ít hơn số lượng byte đã có sẵn?

Giả sử tệp hiện tại là:
`[{"Name":"Chris","Wins":33},{"Name":"Cleo","Wins":10}]`

Và chúng ta ghi đè bằng:
`[{"Name":"Chris","Wins":34}]` (Giả sử chúng ta xóa Cleo)

Tệp sẽ trông như thế này:
`[{"Name":"Chris","Wins":34}]]` (Có một dấu ngoặc vuông dư thừa ở cuối)

Vì vậy, chúng ta cần đảm bảo tệp được cắt bớt (truncate) trước khi ghi.

```go
// file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	player := f.league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		f.league = append(f.league, Player{name, 1})
	}

	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(f.league)
}
```

Để cắt bớt một tệp, chúng ta có thể sử dụng phương thức `Truncate` từ `*os.File`. Tuy nhiên, interface `io.ReadWriteSeeker` của chúng ta không có phương thức này.

Đây là một sự đánh đổi. Chúng ta có thể thay đổi interface của mình để yêu cầu một thứ gì đó có khả năng `Truncate`, hoặc chúng ta có thể bọc `io.ReadWriteSeeker` trong một struct của riêng mình.

Hãy giữ nó đơn giản và sử dụng một helper để thực hiện việc ghi.

```go
// file_system_store.go
func (f *FileSystemPlayerStore) RecordWin(name string) {
	player := f.league.Find(name)

	if player != nil {
		player.Wins++
	} else {
		f.league = append(f.league, Player{name, 1})
	}

	f.save()
}

func (f *FileSystemPlayerStore) save() {
	f.database.Seek(0, io.SeekStart)
	json.NewEncoder(f.database).Encode(f.league)
}
```

Chúng ta sẽ giải quyết vấn đề `Truncate` sau khi chúng ta thực sự cần nó (ví dụ: khi thực hiện chức năng xóa người chơi). Hiện tại, với việc chỉ ghi tăng thêm trận thắng, kích thước tệp sẽ luôn tăng lên hoặc bằng nhau, nên vấn đề này chưa ảnh hưởng.

### Xử lý tệp rỗng

Nếu bạn bắt đầu với một tệp hoàn toàn rỗng, `json.NewDecoder` sẽ trả về một lỗi vì tệp rỗng không phải là JSON hợp lệ.

## Viết bản kiểm thử trước tiên

```go
// file_system_store_test.go
t.Run("works with an empty file", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, "")
	defer cleanDatabase()

	_, err := NewFileSystemPlayerStore(database)

	if err != nil {
		t.Fatalf("unexpected error creating file system player store, %v", err)
	}
})
```

## Thử chạy bản kiểm thử

Bản kiểm thử sẽ thất bại với lỗi phân tích cú pháp JSON.

## Viết đủ mã nguồn để bản kiểm thử vượt qua

```go
// file_system_store.go
func NewFileSystemPlayerStore(database io.ReadWriteSeeker) (*FileSystemPlayerStore, error) {
	database.Seek(0, io.SeekStart)
	league, err := NewLeague(database)

	if err != nil {
		return nil, fmt.Errorf("problem loading player store from file %v", err)
	}

	return &FileSystemPlayerStore{
		database: database,
		league:   league,
	}, nil
}
```

Chúng ta cần cập nhật `NewLeague` để xử lý tệp rỗng hoặc trả về lỗi cụ thể hơn. Nhưng một cách đơn giản là kiểm tra xem tệp có rỗng hay không trước khi giải mã.

Hoặc, chúng ta có thể làm cho `NewFileSystemPlayerStore` khởi tạo tệp với `[]` nếu nó rỗng.

```go
// file_system_store.go
func NewFileSystemPlayerStore(file io.ReadWriteSeeker) (*FileSystemPlayerStore, error) {
	file.Seek(0, io.SeekStart)

	info, err := file.(*os.File).Stat()
	if err != nil {
		return nil, fmt.Errorf("problem getting file info from file %v", err)
	}

	if info.Size() == 0 {
		file.Write([]byte("[]"))
		file.Seek(0, io.SeekStart)
	}

	league, err := NewLeague(file)
	if err != nil {
		return nil, fmt.Errorf("problem loading player store from file %v", err)
	}

	return &FileSystemPlayerStore{
		database: file,
		league:   league,
	}, nil
}
```

Lưu ý: Việc ép kiểu `file.(*os.File)` có thể gây ra lỗi nếu chúng ta truyền một thứ gì đó không phải là tệp. Trong mã nguồn thực tế, bạn nên kiểm tra điều đó.

## Sắp xếp (Sorting)

Bây giờ chúng ta cần hoàn thành yêu cầu cuối cùng: `/league` nên trả về những người chơi được sắp xếp theo số trận thắng.

## Viết bản kiểm thử trước tiên

```go
// file_system_store_test.go
t.Run("league sorted", func(t *testing.T) {
	database, cleanDatabase := createTempFile(t, `[
		{"Name": "Cleo", "Wins": 10},
		{"Name": "Chris", "Wins": 33}]`)
	defer cleanDatabase()

	store, err := NewFileSystemPlayerStore(database)
	assertNoError(t, err)

	got := store.GetLeague()

	want := League{
		{"Chris", 33},
		{"Cleo", 10},
	}

	assertLeague(t, got, want)
})
```

## Thử chạy bản kiểm thử

Bản kiểm thử thất bại vì thứ tự hiện tại là Cleo rồi đến Chris.

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Go có gói `sort` rất mạnh mẽ. Chúng ta có thể sử dụng `sort.Slice`.

```go
// file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() League {
	sort.Slice(f.league, func(i, j int) bool {
		return f.league[i].Wins > f.league[j].Wins
	})
	return f.league
}
```

`sort.Slice` nhận một lát cắt và một hàm so sánh. Hàm so sánh này trả về `true` nếu phần tử tại `i` nên đứng trước phần tử tại `j`. Ở đây chúng ta muốn sắp xếp giảm dần theo số trận thắng.

Tuy nhiên, `GetLeague` không nên có tác dụng phụ là thay đổi thứ tự của `f.league` mỗi khi được gọi (mặc dù trong trường hợp này nó không gây hại lắm). Một cách tốt hơn là sắp xếp một lần khi tải hoặc khi lưu.

## Tái cấu trúc

Hãy di chuyển logic sắp xếp vào kiểu `League` của chính nó.

```go
// league.go
func (l League) Sort() {
	sort.Slice(l, func(i, j int) bool {
		return l[i].Wins > l[j].Wins
	})
}
```

Và cập nhật `GetLeague`:

```go
// file_system_store.go
func (f *FileSystemPlayerStore) GetLeague() League {
	f.league.Sort()
	return f.league
}
```

## Tóm tắt

Trong chương này, chúng ta đã học về:

- **Làm việc với tệp**. Cách sử dụng `os.OpenFile` và `os.CreateTemp`.
- **Interfaces nâng cao**. Tìm hiểu về `io.ReadSeeker` và `io.ReadWriteSeeker` để điều khiển con trỏ đọc/ghi trong tệp.
- **Tái cấu trúc mã nguồn thông minh**. Tách biệt logic xử lý dữ liệu (`League`) khỏi logic lưu trữ (`FileSystemPlayerStore`).
- **Sắp xếp trong Go**. Sử dụng `sort.Slice` để sắp xếp các lát cắt dữ liệu dựa trên các quy tắc tùy chỉnh.
- **Xử lý tệp rỗng**. Cách đảm bảo ứng dụng không bị lỗi khi bắt đầu với một cơ sở dữ liệu mới.

Việc sử dụng TDD đã giúp chúng ta chuyển đổi từ một hệ thống lưu trữ tạm thời trong bộ nhớ sang một hệ thống lưu trữ bền vững trên đĩa một cách an toàn và có kiểm soát.


