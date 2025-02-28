# Hello, World

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/hello-world)**

Thông thường, chương trình đầu tiên mà bạn viết khi học một ngôn ngữ mới sẽ là [Hello, World](https://en.m.wikipedia.org/wiki/%22Hello,_World!%22_program).

-   Tạo một thư mục bất kỳ.
-   Thêm một tệp mới có tên `hello.go` với nội dung như sau:

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, world")
}
```

Để chạy, gõ vào terminal lệnh sau `go run hello.go`.

## Cách nó hoạt động

Khi viết một chương trình Go, bạn cần khai báo package `main` cùng với một hàm (func) `main` bên trong nó. Go tổ chức mã nguồn thành các package để nhóm các chức năng liên quan lại với nhau.

Từ khoá `func` dùng để định nghĩa một hàm với tên và phần thân của nó. Trong Go, chương trình bắt đầu chạy từ hàm `main()`.

Với lệnh `import "fmt"` sẽ dùng để nhập gói `fmt`, trong đó chứa hàm `Println` có tác dụng in nội dung ra màn hình console.

## Cách test (test)

Làm thế nào để test chương trình này?

Một nguyên tắc quan trọng là cần tách biệt phần code xử lý logic khỏi các hiệu ứng phụ (side-effects)

-   `fmt.Println` là một hiệu ứng phụ vì nó in ra màn hình.
-   Chuỗi truyền vào ("Hello, World!") là phần logic chính cần test.

Để dễ test hơn, chúng ta có thể viết một hàm riêng để xử lý phần logic, sau đó gọi hàm này trong `main()`.

```go
package main

import "fmt"

func Hello() string {
	return "Hello, world"
}

func main() {
	fmt.Println(Hello())
}
```

Chúng ta đã tạo một hàm mới bằng từ khoá `func`, nhưng lần này có thêm từ khoá `string` khi định nghĩa. Điều này có nghĩa là hàm sẽ trả về một giá trị `string`.

Bây giờ, hãy tạo một tệp mới có tên `hello_test.go` để viết test cho hàm `Hello`

```go
package main

import "testing"

