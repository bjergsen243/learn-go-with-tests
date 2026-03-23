# Các Anti-pattern trong TDD

Thỉnh thoảng, việc xem lại các kỹ thuật TDD của bạn và tự nhắc nhở bản thân về những hành vi cần tránh là điều rất cần thiết.

Quy trình TDD về lý thuyết rất đơn giản, nhưng thực hành sẽ thử thách kỹ năng thiết kế của bạn. **Đừng nhầm: TDD không khó — thiết kế mới là thứ khó!**

Chương này liệt kê một số anti-pattern (mẫu ngược/lỗi thiết kế) trong TDD và kiểm thử, cũng như cách khắc phục chúng.

(Người dịch) Các bạn có thể đọc thêm về anti-pattern [tại đây](https://en.wikibooks.org/wiki/Introduction_to_Software_Engineering/Architecture/Anti-Patterns) để hiểu thêm về nó.

## Không thực hiện TDD chút nào

Bạn hoàn toàn có thể viết phần mềm tốt mà không cần TDD, nhưng nhiều vấn đề về thiết kế code và chất lượng test mà tôi từng thấy sẽ rất khó xảy ra nếu TDD được áp dụng có kỷ luật.

Một trong những thế mạnh của TDD là nó mang lại cho bạn một quy trình chính thức để chia nhỏ các vấn đề, hiểu những gì bạn đang cố gắng đạt được (đỏ - red), hoàn thành nó (xanh - green), sau đó suy nghĩ kỹ về cách làm cho nó đúng (xanh dương/tái cấu trúc - blue/refactor).

Nếu không có điều này, quy trình thường mang tính tạm thời và lỏng lẻo, điều này _có thể_ làm cho việc kỹ thuật trở nên khó khăn hơn mức cần thiết.

## Hiểu sai các ràng buộc của bước tái cấu trúc (refactoring)

Tôi đã tham gia một số hội thảo, các buổi mobbing hoặc pairing, nơi ai đó đã làm cho một bài kiểm thử vượt qua và đang ở giai đoạn tái cấu trúc. Sau khi suy nghĩ, họ thấy rằng sẽ tốt nếu trừu tượng hóa một số mã nguồn vào một struct mới; một người mới bắt đầu cầu toàn hét lên:

> Bạn không được phép làm điều này! Bạn nên viết một bài kiểm thử cho việc này trước, chúng ta đang làm TDD mà!

Đây dường như là một sự hiểu lầm phổ biến. **Bạn có thể làm bất cứ điều gì bạn thích với mã nguồn khi các bài kiểm thử đang ở trạng thái xanh**, điều duy nhất bạn không được phép làm là **thêm hoặc thay đổi hành vi**.

Mục đích của các bài kiểm thử này là mang lại cho bạn sự _tự do để tái cấu trúc_, tìm ra các mức trừu tượng phù hợp và làm cho mã nguồn dễ thay đổi và dễ hiểu hơn.

## Có các bài kiểm thử không bao giờ thất bại (hay còn gọi là evergreen tests)

Thật đáng kinh ngạc khi điều này lại xảy ra thường xuyên như vậy. Bạn bắt đầu gỡ lỗi hoặc thay đổi một số bài kiểm thử và nhận ra: không có kịch bản nào mà bài kiểm thử này có thể thất bại. Hoặc ít nhất, nó sẽ không thất bại theo cách mà bài kiểm thử _đáng lẽ_ phải bảo vệ.

Điều này _gần như không thể_ xảy ra với TDD nếu bạn tuân thủ **bước đầu tiên**:

> Viết test, xem nó fail

Vấn đề này hầu như chỉ xảy ra khi lập trình viên viết test _sau khi_ viết code, hoặc chạy theo test coverage thay vì tạo bộ test thực sự hữu ích.

## Các khẳng định vô ích (Useless assertions)

Bạn đã bao giờ làm việc trên một hệ thống, và bạn làm hỏng một bài kiểm thử, sau đó bạn thấy thông báo này chưa?

> `false was not equal to true` (false không bằng true)

Tôi biết rằng false không bằng true. Đây không phải là một thông báo hữu ích; nó không cho tôi biết tôi đã làm hỏng cái gì. Đây là triệu chứng của việc không tuân thủ quy trình TDD và không đọc thông báo lỗi khi thất bại.

Quay lại bảng vẽ nào,

> Viết một bài kiểm thử, thấy nó thất bại (và đừng xấu hổ về thông báo lỗi)

## Khẳng định trên các chi tiết không liên quan

Một ví dụ về điều này là thực hiện một khẳng định trên một đối tượng phức tạp, trong khi trên thực tế tất cả những gì bạn quan tâm trong bài kiểm thử là giá trị của một trong các trường.

```go
// đừng làm thế này, giờ đây bài kiểm thử của bạn bị ràng buộc chặt chẽ với toàn bộ đối tượng
if !cmp.Equal(complexObject, want) {
	t.Error("got %+v, want %+v", complexObject, want)
}

// hãy cụ thể, và nới lỏng sự ràng buộc
got := complexObject.fieldYouCareAboutForThisTest
if got != want {
	t.Error("got %q, want %q", got, want)
}
```

Các khẳng định bổ sung không chỉ làm cho bài kiểm thử của bạn khó đọc hơn bằng cách tạo ra "tiếng nhiễu" trong tài liệu của bạn, mà còn ràng buộc bài kiểm thử với các dữ liệu mà nó không quan tâm một cách vô ích. Điều này có nghĩa là nếu bạn tình cờ thay đổi các trường cho đối tượng của mình, hoặc cách chúng hoạt động, bạn có thể gặp các lỗi biên dịch hoặc thất bại không mong muốn với các bài kiểm thử của mình.

Đây là một ví dụ về việc không tuân thủ giai đoạn đỏ (red) một cách đủ nghiêm ngặt.

- Để một thiết kế hiện có ảnh hưởng đến cách bạn viết bài kiểm thử **thay vì nghĩ về hành vi mong muốn**.
- Không cân nhắc đầy đủ thông báo lỗi của bài kiểm thử khi thất bại.

## Có quá nhiều khẳng định trong một kịch bản đơn lẻ cho unit test

Quá nhiều khẳng định có thể làm cho các bài kiểm thử khó đọc và khó gỡ lỗi khi chúng thất bại.

Chúng thường len lỏi vào một cách dần dần, đặc biệt nếu việc thiết lập (setup) bài kiểm thử phức tạp vì bạn ngại phải lặp lại cùng một thiết lập tồi tệ đó để khẳng định về một thứ khác. Thay vì làm điều này, bạn nên khắc phục các vấn đề trong thiết kế của mình - những thứ đang làm cho việc khẳng định các điều mới trở nên khó khăn.

Một nguyên tắc chung hữu ích là đặt mục tiêu thực hiện một khẳng định cho mỗi bài kiểm thử. Trong Go, hãy tận dụng các bài kiểm thử con (subtests) để phân định rõ ràng giữa các khẳng định trong những trường hợp bạn cần. Đây cũng là một kỹ thuật tiện lợi để tách biệt các khẳng định về hành vi so với chi tiết triển khai.

Đối với các bài kiểm thử khác mà thời gian thiết lập hoặc thực thi có thể là một ràng buộc (ví dụ: một bài kiểm thử chấp nhận - acceptance test điều khiển trình duyệt web), bạn cần cân nhắc ưu và nhược điểm của việc các bài kiểm thử hơi khó gỡ lỗi hơn một chút so với thời gian thực thi bài kiểm thử.

## Không lắng nghe các bài kiểm thử của bạn

[Dave Farley trong video của mình "When TDD goes wrong"](https://www.youtube.com/watch?v=UWtEVKVPBQ0&feature=youtu.be) đã chỉ ra rằng,

> TDD mang lại cho bạn phản hồi nhanh nhất có thể về thiết kế của bạn

Từ kinh nghiệm của bản thân, rất nhiều lập trình viên đang cố gắng thực hành TDD nhưng thường xuyên bỏ qua các tín hiệu gửi ngược lại cho họ từ quy trình TDD. Vì vậy, họ vẫn bị mắc kẹt với các hệ thống mỏng manh, gây khó chịu, với một bộ kiểm thử kém chất lượng.

Nói một cách đơn giản, nếu việc kiểm thử mã nguồn của bạn là khó khăn, thì việc _sử dụng_ mã nguồn của bạn cũng sẽ khó khăn. Hãy coi các bài kiểm thử là người dùng đầu tiên của mã nguồn và sau đó bạn sẽ thấy mã của mình có dễ chịu khi làm việc cùng hay không.

Tôi đã nhấn mạnh điều này rất nhiều trong cuốn sách, và tôi sẽ nói lại lần nữa: **hãy lắng nghe các bài kiểm thử của bạn**.

### Thiết lập quá mức, quá nhiều test double, v.v.

Bạn đã bao giờ nhìn vào một bài kiểm thử với 20, 50, 100, 200 dòng mã thiết lập (setup) trước khi có bất kỳ điều gì thú vị trong bài kiểm thử xảy ra chưa? Sau đó, bạn có phải thay đổi mã nguồn và xem lại cái mớ hỗn độn đó và ước rằng mình đã chọn một nghề nghiệp khác không?

Các tín hiệu ở đây là gì? _Hãy lắng nghe_, các bài kiểm thử phức tạp `==` mã nguồn phức tạp. Tại sao mã của bạn lại phức tạp? Nó có nhất thiết phải như vậy không?

- Khi bạn có nhiều test double trong các bài kiểm thử, điều đó có nghĩa là mã bạn đang kiểm thử có nhiều phụ thuộc - nghĩa là thiết kế của bạn cần được cải thiện.
- Nếu bài kiểm thử của bạn dựa vào việc thiết lập các tương tác khác nhau với mock, điều đó có nghĩa là mã của bạn đang thực hiện nhiều tương tác với các phụ thuộc của nó. Hãy tự hỏi liệu những tương tác này có thể đơn giản hơn không.

#### Giao diện bị rò rỉ (Leaky interfaces)

Nếu bạn khai báo một `interface` có nhiều phương thức, đó là dấu hiệu của leaky abstraction. Hãy nghĩ xem có thể gom lại thành ít phương thức hơn không — lý tưởng là chỉ một.

#### Ô nhiễm giao diện (Interface pollution)

Như một câu châm ngôn của Go đã nói, *giao diện càng lớn, sự trừu tượng càng yếu*. Nếu bạn để lộ một giao diện khổng lồ cho người dùng gói của mình, bạn buộc họ phải tạo trong các bài kiểm thử của họ một stub/mock khớp với toàn bộ API, cung cấp triển khai cho cả các phương thức mà họ không sử dụng (đôi khi, họ chỉ gọi panic để làm rõ rằng chúng không nên được sử dụng). Tình huống này là một anti-pattern được gọi là [ô nhiễm giao diện (interface pollution)](https://rakyll.org/interface-pollution/) và đây là lý do tại sao thư viện chuẩn chỉ cung cấp cho bạn những giao diện rất nhỏ.

Thay vào đó, bạn nên để lộ từ gói của mình một struct thuần túy với tất cả các phương thức liên quan được export, nhường cho các khách hàng (clients) sử dụng API của bạn sự tự do để khai báo các giao diện của riêng họ trừu tượng hóa trên tập hợp con các phương thức mà họ cần: ví dụ [go-redis](https://github.com/redis/go-redis) để lộ một struct (`redis.Client`) cho các khách hàng API.

Nói chung, bạn chỉ nên để lộ một giao diện cho khách hàng khi:
- Giao diện bao gồm một tập hợp các hàm nhỏ và mạch lạc.
- Giao diện và triển khai của nó cần được tách rời (ví dụ: vì người dùng có thể chọn giữa nhiều triển khai hoặc họ cần mock một phụ thuộc bên ngoài).

#### Suy nghĩ về các loại test double bạn sử dụng

- Mock đôi khi hữu ích, nhưng chúng cực kỳ mạnh mẽ và do đó dễ bị lạm dụng. Hãy thử tự đặt ra hạn chế cho bản thân là chỉ sử dụng stub.
- Việc xác thực chi tiết triển khai bằng các spy (gián điệp) đôi khi hữu ích, nhưng hãy cố gắng tránh nó. Hãy nhớ rằng chi tiết triển khai của bạn thường không quan trọng, và bạn không muốn các bài kiểm thử của mình bị ràng buộc vào chúng nếu có thể. Hãy tìm cách ràng buộc bài kiểm thử của bạn vào **hành vi hữu ích thay vì các chi tiết ngẫu nhiên**.
- [Đọc các bài viết của tôi về việc đặt tên test double chính xác](https://quii.dev/Start_naming_your_test_doubles_correctly) nếu việc phân loại test double còn hơi mơ hồ với bạn.

#### Hợp nhất các phụ thuộc (Consolidate dependencies)

Dưới đây là một số mã nguồn cho một `http.HandlerFunc` để xử lý việc đăng ký người dùng mới cho một trang web.

```go
type User struct {
	// Một số trường của người dùng
}

type UserStore interface {
	CheckEmailExists(email string) (bool, error)
	StoreUser(newUser User) error
}

type Emailer interface {
	SendEmail(to User, body string, subject string) error
}

func NewRegistrationHandler(userStore UserStore, emailer Emailer) http.HandlerFunc {
	return func(writer http.ResponseWriter, request *http.Request) {
		// trích xuất người dùng từ request body (xử lý lỗi)
		// kiểm tra người dùng đã tồn tại chưa (xử lý trùng lặp, lỗi)
		// lưu trữ người dùng (xử lý lỗi)
		// soạn và gửi email xác nhận (xử lý lỗi)
		// nếu thực hiện được đến đây, trả về phản hồi 2xx
	}
}
```

Ở lần nhìn đầu tiên, có thể nói rằng thiết kế này không quá tệ. Nó chỉ có 2 phụ thuộc!

Hãy đánh giá lại thiết kế bằng cách xem xét các trách nhiệm của handler:

- Phân tích request body thành một `User`: :white_check_mark:
- Sử dụng `UserStore` để kiểm tra xem người dùng có tồn tại không: :question:
- Sử dụng `UserStore` để lưu trữ người dùng: :question:
- Soạn một email: :question:
- Sử dụng `Emailer` để gửi email: :question:
- Trả về phản hồi http phù hợp, tùy thuộc vào thành công, lỗi, v.v.: :white_check_mark:

Để thực thi mã này, bạn sẽ phải viết nhiều bài kiểm thử với các mức độ thiết lập test double, spy, v.v., khác nhau.

- Điều gì xảy ra nếu các yêu cầu mở rộng? Thêm bản dịch cho các email? Gửi cả tin nhắn SMS xác nhận? Bạn có thấy hợp lý không khi bạn phải thay đổi một HTTP handler để đáp ứng sự thay đổi này?
- Bạn có cảm thấy đúng đắn không khi quy tắc quan trọng "chúng ta nên gửi một email" lại nằm trong một HTTP handler?
    - Tại sao bạn lại phải trải qua quy trình tạo các yêu cầu HTTP và đọc các phản hồi để xác thực quy tắc đó?

**Hãy lắng nghe các bài kiểm thử của bạn**. Việc viết các bài kiểm thử cho mã nguồn này theo phong cách TDD sẽ nhanh chóng khiến bạn cảm thấy khó chịu (hoặc ít nhất, khiến lập trình viên lười biếng trong bạn thấy phiền phức). Nếu nó mang lại cảm giác đau đớn, hãy dừng lại và suy nghĩ.

Sẽ ra sao nếu thiết kế giống như thế này thay vì vậy?

```go
type UserService interface {
	Register(newUser User) error
}

func NewRegistrationHandler(userService UserService) http.HandlerFunc {
	return func(writer http.ResponseWriter, request *http.Request) {
		// phân tích người dùng
		// đăng ký người dùng
		// kiểm tra lỗi, gửi phản hồi
	}
}
```

- Dễ dàng kiểm thử handler ✅
- Các thay đổi đối với các quy tắc xung quanh việc đăng ký được tách biệt khỏi HTTP, vì vậy chúng cũng đơn giản hơn để kiểm thử ✅

## Vi phạm tính đóng gói (Violating encapsulation)

Tính đóng gói (Encapsulation) là rất quan trọng. Có lý do để chúng ta không để mọi thứ trong một gói (package) ở trạng thái export (hoặc public). Chúng ta muốn các API mạch lạc với bề mặt tiếp xúc nhỏ để tránh sự ràng buộc chặt chẽ (tight coupling).

Lập trình viên đôi khi sẽ bị cám dỗ để chuyển một hàm hoặc phương thức thành public nhằm mục đích kiểm thử một thứ gì đó. Bằng cách làm này, bạn làm cho thiết kế của mình trở nên tệ hơn và gửi những thông điệp gây nhầm lẫn cho những người bảo trì và người dùng mã nguồn của bạn.

Kết quả của việc này có thể là lập trình viên cố gắng gỡ lỗi một bài kiểm thử và cuối cùng nhận ra hàm đang được kiểm thử _chỉ được gọi từ các bài kiểm thử_. Đây rõ ràng là **một kết quả tồi tệ và lãng phí thời gian**.

Trong Go, hãy coi vị trí mặc định của bạn khi viết bài kiểm thử là _từ quan điểm của một người tiêu dùng gói của bạn_. Bạn có thể đặt điều này thành một ràng buộc tại thời điểm biên dịch bằng cách để các bài kiểm thử nằm trong một gói kiểm thử, ví dụ `package gocoin_test`. Nếu bạn làm điều này, bạn sẽ chỉ có quyền truy cập vào các thành viên được export của gói, vì vậy bạn sẽ không thể tự ràng buộc mình vào chi tiết triển khai.

## Các bài kiểm thử dạng bảng phức tạp (Complicated table tests)

Các bài kiểm thử dạng bảng (Table tests) là một cách tuyệt vời để thực thi một số kịch bản khác nhau khi thiết lập bài kiểm thử là giống nhau và bạn chỉ muốn thay đổi các đầu vào.

_Nhưng_ chúng có thể trở nên lộn xộn khi đọc và khó hiểu khi bạn cố gắng gượng ép các loại bài kiểm thử khác vào dưới cái danh nghĩa là có một bảng kiểm thử "vĩ đại" duy nhất.

```go
cases := []struct {
	X                int
	Y                int
	Z                int
	err              error
	IsFullMoon       bool
	IsLeapYear       bool
	AtWarWithEurasia bool
}{}
```

**Đừng ngần ngại tách ra khỏi bảng của bạn và viết các bài kiểm thử mới** thay vì thêm các trường và các biến boolean mới vào `struct` của bảng.

Một điều cần ghi nhớ khi viết phần mềm là:

> [Đơn giản không có nghĩa là dễ dàng (Simple is not easy)](https://www.infoq.com/presentations/Simple-Made-Easy/)

"Chỉ cần" thêm một trường vào bảng có thể là dễ dàng, nhưng nó có thể làm cho mọi thứ xa rời sự đơn giản.

## Tổng kết

Hầu hết các vấn đề với unit test thường có thể được truy nguyên từ:

- Lập trình viên không tuân thủ quy trình TDD
- Thiết kế kém

Vì vậy, hãy tìm hiểu về thiết kế phần mềm tốt!

Tin tốt là TDD có thể giúp bạn _cải thiện kỹ năng thiết kế của mình_ như đã nêu ở phần đầu:

**Mục đích chính của TDD là cung cấp phản hồi về thiết kế của bạn.** Lần thứ một triệu, hãy lắng nghe các bài kiểm thử của bạn, chúng đang phản chiếu lại thiết kế của bạn cho chính bạn thấy.

Hãy trung thực về chất lượng các bài kiểm thử của bạn bằng cách lắng nghe các phản hồi mà chúng cung cấp cho bạn, và bạn sẽ trở thành một lập trình viên giỏi hơn nhờ điều đó.
