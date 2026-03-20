# OS Exec

**[Bạn có thể tìm thấy toàn bộ source code ở đây](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/os-exec)**

[keith6014](https://www.reddit.com/user/keith6014) đặt câu hỏi trên [reddit](https://www.reddit.com/r/golang/comments/aaz8ji/testdata_and_function_setup_help/)

> Tôi đang gọi chạy một lệnh bằng `os/exec.Command()` thứ sẽ sinh ra dữ liệu XML. Lệnh này sẽ được thực thi trong một hàm có tên `GetData()`.

> Để test `GetData()`, tôi có chuẩn bị một vài dữ liệu testdata do tôi tự tạo.

> Trong file `_test.go` tôi thiết lập một hàm `TestGetData` để gọi tới `GetData()` nhưng nó lại đi chạy lệnh hệ điều hành thông qua thư viện `os.exec`, ngược lại tôi mong nó sử dụng file data thử nghiệm (testdata) của tôi.

> Đâu là cách tối ưu để đạt được điều này? Khi gọi hàm `GetData`, tôi có nên truyền một tham số cờ đại diện cho chế độ "test" (test mode) để nó chọn đọc file theo tình huống (ví dụ `GetData(mode string)`) không?

Một vài điều bạn nên lưu ý:

- Khi một thứ gì đó trở nên khó để test, nguyên nhân thường nằm ở việc phân tách trách nhiệm (separation of concerns) chưa được thực hiện đúng đắn.
- Đừng thêm các "test mode" vào trong source code của bạn, thay vào đó hãy sử dụng kỹ thuật [Dependency Injection (Tiêm phụ thuộc)](./dependency-injection.md) để bạn có thể mô hình hóa các thành phần phụ thuộc và phân tách rõ ràng các ranh giới trong hệ thống.

Tôi xin đưa ra giả định về đoạn code ban đầu trông sẽ như thế nào

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData() string {
	cmd := exec.Command("cat", "msg.xml")

	out, _ := cmd.StdoutPipe()
	var payload Payload
	decoder := xml.NewDecoder(out)

	// 3 thao tác này có thể trả về errors nhưng tôi bỏ qua cho ngắn gọn
	cmd.Start()
	decoder.Decode(&payload)
	cmd.Wait()

	return strings.ToUpper(payload.Message)
}
```

- Nó sử dụng `exec.Command` để thực thi một external command (lệnh bên ngoài) trong một process (tiến trình).
- Chúng ta bắt luồng kết quả output thông qua `cmd.StdoutPipe`, hàm này trả về một `io.ReadCloser` (chi tiết này sẽ trở nên quan trọng ở phần sau).
- Phần còn lại của code ít nhiều được lấy từ [tài liệu chính thức rất tuyệt vời](https://golang.org/pkg/os/exec/#example_Cmd_StdoutPipe) của Go.
    - Chúng ta lấy output từ stdout vào một `io.ReadCloser`, sau đó `Start` lệnh và chờ cho đến khi toàn bộ dữ liệu được đọc xong bằng cách gọi `Wait`. Ở giữa hai lệnh đó, chúng ta decode (giải mã) dữ liệu vào struct `Payload`.

Đây là nội dung bên trong file `msg.xml`

```xml
<payload>
    <message>Happy New Year!</message>
</payload>
```

Tôi đã viết một bài test đơn giản để minh họa cách nó hoạt động

```go
func TestGetData(t *testing.T) {
	got := GetData()
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

## Code có khả năng test được (Testable code)

Code có khả năng test tốt thường là code đã được tách rời các phụ thuộc (decoupled) và mỗi phần chỉ đảm nhận một trách nhiệm duy nhất. Theo tôi, có hai mối quan tâm chính trong đoạn code này:

1. Thu thập dữ liệu XML thô
2. Giải mã dữ liệu XML đó và áp dụng business logic (logic nghiệp vụ) lên nó (trong trường hợp này là gọi `strings.ToUpper` trên nội dung của `<message>`)

Phần đầu tiên thực chất chỉ là sao chép mẫu ví dụ có sẵn từ thư viện chuẩn (standard library).

Phần thứ hai mới là nơi chứa logic riêng của chúng ta. Khi nhìn vào code, chúng ta có thể nhận ra điểm nối (seam) nằm ở đâu: đó chính là `io.ReadCloser`. Chúng ta có thể tận dụng abstraction (tính trừu tượng) có sẵn này để tách hai phần ra và giúp code dễ test hơn.

**Vấn đề chính của `GetData` là phần business logic đang bị gắn chặt (coupled) với phần lấy dữ liệu XML. Để có một thiết kế (design) tốt hơn, chúng ta cần tách rời (decouple) hai phần này.**

Bài test `TestGetData` mà chúng ta đã viết vẫn có thể dùng như một integration test (test tích hợp) vì nó kiểm tra cả hai phần hoạt động cùng nhau, nên chúng ta sẽ giữ nguyên nó để đảm bảo hệ thống vẫn hoạt động đúng.

Dưới đây là code sau khi được tái cấu trúc (refactor) để tách biệt các phần:

```go
type Payload struct {
	Message string `xml:"message"`
}

func GetData(data io.Reader) string {
	var payload Payload
	xml.NewDecoder(data).Decode(&payload)
	return strings.ToUpper(payload.Message)
}

func getXMLFromCommand() io.Reader {
	cmd := exec.Command("cat", "msg.xml")
	out, _ := cmd.StdoutPipe()

	cmd.Start()
	data, _ := io.ReadAll(out)
	cmd.Wait()

	return bytes.NewReader(data)
}

func TestGetDataIntegration(t *testing.T) {
	got := GetData(getXMLFromCommand())
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

Bây giờ `GetData` chỉ nhận đầu vào là một `io.Reader`, nên chúng ta đã giúp nó có khả năng test độc lập. Nó không còn cần quan tâm dữ liệu được lấy từ đâu nữa. Mọi người có thể tái sử dụng hàm này với bất kỳ thứ gì trả về một `io.Reader` (đây là một interface rất phổ biến trong Go). Ví dụ, chúng ta có thể dễ dàng chuyển sang lấy dữ liệu XML từ một URL thay vì từ dòng lệnh command line.

```go
func TestGetData(t *testing.T) {
	input := strings.NewReader(`
<payload>
    <message>Cats are the best animal</message>
</payload>`)

	got := GetData(input)
	want := "CATS ARE THE BEST ANIMAL"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}

```

Đây là một ví dụ về unit test cho hàm `GetData`.

Bằng cách phân tách rõ ràng các mối quan tâm (separation of concerns) và tận dụng các abstraction có sẵn trong Go, việc viết test cho phần business logic quan trọng trở nên đơn giản và dễ dàng.
