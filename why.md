# Tại sao cần kiểm thử đơn vị và cách tận dụng chúng hiệu quả

[Nếu bạn thích xem video, đây là link mà tác giả chia sẻ về chủ đề này](https://www.youtube.com/watch?v=Kwtit8ZEK7U)

Nếu bạn thích đọc hơn, đây là phiên bản bằng chữ.

## Software

Sự hứa hẹn của phần mềm ở khả năng thay đổi của nó. Đó là lý do nó được gọi là _soft_ware - dễ uốn nắn hơn so với phần cứng (hardware). Một đội ngũ kỹ sư giỏi có thể trở thành tài sản quý giá của công ty, xây dựng các hệ thống có thể phát triển theo nhu cầu kinh doanh để tiếp tục tạo ra giá trị ấy.

Vậy tại sao chúng ta lại làm điều này tệ đến vậy? Có bao nhiêu dự án mà bạn biết đã thất bại hoàn toàn? Hoặc trở thành "di sản" (lecacy) rồi phải viết lại từ đầu (và đôi khi bản viết lại cũng thất bại!).

Một hệ thống phần mềm "thất bại" như thế nào? Chẳng phải chúng ta có thể thay đổi nó cho đến khi đúng lúc sao? Đó là những gì chúng ta được hứa hẹn!

Nhiều người chọn Go để phát triển hệ thống vì ngôn ngữ này có những thứ giúp phần mềm ít bị lỗi thời hơn:

- So với những ngôn ngữ phức tạp như Scala – nơi mà [tác giả từng mô tả là “đủ dây để tự treo cổ”](http://www.quii.dev/Scala_-_Just_enough_rope_to_hang_yourself) – Go chỉ có 25 từ khóa (keyword). Nhiều hệ thống có thể được xây dựng chỉ từ thư viện chuẩn và một số thư viện nhỏ khác. Với Go, bạn có thể viết code và quay lại sau 6 tháng mà vẫn hiểu được nó.
- Công cụ hỗ trợ kiểm thử, đo hiệu năng (benchmarking), linting và triển khai (shipping) của Go thuộc hàng tốt nhất so với hầu hết các lựa chọn khác.
- Thư viện chuẩn cực kỳ mạnh mẽ.
- Tốc độ biên dịch nhanh, giúp rút ngắn vòng phản hồi (feedback loop).
- Cam kết về khả năng tương thích ngược của Go: Ngay cả khi Go sắp có generics và nhiều tính năng khác, các nhà thiết kế hứa rằng mã Go viết 5 năm trước vẫn có thể biên dịch được. Tác giả từng mất hàng tuần để nâng cấp một dự án từ Scala 2.8 lên 2.10.

Dù có những đặc điểm tuyệt vời này, chúng ta vẫn có thể tạo ra những hệ thống tệ hại. Vì vậy, hãy nhìn lại quá khứ và rút ra những bài học về kỹ nghệ phần mềm – những bài học này đúng bất kể ngôn ngữ của bạn có “hào nhoáng” thế nào.

Năm 1974, kỹ sư phần mềm [Manny Lehman](https://en.wikipedia.org/wiki/Manny_Lehman_%28computer_scientist%29) đã đưa ra [các định luật tiến hóa phần mềm của Lehman](https://en.wikipedia.org/wiki/Lehman%27s_laws_of_software_evolution):

> Những định luật này mô tả sự cân bằng giữa các lực thúc đẩy phát triển phần mềm và những lực cản làm chậm tiến trình.

Hiểu được những lực tác động này là điều quan trọng nếu chúng ta muốn thoát khỏi vòng lặp vô tận của việc tạo ra các hệ thống rồi biến chúng thành “di sản”, rồi lại viết lại từ đầu.

## Định luật về Sự thay đổi liên tục (The Law of Continuous Change)

> Bất kỳ hệ thống phần mềm nào được sử dụng trong thế giới thực đều phải thay đổi, nếu không nó sẽ ngày càng trở nên kém hữu ích trong môi trường đó (Any software system used in the real-world must change or become less and less useful in the environment).

Điều này nghe có vẻ hiển nhiên – một hệ thống _phải_ thay đổi nếu không sẽ mất dần giá trị. Nhưng thực tế, điều này lại thường bị phớt lờ.

Nhiều nhóm phát triển phần mềm bị thúc đẩy bởi mục tiêu hoàn thành một dự án vào một thời điểm cụ thể, rồi sau đó chuyển sang dự án khác. Nếu phần mềm “may mắn”, nó có thể được bàn giao cho một nhóm khác để bảo trì – nhưng những người đó không phải là người viết ra nó ngay từ đầu.

Mọi người thường tập trung vào việc chọn một framework giúp họ “phát triển nhanh chóng” mà không quan tâm đến tuổi thọ của hệ thống và khả năng tiến hóa của nó.

Ngay cả khi bạn là một kỹ sư phần mềm xuất sắc, bạn cũng không thể đoán trước được tất cả nhu cầu tương lai của hệ thống. Khi doanh nghiệp thay đổi, một số đoạn code tuyệt vời bạn từng viết có thể trở nên không còn phù hợp.

Lehman không dừng lại ở đây – trong thập niên 70, ông tiếp tục đưa ra một định luật khác đáng để suy ngẫm.

## Định luật về Sự gia tăng độ phức tạp (The Law of Increasing Complexity)

> Khi một hệ thống phát triển, độ phức tạp của nó sẽ tăng lên trừ khi có nỗ lực để giảm bớt điều đó (As a system evolves, its complexity increases unless work is done to reduce it).

Điều mà Lehman muốn nói ở đây là chúng ta không thể vận hành các nhóm phát triển phần mềm như những “nhà máy sản xuất tính năng”, liên tục chồng chất thêm tính năng vào hệ thống với hy vọng rằng nó sẽ tồn tại lâu dài.

Chúng ta **phải** quản lý độ phức tạp của hệ thống khi kiến thức về lĩnh vực mà nó phục vụ thay đổi.

## Refactoring

Refactoring là một trong những yếu tố quan trọng giúp phần mềm luôn dễ bảo trì và mở rộng. Tuy nhiên, khái niệm này thường bị hiểu sai hoặc sử dụng không chính xác.

[Martin Fowler đã chỉ ra một sai lầm phổ biến](https://martinfowler.com/bliki/RefactoringMalapropism.html)

> Nếu ai đó nói rằng hệ thống bị lỗi trong vài ngày vì đang refactoring, thì gần như chắc chắn họ không thực sự đang refactor (However the term "refactoring" is often used when it's not appropriate. If somebody talks about a system being broken for a couple of days while they are refactoring, you can be pretty sure they are not refactoring).

Vậy refactoring thực chất là gì?

Refactoring **KHÔNG** phải là:

- Viết lại toàn bộ hệ thống từ đầu.
- Thay đổi cách phần mềm hoạt động.
- Một quá trình kéo dài làm gián đoạn công việc.

Refactoring **ĐÚNG** nghĩa là:

- Cải thiện mã nguồn mà không thay đổi hành vi của chương trình.
- Chia nhỏ thành từng bước nhỏ, mỗi bước đều có thể kiểm tra ngay.
- Dựa vào kiểm thử tự động để đảm bảo không làm hỏng hệ thống.

Refactoring không phải là một nhiệm vụ “làm nếu có thời gian”. Nó là một quá trình liên tục giúp phần mềm duy trì tính linh hoạt, dễ mở rộng và không bị sa lầy vào sự phức tạp ngày càng gia tăng.

### Phân tích và Tương đồng với Toán Học

Trong toán học, bạn có thể đã học về phân tích thừa số (factorisation). Hãy xem một ví dụ đơn giản:

Tính giá trị của `1/2 + 1/4`

Để làm điều này, chúng ta _phân tích mẫu số_ để đưa các phân số về cùng một hệ quy chiếu: `1/2 = 2/4`

Bây giờ, biểu thức trở thành: `2/4 + 1/4 = 3/4`.

Điều quan trọng cần lưu ý ở đây là:

- Chúng ta **không thay đổi giá trị của biểu thức** – cả hai cách viết đều bằng `3/4`.
- Chúng ta chỉ thay đổi cách biểu diễn, giúp việc tính toán trở nên dễ dàng hơn.

Tương tự như vậy, khi bạn refactor code, bạn đang tìm cách làm cho nó dễ hiểu và phù hợp hơn với hệ thống mà không thay đổi hành vi của nó.

- Refactoring không phải là viết lại code từ đầu.
- Refactoring giúp mã nguồn dễ bảo trì và mở rộng hơn.
- Một hệ thống tốt là hệ thống có thể được cải thiện liên tục mà không gây gián đoạn.

Tóm lại, refactoring chính là cải thiện cách tổ chức mà **không làm thay đổi kết quả** – giống như cách ta làm với toán học!

#### Ví dụ với Go

Dưới đây là một hàm nhận vào `name` và `language`:

    func Hello(name, language string) string {
    
      if language == "es" {
         return "Hola, " + name
      }
    
      if language == "fr" {
         return "Bonjour, " + name
      }
      
      // imagine dozens more languages
    
      return "Hello, " + name
    }

Vấn đề của đoạn code này:

- Quá nhiều câu lệnh `if`, làm cho mã nguồn trở nên khó đọc và bảo trì.
- Trùng lặp logic: Mỗi dòng đều ghép chuỗi `", " + name`, gây dư thừa.
- Nếu muốn thêm một ngôn ngữ mới, ta phải chỉnh sửa code, vi phạm nguyên tắc Open/Closed Principle (OCP).

    func Hello(name, language string) string {
      	return fmt.Sprintf(
      		"%s, %s",
      		greeting(language),
      		name,
      	)
    }
    
    var greetings = map[string]string {
      "es": "Hola",
      "fr": "Bonjour",
      //etc..
    }
    
    func greeting(language string) string {
      greeting, exists := greetings[language]
      
      if exists {
         return greeting
      }
      
      return "Hello"
    }

Bản chất của việc refactor không quan trọng, điều quan trọng là hành vi của chương trình không thay đổi.

Khi refactor, bạn có thể làm bất cứ điều gì:

- Thêm interface để code linh hoạt hơn
- Định nghĩa thêm struct, method để tổ chức lại logic
- Tách nhỏ function để dễ đọc, dễ hiểu hơn

Nhưng kết quả đầu ra của chương trình phải giữ nguyên.

Vì vậy, viết test trước khi refactor là bắt buộc. Sau khi thay đổi code, chỉ cần chạy test để đảm bảo mọi thứ vẫn hoạt động đúng.

### Khi refactor code, bạn không được thay đổi hành vi

Là lập trình viên, chúng ta luôn cố gắng chia nhỏ hệ thống thành các file, package, function… để dễ hiểu hơn. Nếu làm quá nhiều thứ cùng lúc, bạn sẽ dễ mắc sai lầm.

Nhiều dự án thất bại khi refactor vì các lập trình viên cố gắng làm quá nhiều thứ cùng lúc.

Vậy làm sao để đảm bảo refactor không thay đổi hành vi?

Khi làm toán, chúng ta phải tự kiểm tra xem kết quả sau khi biến đổi có giống ban đầu không. Với code, cách tốt nhất để kiểm tra là viết unit test trước khi refactor.

Những ai không viết test thường phải kiểm thử thủ công, điều này tốn thời gian và không thể mở rộng.

**Lợi ích của unit test khi refactor**:

- Giúp bạn tự tin thay đổi code mà không làm thay đổi hành vi
- Là tài liệu mô tả cách hệ thống hoạt động
- Cung cấp phản hồi nhanh hơn nhiều so với kiểm thử thủ công

#### Ví dụ với Go

Một unit test cho hàm `Hello` như sau

    func TestHello(t *testing.T) {
      got := Hello(“Chris”, es)
      want := "Hola, Chris"
    
      if got != want {
         t.Errorf("got %q want %q", got, want)
      }
    }

Trên dòng lệnh, bạn có thể chạy `go test` để nhận phản hồi ngay lập tức về việc refactor có làm thay đổi hành vi hay không. Nhưng trong thực tế, bạn nên tìm cách chạy test nhanh nhất trong editor/IDE của mình.

Bạn nên duy trì một vòng lặp chặt chẽ:

- Refactor từng bước nhỏ
- Chạy tests
- Lặp lại

Điều này giúp bạn tránh đi lạc hướng hoặc mắc sai lầm trong quá trình refactor.

Nếu dự án của bạn có đầy đủ unit test cho các hành vi quan trọng và có thể chạy test trong chưa đến một giây, bạn sẽ có một mạng lưới an toàn để refactor một cách tự tin.

Điều này giúp bạn kiểm soát sự phức tạp ngày càng tăng của hệ thống, như Lehman đã mô tả.

## Nếu unit test tuyệt vời như vậy, tại sao đôi khi vẫn có sự phản đối khi viết chúng?

Một số người (như tác giả) cho rằng unit test rất quan trọng để duy trì sự linh hoạt của hệ thống trong dài hạn, vì nó giúp bạn refactor một cách tự tin.

Nhưng cũng có nhiều người cảm thấy unit test _gây cản trở_ quá trình refactor.

Hãy tự hỏi: Bạn có phải thay đổi quá nhiều test khi refactor không?

Trong nhiều dự án, dù có test coverage tốt, các kỹ sư vẫn e ngại refactor vì cho rằng việc sửa test quá tốn công sức.

Điều này đi ngược lại với những gì unit test hứa hẹn!

### Tại sao điều này xảy ra?

Hãy tưởng tượng bạn được yêu cầu tạo ra một hình vuông, và bạn nghĩ cách tốt nhất là ghép hai tam giác vuông lại với nhau.

![Two right-angled triangles to form a square](https://i.imgur.com/ela7SVf.jpg)

Bạn viết unit test để kiểm tra rằng các cạnh của hình vuông bằng nhau, đồng thời kiểm tra tam giác để đảm bảo tổng góc là 180 độ, số lượng tam giác đúng, v.v. Viết những test này khá dễ và test coverage cao là điều tốt, vậy tại sao không làm?

Vài tuần sau, “Định luật Thay đổi Liên tục” (The Law of Continuous Change) xuất hiện và một lập trình viên mới vào dự án. Cô ấy cho rằng hình vuông nên được tạo từ hai hình chữ nhật thay vì hai tam giác.

![Two rectangles to form a square](https://i.imgur.com/1G6rYqD.jpg)

Cô ấy thử refactor nhưng nhận được hàng loạt test bị lỗi. Câu hỏi đặt ra là: cô ấy có thực sự làm hỏng hành vi quan trọng không? Bây giờ cô ấy phải đào sâu vào các test cũ về tam giác để hiểu chuyện gì đang xảy ra.

_Việc hình vuông được tạo từ tam giác hay chữ nhật không quan trọng_, nhưng các **test đã vô tình làm cho chi tiết này trở nên quan trọng**. Chúng ta đã viết test không đúng cách: thay vì kiểm tra hành vi, chúng ta kiểm tra cách triển khai.

## Ưu tiên kiểm tra hành vi hơn là chi tiết triển khai

Khi nghe ai đó phàn nàn về unit test, thường thì nguyên nhân là do test được viết ở sai mức trừu tượng.

- Họ test quá nhiều chi tiết triển khai, theo dõi quá mức cách các module tương tác với nhau.
- Họ lạm dụng mocking, khiến test trở nên mong manh và khó bảo trì.
- Họ chạy theo chỉ số test coverage mà không quan tâm test đó có thực sự hữu ích hay không.

Nếu chỉ cần kiểm tra hành vi, vậy có nên chỉ viết test hệ thống (system test/black-box test)?

 Không hẳn!

- System test rất hữu ích trong việc kiểm tra hành trình người dùng, đảm bảo hệ thống hoạt động đúng.
- Nhưng chúng thường đắt đỏ để viết, chạy chậm và khó tìm nguyên nhân lỗi khi có vấn đề.
- Vì vậy, chúng không lý tưởng cho việc refactor vì vòng lặp phản hồi (feedback loop) quá chậm.

Vậy mức trừu tượng đúng khi viết test là gì?

Câu trả lời là: Kiểm tra hành vi ở đúng cấp độ, không đi quá sâu vào chi tiết triển khai.

## Viết unit test hiệu quả là một bài toán thiết kế

Bỏ qua chuyện test một chút, trong một hệ thống phần mềm, chúng ta luôn muốn có các “đơn vị” (units) độc lập, ít phụ thuộc, xoay quanh các khái niệm quan trọng trong domain.

Hãy hình dung những đơn vị này như các mảnh Lego, mỗi mảnh có một API rõ ràng, dễ sử dụng, không để lộ chi tiết triển khai. Khi ghép chúng lại với nhau, ta có thể xây dựng những hệ thống lớn hơn mà không bị ràng buộc vào cách chúng hoạt động bên trong.

Ví dụ, nếu bạn đang viết một hệ thống ngân hàng bằng Go, bạn có thể có một package account:

- Nó cung cấp một API sạch, dễ hiểu.
- Bên trong có thể có hàng tá types, functions hợp tác với nhau để hoạt động.
- Nhưng API bên ngoài không bị ảnh hưởng bởi cách nó triển khai nội bộ.

Áp dụng vào unit test:

- Nếu một hệ thống có những unit như vậy, unit test chỉ cần kiểm tra API công khai của chúng.
- Điều này đảm bảo rằng test chỉ kiểm tra hành vi thực sự hữu ích, thay vì ràng buộc vào chi tiết triển khai.
- Khi đó, ta có thể thoải mái refactor bên trong mà không phải sửa hàng loạt test không cần thiết.

### Đây có phải là unit test không?

**CÓ**. Unit test là kiểm thử các “đơn vị” (units) trong hệ thống, như cách tôi đã mô tả trước đó. Unit test **KHÔNG** đơn giản là kiểm thử một class, function hay một đoạn code cụ thể. Chúng kiểm tra một **đơn vị** có **ý nghĩa trong domain**, với API rõ ràng và không ràng buộc vào cách triển khai bên trong.

## Kết nối các khái niệm với nhau

Chúng ta đã đề cập đến:

- Refactoring (Tái cấu trúc)
- Unit tests (Kiểm thử đơn vị)
- Unit design (Thiết kế đơn vị)

Chúng ta có thể thấy rằng Refactoring (tái cấu trúc), Unit Test (kiểm thử đơn vị), và Thiết kế Unit tốt bổ trợ lẫn nhau để giúp phần mềm dễ bảo trì và mở rộng.

### Refactoring

- Cho ta thấy chất lượng của unit test. Nếu sau khi tái cấu trúc, ta phải kiểm thử thủ công, nghĩa là unit test chưa đủ. Nếu test bị fail mà không rõ lý do, có thể test đang ở mức trừu tượng sai (hoặc thậm chí không có giá trị và nên bị xóa).
- Giúp quản lý độ phức tạp bên trong và giữa các module.

### Unit tests

- Là “tấm lưới an toàn” giúp ta tái cấu trúc mà không lo làm hỏng chức năng
- Xác minh và ghi lại hành vi mong muốn của hệ thống.

### Unit (có thiết kế tốt)

- Dễ viết unit test _có ý nghĩa_, tập trung vào hành vi thay vì chi tiết triển khai.
- Dễ dàng tái cấu trúc mà không cần sửa đổi quá nhiều test.

Làm thế nào để duy trì khả năng tái cấu trúc liên tục?

## Tại sao nên dùng Test Driven Development (TDD)

Nhiều người khi hiểu rằng phần mềm luôn phải thay đổi (theo định luật của Lehman) sẽ cố gắng thiết kế một hệ thống “hoàn hảo” ngay từ đầu. Họ dành nhiều tháng lên kế hoạch nhưng cuối cùng lại đi vào ngõ cụt vì dự đoán sai nhu cầu thực tế.

Đây chính là sai lầm của thời kỳ phát triển phần mềm kiểu cũ, nơi:

- Nhóm phân tích dành 6 tháng viết tài liệu yêu cầu.
- Nhóm kiến trúc dành 6 tháng lên thiết kế hệ thống.
- Sau vài năm, dự án thất bại vì không đáp ứng đúng nhu cầu thực tế.

Chuyện này vẫn đang xảy ra!

Agile dạy chúng ta rằng phần mềm nên được phát triển theo hướng lặp lại (iterative), bắt đầu nhỏ, nhận phản hồi nhanh từ người dùng, rồi tiến hóa dần dần. **TDD giúp thực thi triết lý này trong code**.

### Phát triển theo từng bước nhỏ

- Viết một test nhỏ cho một hành vi cụ thể của hệ thống.
- Chạy test để đảm bảo nó fail (red).
- Viết đúng lượng code tối thiểu để làm test pass (green).
- Refactor
- Lặp lại

Lợi ích của việc làm theo TDD:

- Giúp bạn không đi lạc hướng, luôn tập trung vào giá trị thực tế.
- Phản hồi nhanh chóng, dễ dàng điều chỉnh thiết kế khi cần.
- Dần hình thành tư duy phát triển theo hướng hành vi (behavior-driven).
- Khi đã quen, bạn sẽ thấy cảm giác khó chịu nếu hệ thống không “xanh” vì đó là dấu hiệu bạn có thể đang đi sai hướng.

## Tổng kết 

- Sức mạnh của phần mềm là nó có thể thay đổi. _Hầu hết_ phần mềm sẽ cần thay đổi theo thời gian theo những cách không thể đoán trước. Nhưng đừng cố gắng thiết kế quá phức tạp ngay từ đầu, vì dự đoán tương lai là điều rất khó.
- Thay vào đó, hãy tập trung làm cho phần mềm dễ thay đổi (malleable). Nếu không refactor liên tục khi phần mềm phát triển, nó sẽ trở thành một mớ hỗn độn.
- Bộ test tốt giúp bạn refactor nhanh hơn và ít căng thẳng hơn.
- Viết unit test tốt là một bài toán thiết kế. Hãy tổ chức code theo các đơn vị (unit) có ý nghĩa, có thể kết hợp như các viên gạch Lego.
- TDD giúp bạn phát triển phần mềm một cách có tổ chức, từng bước nhỏ, được hỗ trợ bởi test để đảm bảo phần mềm luôn sẵn sàng cho những thay đổi trong tương lai.
