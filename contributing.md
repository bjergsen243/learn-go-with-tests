# Đóng góp

Mọi đóng góp đều được hoan nghênh! Tôi hy vọng đây sẽ trở thành một tài liệu hướng dẫn tuyệt vời giúp bạn học Go thông qua việc viết test. Nếu có ý tưởng cải thiện, bạn có thể gửi PR hoặc tạo issue [tại đây](https://github.com/quii/learn-go-with-tests/issues)

## Những gì chúng tôi tìm kiếm

* Hướng dẫn về các tính năng của Go \(ví dụ: `if`, `select`, struct, method, v.v.\).
* Giới thiệu các chức năng thú vị trong thư viện chuẩn, chẳng hạn như cách áp dụng TDD để xây dựng một HTTP server
* Hướng dẫn sử dụng các công cụ của Go như benchmark, race detector,… để giúp bạn viết phần mềm chất lượng cao.

Nếu bạn chưa tự tin viết hướng dẫn của riêng mình, bạn vẫn có thể đóng góp bằng cách tạo issue về những chủ đề bạn muốn học.

### ⚠️ Nhận phản hồi nhanh chóng cho nội dung mới ⚠️

- TDD dạy chúng ta làm việc theo từng bước nhỏ và nhận phản hồi sớm. Tôi khuyến khích bạn làm điều tương tự khi đóng góp
    - Hãy mở PR với bài test đầu tiên và phần triển khai của bạn, sau đó thảo luận về cách tiếp cận để tôi có thể đưa ra phản hồi và điều chỉnh nếu cần.
- Dù đây là một dự án mã nguồn mở, tôi vẫn có quan điểm rõ ràng về nội dung. Càng trao đổi sớm, bạn càng có thể đi đúng hướng.

## Hướng dẫn phong cách viết

* Luôn nhấn mạnh vào chu trình TDD. Hãy tham khảo [Chapter Template](template.md)
* Tập trung vào phát triển tính năng dựa trên các bài test. Ví dụ “Hello, world” hoạt động tốt vì chúng ta dần làm nó phức tạp hơn và học các kỹ thuật mới _dựa trên các bài test_:
  * `Hello()` &lt;- cách viết hàm, kiểu trả về.
  * `Hello(name string)` &lt;- tham số, hằng số.
  * `Hello(name string)` &lt;- giá trị mặc định “world” với `if`.
  * `Hello(name, language string)` &lt;- `switch`.
* Giảm thiểu lượng kiến thức cần thiết.
  * Cần chọn ví dụ minh họa rõ ràng cho nội dung muốn truyền đạt mà không làm người đọc bối rối bởi những tính năng khác.
  * Ví dụ, bạn có thể học về `struct` mà không cần hiểu con trỏ ngay từ đầu
  * Ngắn gọn là ưu tiên hàng đầu.
* Tuân theo [hướng dẫn phong cách Code Review Comments](https://go.dev/wiki/CodeReviewComments) để đảm bảo tính nhất quán.
* Mỗi phần nên có một ứng dụng có thể chạy được ở cuối (ví dụ: `package main` với một hàm `main`), giúp người đọc có thể chạy thử và trải nghiệm.
* Tất cả các bài test phải chạy thành công.
* Chạy `./build.sh` trước khi gửi PR.
