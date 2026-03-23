# Bước Refactoring, danh sách kiểm tra khởi đầu

Refactoring là một kỹ năng mà khi thực hành đủ nhiều, sẽ trở thành bản năng tự nhiên và khá dễ dàng trong hầu hết các trường hợp.

Nhiều người nhầm lẫn refactoring với việc thay đổi lớn về thiết kế, nhưng thực ra chúng hoàn toàn khác nhau. Phân biệt rõ ràng giữa refactoring và các hoạt động lập trình khác giúp bạn làm việc có kỷ luật và mạch lạc hơn.

## Refactoring so với các hoạt động khác

Refactoring đơn giản là cải thiện chất lượng code hiện có mà <u>không làm thay đổi hành vi</u>. Do đó, các bài test không cần phải thay đổi.

Đây là lý do refactoring là bước thứ 3 trong chu trình TDD. Khi bạn đã thêm một hành vi mới và viết test để kiểm chứng, bước refactoring tiếp theo không được đòi hỏi thay đổi code test. **Nếu bạn phải sửa cả test khi refactor, thì bạn đang làm việc khác chứ không phải refactoring.**

Nhiều thao tác refactoring rất dễ học và thực hiện (IDE thường tự động hóa phần lớn), nhưng theo thời gian, chúng tạo ra tác động rất lớn đến chất lượng hệ thống.

### Các hoạt động khác, ví dụ như thiết kế "lớn"

> Vậy có phải tôi không thay đổi hành vi "thực sự", nhưng tôi vẫn phải thay đổi các bài tests của mình? Quá trình đó gọi là gì thế?

Hãy giả dụ bạn đang làm việc trên một kiểu type và muốn cải thiện chất lượng code của nó. *Hoạt động refactoring không nên bắt buộc bạn phải thay đổi các bài test*, vì vậy bạn không được phép:

- Thay đổi hành vi của chương trình
- Thay đổi chữ ký của method (method signatures)

...vì các bài tests của bạn đang bị phụ thuộc (coupled) vào hai điều trên, nhưng bạn lại có thể:

- Giới thiệu các methods, fields dạng private hay thậm chí là các kiểu types & interfaces mới.
- Thay đổi cấu trúc code nội bộ của public methods.

Điều gì xảy ra nếu bạn muốn thay đổi chữ ký của một method?

```go
func (b BirthdayGreeter) WishHappyBirthday(age int, firstname, lastname string, email Email) {
	// some fascinating emailing code
}
```

Bạn có thể cảm thấy danh sách tham số của nó quá dài và muốn mang lại tính gắn kết và ngữ nghĩa tốt hơn cho đoạn code.

```go
func (b BirthdayGreeter) WishHappyBirthday(person Person)
```

Lúc này bạn đang bước vào quá trình **thiết kế**, và cần đi từng bước cẩn thận. Nếu không có kỷ luật, bạn có thể gây rối cho code, test, *và* cả những phần phụ thuộc vào nó — hãy nhớ rằng không chỉ test mà cả code chính thức cũng đang gọi `WishHappyBirthday`.

**Bạn vẫn có thể thực hiện thay đổi này theo hướng viết test trước.** Có thể tranh luận về việc đây có phải là thay đổi "hành vi" hay không, nhưng rõ ràng bạn đang muốn method hoạt động khác đi.

Vì đây là thay đổi hành vi, hãy áp dụng TDD. TDD mang lại một quy trình đơn giản, an toàn và lặp lại được để thay đổi hành vi hệ thống. Tại sao lại bỏ qua nó chỉ vì nó *cảm giác* khác?

Trong trường hợp này, bạn sẽ sửa các test hiện có để sử dụng type mới. Các bước nhỏ, lặp lại quen thuộc trong TDD — giúp giảm rủi ro và giữ kỷ luật — sẽ rất hữu ích ở đây.

Có thể bạn có nhiều test đang gọi `WishHappyBirthday`. Trong trường hợp này, hãy comment tất cả trừ một test, thực hiện thay đổi cho test đó, rồi mở lại và cập nhật từng test còn lại.