func TestHello(t *testing.T) {
	got := Hello()
	want := "Hello, world"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

## Go modules?

Bước tiếp theo là chạy test. Hãy nhập lệnh `go test` trong terminal.

Nếu test chạy thành công, có thể bạn đang sử dụng phiên bản Go cũ hơn. Tuy nhiên, nếu bạn đang dùng Go 1.16 trở lên, có khả năng test sẽ không chạy mà thay vào đó, bạn sẽ thấy lỗi như sau:

```shell
$ go test
go: cannot find main module; see 'go help modules'
```

Vấn đề ở đây do `modules` chưa được khởi tạo. Nhập lệnh sau vào terminal `go mod init example.com/hello`. Lúc này một tệp mới sẽ được tạo với nội dung như sau:

```
module example.com/hello

go 1.16
```

Tệp này cung cấp cho `go` tools các thông tin quan trọng về mã nguồn của bạn. Nếu bạn dự định phân phối ứng dụng, tệp này sẽ chứa thông tin về nơi tải mã nguồn cũng như các thư viện phụ thuộc.

Tên module, example.com/hello, thường đại diện cho một URL nơi module có thể được tìm thấy và tải xuống. Để đảm bảo khả năng tương thích với các công cụ mà chúng ta sẽ sử dụng sau này, hãy đảm bảo rằng tên module có chứa dấu chấm (.), chẳng hạn như .com trong example.com/hello

Hiện tại, module file này chỉ cần ở mức tối thiểu và bạn có thể để nguyên như vậy. Nếu muốn tìm hiểu thêm về modules, bạn có thể [tham khảo tài liệu của Golang](https://golang.org/doc/modules/gomod-ref).

Bây giờ, bạn có thể tiếp tục chạy test và học Go, ngay cả khi đang sử dụng Go 1.16 trở lên.

Lưu ý cho các chương sau

-   Khi làm việc trong các thư mục mới, bạn cần chạy lệnh `go mod init SOMENAME` trước khi thực hiện các lệnh như `go test` hoặc `go build`.

## Tiếp tục test

Chạy lệnh `go test` trong terminal. Lúc này nó sẽ chạy thành công. Để kiểm tra, hãy thử thay đổi giá trị của biến `want`.

Bạn có thể thấy rằng không cần phải lựa chọn giữa nhiều framework test khác nhau hay tìm cách cài đặt chúng. Mọi thứ cần thiết đều được tích hợp sẵn trong ngôn ngữ Go, và cú pháp test cũng giống với phần còn lại của mã bạn viết.

### Writing tests

Viết một test trong Go cũng giống như viết một hàm bình thường, nhưng cần tuân theo một số quy tắc sau:

-   Tệp chứa test phải có tên theo định dạng `xxx_test.go`
-   Hàm test phải bắt đầu bằng từ `Test`
-   Hàm test chỉ nhận một tham số duy nhất là `t *testing.T`
-   Để sử dụng `*testing.T`, bạn chỉ cần `import "testing"`, tương tự như cách bạn đã import `fmt`.

Hiện tại, bạn chỉ cần hiểu rằng biến `t` có kiểu `*testing.T` chính là "cầu nối" giúp bạn tương tác với framework test. Chẳng hạn, có thể gọi `t.Fail()` để làm cho test thất bại.

Chúng ta vừa đề cập đến một số khái niệm mới:

#### `if`

Câu lệnh `if` trong Go hoạt động tương tự như đa số ngôn ngữ lập trình khác.

#### Declaring variables

Chúng ta có thể khai báo biến bằng cú pháp `varName := value`. Cách này giúp tái sử dụng giá trị trong test, làm cho mã dễ đọc hơn.

#### `t.Errorf`

Chúng ta gọi _phương thức_ `Errorf` trên `t`, phương thức này sẽ in ra một thông báo lỗi và làm cho test thất bại. Chữ `f` trong Errorf là viết tắt của format, cho phép chúng ta chèn giá trị vào chuỗi theo định dạng với các ký tự giữ chỗ như `%q`.

Khi test thất bại, thông báo lỗi sẽ giúp bạn hiểu rõ nguyên nhân. Bạn có thể đọc thêm về các ký tự giữ chỗ trong [tài liệu fmt](https://pkg.go.dev/fmt#hdr-Printing). Trong test, `%q` rất hữu ích vì nó tự động bọc giá trị trong dấu nháy kép.

Chúng ta sẽ tìm hiểu sự khác biệt giữa _hàm_ (function) và _phương thức_ (method) trong các phần sau.

### Tài liệu của Go

Một điểm hữu ích khác của Go là hệ thống tài liệu. Chúng ta vừa xem tài liệu của package fmt trên trang web chính thức, nhưng Go cũng cung cấp cách để truy cập tài liệu nhanh chóng ngay cả khi không có internet.

Go có một công cụ tích hợp sẵn gọi là `doc`, cho phép bạn xem tài liệu của bất kỳ package nào đã cài đặt trên hệ thống hoặc trong module mà bạn đang làm việc. Để xem lại tài liệu về các ký tự định dạng (Printing verbs), bạn có thể sử dụng lệnh sau:

```
$ go doc fmt
package fmt // import "fmt"

Package fmt implements formatted I/O with functions analogous to C's printf and
scanf. The format 'verbs' are derived from C's but are simpler.

# Printing

The verbs:

General:

    %v	the value in a default format
    	when printing structs, the plus flag (%+v) adds field names
    %#v	a Go-syntax representation of the value
    %T	a Go-syntax representation of the type of the value
    %%	a literal percent sign; consumes no value
...
```

Công cụ thứ hai của Go để xem tài liệu là lệnh pkgsite, đây cũng là công cụ vận hành trang web tài liệu chính thức của Go. Bạn có thể cài đặt pkgsite bằng lệnh `go install golang.org/x/pkgsite/cmd/pkgsite@latest`, sau đó chạy với `pkgsite -open .`. Lệnh go install sẽ tải mã nguồn từ kho lưu trữ, sau đó biên dịch thành một tệp thực thi. Theo mặc định, tệp này sẽ nằm trong: `$HOME/go/bin` cho Linux, macOS, và `%USERPROFILE%\go\bin` cho Windows. Nếu bạn chưa thêm các đường dẫn này vào biến môi trường $PATH, bạn nên làm vậy để có thể chạy các lệnh Go một cách thuận tiện hơn.

Hầu hết thư viện tiêu chuẩn của Go đều có tài liệu rất chi tiết kèm theo ví dụ. Bạn có thể mở [http://localhost:8080/testing](http://localhost:8080/testing) để khám phá thêm các tài nguyên hữu ích.

### Hello, YOU

Bây giờ chúng ta đã có 1 chương trình test, từ đây có thể tiếp tục cải tiến phần mềm một cách an toàn.

Ở ví dụ trước, chúng ta viết test _sau_ khi đã viết mã để bạn có thể hiểu cách viết một test và khai báo một hàm. Từ thời điểm này, chúng ta sẽ _viết test trước_.

Yêu cầu tiếp theo là cho phép chỉ định người nhận lời chào.

Hãy bắt đầu bằng cách viết một test để xác định các yêu cầu này. Đây là nguyên tắc cơ bản của phát triển phần mềm theo hướng test (TDD), giúp đảm bảo test thực sự kiểm tra đúng những gì chúng ta mong muốn.

Khi bạn viết test một cách hồi tố (tức là sau khi đã viết mã), có nguy cơ test vẫn chạy đúng ngay cả khi mã không hoạt động như mong đợi.

```go
package main

import "testing"

func TestHello(t *testing.T) {
	got := Hello("Chris")
	want := "Hello, Chris"

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

Bây giờ hãy chạy lệnh `go test`, bạn sẽ thấy một lỗi biên dịch

```text
./hello_test.go:6:18: too many arguments in call to Hello
    have (string)
    want ()
```

Khi làm việc với một ngôn ngữ có kiểu dữ liệu tĩnh như Go, điều quan trọng là _chú ý đến trình biên dịch_ (compiler). Trình biên dịch hiểu cách các phần của mã nên kết hợp với nhau để hoạt động, vì vậy bạn không cần phải tự kiểm tra mọi thứ

Trong trường hợp này, trình biên dịch đang cho bạn biết cần làm gì tiếp theo. Chúng ta phải thay đổi hàm `Hello` để nhận một tham số.

Hãy chỉnh sửa hàm `Hello` để chấp nhận một đối số kiểu string.

```go
func Hello(name string) string {
	return "Hello, world"
}
```

Nếu bạn chạy lại test, chương trình `hello.go` sẽ không thể biên dịch vì bạn chưa truyền đối số. Hãy truyền vào chuỗi "world" để chương trình có thể biên dịch được.

```go
func main() {
	fmt.Println(Hello("world"))
}
```

Bây giờ khi chạy bài test, bạn sẽ thấy một thông báo lỗi.

```text
hello_test.go:10: got 'Hello, world' want 'Hello, Chris''
```

Cuối cùng, chương trình đã có thể biên dịch, nhưng nó chưa đáp ứng được yêu cầu test.

Hãy chỉnh sửa hàm Hello để sử dụng đối số name và nối chuỗi với `Hello,`.

```go
func Hello(name string) string {
	return "Hello, " + name
}
```

Sau khi chạy test, chúng sẽ thành công (pass). Thông thường, trong chu trình TDD, bước tiếp theo là _tinh chỉnh mã nguồn_ (refactor).

### Lưu ý về kiểm soát phiên bản (source control)

Tại thời điểm này, nếu bạn đang sử dụng hệ thống kiểm soát phiên bản \(và bạn nên làm vậy!\), tôi khuyên bạn nên thực hiện `commit` mã nguồn hiện tại. Chúng ta đã có một chương trình hoạt động được và có test đi kèm.

Tuy nhiên, tôi _không_ khuyên bạn nên đẩy (push) lên nhánh main ngay, vì chúng ta sẽ thực hiện bước tinh chỉnh mã nguồn (refactor) tiếp theo. Việc commit ngay lúc này giúp bạn có một điểm khôi phục an toàn. Nếu chẳng may gặp lỗi trong quá trình refactor, bạn luôn có thể quay lại phiên bản đang hoạt động.

Dù không có quá nhiều thứ cần tối ưu hóa ở đây, nhưng chúng ta có thể giới thiệu một tính năng mới của Go: _hằng số_ (constants).

### Hằng số (Constants)

Hằng số được định nghĩa như sau

```go
const englishHelloPrefix = "Hello, "
```

Bây giờ chúng ta sẽ refactor code

```go
const englishHelloPrefix = "Hello, "

func Hello(name string) string {
	return englishHelloPrefix + name
}
```

Sau khi refactor, chạy lại các tests để đảm bảo chúng vẫn hoạt động như cũ.

Bạn nên cân nhắc sử dụng hằng số (constants) để làm rõ ý nghĩa của các giá trị trong mã nguồn và đôi khi giúp cải thiện hiệu suất.

## Hello, world... again

Yêu cầu tiếp theo là khi hàm của chúng ta được gọi với một chuỗi rỗng, nó sẽ mặc định in ra "Hello, World" thay vì "Hello, ".

Hãy bắt đầu bằng cách viết một bài test (test) mới và để nó thất bại trước.

```go
func TestHello(t *testing.T) {
	t.Run("saying hello to people", func(t *testing.T) {
		got := Hello("Chris")
		want := "Hello, Chris"

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})
	t.Run("say 'Hello, World' when an empty string is supplied", func(t *testing.T) {
		got := Hello("")
		want := "Hello, World"

		if got != want {
			t.Errorf("got %q want %q", got, want)
		}
	})
}
```

Ở đây, chúng ta đang giới thiệu một công cụ khác trong bộ test: subtests. Đôi khi, việc nhóm các bài test theo một chủ đề chung và sau đó chia nhỏ thành các tình huống cụ thể sẽ rất hữu ích.

Lợi ích của cách tiếp cận này là bạn có thể thiết lập một phần mã chung để sử dụng trong các bài test khác.

Bây giờ, khi bài test của chúng ta đang thất bại, hãy sửa mã bằng cách sử dụng câu lệnh `if`.

```go
const englishHelloPrefix = "Hello, "

func Hello(name string) string {
	if name == "" {
		name = "World"
	}
	return englishHelloPrefix + name
}
```

Khi chạy lại các bài test, chúng ta sẽ thấy rằng mã đã đáp ứng yêu cầu mới mà không vô tình làm hỏng các chức năng khác.

Điều quan trọng là các bài test cần _phải rõ ràng_ để thể hiện chính xác những gì mã của bạn cần làm. Tuy nhiên, chúng ta đang có một số đoạn mã lặp lại khi kiểm tra xem thông điệp có đúng như mong đợi hay không.

Refactoring (tái cấu trúc mã) _không chỉ_ áp dụng cho production code!

Bây giờ, khi tất cả bài test đã chạy thành công, chúng ta có thể (và nên) tái cấu trúc mã test để làm cho nó gọn gàng hơn.

```go
func TestHello(t *testing.T) {
	t.Run("saying hello to people", func(t *testing.T) {
		got := Hello("Chris")
		want := "Hello, Chris"
		assertCorrectMessage(t, got, want)
	})

	t.Run("empty string defaults to 'world'", func(t *testing.T) {
		got := Hello("")
		want := "Hello, World"
		assertCorrectMessage(t, got, want)
	})

}

func assertCorrectMessage(t testing.TB, got, want string) {
	t.Helper()
	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

Chúng ta đã làm gì ở đây?

Chúng ta đã tái cấu trúc phần kiểm tra kết quả thành một hàm riêng. Điều này giúp giảm sự trùng lặp và cải thiện khả năng đọc của mã test.

Một số điểm cần lưu ý:

-   Chúng ta truyền `t *testing.T` vào hàm kiểm tra để có thể gọi t.Fail() khi cần thiết.
-   Đối với các hàm hỗ trợ test, nên dùng `testing.TB` thay vì `*testing.T`. `testing.TB` là một interface chung cho cả `*testing.T` (test) và `*testing.B` (benchmark), giúp bạn có thể sử dụng cùng một hàm hỗ trợ trong cả test và đo hiệu suất.
-   `t.Helper()` giúp báo cho bộ test rằng đây là một hàm hỗ trợ. Nhờ đó, nếu bài test thất bại, Go sẽ báo lỗi ở dòng gọi hàm thay vì bên trong hàm hỗ trợ, giúp dễ dàng tìm ra lỗi hơn.
-   Nếu bạn chưa hiểu, hãy thử comment (`//`) dòng `t.Helper()`, sau đó thay đổi để làm bài test thất bại để so sánh kết quả.

Khi có nhiều tham số cùng kiểu dữ liệu (ví dụ: hai chuỗi string), bạn có thể viết gọn thay vì `func testHelper(got string, want string)`, có thể dùng `func testHelper(got, want string)`.

### Quay lại với source control

Bây giờ, khi chúng ta đã hài lòng với mã nguồn, nên sửa đổi (amend) commit trước đó thay vì tạo một commit mới. Điều này giúp giữ lịch sử sạch sẽ, chỉ lưu lại phiên bản tốt nhất của mã nguồn kèm theo bài test hoàn chỉnh.

### Sự kỷ luật

Hãy cùng xem lại quy trình phát triển theo hướng test (TDD):

-   Viết một bài test (test)
-   Làm cho trình biên dịch không báo lỗi
-   Chạy test, xác nhận rằng nó thất bại và kiểm tra xem thông báo lỗi có dễ hiểu không
-   Viết đủ mã nguồn để làm bài test vượt qua
-   Refactor

Bề ngoài, quy trình này có vẻ tốn công, nhưng tuân thủ chu trình phản hồi (feedback loop) này rất quan trọng.

-   Đảm bảo bài test có ý nghĩa, không chỉ kiểm tra đúng chức năng mà còn giúp thiết kế phần mềm tốt hơn.
-   Nhìn thấy bài test thất bại là bước quan trọng, vì nó giúp xác định rõ lỗi, tránh tình trạng test kém chất lượng với thông báo mơ hồ.
-   test nhanh chóng, kết hợp với công cụ hỗ trợ giúp bạn duy trì trạng thái flow khi viết mã.

Ngược lại, không viết test đồng nghĩa với việc bạn phải kiểm tra thủ công bằng cách chạy phần mềm liên tục, làm gián đoạn luồng công việc và không tiết kiệm thời gian về lâu dài.

## Tiếp tục nào! Thêm yêu cầu mới

Chà, chúng ta có thêm yêu cầu mới đây. Bây giờ, chúng ta cần hỗ trợ một tham số thứ hai, chỉ định ngôn ngữ của lời chào. Nếu ngôn ngữ được truyền vào không được nhận diện, hãy mặc định sử dụng tiếng Anh.

Chúng ta nên tự tin rằng có thể dễ dàng áp dụng TDD để phát triển tính năng này!

Hãy viết một bài test cho trường hợp người dùng truyền vào tiếng Tây Ban Nha và thêm nó vào bộ test hiện có.

```go
	t.Run("in Spanish", func(t *testing.T) {
		got := Hello("Elodie", "Spanish")
		want := "Hola, Elodie"
		assertCorrectMessage(t, got, want)
	})
```

Nhớ đừng gian lận! _Viết test trước_. Khi bạn cố gắng chạy test, trình biên dịch _sẽ báo lỗi_ vì bạn đang gọi hàm `Hello` với hai đối số thay vì một.

```text
./hello_test.go:27:19: too many arguments in call to Hello
    have (string, string)
    want (string)
```

Sửa lỗi biên dịch bằng cách thêm một đối số kiểu string nữa vào hàm `Hello`.

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}
	return englishHelloPrefix + name
}
```

Khi bạn chạy lại bài kiểm tra, nó sẽ báo lỗi vì các bài kiểm tra khác và trong `hello.go` chưa truyền đủ đối số cho hàm `Hello`.

```text
./hello.go:15:19: not enough arguments in call to Hello
    have (string)
    want (string, string)
```

Hãy sửa lỗi bằng cách truyền vào chuỗi rỗng (`""`). Bây giờ, tất cả các bài kiểm tra của bạn sẽ biên dịch thành công và chạy đúng, ngoại trừ kịch bản mới mà chúng ta vừa thêm vào.

```text
hello_test.go:29: got 'Hello, Elodie' want 'Hola, Elodie'
```

Chúng ta có thể sử dụng câu lệnh `if` để kiểm tra xem ngôn ngữ có phải là “Spanish” hay không, và nếu đúng, thì thay đổi thông điệp phù hợp.

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	if language == "Spanish" {
		return "Hola, " + name
	}
	return englishHelloPrefix + name
}
```

Các bài test của bạn bây giờ sẽ vượt qua.

Đây là lúc để _refactor_. Bạn có thể nhận thấy một số vấn đề trong mã, chẳng hạn như các chuỗi "ma thuật" (magic strings) xuất hiện lặp lại. Hãy thử cải tiến nó bằng cách thay thế các giá trị cố định này bằng hằng số hoặc một cấu trúc dễ quản lý hơn.

```go
	const spanish = "Spanish"
	const englishHelloPrefix = "Hello, "
	const spanishHelloPrefix = "Hola, "

	func Hello(name string, language string) string {
		if name == "" {
			name = "World"
		}

		if language == spanish {
			return spanishHelloPrefix + name
		}
		return englishHelloPrefix + name
	}
```

### Tiếng Pháp

-   Viết một bài test xác nhận rằng nếu bạn truyền vào `"French"`, bạn sẽ nhận được `"Bonjour, "`.
-   Chạy bài test và kiểm tra xem nó thất bại với thông báo lỗi dễ đọc.
-   Thực hiện thay đổi nhỏ nhất có thể trong mã để làm cho bài test vượt qua.

Bạn có thể đã viết một đoạn mã trông tương tự như thế này.

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	if language == spanish {
		return spanishHelloPrefix + name
	}
	if language == french {
		return frenchHelloPrefix + name
	}
	return englishHelloPrefix + name
}
```

## `switch`

Khi bạn có nhiều câu lệnh `if` để kiểm tra một giá trị cụ thể, việc sử dụng `switch` sẽ giúp mã của bạn dễ đọc hơn và dễ mở rộng nếu sau này bạn muốn hỗ trợ thêm nhiều ngôn ngữ.

Chúng ta có thể refactor mã bằng cách sử dụng switch để làm cho nó gọn gàng và dễ bảo trì hơn.

```go
func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	prefix := englishHelloPrefix

	switch language {
	case spanish:
		prefix = spanishHelloPrefix
	case french:
		prefix = frenchHelloPrefix
	}

	return prefix + name
}
```

Viết một bài test mới để thêm một lời chào bằng ngôn ngữ bạn chọn. Bạn sẽ thấy việc mở rộng hàm của chúng ta trở nên đơn giản như thế nào!

### Refactor...lần...cuối?

Bạn có thể thấy rằng hàm của chúng ta đang trở nên hơi dài. Một cách đơn giản để tối ưu lại là tách một số chức năng ra thành một hàm khác.

```go

const (
	spanish = "Spanish"
	french  = "French"

	englishHelloPrefix = "Hello, "
	spanishHelloPrefix = "Hola, "
	frenchHelloPrefix  = "Bonjour, "
)

func Hello(name string, language string) string {
	if name == "" {
		name = "World"
	}

	return greetingPrefix(language) + name
}

func greetingPrefix(language string) (prefix string) {
	switch language {
	case french:
		prefix = frenchHelloPrefix
	case spanish:
		prefix = spanishHelloPrefix
	default:
		prefix = englishHelloPrefix
	}
	return
}
```

Một số khái niệm mới:

-   Trong phần khai báo hàm, chúng ta đã sử dụng _giá trị trả về có tên_ `(prefix string)`.
-   Điều này tạo ra một biến có tên là `prefix` ngay trong hàm `prefix`.
    -   Biến này sẽ được gán giá trị mặc định của kiểu dữ liệu tương ứng (ví dụ: `int` mặc định là 0, còn `string` mặc định là `""`).
        -   Bạn có thể trả về giá trị của `prefix` bằng cách chỉ gọi `return` mà không cần viết `return prefix`
    -   Khi xem tài liệu Go Doc, giá trị trả về có tên giúp làm rõ ý nghĩa của hàm hơn.
-   `default` trong switch case sẽ được thực thi nếu không có trường hợp `case` nào phù hợp.
-   Tên hàm bắt đầu bằng chữ cái viết thường. Trong Go:
    -   Các hàm public (có thể được gọi từ bên ngoài) bắt đầu bằng chữ cái viết hoa
    -   Các hàm private (chỉ có thể sử dụng trong cùng một package) bắt đầu bằng chữ cái viết thường.
    -   Vì đây là một phần bên trong thuật toán, không cần lộ ra ngoài, nên chúng ta đặt tên hàm ở dạng private.
-   Có thể nhóm các hằng số (const) lại với nhau trong một khối thay vì khai báo từng dòng riêng lẻ. Để giúp mã nguồn dễ đọc hơn, nên có một dòng trống giữa các nhóm hằng số có liên quan.

## Tổng kết

Ai mà ngờ được rằng một chương trình đơn giản như `Hello, world`lại có thể giúp ta học được nhiều thứ đến vậy!

### Một số cú pháp của Go mà bạn đã tìm hiểu

-   Viết tests
-   Khai báo hàm, sử dụng tham số và kiểu trả về
-   `if`, `const` và `switch`
-   Khai báo biến và hằng số

### Quy trình TDD và tầm _quan trọng_ của từng bước

-   Viết một bài test thất bại và quan sát lỗi, đảm bảo rằng bài test thực sự phản ánh đúng yêu cầu của bài toán. Điều này cũng giúp ta thấy được mô tả lỗi có dễ hiểu hay không.
-   Viết ít code nhất có thể để bài test pass, từ đó đảm bảo rằng phần mềm của ta hoạt động đúng.
-   Sau đó mới refactor, giúp cải thiện chất lượng mã nguồn mà vẫn được bảo vệ bởi các bài test.

Chúng ta đã đi từ một hàm đơn giản `Hello()` đến `Hello("name")`, rồi phát triển thành `Hello("name", "French")` với những bước nhỏ, dễ hiểu.

Tất nhiên, đây chỉ là một bài tập đơn giản so với phần mềm thực tế, nhưng các nguyên tắc của TDD vẫn luôn có giá trị. Việc luyện tập TDD thường xuyên giúp bạn rèn luyện khả năng chia nhỏ bài toán thành các phần có thể test, giúp việc viết phần mềm dễ dàng hơn rất nhiều.
