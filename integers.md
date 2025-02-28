# Số nguyên (Integers)

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/integers)**

Số nguyên trong Go hoạt động đúng như bạn mong đợi. Bây giờ, hãy thử viết một hàm `Add` để kiểm tra cách chúng hoạt động.

Trước tiên, tạo một tệp kiểm thử có tên `adder_test.go` và thêm đoạn mã sau.

**Lưu ý:** Mọi tệp Go trong cùng một thư mục phải thuộc cùng một `package`, vì vậy hãy đảm bảo tổ chức mã nguồn một cách hợp lý. Nếu bạn chưa rõ về cách sắp xếp, [bài viết này](https://dave.cheney.net/2014/12/01/five-suggestions-for-setting-up-a-go-project) sẽ giúp bạn hiểu rõ hơn.

Khi đó, cấu trúc thư mục của bạn có thể trông như thế này:

```
learnGoWithTests
    |
    |-> helloworld
    |    |- hello.go
    |    |- hello_test.go
    |
    |-> integers
    |    |- adder_test.go
    |
    |- go.mod
    |- README.md
```

## Viết test trước tiên

```go
package integers

import "testing"

func TestAdder(t *testing.T) {
	sum := Add(2, 2)
	expected := 4

	if sum != expected {
		t.Errorf("expected '%d' but got '%d'", expected, sum)
	}
}
```

Bạn sẽ thấy rằng chúng ta đang sử dụng `%d `thay vì `%q` trong chuỗi định dạng. Lý do là `%d` dùng để hiển thị số nguyên, trong khi `%q` dành cho chuỗi.

Ngoài ra, lần này chúng ta không còn sử dụng package `main` nữa, mà thay vào đó đã định nghĩa một package có tên là `integers`. Đúng như tên gọi, package này sẽ chứa các hàm làm việc với số nguyên, chẳng hạn như `Add`.

## Chạy thử test

Chạy lệnh `go test`

Quan sát lỗi biên dịch

`./adder_test.go:6:9: undefined: Add`

Lỗi này nghĩa là hàm `Add` chưa được định nghĩa

## Viết đủ code và thử lại

Chỉ viết đủ code để trình biên dịch không báo lỗi – _không hơn không kém_.
Điều này giúp đảm bảo rằng bài kiểm thử sẽ thất bại với lý do chính xác mà chúng ta mong đợi.

```go
package integers

func Add(x, y int) int {
	return 0
}
```

Nhớ rằng, khi có nhiều tham số cùng kiểu dữ liệu (trong trường hợp này là hai số nguyên), thay vì viết `(x int, y int)`, bạn có thể rút gọn thành `(x, y int)`.

Bây giờ hãy chạy lại test, chúng ta sẽ thấy kết quả báo lỗi chính xác:

`adder_test.go:10: expected '4' but got '0'`

(Lỗi này có nghĩa là chúng ta mong đợi kết quả là 4, nhưng lại nhận được 0.)

Ngoài ra, trong [phần trước](hello-world.md#one...last...refactor?), chúng ta đã học về _giá trị trả về có tên_ (named return value), nhưng ở đây lại không sử dụng nó. Thông thường, named return value nên được dùng khi ý nghĩa của giá trị trả về không rõ ràng trong ngữ cảnh. Nhưng trong trường hợp này, hàm `Add` đã rất rõ ràng là để cộng hai số, nên không cần thiết phải dùng named return value.

Bạn có thể tham khảo thêm về vấn đề này trong [wiki chính thức của Go](https://go.dev/wiki/CodeReviewComments#named-result-parameters).

## Viết đủ code để test thành công

Theo đúng quy trình TDD, bây giờ chúng ta chỉ nên viết lượng mã _tối thiểu cần thiết_ để test _chạy thành công_. Một lập trình viên cẩn thận (pragmatic developer) có thể viết như sau:

```go
func Add(x, y int) int {
	return 4
}
```

Bạn có thể nghĩ rằng TDD thật vô nghĩa khi chúng ta chỉ viết đúng đủ để bài kiểm thử vượt qua mà không thực sự giải quyết vấn đề.

Chúng ta có thể viết thêm một test với những số khác để buộc nó thất bại, nhưng điều đó giống như một [trò đuổi bắt vô tận](https://en.m.wikipedia.org/wiki/Cat_and_mouse).

Sau này, khi bạn đã quen với cú pháp của Go, chúng ta sẽ tìm hiểu về _“Kiểm thử dựa trên tính chất”_ (Property-Based Testing), một kỹ thuật giúp phát hiện lỗi mà không cần chơi trò mèo vờn chuột với các bài kiểm thử.

Còn bây giờ, hãy sửa lỗi đúng cách!

```go
func Add(x, y int) int {
	return x + y
}
```

Nếu bạn chạy lại test, nó nên thành công.

## Refactor

Không có nhiều điều cần cải thiện trong đoạn code ở đây.

Trước đó, chúng ta đã thấy rằng khi đặt tên cho giá trị trả về, nó sẽ xuất hiện trong tài liệu và hầu hết các trình soạn thảo của lập trình viên.

Điều này rất hữu ích vì nó giúp mã dễ sử dụng hơn. Tốt nhất là người dùng có thể hiểu cách sử dụng hàm chỉ bằng cách nhìn vào kiểu dữ liệu trả về và tài liệu đi kèm.

Bạn có thể thêm tài liệu cho hàm bằng cách viết chú thích phía trên nó. Những chú thích này sẽ xuất hiện trong Go Doc, giống như khi bạn xem tài liệu của thư viện tiêu chuẩn.

```go
// Add takes two integers and returns the sum of them.
func Add(x, y int) int {
	return x + y
}
```

### Test Ví Dụ (Testable Examples)

Nếu muốn làm tốt hơn nữa, bạn có thể tạo các [Test Ví Dụ](https://blog.golang.org/examples). Bạn sẽ thấy nhiều ví dụ như vậy trong tài liệu của thư viện chuẩn.

Thường thì các code ví dụ nằm bên ngoài mã thực tế, chẳng hạn trong file README, dễ bị lỗi thời và không còn chính xác vì chúng không được kiểm tra thường xuyên.

Các hàm ví dụ trong Go được biên dịch mỗi khi chạy bài kiểm thử. Vì vậy, bạn có thể chắc chắn rằng ví dụ trong tài liệu luôn phản ánh đúng hành vi hiện tại của mã nguồn

Các hàm ví dụ phải bắt đầu bằng từ khóa `Example` (giống như hàm kiểm thử bắt đầu bằng `Test`) và được đặt trong các file `_test.go` của package.

Hãy thêm hàm `ExampleAdd` vào file `adder_test.go`.

```go
func ExampleAdd() {
	sum := Add(1, 5)
	fmt.Println(sum)
	// Output: 6
}
```

(Nếu trình soạn thảo của bạn không tự động nhập các gói cần thiết, bước biên dịch sẽ thất bại vì thiếu `import "fmt"` trong `adder_test.go`. Bạn nên tìm hiểu cách cấu hình trình soạn thảo để tự động xử lý các lỗi kiểu này.)

Thêm đoạn code này sẽ giúp ví dụ xuất hiện trong tài liệu, làm cho code của bạn dễ tiếp cận hơn. Nếu sau này mã thay đổi khiến ví dụ không còn hợp lệ, quá trình biên dịch sẽ thất bại, giúp bạn phát hiện lỗi sớm.

Khi chạy bộ kiểm thử của gói, bạn sẽ thấy hàm `ExampleAdd` được thực thi mà không cần bổ sung bất kỳ thiết lập nào.

```bash
$ go test -v
=== RUN   TestAdder
--- PASS: TestAdder (0.00s)
=== RUN   ExampleAdd
--- PASS: ExampleAdd (0.00s)
```

Lưu ý định dạng đặc biệt của chú thích `// Output: 6`. Khi có chú thích này, ví dụ không chỉ được biên dịch mà còn được thực thi trong quá trình kiểm thử. Hãy thử xóa dòng `// Output: 6`, sau đó chạy `go test`, bạn sẽ thấy `ExampleAdd` không còn được chạy nữa.

Các ví dụ không có chú thích đầu ra vẫn hữu ích để minh họa cách sử dụng mã, đặc biệt đối với các trường hợp không thể kiểm thử đơn vị, chẳng hạn như truy cập mạng. Tuy nhiên, chúng vẫn đảm bảo mã có thể biên dịch được.

Để xem tài liệu ví dụ, bạn có thể sử dụng {pkgsite}. Điều hướng đến thư mục dự án, sau đó chạy `pkgsite -open .`. Trình duyệt sẽ mở trang `http://localhost:8080`, nơi bạn có thể xem danh sách tất cả các gói trong thư viện chuẩn của Go cũng như các gói bên thứ ba mà bạn đã cài đặt.

Tại đây, bạn sẽ tìm thấy tài liệu cho gói của mình. Điều hướng đến `github.com/quii/learn-go-with-tests`, vào phần `Integers`, chọn `func Add`, rồi mở rộng mục `Example`. Bạn sẽ thấy ví dụ mà bạn đã thêm cho `sum := Add(1, 5)`.

Nếu bạn công khai mã nguồn cùng các ví dụ, tài liệu của bạn cũng sẽ xuất hiện trên [pkg.go.dev](https://pkg.go.dev/). Ví dụ, [đây](https://pkg.go.dev/github.com/quii/learn-go-with-tests/integers/v2) là API cuối cùng của chương này. Trang web này giúp bạn tìm kiếm tài liệu của các gói trong thư viện chuẩn và các gói bên thứ ba dễ dàng hơn.

## Tổng kết

Những gì chúng ta đã học được:

*   Thực hành thêm về quy trình TDD
*   Làm việc với số nguyên và phép cộng
*   Viết tài liệu rõ ràng hơn để người dùng có thể nhanh chóng hiểu cách sử dụng mã của chúng ta
*   Cách viết ví dụ minh họa cho mã, đồng thời đảm bảo chúng luôn được kiểm tra cùng với bộ kiểm thử