### Big design

Thiết kế có thể đòi hỏi những thay đổi lớn hơn và nhiều thảo luận hơn, thường mang tính chủ quan. Thay đổi cấu trúc nhiều thành phần trong hệ thống thường tốn thời gian hơn refactoring. Tuy nhiên, bạn vẫn nên cố gắng giảm rủi ro bằng cách chia nhỏ quá trình thành nhiều bước.

### Nhìn rừng chứ không chỉ nhìn cây

> [Thành ngữ "can't see the wood for the trees" nghĩa là quá tập trung vào chi tiết nhỏ mà quên mất bức tranh tổng thể.](https://www.collinsdictionary.com/dictionary/english/cant-see-the-wood-for-the-trees)

Thảo luận về thiết kế tổng thể sẽ dễ hơn rất nhiều khi **code đã được tổ chức tốt**. Nếu mỗi lần mở file, bạn và đồng nghiệp đều phải mất thời gian giải mã đống code rối, thì làm sao nghĩ được về thiết kế tổng thể?

Đó là lý do **refactoring liên tục đóng vai trò quan trọng trong TDD**. Nếu bỏ qua các vấn đề thiết kế nhỏ, bạn sẽ rất khó hình dung được bức tranh toàn cảnh của hệ thống.

Code có cấu trúc kém sẽ xuống cấp theo cấp số nhân khi các kỹ sư tiếp tục thêm tính năng phức tạp lên một nền tảng không vững.

## Khởi đầu danh sách kiểm tra trong tâm trí (mental-checklist)

**Hãy hình thành thói quen lướt qua một danh sách checklist trong đầu ở mỗi vòng lặp TDD.** Càng thực hành nhiều, việc này sẽ càng trở nên tự nhiên. **Đây là kỹ năng cần được rèn luyện.** Hãy nhớ rằng, mỗi bước refactoring này đều không yêu cầu bất kỳ thay đổi nào trong các bài test của bạn.

Tôi đã đính kèm các phím tắt dành cho IntelliJ/GoLand, bộ công cụ mà tôi và đội nhóm đang sử dụng. Mỗi khi hướng dẫn các kỹ sư mới, tôi thường nhấn mạnh rằng họ nên tập thành thói quen sử dụng phím tắt và thành thạo các công cụ refactor để thao tác nhanh chóng và an toàn.

### Inline variables

Nếu bạn khai báo một biến chỉ để truyền nó vào một method/function khác:

```go
url := baseURL + "/user/" + id
res, err := client.Get(url)
```

Hãy cân nhắc gộp biến đó (`command+option+n`), *trừ khi* tên biến mang thêm một lớp ý nghĩa quan trọng.

```go
res, err := client.Get(baseURL + "/user/" + id)
```

Đừng lạm dụng inline quá mức. Mục tiêu không phải là loại bỏ mọi biến để rồi tạo ra những dòng code dài ngoằng khó đọc. Nếu bạn tìm được một cái tên mô tả đúng ý nghĩa cho giá trị đó, thì giữ nguyên biến có lẽ là lựa chọn tốt hơn.

### DRY up values with extract variables

"Đừng lặp lại chính mình" - Don't Repeat Yourself (DRY). Nếu cùng một giá trị xuất hiện nhiều lần trong code, hãy cân nhắc trích xuất nó thành một biến có tên mô tả rõ ràng (`command+option+v`).

Việc này không chỉ cải thiện readability của code, mà còn giúp việc cập nhật giá trị đó dễ dàng hơn nhiều, vì bạn chỉ cần sửa ở một chỗ duy nhất thay vì tìm và sửa ở nhiều nơi.

### DRY trong trường hợp tổng quát

[DRY (Đừng lặp lại chính mình)](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) ngày nay nhận nhiều ý kiến tiêu cực, và điều đó có lý do của nó. DRY là một khái niệm rất dễ bị hiểu sai ở mức độ hời hợt và sau đó bị áp dụng sai.

Một lập trình viên có thể áp dụng DRY đi quá xa, tạo ra những lớp trừu tượng phức tạp khó hiểu chỉ để tiết kiệm vài dòng code, thay vì ý tưởng *thực sự* của DRY là đặt một _ý niệm_ ở một nơi duy nhất. Việc giảm số lượng dòng code thường chỉ là hiệu ứng phụ của DRY, **chứ đó không phải là mục tiêu thực tế**.

Đúng, DRY có thể bị áp dụng sai, nhưng ngược lại — từ chối DRY mọi thứ — cũng tệ không kém. Code lặp lại tạo thêm nhiễu và tăng chi phí bảo trì. Không gom những khái niệm liên quan lại với nhau chỉ vì sợ dùng sai DRY sẽ gây ra những vấn đề *khác*.

Thay vì cực đoan — "phải DRY mọi thứ" hay "DRY là tệ" — hãy nhìn vào code trước mắt. Cái gì đang lặp lại? Có cần thiết không? Nếu gom vào một method, danh sách tham số có hợp lý không? Code mới có tự giải thích được ý nghĩa không?

Chín phần mười trường hợp, bạn có thể nhìn vào danh sách tham số của một hàm, và nếu nó trông rối rắm khó hiểu, thì rất có thể đó là một cách áp dụng DRY tồi.

Nếu bạn cảm thấy việc làm cho code DRY trở nên khó khăn, có thể bạn đang làm cho mọi thứ phức tạp hơn; hãy cân nhắc dừng lại.

Hãy DRY một cách cẩn thận, **nhưng thực hành nó thường xuyên sẽ cải thiện khả năng đánh giá của bạn**. Tôi khuyến khích các đồng nghiệp của mình "cứ thử xem" và sử dụng source control để quay lại trạng thái an toàn nếu nó sai.

<u>**Tự mình thử nghiệm những điều này sẽ dạy bạn nhiều bài học hơn là chỉ ngồi thảo luận về nó**</u>, và source control kết hợp với các automated test tốt sẽ mang đến cho bạn một môi trường hoàn hảo để thử nghiệm và học hỏi.

### Extract các Magic values.

> [Những giá trị đơn lẻ với ý nghĩa không rõ ràng hoặc lặp lại nhiều lần, (tốt nhất là) nên được thay thế bằng những hằng số có tên gọi (named constants).](https://en.wikipedia.org/wiki/Magic_number_(programming))

Sử dụng chức năng trích xuất biến (`command+option+v`) hoặc trích xuất hằng số (`command+option+c`) để đặt tên cho các giá trị ma thuật. Bạn có thể coi đây là thao tác ngược lại của việc inlining. Tôi thường xuyên chuyển đổi qua lại giữa inline và extract để xem cách viết nào thì dễ đọc hơn.

Hãy nhớ rằng trích xuất các giá trị lặp lại cũng tạo ra một mức độ _phụ thuộc (coupling)_. Mọi thứ đang dùng chung giá trị đó bây giờ sẽ bị liên kết với nhau. Xem xét đoạn code sau:

```go
func main() {
	api1Client := http.Client{
		Timeout: 1 * time.Second,
	}
	api2Client := http.Client{
		Timeout: 1 * time.Second,
	}
	api3Client := http.Client{
		Timeout: 1 * time.Second,
	}
	//etc
}
```

Chúng ta đang thiết lập một vài HTTP clients cho chương trình của mình. Đang có vài _giá trị ma thuật_ ở đây, và chúng ta có thể DRY thuộc tính `Timeout` bằng cách trích xuất (extract) một biến mới và đặt cho nó một cái tên có nghĩa.

![Ảnh chụp màn hình lúc tôi trích xuất biến](https://i.imgur.com/4sgUG7L.png)

Bây giờ đoạn code trông như thế này:

```go
func main() {
	timeout := 1 * time.Second
	api1Client := http.Client{
		Timeout: timeout,
	}
	api2Client := http.Client{
		Timeout: timeout,
	}
	api3Client := http.Client{
		Timeout: timeout,
	}
	// etc..
}
```

Chúng ta không còn giá trị ma thuật nữa; chúng ta đã gắn cho nó một cái tên mang ý nghĩa, nhưng chúng ta cũng vô tình làm cho cả ba clients này **chia sẻ chung một biến timeout**. Điều đó _có thể_ là điều bạn mong muốn; refactoring thường phụ thuộc vào hoàn cảnh riêng, nhưng đó là một điểm mà bạn nên chú ý.

Nếu bạn sành sỏi cách sử dụng IDE của mình, bạn có thể thực hiện refactor gộp biến ngược lại (_inline_) để cho phép các clients có giá trị `Timeout` riêng biệt một lần nữa.

### Giúp cho public methods/functions dễ đọc lướt

Code của bạn có đang chứa những public methods hoặc functions quá dài dòng không?

Hãy bao bọc các thao tác trên vào trong những methods/functions cấu trúc private thông qua công cụ refactor extract method (`command+option+m`).

Đoạn code bên dưới có một số chi tiết rườm rà: tạo chuỗi JSON, chuyển thành `io.Reader`, rồi gửi qua HTTP `POST`.

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	payload := []byte(`{"name": "` + name + `"}`)

	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer(payload),
	)
	//todo: handle codes, err etc
}
```

Đầu tiên, sử dụng chức năng inline variable (`command+option+n`) để gộp `payload` vào quá trình tạo buffer.

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer([]byte(`{"name": "`+name+`"}`)),
	)
	// etc
}
```

Bây giờ, chúng ta có thể trích xuất quá trình tạo JSON payload vào một hàm mới bằng cách sử dụng công cụ refactor extract method (`command+option+m`) để loại bỏ sự nhiễu loạn khỏi method này.

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		createWidgetPayload(name),
	)
	// etc
}
```

Các public methods và functions nên mô tả *những gì* chúng làm (what) thay vì *làm như thế nào* (how).

> **Bất cứ khi nào tôi phải suy nghĩ để hiểu code đang làm gì, tôi sẽ tự hỏi bản thân liệu tôi có thể refactor lại code để giúp cho sự hiểu biết đó trở nên rõ ràng hơn ngay lập tức không**

-- Martin Fowler

Điều này giúp bạn hiểu thiết kế tổng thể tốt hơn, và sau đó nó cho phép bạn đặt câu hỏi về các trách nhiệm:

> Tại sao method này lại làm X? Đáng lẽ nó phải nằm trong Y chứ?

> Tại sao method này lại làm quá nhiều nhiệm vụ? Chúng ta có thể gộp chúng lại ở nơi khác không?

Các private functions và methods rất tuyệt vời; chúng cho phép bạn gói gọn những cái "như thế nào" (how) không liên quan thành những cái "làm cái gì" (what).

#### Nhưng bây giờ tôi không biết nó hoạt động như thế nào!

Một phản đối phổ biến với cách refactoring này — chia thành nhiều function/method nhỏ — là nó có thể khó theo dõi luồng code. Câu trả lời thẳng thắn của tôi là:

> Bạn đã học cách điều hướng codebases bằng công cụ của mình một cách hiệu quả chưa?

Với tư cách người viết `CreateWidget`, tôi cố ý không muốn việc tạo chuỗi JSON chiếm spotlight trong method này. Nó gây phân tâm và 99% thời gian không phải thứ người đọc cần quan tâm.

Tuy nhiên, nếu ai đó _thực sự_ quan tâm, bạn chỉ cần nhấn `command+b` (hoặc phím tắt "điều hướng đến ký hiệu" (navigate to symbol) của bạn) trên `createWidgetPayload`... và đọc nó. Nhấn `command+left-arrow` để quay lại.

### Đưa quá trình tạo giá trị về construction time.

Các methods thường xuyên phải tạo giá trị và dùng chúng, giống như biến `url` trong method `CreateWidget` của chúng ta ở trên.

```go
type WidgetService struct {
	baseURL string
	client  *http.Client
}

