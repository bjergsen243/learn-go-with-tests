# Mở rộng Acceptance Tests — và giới thiệu gRPC

Chương này là phần tiếp theo của chương [Giới thiệu về Acceptance Tests](https://quii.gitbook.io/learn-go-with-tests/testing-fundamentals/intro-to-acceptance-tests). Bạn có thể tìm thấy [mã nguồn hoàn chỉnh cho chương này trên GitHub](https://github.com/quii/go-specs-greet).

Acceptance test cực kỳ quan trọng. Chúng ảnh hưởng trực tiếp đến khả năng phát triển hệ thống lâu dài với chi phí bảo trì hợp lý.

Chúng cũng rất hữu ích khi làm việc với legacy code. Khi gặp codebase thiếu test, đừng vội refactor. Thay vào đó, viết vài acceptance test làm mạng lưới an toàn — giúp bạn thay đổi cấu trúc bên trong mà không ảnh hưởng hành vi bên ngoài. Acceptance test không quan tâm đến code bên trong, nên rất phù hợp cho tình huống này.

Sau khi đọc xong chương này, bạn sẽ hiểu rằng acceptance tests không chỉ hữu ích cho việc xác minh tính đúng đắn, mà còn có thể hướng dẫn quá trình phát triển bằng cách giúp chúng ta thay đổi hệ thống một cách có chủ đích và có phương pháp, giảm bớt công sức lãng phí.

## Tài liệu nền tảng

Cảm hứng cho chương này đến từ nhiều năm kinh nghiệm làm việc với acceptance tests. Hai video mà tôi khuyên bạn nên xem:

- Dave Farley - [How to write acceptance tests](https://www.youtube.com/watch?v=JDD5EEJgpHU)
- Nat Pryce - [E2E functional tests that can run in milliseconds](https://www.youtube.com/watch?v=Fk4rCn4YLLU)

"Growing Object Oriented Software" (GOOS) là một cuốn sách vô cùng quan trọng đối với nhiều kỹ sư phần mềm, trong đó có tôi. Đây là cách tiếp cận mà tôi luôn hướng dẫn các đồng nghiệp của mình.

- [GOOS](http://www.growing-object-oriented-software.com) - Tác giả Nat Pryce & Steve Freeman

Cuối cùng, tôi và [Riya Dattani](https://twitter.com/dattaniriya) đã thảo luận về chủ đề này trong bối cảnh BDD (Behavior Driven Development - Phát triển Hướng Hành vi) qua buổi nói chuyện [Acceptance tests, BDD and Go](https://www.youtube.com/watch?v=ZMWJCk_0WrY).

## Tóm tắt

Chúng ta đang nói về black-box testing — xác minh hệ thống hoạt động đúng từ **góc nhìn nghiệp vụ**. Các test này không truy cập vào chi tiết bên trong. Chúng chỉ quan tâm hệ thống làm **gì**, không phải **bằng cách nào**.

## Phân tích các acceptance tests bị hỏng cấu trúc

Qua nhiều năm, tôi đã làm việc với rất nhiều công ty và đội ngũ phát triển. Tất cả đều nhận thấy nhu cầu cần có acceptance tests - một hình thức nào đó để kiểm tra xem hệ thống có hoạt động đúng hay không từ góc nhìn người dùng. Tuy nhiên, vấn đề truyền thống (và gần như không tránh khỏi) là các bài acceptance tests thường trở thành gánh nặng cho đội phát triển:

- Mất nhiều thời gian để chạy
- Dễ bị hỏng (brittle)
- Không ổn định (flaky)
- Tốn kém để bảo trì, khiến việc thay đổi phần mềm trở nên khó khăn hơn thay vì dễ dàng hơn
- Chỉ chạy được trong môi trường chuyên dụng, gây ra vòng phản hồi chậm và độ tin cậy thấp

Giả sử bạn muốn viết một acceptance test cho website của mình. Bạn dùng một headless browser (ví dụ [Selenium](https://www.selenium.dev)) để mô phỏng người dùng nhấp chuột vào các chức năng nhằm xác minh website hoạt động đúng.

Theo thời gian, phần HTML/Markup thay đổi liên tục khi có thêm chức năng mới, các lập trình viên thay đổi cấu trúc markup - ví dụ dùng `<article>` thay vì `<section>`.

Đây chỉ là những thay đổi nhỏ mà người dùng có thể không nhận ra, nhưng bạn lại phải tốn công cập nhật toàn bộ acceptance tests.

### Liên kết chặt (Tight-coupling)

Hãy suy nghĩ xem điều gì nên kích hoạt việc cập nhật acceptance tests:

- **Thay đổi hành vi bên ngoài** (An external behaviour change). Khi tính năng nghiệp vụ thay đổi, việc cập nhật acceptance tests là hợp lý và cần thiết.
- **Thay đổi chi tiết triển khai hoặc refactor** (An implementation detail/refactoring). Những thay đổi này không nên buộc bạn phải sửa test.

Tuy nhiên, trong thực tế, hầu hết các lần sửa acceptance tests lại xuất phát từ nguyên nhân thứ hai. Điều này dẫn đến tình trạng các lập trình viên ngại thay đổi ứng dụng vì sợ phải sửa hàng loạt test.

![Riya và tôi đang nói chuyện về việc tách biệt các mối quan tâm trong test](https://i.imgur.com/bbG6z57.png)

Gốc rễ của vấn đề này nằm ở việc chúng ta thiếu áp dụng các nguyên tắc kỹ thuật phần mềm đã được đúc kết. **Bạn không thể áp dụng cùng một cách viết unit tests cho acceptance tests** - chúng cần một cách tiếp cận và tư duy khác.

## Cấu trúc của một acceptance test tốt

Chúng ta muốn acceptance tests chỉ thay đổi khi hành vi của hệ thống thay đổi, chứ không phải khi chi tiết triển khai thay đổi. Cách hợp lý nhất là tách chúng thành hai phần riêng biệt.

### Các loại độ phức tạp (Types of complexity)

Với vai trò kỹ sư, chúng ta phải đối mặt với hai loại phức tạp:

- **Accidental complexity** (Độ phức tạp ngẫu nhiên) - phức tạp phát sinh từ việc làm việc với máy tính: mạng, ổ đĩa, API, v.v.
- **Essential complexity** (Độ phức tạp cốt lõi) - còn gọi là "domain logic" (logic nghiệp vụ). Đây là các quy tắc và nguyên lý mà lĩnh vực kinh doanh yêu cầu.
  - Ví dụ: "Khi một tài khoản rút số tiền lớn hơn số dư hiện có, tài khoản đó bị đánh dấu là thấu chi." Quy tắc này không liên quan gì đến máy tính - nó đã tồn tại trước khi có phần mềm ngân hàng.

Khi essential complexity được mô tả rõ ràng cho cả những người không chuyên kỹ thuật, giá trị sẽ rất lớn nếu bạn thể hiện chúng qua domain code thuần túy và kiểm tra bằng acceptance tests.

### Phân tách trách nhiệm (Separation of concerns)

Theo lời Dave Farley trong video ở trên, và qua thảo luận của Riya và tôi, chúng ta nên viết các bài test dưới dạng **specifications** (đặc tả). Specification mô tả hành vi mà chúng ta mong muốn, không đề cập đến accidental complexity hay chi tiết triển khai (implementation details).

Điều này nghe quen thuộc. Trong quá trình phát triển sản phẩm, chúng ta luôn cố gắng phân tách trách nhiệm cho các module. Chúng ta sử dụng `interface` để tách HTTP handler khỏi những thứ không liên quan đến HTTP. Vậy hãy mạnh dạn áp dụng tư duy tương tự cho acceptance tests.

Dave Farley mô tả rõ ràng cách tổ chức phân lớp.

![Sơ đồ của Dave Farley về Acceptance Tests](https://i.imgur.com/nPwpihG.png)

Trong buổi nói chuyện tại GopherconUK, Riya và tôi đã diễn đạt điều này bằng thuật ngữ Go.

![Separation of concerns - Phân tách trách nhiệm](https://i.imgur.com/qdY4RJe.png)

### Sức mạnh vượt trội của kiểm thử (Testing on steroids)

Khi chúng ta tách biệt specification khỏi chi tiết triển khai, chúng ta có thể tái sử dụng specification trong nhiều bối cảnh khác nhau.

#### Cấu hình linh hoạt cho driver (Make our drivers configurable)

Với cách tiếp cận này, bạn có thể chạy acceptance tests trên nhiều môi trường: local, staging, và thậm chí cả production.

- Nhiều đội phát triển gặp khó khăn khi code bị ràng buộc chặt đến mức không thể chạy test ở local. Điều này tạo thêm một tầng phản hồi chậm. Bạn có muốn chắc chắn rằng acceptance tests pass _trước khi_ commit code không? Nếu test chỉ chạy được trên môi trường riêng, bạn phải chờ đợi rất lâu mới biết kết quả.
- Ngay cả khi test pass trên staging, đừng cho rằng hệ thống đã ổn. Staging và production không bao giờ giống hệt nhau. Bạn có thể tham khảo: [I test in production](https://increment.com/testing/i-test-in-production/).
- Các môi trường có thể khác nhau về hành vi: CDN cache sai header, dịch vụ phụ thuộc phản hồi khác, file cấu hình lệch nhau. Sẽ rất tuyệt nếu bạn chạy được test specification trực tiếp trên production để phát hiện lỗi nhanh chóng.

#### Sử dụng các driver khác nhau để test các phần khác nhau (Plug in different drivers to test other parts of your system)

Với cách tiếp cận linh hoạt này, chúng ta có thể kiểm tra hành vi qua các tầng trừu tượng khác nhau.

- Ví dụ: nếu bạn có website với một API phía sau, bạn có thể dùng cùng một specification để test cả hai - thông qua web browser cho web page, và thông qua HTTP client cho API.
- Lý tưởng hơn nữa, tôi muốn viết domain code (mã nghiệp vụ) để có thể chạy specification dưới dạng unit tests. Điều này mang lại phản hồi cực nhanh, xác nhận rằng essential complexity được triển khai đúng.

### Acceptance tests thay đổi vì lý do đúng đắn (Acceptance tests changing for the right reasons)

Với cách tổ chức này, specification chỉ cần thay đổi khi hành vi của ứng dụng thay đổi - điều này hoàn toàn hợp lý.

- Nếu API thay đổi, bạn chỉ cần cập nhật driver tương ứng.
- Nếu markup thay đổi, bạn cũng chỉ cần sửa driver cụ thể đó.

Khi ứng dụng phát triển, bạn sẽ nhận ra mình có thể tái sử dụng các driver cho nhiều test. Mỗi khi chi tiết triển khai thay đổi, bạn chỉ cần sửa ở một điểm duy nhất.

Nếu làm đúng cách, bạn đạt được sự tách biệt hoàn toàn giữa chi tiết triển khai và specification. Cách tổ chức này tạo ra một cấu trúc đơn giản và rõ ràng để quản lý sự thay đổi - điều cốt lõi khi hệ thống phát triển lớn hơn.

### Acceptance tests như phương pháp phát triển phần mềm (Acceptance tests as a method for software development)

Trong buổi nói chuyện, Riya và tôi cũng thảo luận về mối liên hệ giữa acceptance tests và BDD. Chúng tôi nhấn mạnh việc _hiểu rõ vấn đề cần giải quyết_ trước khi viết code, và dùng specification để định hướng quá trình phát triển.

Tôi đã học được cách tiếp cận này từ cuốn GOOS. Tôi đã tổng hợp các ý tưởng chính trong bài viết [Why TDD](https://quii.dev/The_Why_of_TDD).

---

TDD mở đường để xây dựng phần mềm hoạt động đúng hành vi mong muốn thông qua vòng lặp liên tục. Khi bắt đầu, điều quan trọng là tập trung vào hành vi cốt lõi và giới hạn phạm vi chức năng.

Theo hướng tiếp cận "từ trên xuống" (top-down), chúng ta bắt đầu với acceptance test - bài test kiểm tra hành vi từ bên ngoài. Bài test này đóng vai trò "ngôi sao Bắc Đẩu" (north star) để định hướng công việc. Hãy tập trung để bài test này pass. Ban đầu, nó sẽ fail một thời gian trong khi bạn viết đủ code cần thiết.

![](https://i.imgur.com/pxTaYu4.png)

Sau khi có acceptance test, bạn quay lại vòng lặp TDD nhỏ hơn để xây dựng từng unit cần thiết giúp acceptance test pass. Ưu điểm ở giai đoạn này là bạn không cần lo lắng về thiết kế tổng thể - acceptance test đã giúp bạn hiểu rõ những gì cần xây dựng.

Bước đầu tiên để acceptance test pass thường tốn nhiều công sức hơn bạn nghĩ: thiết lập web server, routing, cấu hình, v.v. Nhưng đây là đầu tư xứng đáng. Hãy tập trung xây dựng nền tảng tối thiểu để acceptance test pass, từ đó bạn có thể lặp lại nhanh chóng và an toàn (iterate quickly and safely).

![](https://i.imgur.com/t5y5opw.png)

Khi viết code, hãy lắng nghe phản hồi từ các test. Chúng cho bạn tín hiệu để cải thiện thiết kế. Tuy nhiên, hãy đảm bảo rằng bạn tập trung vào hành vi chứ không phải tưởng tượng ra những gì chưa cần.

Thường thì unit đầu tiên phải gánh toàn bộ trách nhiệm để acceptance test pass sẽ khá lớn. Đây là dấu hiệu cho thấy bạn cần chia nhỏ và tách logic ra.

![](https://i.imgur.com/UYqd7Cq.png)

Lúc này, bạn có thể sử dụng test doubles (ví dụ: fakes, mocks) vì phần lớn sự phức tạp nằm "giữa" các unit - cách chúng tương tác với nhau.

#### Hiểm họa của cách tiếp cận từ dưới lên (The perils of bottom-up)

Rõ ràng là cách tiếp cận "từ trên xuống" (top-down) khác với "từ dưới lên" (bottom-up). Cách bottom-up có ưu điểm riêng, nhưng cũng tiềm ẩn rủi ro. Khi bạn xây dựng các service và viết code mà chưa tích hợp vào ứng dụng tổng thể và chưa được kiểm tra bằng acceptance test, **bạn có nguy cơ lãng phí công sức vào những thứ dựa trên giả định chưa được kiểm chứng**.

Việc kết nối công việc với acceptance test rất quan trọng để đảm bảo code hoạt động đúng trong bối cảnh thực tế.

Tôi đã nhiều lần chứng kiến các đồng nghiệp viết code theo hướng bottom-up với hy vọng giải quyết vấn đề, nhưng khi tích hợp lại thì phát hiện:

- Hành vi không đúng như mong đợi
- Làm những thứ không cần thiết (Does stuff we don't need)
- Khó tích hợp (Doesn't integrate easily)
- Phải viết lại rất nhiều (Requires a ton of re-writing)

Đó chính là lãng phí (waste).

## Đủ lý thuyết, hãy bắt đầu viết code (Enough talk, time to code)

Khác với phần còn lại của cuốn sách, bạn sẽ cần cài [Docker](https://www.docker.com) vì chúng ta sẽ đóng gói ứng dụng vào container. Từ chương này trở đi, giả định rằng bạn đã quen thuộc với việc viết Go, import package, v.v.

Khởi tạo project với lệnh `go mod init github.com/quii/go-specs-greet` (bạn có thể đặt tên khác, nhưng nếu thay đổi đường dẫn thì cần chỉnh lại các import tương ứng).

Tạo thư mục `specifications`, trong đó tạo file `greet.go`:

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet() (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet()
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, world")
}
```

IDE của bạn (ví dụ Goland) có thể tự động tải dependency, nhưng nếu cần làm thủ công, hãy chạy:

`go get github.com/alecthomas/assert/v2`

Theo sơ đồ thiết kế của Farley (Specification -> DSL -> Driver -> System), chúng ta đang viết specification tách biệt khỏi implementation. Chúng ta không quan tâm đến việc `Greet` được thực hiện _như thế nào_ (how). Specification chỉ xử lý essential complexity thuộc domain nghiệp vụ. Hiện tại điều này có vẻ đơn giản, nhưng sẽ có giá trị hơn khi chúng ta bổ sung thêm chức năng. Nguyên tắc cơ bản là bắt đầu nhỏ.

Bạn có thể dùng `interface` như bước đầu tiên của DSL. Khi ứng dụng phát triển lớn hơn, có thể cần thêm các lớp trừu tượng khác, nhưng hiện tại thế này là đủ.

Tại thời điểm này, có thể có người cho rằng việc tách specification khỏi implementation là "trừu tượng quá mức". **Tôi cam đoan rằng, acceptance tests bị ràng buộc chặt với implementation sẽ trở thành gánh nặng cho cả đội phát triển**. Theo kinh nghiệm của tôi, phần lớn acceptance tests tốn kém bảo trì là do chúng bị ràng buộc chặt với chi tiết triển khai, chứ không phải vì trừu tượng quá mức.

Với specification này, chúng ta có thể test bất kỳ "hệ thống" nào có khả năng thực hiện `Greet`.

### Mặt trận đầu tiên: HTTP API

Chúng ta được yêu cầu xây dựng một greeter service qua HTTP. Cần chuẩn bị:

1. Một **driver** - trong trường hợp này sẽ là HTTP client. Code này biết cách tương tác với API. Driver có nhiệm vụ chuyển đổi DSL thành các lời gọi đến hệ thống cụ thể. Trong ví dụ này, driver sẽ implement interface được định nghĩa trong specification.
2. Một **HTTP server** với endpoint greet
3. Một **test** có nhiệm vụ tự động khởi tạo, chạy web server, kết nối driver với specification và chạy test.

## Viết test trước (Write the test first)

Bước đầu tiên - tạo một bài black-box test có khả năng tự động build, chạy test và dọn dẹp - thường tốn nhiều công sức. Nhưng việc làm điều này ngay từ đầu dự án sẽ đơn giản hơn rất nhiều. Cá nhân tôi, mỗi dự án mới, tôi đều bắt đầu bằng việc thiết lập server "hello world" cùng với acceptance test.

Việc thiết kế với "specifications", "drivers", và "acceptance tests" cần thời gian để quen. Gợi ý là hãy "đi ngược" (work backwards) - bắt đầu từ specification.

Tạo cấu trúc thư mục cho chương trình:

`mkdir -p cmd/httpserver`

Trong thư mục đó, tạo file `greeter_server_test.go`:

```go
package main_test

import (
	"testing"

	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	specifications.GreetSpecification(t, nil)
}
```

Ý tưởng là đưa specification vào một Go test bình thường. Chúng ta có `*testing.T`, nhưng argument thứ hai thì sao?

Vì `specifications.Greeter` là interface, chúng ta cần một `Driver` để implement nó. Cập nhật `TestGreeterServer`:

```go
import (
	go_specs_greet "github.com/quii/go-specs-greet"
)

func TestGreeterServer(t *testing.T) {
	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

`Driver` có `BaseURL` cho phép cấu hình để chạy test trên các môi trường khác nhau, bao gồm cả local.

## Thử chạy test (Try to run the test)

```
./greeter_server_test.go:46:12: undefined: go_specs_greet.Driver
```

Chúng ta vẫn đang đi theo dòng TDD. Có một bước nhảy khá lớn cần thực hiện: viết nhiều code hơn bình thường, nhưng khi mới bắt đầu dự án, điều này là bình thường. Hãy tuân thủ quy tắc của bước red.

> Commit bao nhiêu "tội" cũng được miễn là test pass (Commit as many sins as necessary to get the test passing)

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Hãy kiên nhẫn; khi test pass, chúng ta sẽ refactor. Tạo file `driver.go` ở thư mục gốc (project root):

```go
package go_specs_greet

import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
}

func (d Driver) Greet() (string, error) {
	res, err := http.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

Một vài lưu ý:

- Bạn có thể thắc mắc tại sao không unit test phần xử lý `if err != nil`. Xét giá trị thực tế, chúng chỉ trả về error nhận được - đây là test có giá trị thấp (low value).
- **Không dùng default HTTP client của Go**. Chúng ta sẽ thêm HTTP client có cấu hình timeout sau. Nhưng hiện tại, hãy tập trung vào việc làm test pass.
- Trong file `greeter_server_test.go`, nhớ import `github.com/quii/go-specs-greet`.

Thử chạy test. Lần này code biên dịch được nhưng test fail:

```
Get "http://localhost:8080/greet": dial tcp [::1]:8080: connect: connection refused
```

Mặc dù `Driver` đã biên dịch được, nhưng chưa có application nào chạy, nên HTTP request thất bại. Acceptance test cần phải tự động build, chạy và dọn dẹp hệ thống.

### Chạy ứng dụng (Running our application)

Vì chúng ta đóng gói application bằng Docker image để triển khai, test cũng nên làm tương tự.

Để sử dụng Docker trong test, chúng ta dùng [Testcontainers](https://golang.testcontainers.org). Testcontainers cung cấp API lập trình để quản lý Docker container trong test.

`go get github.com/testcontainers/testcontainers-go`

Cập nhật file `cmd/httpserver/greeter_server_test.go`:

```go
package main_test

import (
	"context"
	"testing"

	"github.com/alecthomas/assert/v2"
	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func TestGreeterServer(t *testing.T) {
	ctx := context.Background()

	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "./cmd/httpserver/Dockerfile",
			// set to false if you want less spam, but this is helpful if you're having troubles
			PrintBuildLog: true,
		},
		ExposedPorts: []string{"8080:8080"},
		WaitingFor:   wait.ForHTTP("/").WithPort("8080"),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})

	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

Thử chạy test:

```
=== RUN   TestGreeterHandler
2022/09/10 18:49:44 Starting container id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Waiting for container id 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Container is ready id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
    greeter_server_test.go:32: Did not expect an error but got:
        Error response from daemon: Cannot locate specified Dockerfile: ./cmd/httpserver/Dockerfile: failed to create container
--- FAIL: TestGreeterHandler (0.59s)
```

Chúng ta cần tạo `Dockerfile` cho ứng dụng. Trong thư mục `httpserver`, tạo file `Dockerfile`:

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
# For example, golang:1.22.1-alpine.
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/httpserver/*.go

EXPOSE 8080
CMD [ "./svr" ]
```

Đừng lo lắng về chi tiết lúc này; chúng ta có thể tối ưu Dockerfile sau. Ưu điểm là khi nâng cấp Dockerfile, test sẽ giúp xác nhận mọi thứ vẫn hoạt động. Đây chính là sức mạnh của black-box tests.

Chạy test lại. Lần này sẽ báo lỗi build image thất bại vì chúng ta chưa viết chương trình.

Để test chạy được, chúng ta cần một chương trình lắng nghe trên cổng `8080`, **nhưng chỉ vậy thôi**. Theo kỷ luật TDD, không viết production code nào ngoài những gì cần thiết để thấy test fail đúng cách.

Tạo file `main.go` trong thư mục `httpserver`:

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(writer http.ResponseWriter, request *http.Request) {
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

Chạy test, bạn sẽ thấy lỗi sau:

```
    greet.go:16: Expected values to be equal:
        +Hello, World
        \ No newline at end of file
--- FAIL: TestGreeterHandler (2.09s)
```

## Viết đủ code để test pass

Cập nhật handler để trả về kết quả đúng theo specification:

```go
import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		fmt.Fprint(w, "Hello, world")
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

## Refactor

Mặc dù không phải refactor theo nghĩa chặt chẽ, đây là thời điểm tốt để cải thiện. Chúng ta không nên dùng default HTTP client. Hãy thêm một HTTP client có cấu hình vào `Driver`:

```go
import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
	Client  *http.Client
}

func (d Driver) Greet() (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

Trong file test `cmd/httpserver/greeter_server_test.go`, tạo client với timeout:

```go
client := http.Client{
	Timeout: 1 * time.Second,
}

driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080", Client: &client}
specifications.GreetSpecification(t, driver)
```

Giữ cho `main.go` đơn giản nhất có thể - nó chỉ nên chịu trách nhiệm kết nối các thành phần lại với nhau.

Tạo file `handler.go` ở thư mục gốc:

```go
package go_specs_greet

import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello, world")
}
```

Cập nhật `main.go` để import và sử dụng handler:

```go
package main

import (
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet"
)

func main() {
	handler := http.HandlerFunc(go_specs_greet.Handler)
	http.ListenAndServe(":8080", handler)
}
```

## Nhìn lại (Reflect)

Bước khởi đầu tốn nhiều công sức. Chúng ta tạo ra nhiều file Go để xây dựng và test một HTTP handler chỉ trả về chuỗi cố định. Dù "vòng lặp đầu tiên" (iteration 0) có phần nặng nề, nhưng nó sẽ phục vụ tốt cho các vòng lặp tiếp theo.

Việc thay đổi chức năng giờ đây không quá khó - chỉ cần cập nhật specification rồi xử lý các thay đổi kéo theo. Giờ chúng ta đã có Dockerfile và testcontainers cho acceptance test, bạn không cần lo lắng về cơ sở hạ tầng nữa.

Hãy xem yêu cầu tiếp theo: "chào một người cụ thể theo tên".

## Viết test trước (Write the test first)

Cập nhật specification:

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet(name string) (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet("Mike")
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, Mike")
}
```

Để đón nhận tên người dùng, chúng ta thay đổi interface để nhận tham số `name`.

## Thử chạy test (Try to run the test)

```
./greeter_server_test.go:48:39: cannot use driver (variable of type go_specs_greet.Driver) as type specifications.Greeter in argument to specifications.GreetSpecification:
	go_specs_greet.Driver does not implement specifications.Greeter (wrong type for Greet method)
		have Greet() (string, error)
		want Greet(name string) (string, error)
```

Thay đổi specification kéo theo driver cũng cần được cập nhật.

## Viết lượng code tối thiểu để chạy test và kiểm tra lỗi

Cập nhật driver để gửi `name` dưới dạng query parameter trong request:

```go
import "io"

func (d Driver) Greet(name string) (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet?name=" + name)
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

Test đã biên dịch được và trả về lỗi:

```
    greet.go:16: Expected values to be equal:
        -Hello, world
        \ No newline at end of file
        +Hello, Mike
        \ No newline at end of file
--- FAIL: TestGreeterHandler (1.92s)
```

## Viết đủ code để test pass

Lấy tên từ request và trả về lời chào:

```go
import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, %s", r.URL.Query().Get("name"))
}
```

Test pass.

## Refactor

Trong chương [HTTP Handlers Revisited](https://github.com/quii/learn-go-with-tests/blob/main/http-handlers-revisited.md), chúng ta đã thảo luận về tầm quan trọng của việc giữ cho HTTP handler chỉ xử lý HTTP, tách riêng domain logic ra ngoài. Điều này giúp phát triển domain logic độc lập và dễ test hơn.

Hãy tách các mối quan tâm (concerns) ra.

Cập nhật handler trong `./handler.go`:

```go
func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, Greet(name))
}
```

Tạo file `./greet.go`:
```go
package go_specs_greet

import "fmt"

func Greet(name string) string {
	return fmt.Sprintf("Hello, %s", name)
}
```

## Một chút về adapter pattern

Khi đã tách domain logic ra thành function độc lập, chúng ta có thể viết unit test trực tiếp cho hàm `Greet`. Điều này đơn giản hơn nhiều so với việc chạy specification qua driver, khởi động server chỉ để lấy một chuỗi.

Nhưng liệu chúng ta có thể tái sử dụng specification ở đây không? Vì specification được thiết kế để ẩn đi chi tiết triển khai, nếu nó nắm bắt được essential complexity và domain code cũng thể hiện đúng essential complexity đó, chúng ta hoàn toàn có thể kết nối chúng.

Thử tạo file `./greet_test.go`:

```go
package go_specs_greet_test

import (
	"testing"

	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(t, go_specs_greet.Greet)
}

```

Tuy nhiên, code này không biên dịch được:

```
./greet_test.go:11:39: cannot use go_specs_greet.Greet (value of type func(name string) string) as type specifications.Greeter in argument to specifications.GreetSpecification:
	func(name string) string does not implement specifications.Greeter (missing Greet method)
```

Specification yêu cầu một đối tượng có method `Greet()`, không phải một function đơn lẻ.

Lỗi biên dịch này cho thấy chúng ta đã có thứ mang đúng hành vi (behaviour) của `Greeter`, nhưng không đúng hình dạng (shape) để thỏa mãn compiler. Đây là lúc cần dùng **adapter pattern**.

> Trong [kỹ thuật phần mềm](https://en.wikipedia.org/wiki/Software_engineering), **adapter pattern** là một [design pattern](https://en.wikipedia.org/wiki/Software_design_pattern) (hay còn gọi là [wrapper](https://en.wikipedia.org/wiki/Wrapper_function)) cho phép [interface](https://en.wikipedia.org/wiki/Interface_(computer_science)) của một [class](https://en.wikipedia.org/wiki/Class_(computer_science)) được sử dụng như một interface khác.[[1\]](https://en.wikipedia.org/wiki/Adapter_pattern#cite_note-HeadFirst-1) Pattern này thường được dùng để các class hiện có làm việc với nhau mà không cần sửa [mã nguồn](https://en.wikipedia.org/wiki/Source_code).

Nghe có vẻ phức tạp, nhưng thực ra design pattern chỉ là từ vựng chung để mô tả các giải pháp cho vấn đề thường gặp. Có chung ngôn ngữ giúp giao tiếp hiệu quả hơn.

Tạo file `./specifications/adapters.go`:

```go
type GreetAdapter func(name string) string

func (g GreetAdapter) Greet(name string) (string, error) {
	return g(name), nil
}
```

Giờ chúng ta có thể dùng adapter trong test để kết nối function `Greet` với specification:

```go
package go_specs_greet_test

import (
	"testing"

	gospecsgreet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(
		t,
		specifications.GreetAdapter(gospecsgreet.Greet),
	)
}
```

Adapter hữu ích khi bạn có một type có đúng hành vi (behaviour) mà interface yêu cầu, nhưng không đúng hình dạng (shape).

## Nhìn lại (Reflect)

Quá trình thay đổi diễn ra khá suôn sẻ. Có thể do bản chất đơn giản của bài toán, nhưng kỷ luật kỹ thuật đã giúp việc thay đổi trở nên dễ dàng từ trên xuống dưới:

- Mô tả hành vi mới trong specification
- Nắm bắt essential complexity mới trong specification
- Để compiler dẫn đường đến khi acceptance test pass
- Cập nhật implementation để đáp ứng specification
- Refactor

Sau vòng lặp đầu tiên tốn nhiều công sức, các vòng tiếp theo dễ dàng hơn vì specification, driver và implementation đã được tách biệt. Thay đổi specification chỉ yêu cầu cập nhật driver và implementation, không ảnh hưởng đến cơ sở hạ tầng (container, Docker).

Dù mất thêm chi phí build Docker image và khởi tạo container, chúng ta có vòng phản hồi (feedback loop) cho phép test **toàn bộ** ứng dụng:

```
quii@Chriss-MacBook-Pro go-specs-greet % go test ./...
ok  	github.com/quii/go-specs-greet	0.181s
ok  	github.com/quii/go-specs-greet/cmd/httpserver	2.221s
?   	github.com/quii/go-specs-greet/specifications	[no test files]
```

Giả sử CTO mới tuyên bố rằng gRPC mới là _tương lai_. Bạn cần phơi chức năng này qua gRPC server, đồng thời vẫn giữ HTTP server hiện tại.

Đây là ví dụ điển hình của **accidental complexity**. Hãy nhớ, accidental complexity là sự phức tạp phát sinh từ máy tính, mạng, ổ đĩa, API, v.v. **Essential complexity không hề thay đổi**, vì vậy chúng ta không cần thay đổi specification.

Có nhiều mô hình kiến trúc mô tả việc tách biệt hai loại phức tạp này. Ví dụ "ports and adapters" - giữ domain code tách biệt khỏi accidental complexity, đặt trong các "adapters" riêng.

### Làm cho thay đổi trở nên dễ dàng (Making the change easy)

Đôi khi, bạn cần refactor _trước khi_ thêm chức năng mới.

> Đầu tiên hãy làm cho thay đổi trở nên dễ dàng, rồi mới thực hiện thay đổi đó (First make the change easy, then make the easy change)

~Kent Beck

Hãy di chuyển `driver.go` và `handler.go` vào package `httpserver` trong thư mục `adapters`, đổi tên package thành `httpserver`.

Trong `handler.go`, import gói root để gọi method `Greet`:

```go
package httpserver

import (
	"fmt"
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet/domain/interactions"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, go_specs_greet.Greet(name))
}

```

Cập nhật import adapter httpserver trong `main.go`:

```go
package main

import (
	"net/http"

	"github.com/quii/go-specs-greet/adapters/httpserver"
)

func main() {
	handler := http.HandlerFunc(httpserver.Handler)
	http.ListenAndServe(":8080", handler)
}
```

Cập nhật import và tham chiếu `Driver` trong `greeter_server_test.go`:

```go
driver := httpserver.Driver{BaseURL: "http://localhost:8080", Client: &client}
```

Cuối cùng, nên đặt domain code trong thư mục riêng. Đừng để thư mục `domain` trở thành nơi chứa mọi thứ hỗn loạn. Hãy suy nghĩ về cách tổ chức domain và nhóm các khái niệm liên quan lại với nhau. Điều này giúp dự án dễ hiểu hơn và import rõ ràng hơn.

Thay vì:

```go
domain.Greet
```

Chúng ta có:

```go
interactions.Greet
```

Tạo thư mục `domain/interactions` để chứa domain code. Sử dụng công cụ của IDE để cập nhật import và tham chiếu.

Cấu trúc dự án giờ trông như thế này:

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   └── httpserver
|       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── adapters.go
    └── greet.go

```

Domain code (**essential complexity**) nằm ở tầng gốc của Go module, trong khi code tương tác với "thế giới bên ngoài" được đặt trong các thư mục **adapters** riêng biệt. Thư mục `cmd` chứa các ứng dụng thực tế, kèm theo black-box tests để xác minh chúng hoạt động đúng.

Cuối cùng, hãy dọn dẹp acceptance test. Nhìn lại các bước ở mức cao, một acceptance test bao gồm:

- Build Docker image
- Chờ server lắng nghe trên port cụ thể
- Tạo driver biết cách chuyển đổi DSL thành lời gọi hệ thống
- Kết nối driver với specification

... và chúng ta sẽ cần lặp lại chính xác các bước này cho acceptance test của gRPC server.

Trong thư mục `adapters`, tạo file `docker.go` để đưa hai bước đầu tiên vào một hàm tái sử dụng:

```go
package adapters

import (
	"context"
	"fmt"
	"testing"
	"time"

	"github.com/alecthomas/assert/v2"
	"github.com/docker/go-connections/nat"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func StartDockerServer(
	t testing.TB,
	port string,
	dockerFilePath string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:       "../../.",
			Dockerfile:    dockerFilePath,
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

Điều này giúp đơn giản hóa acceptance test đáng kể:

```go
func TestGreeterServer(t *testing.T) {
	var (
		port           = "8080"
		dockerFilePath = "./cmd/httpserver/Dockerfile"
		baseURL        = fmt.Sprintf("http://localhost:%s", port)
		driver         = httpserver.Driver{BaseURL: baseURL, Client: &http.Client{
			Timeout: 1 * time.Second,
		}}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, driver)
}
```

Việc thiết lập test tiếp theo cũng sẽ gọn gàng hơn.

## Viết test trước (Write the test first)

Việc thêm chức năng mới giờ đơn giản hơn - chỉ cần thêm adapter mới sử dụng cùng domain code. Cụ thể:

- Không cần thay đổi specification
- Tái sử dụng specification hiện có
- Tái sử dụng domain code

Tạo thư mục mới `grpcserver` trong `cmd` cho ứng dụng mới và acceptance test. Trong file `cmd/grpcserver/greeter_server_test.go`:

```go
package main_test

import (
	"fmt"
	"testing"

	"github.com/quii/go-specs-greet/adapters"
	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	var (
		port           = "50051"
		dockerFilePath = "./cmd/grpcserver/Dockerfile"
		driver         = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, &driver)
}
```

Điểm khác biệt chỉ là:

- Sử dụng Dockerfile khác, cho ứng dụng mới
- Sử dụng `Driver` mới, tương tác với hệ thống qua gRPC thay vì HTTP

## Thử chạy test (Try to run the test)

```
./greeter_server_test.go:26:12: undefined: grpcserver
```

Chưa có `Driver` nào được tạo, nên biên dịch lỗi.

## Viết lượng code tối thiểu để chạy test và kiểm tra lỗi

Tạo thư mục `grpcserver` trong `adapters`, tạo file `driver.go`:

```go
package grpcserver

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	return "", nil
}
```

Chạy lại test. Code biên dịch được nhưng test fail vì chưa có Dockerfile và chương trình.

Tạo `Dockerfile` trong `cmd/grpcserver`:

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/grpcserver/*.go

EXPOSE 50051
CMD [ "./svr" ]
```

Thêm `main.go`:

```go
package main

import "fmt"

func main() {
	fmt.Println("implement me")
}
```

Test sẽ fail vì server không lắng nghe trên port. Giờ hãy xây dựng client và server gRPC.

## Viết đủ code để test pass

### Giới thiệu gRPC

Nếu bạn chưa biết về gRPC, hãy tham khảo [trang web gRPC](https://grpc.io). Trong chương này, gRPC đơn giản là một adapter khác cho hệ thống, cho phép gọi thủ tục từ xa (**r**emote **p**rocedure **c**all) đến domain code.

Với gRPC, bạn định nghĩa service bằng Protocol Buffers. Từ đó, code cho cả client và server được tự động sinh ra. Protocol Buffers không chỉ dùng cho Go mà hỗ trợ nhiều ngôn ngữ, giúp giao tiếp service-to-service dễ dàng.

Bạn cần cài **Protocol buffer compiler** và **Go plugins**. [Hướng dẫn chi tiết trên trang gRPC](https://grpc.io/docs/languages/go/quickstart/).

Trong thư mục `adapters/grpcserver`, tạo file `greet.proto`:

```protobuf
syntax = "proto3";

option go_package = "github.com/quii/adapters/grpcserver";

package grpcserver;

service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
}

message GreetRequest {
  string name = 1;
}

message GreetReply {
  string message = 1;
}
```

Bạn không cần trở thành chuyên gia Protocol Buffers. Chúng ta chỉ định nghĩa service với method `Greet`, mô tả kiểu dữ liệu đầu vào và đầu ra (message types).

Từ thư mục `adapters/grpcserver`, chạy lệnh sau để sinh code cho client và server:

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

Lệnh này sinh ra code mà chúng ta có thể sử dụng. Hãy cập nhật `Driver` để dùng client code đã sinh:

```go
package grpcserver

import (
	"context"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	//todo: cần refactor sau, không nên dial lại mỗi lần gọi Greet
	conn, err := grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		return "", err
	}
	defer conn.Close()

	client := NewGreeterClient(conn)
	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

Sau khi có client, hãy cập nhật `main.go` để thêm server. Nhớ rằng ở giai đoạn này, mục tiêu là làm test pass, chưa cần quan tâm đến chất lượng code.

```go
package main

import (
	"context"
	"log"
	"net"

	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"google.golang.org/grpc"
)

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatal(err)
	}
	s := grpc.NewServer()
	grpcserver.RegisterGreeterServer(s, &GreetServer{})

	grpcserver.RegisterGreeterServer(s, &GreetServer{})

	if err := s.Serve(lis); err != nil {
		log.Fatal(err)
	}
}

type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: "fixme"}, nil
}
```

Để tạo gRPC server, bạn cần implement interface mà code đã sinh ra:

```go
// GreeterServer is the server API for Greeter service.
// All implementations must embed UnimplementedGreeterServer
// for forward compatibility
type GreeterServer interface {
	Greet(context.Context, *GreetRequest) (*GreetReply, error)
	mustEmbedUnimplementedGreeterServer()
}
```

Hàm `main` chịu trách nhiệm:

- Lắng nghe trên một port
- Tạo `GreetServer` implement interface, đăng ký với `grpcServer.RegisterGreeterServer` kèm theo `grpc.Server`
- Khởi chạy server với listener

Tôi cố tình dùng chuỗi `fixme` thay vì gọi domain code ngay vì muốn kiểm tra xem các lớp transport hoạt động đúng chưa, và muốn thấy kết quả fail rõ ràng:

```
greet.go:16: Expected values to be equal:
-fixme
\ No newline at end of file
+Hello, Mike
\ No newline at end of file
```

Tốt! Driver đã kết nối thành công với gRPC server.

Giờ hãy gọi domain code trong `GreetServer`:

```go
type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

Test pass! Chúng ta có acceptance test xác nhận gRPC greeter server hoạt động đúng.

## Refactor

Chúng ta đã phạm một số "tội" để test pass. Giờ có mạng lưới an toàn rồi, hãy refactor.

### Giữ `main` gọn nhẹ

Không nên nhồi nhét quá nhiều code vào `main`. Di chuyển `GreetServer` vào `adapters/grpcserver` cho đúng vị trí. Nguyên tắc là đảm bảo tính gắn kết (cohesion) - khi thay đổi service, phạm vi ảnh hưởng (blast-radius) chỉ nằm trong khu vực code liên quan.

### Không nên dial lại mỗi lần gọi qua Driver

Nếu specification mở rộng thêm (và sẽ như vậy), việc Driver phải redial cho mỗi lần gọi RPC sẽ lãng phí.

```go
package grpcserver

import (
	"context"
	"sync"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string

	connectionOnce sync.Once
	conn           *grpc.ClientConn
	client         GreeterClient
}

func (d *Driver) Greet(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}

func (d *Driver) getClient() (GreeterClient, error) {
	var err error
	d.connectionOnce.Do(func() {
		d.conn, err = grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
		d.client = NewGreeterClient(d.conn)
	})
	return d.client, err
}
```

Ở đây chúng ta dùng [`sync.Once`](https://pkg.go.dev/sync#Once) để đảm bảo Driver chỉ kết nối một lần duy nhất.

Hãy xem lại cấu trúc dự án:

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   ├── docker.go
│   ├── grpcserver
│   │   ├── driver.go
│   │   ├── greet.pb.go
│   │   ├── greet.proto
│   │   ├── greet_grpc.pb.go
│   │   └── server.go
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   ├── grpcserver
│   │   ├── Dockerfile
│   │   ├── greeter_server_test.go
│   │   └── main.go
│   └── httpserver
│       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── greet.go
```

- Thư mục `adapters` chứa các nhóm chức năng gắn kết (cohesive units)
- Thư mục `cmd` chứa các ứng dụng cùng acceptance tests
- Domain code hoàn toàn tách biệt (totally decoupled) khỏi accidental complexity

### Gộp Dockerfile

Có thể bạn đã nhận ra cả hai `Dockerfiles` gần như giống hệt nhau, chỉ khác đường dẫn binary.

Dockerfile hỗ trợ arguments, cho phép tái sử dụng trong nhiều bối cảnh. Hãy thay thế cả hai Dockerfile bằng một file duy nhất ở thư mục gốc:

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
FROM golang:1.18-alpine

WORKDIR /app

ARG bin_to_build

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/${bin_to_build}/main.go

CMD [ "./svr" ]
```

Cập nhật hàm `StartDockerServer` để truyền argument khi build image:

```go
func StartDockerServer(
	t testing.TB,
	port string,
	binToBuild string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "Dockerfile",
			BuildArgs: map[string]*string{
				"bin_to_build": &binToBuild,
			},
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

Cập nhật test để truyền tên image cần build (tương tự cho cả test HTTP):

```go
func TestGreeterServer(t *testing.T) {
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
}
```

### Tách riêng các test (Separating test suites)

Một lợi thế lớn của acceptance tests là khả năng kiểm tra toàn bộ hệ thống từ góc nhìn người dùng (user-facing, behavioural POV). Tuy nhiên, so với unit tests, chúng có nhược điểm:

- Chậm hơn (Slower)
- Phản hồi kém chi tiết - không tập trung như unit tests
- Không giúp nhiều cho chất lượng nội bộ (internal quality) và thiết kế (design)

[Kim tự tháp test (The Test Pyramid)](https://martinfowler.com/articles/practical-test-pyramid.html) gợi ý cách kết hợp hợp lý các loại test. Bạn nên đọc bài của Fowler để hiểu chi tiết. Tóm lại: nên có nhiều unit tests và ít acceptance tests.

Do đó, khi dự án phát triển, acceptance tests có thể mất vài phút để chạy. Để giữ trải nghiệm phát triển tốt, lập trình viên cần có cách chạy riêng từng loại test.

`go test ./...` không cần cấu hình phức tạp, nhưng cần Docker. Go cung cấp [short flag](https://pkg.go.dev/testing#Short) để chạy "test ngắn":

`go test -short ./...`

Thêm đoạn sau vào đầu acceptance tests để bỏ qua khi dùng flag `-short`:

```go
if testing.Short() {
	t.Skip()
}
```

Ví dụ `Makefile`:

```makefile
build:
	golangci-lint run
	go test ./...

unit-tests:
	go test -short ./...
```

### Khi nào nên viết acceptance test?

Quy tắc chung là viết nhiều unit tests và ít acceptance tests. Nhưng khi nào nên viết acceptance test so với unit test?

Dưới đây là một vài gợi ý:

- Đây là trường hợp biên (edge case)? Hãy dùng unit test.
- Đây là tính năng mà người không chuyên kỹ thuật (non-computer people) có thể mô tả? Bạn muốn chắc chắn rằng tính năng cốt lõi này "rõ ràng hoạt động"? Hãy thêm acceptance test.
- Đây là hành trình người dùng (user journey) hoàn chỉnh, không chỉ là một function đơn lẻ? Hãy dùng acceptance test.
- Unit tests đã cho bạn đủ tự tin (confidence) chưa? Nếu bạn đã có acceptance test cho user journey và cần kiểm tra thêm các input khác nhau, unit tests có thể đủ - thêm acceptance test ở đây sẽ ít giá trị.

## Tiếp tục phát triển (Iterating on our work)

Sau tất cả công sức trên, hệ thống vẫn rất đơn giản. Nhưng chúng ta đã xây dựng một cấu trúc tuy không đơn giản (simple is not the same as easy), nhưng đáng giá vì nó giúp mở rộng dễ dàng hơn khi bắt đầu dự án mới.

Hãy thêm chức năng "curse" (chửi) vào API.

## Viết test trước (Write the test first)

Vì đây là hành vi hoàn toàn mới, chúng ta bắt đầu với acceptance test. Thêm vào specification:

```go
type MeanGreeter interface {
	Curse(name string) (string, error)
}

func CurseSpecification(t *testing.T, meany MeanGreeter) {
	got, err := meany.Curse("Chris")
	assert.NoError(t, err)
	assert.Equal(t, got, "Go to hell, Chris!")
}
```

Sử dụng specification mới trong acceptance test:

```go
func TestGreeterServer(t *testing.T) {
	if testing.Short() {
		t.Skip()
	}
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	t.Cleanup(driver.Close)
	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
	specifications.CurseSpecification(t, &driver)
}
```

## Thử chạy test (Try to run the test)

```
# github.com/quii/go-specs-greet/cmd/grpcserver_test [github.com/quii/go-specs-greet/cmd/grpcserver.test]
./greeter_server_test.go:27:39: cannot use &driver (value of type *grpcserver.Driver) as type specifications.MeanGreeter in argument to specifications.CurseSpecification:
	*grpcserver.Driver does not implement specifications.MeanGreeter (missing Curse method)
```

`Driver` chưa có method `Curse`.

## Viết lượng code tối thiểu để chạy test và kiểm tra lỗi

Mục tiêu là chỉ làm cho code biên dịch được. Thêm method vào `Driver`:

```go
func (d *Driver) Curse(name string) (string, error) {
	return "", nil
}
```

Chạy lại test, code biên dịch được và fail đúng cách:

```
greet.go:26: Expected values to be equal:
+Go to hell, Chris!
\ No newline at end of file
```

## Viết đủ code để test pass

Thêm method `Curse` vào protocol buffer, rồi sinh lại code:

```protobuf
service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
  rpc Curse (GreetRequest) returns (GreetReply) {}
}
```

Dù việc dùng chung type `GreetRequest` và `GreetReply` cho cả `Greet` và `Curse` tạo ra coupling, chúng ta sẽ xử lý trong bước refactor. Nhớ rằng mục tiêu là làm test pass trước.

Sinh lại code (từ thư mục `adapters/grpcserver`):

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

### Cập nhật driver

Client code đã được sinh ra, giờ implement hàm `Curse` trong `Driver`:

```go
func (d *Driver) Curse(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Curse(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

### Cập nhật server

Thêm method `Curse` vào `Server`:

```go
package grpcserver

import (
	"context"
	"fmt"

	"github.com/quii/go-specs-greet/domain/interactions"
)

type GreetServer struct {
	UnimplementedGreeterServer
}

func (g GreetServer) Curse(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: fmt.Sprintf("Go to hell, %s!", request.Name)}, nil
}

func (g GreetServer) Greet(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

Test pass!

## Refactor

Thử tự làm phần refactor:

- Tách domain logic của `Curse` ra khỏi gRPC server, tương tự như đã làm với `Greet`. Sử dụng specification làm unit test cho domain logic.
- Tạo các message type riêng trong protobuf để `Greet` và `Curse` không bị coupling.

## Thêm chức năng `Curse` vào HTTP server

Phần này dành cho bạn tự thực hành. Vì đã có specification và domain code tách biệt, việc này rất đơn giản:

- Thêm specification vào acceptance test của HTTP server
- Cập nhật `Driver`
- Thêm endpoint mới vào server và gọi domain code. Bạn có thể dùng `http.NewServeMux` để quản lý nhiều endpoint riêng biệt.

Hãy thực hiện từng bước nhỏ (small steps), commit thường xuyên sau mỗi lần test pass. Nếu gặp khó khăn, tham khảo [mã nguồn trên GitHub](https://github.com/quii/go-specs-greet).

## Kiểm tra chéo domain logic bằng unit test

Càng phát triển, nếu không có thay đổi lớn về cấu trúc, bạn không cần thêm acceptance test mới. Các quy tắc nghiệp vụ phức tạp và edge cases có thể được kiểm tra dễ dàng bằng unit test vì chúng ta đã tách biệt các mối quan tâm (separated concerns).

Ví dụ: thêm unit test cho hàm `Greet` để xử lý trường hợp `name` rỗng, trả về "World" thay thế. Đây là cách đơn giản để thể hiện business rules trong test.

## Tổng kết

Xây dựng hệ thống với chi phí thay đổi hợp lý đòi hỏi thiết kế acceptance tests để hỗ trợ bạn, không phải trở thành gánh nặng bảo trì. Bạn có thể dùng chúng như cọc tiêu định hướng, hoặc như GOOS nói, để "phát triển" phần mềm có phương pháp.

Hy vọng qua ví dụ này, bạn thấy cách chúng ta thay đổi ứng dụng bằng quy trình có cấu trúc, dễ dự đoán (predictable, structured workflow) và cách ứng dụng phát triển một cách có tổ chức.

Hãy suy nghĩ về việc thu thập yêu cầu khi muốn mở rộng dự án. Tập trung vào domain, tạo specification không phụ thuộc vào triển khai (implementation-agnostic) để dùng làm kim chỉ nam cho cả đội. Riya và tôi có thảo luận về cách dùng BDD và "Example Mapping" [trong buổi nói chuyện tại GopherconUK](https://www.youtube.com/watch?v=ZMWJCk_0WrY) để nắm bắt essential complexity và viết specification hiệu quả.

Tách biệt essential complexity và accidental complexity sẽ mang lại cấu trúc rõ ràng (structured) và có chủ đích (deliberate) cho quy trình làm việc. Điều này giúp acceptance tests trở nên linh hoạt và giảm gánh nặng bảo trì (maintenance burden).

Dave Farley đưa ra lời khuyên sau:

> Hãy tưởng tượng người ít kỹ thuật nhất (the least technical person) nhưng hiểu rõ lĩnh vực nghiệp vụ (problem-domain) đọc acceptance tests của bạn. Các bài test đó phải có ý nghĩa với họ.

Specification nên đóng vai trò như tài liệu (documentation), mô tả cách hệ thống hoạt động ở mức hành vi. Đây cũng là nền tảng cho các công cụ như [Cucumber](https://cucumber.io), giúp bạn viết DSL mô tả hành vi dưới dạng code, rồi chuyển đổi DSL thành lời gọi hệ thống.

### Tóm tắt những gì đã học

- Viết specification thể hiện essential complexity, tách biệt khỏi accidental complexity. Tái sử dụng specification trong nhiều bối cảnh khác nhau.
- Sử dụng [Testcontainers](https://golang.testcontainers.org) để quản lý vòng đời acceptance tests. Build Docker image và chạy container ngay trên máy local để có phản hồi nhanh.
- Đóng gói ứng dụng bằng Docker (containerising)
- gRPC
- Cấu trúc thư mục không chỉ là hình thức - nó phản ánh kiến trúc và giúp tổ chức code hợp lý.

### Đọc thêm

- Khi hệ thống phát triển lớn, việc chỉ dùng interface cho DSL có thể không đủ. Lúc đó cần các tầng trừu tượng cao hơn. Hãy tham khảo [Screenplay Pattern](https://cucumber.io/blog/bdd/understanding-screenplay-(part-1)/) để tìm cách tổ chức specification hiệu quả hơn.
- Cuốn [Growing Object-Oriented Software, Guided by Tests](http://www.growing-object-oriented-software.com) là tài liệu kinh điển. Phong cách "London style" với cách tiếp cận top-down là nền tảng của chương này. "Learn Go with Tests" rất phù hợp làm bổ sung cho cuốn GOOS.
- [Trong kho mã nguồn ví dụ](https://github.com/quii/go-specs-greet), có thêm code mà tôi viết trong chương này, bao gồm việc thiết lập Docker build. Để vui, tôi còn tạo thêm **chương trình thứ ba** - một trang web sử dụng HTML forms gọi `Greet` và `Curse`. `Driver` trong chương trình này dùng [https://github.com/go-rod/rod](https://github.com/go-rod/rod) để tương tác với website qua trình duyệt, giống như cách người dùng thực sử dụng. Hãy xem git history để thấy cách tôi bắt đầu đơn giản rồi mở rộng, tận dụng sự tự do refactor mà không sợ phá hỏng nhờ acceptance test bảo vệ.
