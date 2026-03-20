# OS Exec

**[Bạn có thể tìm thấy toàn bộ source code ở đây](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/os-exec)**

[keith6014](https://www.reddit.com/user/keith6014) đặt câu hỏi trên [reddit](https://www.reddit.com/r/golang/comments/aaz8ji/testdata_and_function_setup_help/)

> Tôi đang gọi chạy một lệnh bằng `os/exec.Command()` thứ sẽ sinh ra dữ liệu XML. Lệnh này sẽ được thực thi trong một hàm có tên `GetData()`.

> Để test `GetData()`, tôi có chuẩn bị một vài dữ liệu testdata do tôi tự tạo.

> Trong file `_test.go` tôi thiết lập một hàm `TestGetData` để gọi tới `GetData()` nhưng nó lại đi chạy lệnh hệ điều hành thông qua thư viện `os.exec`, ngược lại tôi mong nó sử dụng file data thử nghiệm (testdata) của tôi.

> Đâu là cách tối ưu để đạt được điều này? Khi gọi lệnh hàm `GetData`, tôi có nên truyền nhận tham số cờ đại diện cho chế độ "test" (test mode) để nó chọn đọc file text theo tình huống (ví dụ `GetData(mode string)`) không?

Một vài điều bạn nên lưu ý:

- Khi một thứ gì đó trở nên khó để test, nguyên nhân thường xuyên nằm ở việc sự phân công trách nhiệm và tách bạch thành phần (separation of concerns) chưa được thực hiện chuẩn xác.
- Đừng thêm các công tắc chế độ "test modes" vào trong source code của bạn, thay vào đó hãy sử dụng kỹ thuật [Dependency Injection (Tiêm phụ thuộc)](./dependency-injection.md) để bạn có thể mô hình hóa các tệp thành phần hệ thống và xử lý phân tách các ranh giới mạch lạc.

Tôi xin đưa ra giả định xem đoạn code ban đầu trông sẽ như thế nào

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

- Nó sử dụng `exec.Command` để cung cấp cho bạn quyền vận hành thực thi một external command (lệnh bên ngoài) vào một process (tiến trình đang chạy).
- Chúng ta đón hứng bắt lấy luồng kết quả output gán vào `cmd.StdoutPipe`, hàm này trả về cho ta một `io.ReadCloser` (chi tiết này sẽ mang tầm quan trọng về sau).
- Phần còn lại của code ít nhiều được copy dán và tùy chỉnh từ [tài liệu chính thức siêu tuyệt vời](https://golang.org/pkg/os/exec/#example_Cmd_StdoutPipe) của ngôn ngữ .
    - Chúng ta hứng lấy bất kỳ output nào đổ ra từ stdout và gán vào một cấu trúc `io.ReadCloser` và sau đó chúng ta `Start` quá trình chạy script lệnh rồi kiên trì chờ thao tác hệ thống tới khi toàn bộ phần data được đọc ra bằng cách gọi chuỗi lệnh `Wait`. Nằm giữa khoảng trống của hai lần gọi hàm kia, chúng ta decode (giải mã) dữ liệu được gán vào struct `Payload` của mình.

Đây là nội dung được chứa nằm bên trong file `msg.xml`

```xml
<payload>
    <message>Happy New Year!</message>
</payload>
```

Tôi đã viết một bài test đơn giản để minh họa cách nó đánh giá thực chiến

```go
func TestGetData(t *testing.T) {
	got := GetData()
	want := "HAPPY NEW YEAR!"

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

## Đoạn code mang khả năng chạy test (Testable code)

Đoạn code test hiệu quả cần phải giảm sự phụ thuộc trói buộc đa phần (decoupled) và chỉ gánh duy nhất một mục đích. Với cá nhân tôi, hình như có chừng hai băn khoăn chính đáng lưu tâm đối với đoạn code này

1. Thu thập dữ liệu XML thô dạng chưa qua biên dịch
2. Quá trình giải mã chuyển đổi đoạn data XML và vận dụng tầng logic nghiệp vụ (business logic) của cấu trúc xử lý (nhờ vậy gọi chạy `strings.ToUpper` xử lý text trong cái `<message>`)

Phần đầu thực ra chỉ là sao chép y chang cấu trúc dùng mẫu ví dụ cho sẵn trên thư viện chuẩn (standard lib).

Phần việc phía sau kia mới là phân khu chứa chuỗi tính logic vận hành riêng biệt của chúng ta và thông qua cách soi lướt vào code, chúng ta có thể khoanh ra mạch ranh giới "đường giao" (seam) kết nối quy trình tính toán rẽ vào nằm ở đâu; điểm nối đó chính là nơi ta tóm được luồng cấp vào `io.ReadCloser`. Chúng ta có năng lực ứng dụng sự trừu tượng hóa có sẵn đấy nhằm đả phá bẻ gãy phân nhỏ chia bạch ranh những thao tác việc này cũng như thúc giúp đoạn mã hoàn thiện hơn ở năng lực dễ test.

**Vấn đề lớn gặp ở hàm GetData là thứ mạch quy trình tính toán xử lý nghiệp vụ nội tại đang lỡ cắm nối (coupled) với nhóm dòng mã lo thực thi quy trình tìm bốc lấy dữ liệu cấu trúc XML. Biện giải làm cấu hình toàn mảng (design) được thiết chật tối ưu chuẩn điểm hơn ta sẽ yêu cầu đem tách bạch giải trói (decouple) hai tầng ứng dụng này.**

Gói function bộ test bài chuẩn `TestGetData` thiết kế bởi chúng ta vốn vẫn xài chung cho phần tác vụ có chức trách tựa mô-đun móc nối (integration test) kết trạm chạy chéo hai nhóm nghiệp vụ nên rồi chúng ta quyết bảo lưu cất nguyên nó nhằm rào chắc chốt kiểm đảm hệ thống vẫn trôi chảy thực thi chuẩn. 

Ngay đây mời bạn ngó tổng diện qua hình hài chuỗi hệ mã source code sau thiết phân mảnh độc lập hoàn mĩ:

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

Bây giờ khi hàm `GetData` chỉ nhận luồng dữ liệu đầu vào chạy từ một đối tượng `io.Reader`, chúng ta đã làm cho nó sở hữu khả năng test độc lập và nó không còn phải bận tâm việc dữ liệu được lấy ra theo phương thức nào nữa; mọi người có thể tái sử dụng function này với dẫu bất cứ thực thể chức năng gì mang bản chất trả hồi về một đại lượng `io.Reader` (một dạng vốn thuộc thiết cực kỳ phổ quát). Chẳng tỷ dụ minh chứng sát sườn là hệ thống chúng ta có thể thoải mái chuyển qua trích xuất chóp dữ liệu XML nọ trên một đường dẫn URL web thay cho cách lấy gọi mã lệnh từ nền command line thô kệch ngày gốc.

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

Dưới đây chính là một minh chứng phác dựng về dạng bài unit test đặc dọn riêng dành cho hàm `GetData`.

Bằng khâu chuẩn bị phân tách rành rẽ các khối công sự phận bận tâm, đồng thời tích cực tận dụng khả năng khai thác mảng trừu tượng hóa vốn có sẵn bên trong bộ xương ngôn ngữ lập trình Go, cái việc viết cấu trúc bài chạy đi test test bảo trì mạch hệ nghiệp vụ logic (business logic) tối cao quan trọng kia đã trở thành việc nhẹ nhàng như một cơn gió mát.