func NewWidgetService(baseURL string) *WidgetService {
	client := http.Client{
		Timeout: 10 * time.Second,
	}
	return &WidgetService{baseURL: baseURL, client: &client}
}

func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		createWidgetPayload(name),
	)
	// etc
}
```

Một kỹ thuật refactoring bạn có thể áp dụng ở đây là, nếu một giá trị đang được tạo ra mà **không phụ thuộc vào các tham số truyền vào của method**, thì bạn có thể thiết lập một _field_ trong type của bạn và tính toán nó tại constructor function.

```go
type WidgetService struct {
	client          *http.Client
	createWidgetURL string
}

func NewWidgetService(baseURL string) *WidgetService {
	client := http.Client{
		Timeout: 10 * time.Second,
	}
	return &WidgetService{
		createWidgetURL: baseURL + "/widgets",
		client:          &client,
	}
}

func (ws *WidgetService) CreateWidget(name string) error {
	req, err := http.NewRequest(
		http.MethodPost,
		ws.createWidgetURL,
		createWidgetPayload(name),
	)
	// etc
}
```

Bằng cách đưa chúng về thời điểm khởi tạo, bạn có thể tinh giản các method của mình.

#### So sánh và đối chiếu hàm `CreateWidget`

Bắt đầu với

```go
func (ws *WidgetService) CreateWidget(name string) error {
	url := ws.baseURL + "/widgets"
	payload := []byte(`{"name": "` + name + `"}`)
	req, err := http.NewRequest(
		http.MethodPost,
		url,
		bytes.NewBuffer(payload),
	)
	// etc
}

