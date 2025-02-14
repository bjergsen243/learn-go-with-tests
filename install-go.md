# Cài đặt Go, thiết lập môi trường

Hướng dẫn cài đặt Go [tại đây](https://golang.org/doc/install).

## Go Environment

### Go Modules

Go 1.11 giới thiệu [Modules](https://go.dev/wiki/Modules). Kể từ Go 1.16, đây là cách mặc định để build, nên bạn không cần sử dụng `GOPATH` nữa.

Modules được thiết kế để giải quyết các vấn đề liên quan đến quản lý phụ thuộc, chọn phiên bản và tái tạo bản build một cách nhất quán. Ngoài ra, nó còn cho phép bạn chạy code bên ngoài `GOPATH`

Sử dụng Modules khá là đơn giản. Bạn chỉ cần chọn một thư mục bên ngoài `GOPATH` làm thư mục gốc cho dự án, sau đó khởi tạo module mới bằng lệnh `go mod init`.

Lệnh này sẽ tạo tệp `go.mod`, trong đó chứa

* Đường dẫn module (module path)
* Phiên bản Go
* Danh sách phụ thuộc (các module cần thiết để biên dịch thành công)

Nếu không chỉ định  `<modulepath>`, Go sẽ tự suy đoán từ cấu trúc thư mục. Bạn cũng có thể ghi đè giá trị này bằng cách cung cấp đường dẫn module theo ý muốn.

```sh
mkdir my-project
cd my-project
go mod init <modulepath>
```

Tệp `go.mod` sẽ như sau:

```
module cmd

go 1.16

```

Bạn có thể xem tất cả các lệnh `go mod` có sẵn bằng cách.

```sh
go help mod
go help mod init
```

## Go Linting

Bạn có thể sử dụng [GolangCI-Lint](https://golangci-lint.run) thay thế cho linter mặc định, giúp cải thiện chất lượng code cũng như phát hiện lỗi tốt hơn.

Cài đặt (MacOS):

```sh
brew install golangci-lint
```

## Refactoring và công cụ hỗ trợ

Một trong những trọng tâm của cuốn sách này là tầm quan trọng của refactoring (tái cấu trúc mã nguồn).

Các công cụ phù hợp sẽ giúp bạn thực hiện refactoring một cách an toàn và hiệu quả hơn.

Bạn nên làm quen với trình soạn thảo của mình để có thể thực hiện các thao tác sau chỉ bằng một tổ hợp phím:

- **Tách/Gộp biến**. Đặt tên cho các giá trị giúp mã nguồn dễ hiểu và bảo trì hơn. Hạn chế tạo các [magic values](https://en.wikipedia.org/wiki/Magic_number_(programming)).
- **Tách phương thức/hàm**. Việc tách một đoạn mã thành hàm hoặc phương thức riêng biệt là rất quan trọng để cải thiện tính tải sử dụng và dễ đọc.
- **Đổi tên**. Bạn nên có khả năng đổi tên các biến, hàm hoặc cấu trúc trên nhiều tệp một cách an toàn.
- **go fmt**. Go có trình định dạng mã mặc định là `go fmt`. Trình soạn thảo của bạn nên tự động chạy lệnh này mỗi khi lưu tệp.
- **Chạy kiểm thử**. Sau khi thực hiện các thay đổi trên, bạn cần có khả năng chạy lại kiểm thử nhanh chóng để đảm bảo mã nguồn vẫn hoạt động chính xác.

Ngoài ra, để làm việc hiệu quả hơn, bạn nên làm quen với các tính năng sau:

- **Xem signature của hàm**. Bạn không nên mơ hồ về cách gọi một hàm trong Go. Trinh soạn thảo của bạn nên hiển thị tài liệu, danh sách tham số và giá trị trả về của hàm.
- **Xem định nghĩa của hàm**. Nếu vẫn chưa rõ một hàm hoạt động thế nào, bạn nên biết cách mở mã nguồn của nó để tìm hiểu.
- **Tìm tất cả vị trí sử dụng của một symbol**. Biết được vị trí và cách một hàm được sử dụng sẽ giúp bạn đưa ra quyết định chính xác hơn khi refactoring.

Mastering your tools will help you concentrate on the code and reduce context switching.

## Tổng kết

Đến đây, bạn nên cài đặt xong Go, một trình soạn thảo phù hợp và thiết lập một số công cụ hỗ trợ phù hợp với mình. Go có một hệ sinh thái rất lớn với nhiều công cụ và thư viện từ bên thứ ba. Trong phần này, tác giả đã giới thiệu một số công cụ hữu ích. Nếu bạn muốn khám phá thêm, hãy xem danh sách đầy đủ tại [https://awesome-go.com](https://awesome-go.com).