```

Với một vài bước refactor cơ bản, hầu như hoàn toàn được thúc đẩy bằng cách sử dụng công cụ tự động, chúng ta thu được kết quả:

```go
func (ws *WidgetService) CreateWidget(name string) error {
	req, err := http.NewRequest(
		http.MethodPost,
		ws.createWidgetURL,
		createWidgetPayload(name),
	)
	// etc
}
```

Đây là một cải thiện nhỏ, nhưng rõ ràng nó dễ đọc hơn. Nếu bạn thực hành thành thạo, kiểu cải thiện này sẽ tốn của bạn chưa tới một phút, và miễn là bạn áp dụng TDD tốt, bạn sẽ có lưới an toàn của các bài test để đảm bảo bạn không làm hỏng bất cứ thứ gì. Những cải tiến nhỏ liên tục này là rất quan trọng đối với sức khỏe lâu dài của một codebase.

### Cố gắng loại bỏ các comments.

> Một nguyên tắc nền tảng mà chúng tôi tuân theo là bất cứ khi nào chúng tôi cảm thấy cần phải comment một điều gì đó, thay vào đó chúng tôi sẽ viết một method.

-- Martin Fowler

Một lần nữa, refactor trích xuất method có thể là người bạn của bạn ở đây.

## Ngoại lệ của quy tắc

Có những sự cải tiến cho code yêu cầu sự thay đổi trong các bài tests của bạn, mà tôi vẫn sẽ rất sẵn lòng xếp vào nhóm "refactoring", mặc dù nó phá vỡ quy tắc.

Một ví dụ đơn giản sẽ là việc đổi tên một public symbol (ví dụ: một method, type, hoặc function) bằng `shift+F6`. Điều này tất nhiên sẽ thay đổi cả code production và test.

Tuy nhiên, vì nó là một sự thay đổi **tự động và an toàn (automated and safe)**, rủi ro rơi vào vòng xoáy của việc phá vỡ code production và test mà rất nhiều người gặp phải với các loại cập nhật *thiết kế* khác là rất nhỏ.

Vì lý do đó, bất kỳ thay đổi nào mà bạn có thể thực hiện một cách an toàn bằng IDE/editor của mình, tôi vẫn sẽ vui vẻ gọi là refactoring.

## Sử dụng các công cụ của bạn để giúp bạn luyện tập refactoring.

- Bạn nên chạy các unit tests của mình mỗi khi bạn thực hiện một trong những thay đổi nhỏ này. Chúng ta đầu tư thời gian vào việc làm cho code có thể unit-test được, và feedback loop kéo dài vài phần nghìn giây là một trong những lợi ích lớn nhất; hãy sử dụng nó!
- Dựa vào source control. Bạn không nên cảm thấy ngại ngùng trong việc thử nghiệm các ý tưởng. Nếu bạn vui vẻ, hãy commit nó; nếu không, hãy revert. Việc này nên mang lại cảm giác thoải mái và dễ dàng chứ không phải chuyện to tát.
- Càng tận dụng tốt unit tests và source control, bạn càng dễ dàng *thực hành* refactoring. Khi bạn làm chủ được kỷ luật này, **kỹ năng thiết kế của bạn sẽ tăng lên nhanh chóng** bởi vì bạn có một vòng lặp phản hồi và lưới an toàn đáng tin cậy và hiệu quả.
- Rất nhiều lần trong sự nghiệp của mình, tôi vô tình nghe thấy các developer phàn nàn về việc không có thời gian để refactor; nhưng thật không may, rõ ràng lý do khiến họ tốn quá nhiều thời gian là vì họ làm việc đó không có kỷ luật - và họ đã chưa thực hành đủ nhiều.
- Mặc dù tốc độ đánh máy không bao giờ là điểm nghẽn, bạn vẫn phải có khả năng sử dụng bất cứ editor/IDE nào của mình để refactor một cách nhanh chóng và an toàn. Ví dụ, nếu công cụ của bạn không cho phép bạn extract variable chỉ bằng một phím bấm, bạn sẽ làm việc đó ít đi bởi vì nó tốn nhiều công sức và có độ rủi ro cao.

## Đừng xin phép để refactor

Refactoring nên là một sự xuất hiện thường xuyên trong công việc của bạn, thứ gì đó mà bạn thực hiện liên tục. Nó cũng không nên là một cái hố đen hút thời gian (time-sink), đặc biệt nếu nó được thực hiện với số lượng ít ỏi và diễn ra thường xuyên.

Nếu bạn không refactor, chất lượng nội bộ của bạn sẽ bị ảnh hưởng, năng suất của đội sẽ giảm xuống và áp lực sẽ gia tăng.

Martin Fowler có thêm một câu quote rất tuyệt vời cho chúng ta.

> Mặc dù vậy, ngoài trừ trường hợp bạn đang ở rất gần một deadline, bạn không bao giờ nên trì hoãn việc refactoring viện cớ mình không có thời gian. Kinh nghiệm từ một vài dự án đã chỉ ra rằng một đợt refactoring dài sẽ dẫn đến việc tăng mạnh năng suất (productivity). Không có đủ quỹ thời gian thường lại chính là một dấu hiệu chứng tỏ rằng bạn cần phải tiến hành refactoring ngay.

## Tổng kết

Đây không phải là một danh sách toàn diện, chỉ là một khởi đầu. Mời tìm đọc quyển sách Refactoring (ấn bản 2) của Martin Fowler để trở thành một chuyên gia (pro).

Refactoring nên diễn ra cực kỳ nhanh chóng và an toàn khi bạn đã thực hành thuần thục, vì thế không có một lý do biện hộ nào để từ chối không làm cả. Có quá nhiều người coi refactoring là một quyết định cần những người khác duyệt qua thay vì coi nó như một phần kỹ năng phải học đưa vào làm một phần công việc thường nhật.

Chúng ta nên luôn hướng tới việc bàn giao hệ thống code trong một trạng thái *mẫu mực*.

Refactoring tốt giúp code dễ đọc hiểu hơn. Khi code dễ hiểu, các design trở nên dễ phát hiện và nắm bắt hơn. Rất khó để nhìn ra thiết kế tốt trong những hệ thống chứa đầy function khổng lồ, code trùng lặp không cần thiết, và deep nesting. **Việc refactor nhỏ nhưng thường xuyên là yếu tố thiết yếu cho một thiết kế tốt hơn.**

