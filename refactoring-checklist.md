# Bước Refactoring, danh sách kiểm tra khởi đầu

Refactoring (tái cấu trúc code) là một kỹ năng mà khi được thực hành đủ nhiều, sẽ trở thành bản năng thứ hai và trở nên khá dễ dàng trong hầu hết các trường hợp.

Hoạt động này thường bị nhầm lẫn với những sự thay đổi lớn về mặt thiết kế, nhưng thực ra chúng hoàn toàn tách biệt. Việc phân định rạch ròi giữa refactoring và các loại hoạt động lập trình khác là rất hữu ích, vì nó cho phép tôi làm việc với sự rõ ràng và kỷ luật.

## Refactoring so với các hoạt động khác

Refactoring đơn thuần là cải thiện chất lượng mã code hiện có và <u>không làm thay đổi hành vi (behaviour)</u>; do đó, các bài test không cần thiết phải thay đổi.

Đây là lý do tại sao nó lại là bước thứ 3 trong chu trình TDD. Khi bạn đã thêm một hành vi và một bài test để chứng thực cho nó, quá trình refactoring tiếp theo đây phải là một hoạt động không đòi hỏi bất kỳ sự thay đổi nào đối với code bài test. **Bạn đang làm một hoạt động gì đó khác thuật ngữ "refactoring"** nếu bạn đang thay đổi code rồi sau đó bắt buộc phải sửa lại cả bài test cùng một lúc.

Nhiều thao tác refactoring vô cùng hữu ích lại rất dễ học và thực hiện (IDE của bạn hầu như tự động hóa rất nhiều trong số đó) nhưng, theo thời gian, chúng tạo ra ảnh hưởng vô cùng to lớn đến chất lượng hệ thống của chúng ta.

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

Chà, giờ thì bạn đang bước vào quá trình **thiết kế (designing)** rồi đấy, và bạn phải đảm bảo mình đi từng bước cẩn trọng. Nếu bạn không làm điều này một cách có kỷ luật, bạn có thể tạo ra một đống lộn xộn cho code của mình, bài test phía sau nó, *và* có lẽ là cả những thứ phụ thuộc vào nó nữa - hãy nhớ rằng, không chỉ có mỗi bài test của bạn đang sử dụng `WishHappyBirthday`. Hy vọng rằng nó vẫn đang được sử dụng trơn tru bởi mã nguồn "thực sự" nữa!

**Bạn vẫn có khả năng điều hướng sự thay đổi này thông qua phương thức viết test trước (test first)**. Bạn có thể chẻ sợi tóc làm tư chỉ để biện luận xem thứ này có phải là quy trình đổi thay "hành vi" không, nhưng rõ ràng là bạn đang muốn method của mình cư xử theo một lối khác.

Do đây là một sự thay đổi về hành vi, hãy áp dụng quy trình TDD ở đây. Một lợi ích của TDD là nó cung cấp cho bạn một luồng kiểm soát đơn giản, an toàn, và có thể lặp đi lặp lại để thúc đẩy sự thay đổi hành vi trong hệ thống; tại sao lại từ bỏ nó trong những tình huống này chỉ vì nó mang lại *cảm giác* khác biệt?

Trong trường hợp này, bạn sẽ thay đổi các tests hiện có để chúng tiến tới sử dụng kiểu type mới. Các bước tiến vòng lặp ngắt quãng quy mô nhỏ bé mà bạn thường quen thuộc mỗi khi xài TDD nhằm giảm thiểu độ rủi ro cũng như củng cố kỷ luật và tính rõ ràng sẽ giúp ích rất nhiều cho bạn trong những tình huống như thế này.

Nhiều khả năng bạn sẽ có sẵn nhiều bài test đang triệu gọi hàm `WishHappyBirthday`; trong những tình huống này, tôi khuyên bạn nên đóng comment tất cả lại và chỉ trừ lại một test duy nhất, điều khiển cho sự thay đổi diễn ra, và sau đó tiếp tục tháo gỡ lại các tests còn lại theo sự phù hợp.

### Thiết kế hệ thống lớn (Big design)

Việc thiết kế có thể đòi hỏi những thay đổi đáng kể hơn và những cuộc hội thoại mở rộng toàn diện hơn, và thông thường nó luôn mang trên mình một mức độ tính chủ quan. Thay đổi cấu trúc thiết kế ở nhiều thành phần trong hệ thống của bạn thường là một quá trình tốn thời gian hơn refactoring; dẫu vậy, bạn vẫn nên nỗ lực giảm thiểu rủi ro bằng việc suy tính tư duy xem làm thế nào để thực thi quá trình ấy bằng vô vàn các bước đi nhỏ.

### Mất tầm nhìn tổng thể do quá chú tâm chi tiết (Seeing the wood for the trees)

> [Nếu một cá nhân không thể **nhìn thấy khu rừng vì bị che khuất bởi bạt ngàn cây cối** trong văn hóa Anh Quốc, điều đó có nghĩa là họ đang trở nên quá gắn chặt vào một tiểu tiết vụn vặt nào đó và vì vậy không thể nhận thức được đặc điểm quan trọng đối với tổng thể toàn cục vấn đề.](https://www.collinsdictionary.com/dictionary/english/cant-see-the-wood-for-the-trees)

Việc đàm luận về các vấn đề thiết kế "vĩ mô" sẽ dễ tiếp cận hơn nhiều khi **mã nguồn tầng dưới đã được tổ chức cấu trúc tốt (well-factored)**. Giả dụ bạn và các đồng nghiệp cứ phải hộc tốc tốn nhường ấy khoảng lượng lớn quỹ thời gian chỉ để phân tích thấu hiểu một mớ bòng bong code sau mỗi lúc mở file, hệ thống chứa cơ hội nào để các bạn ngẫm tới thiết kế bao quát của dự án code đó?

Đó chính là nguyên nhân tại sao **khâu refactoring liên tục luôn đóng một vai trò vô cùng quan trọng trong quy trình TDD**. Nếu chúng ta lơ là bỏ quên thất bại ở khâu giải quyết các vấn đề thiết kế cỏn con, chúng ta rồi sẽ phải đối mặt vất vả khó nhằn mỗi lúc muốn vẽ lên mô hình tổng quan của hệ thống đồ sộ đang phải quản.

Trớ trêu thay, những bộ source code được cấu trúc tồi tệ sẽ bị làm cho vỡ nát theo thứ bậc luỹ thừa khi những kỹ sư tiếp tục nhồi nhét sự rườm rà (complexity) lên trên một khối móng nền lung lay dễ đổ.

## Khởi đầu danh sách kiểm tra trong tâm trí (mental-checklist)

**Hãy hình thành thói quen lướt qua một danh sách checklist cắm sẵn trong đầu ở mọi vòng đời TDD.** Càng ép bản thân trui rèn và thực hành nhiều tới đâu, chuyện ấy sẽ càng thuận tiện hơn tới đó. **Đó là dạng kỹ năng bắt buộc phải được thực hành rèn dũa.** Hãy nhớ lấy, mỗi bước trong hàng loạt những thao tác tu chỉnh này đều không làm phát sinh một bất cứ thứ bắt buộc đổi thay nào tại khâu tests mã của bạn.

Tôi đã đính kèm các mã lệnh phím tắt thao tác nhanh dành cho IntelliJ/GoLand, bộ dụng cụ tôi cùng đội nhóm đang chuyên dùng. Mỗi bất cứ thời khắc nào bắt tay qua bộ môn đào tạo gửi thông điệp huấn luyện đôn đốc mảng kĩ sư lớp mới, tôi thường tạo điểm nhấn đẩy khuyên họ hãy tập quen rồi nắm nhuần thói quen dùng sức mạnh cơ tay thuộc nằm phím (muscle memory) cũng như nề nếp dùng thành thạo các tiện ích kia hòng múc đích thao tác refactor gọn lẹ và tĩnh tại an toàn.

### Gộp chung biến (Inline variables)

Trường hợp bạn đi thiết lập khởi tạo một biến, mà nó chỉ để đem trao trả nhồi làm biến truyền số vào ở một cửa hàm method/function khác:

```go
url := baseURL + "/user/" + id
res, err := client.Get(url)
```

Nên suy tư về thao tác gộp thẳng dòng biến đó (`command+option+n`) *phân tích trừ phi* bản dạng đặc tả của biến chứa thêm một lớp ranh nghĩa rất rõ rệt quan trọng.

```go
res, err := client.Get(baseURL + "/user/" + id)
```

Xin chớ múa lộng quá mức "quá độ thông minh" ở khu vực mảng dùng lệnh gộp inline; chuẩn điểm hướng về không phải vì nhằm diệt sạch chùi mất độ tồn tại của bất kì một cái biến số nào để đánh đổi lấy vài mảng khối hàm rụt lòi chỉ sót đúng dòng lệnh nực cười thảm bại cực kì đau mắt khiến chả có ai luận đọc code ra. Nếu như bạn kiếm mớm ra được một nhãn từ tên đặt gán lột tả đúng nghĩa quan yếu nhất cho mẫu giá trị đó, chắc khả năng bảo lưu vẹn nguyên nó lại là đỉnh quyết định tối cao rồi.

### Tránh dư thừa lặp giá trị qua công đoạn trích gán tách biến (DRY up values with extract variables)

"Không được lặp lại chính bạn - Don't repeat yourself" (DRY). Đang đem gán lấy chính cùng chung một cái giá trị xuất hiện lọt thỏm không đếm xuể trong cấu bộ hàm? Xin hãy bớt một giây ngẫm đoái đến thao tác móc bóc trích gán khối value nọ trao qua cho một cái tên biến có gột mang sắc tính biểu nghĩa (`command+option+v`).

Quy chuẩn này thúc làm đòn bẩy độ trong sủa cho mã lọt mắt độ đọc hiểu (readability), mà còn gỡ rối giúp con đường sửa tu bồi đắp biến số đó thuận xu hướng hơn gấp trăm vạn lần, tại cớ rành rành đấy bạn sẽ thoát gánh chịu nỗi nhớ nhớ quên quên tự tay truy gõ sửa lặp bao hồi chung chạ một mã gán đấy mọi bề.

### Tránh lặp lại code (DRY) trong trường hợp tổng quát

[DRY (Đừng lặp lại chính mình)](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) ngày nay nhận nhiều ý kiến tiêu cực, và điều đó có lý do của nó. DRY là một khái niệm rất dễ bị hiểu sai ở mức độ hời hợt và sau đó bị áp dụng sai.

Một lập trình viên có thể áp dụng DRY đi quá xa, tạo ra những lớp trừu tượng phức tạp khó hiểu chỉ để tiết kiệm vài dòng code, thay vì ý tưởng *thực sự* của DRY là đặt một _ý niệm_ ở một nơi duy nhất. Việc giảm số lượng dòng code thường chỉ là hiệu ứng phụ của DRY, **chứ đó không phải là mục tiêu thực tế**.

Nên đúng vậy, DRY có thể bị áp dụng sai, nhưng chiều hướng ngược lại là từ chối việc DRY bất cứ thứ gì cũng tồi tệ không kém. Code bị lặp lại thêm vào sự nhiễu loạn và làm tăng các chi phí bảo trì. Việc từ chối tập hợp những khái niệm có liên quan vào cùng một chỗ chỉ vì sợ dùng sai DRY sẽ gây ra những vấn đề *khác biệt*.

Nên thay vì cực đoan ở cả hai phía "phải DRY mọi thứ" hoặc "DRY là tồi tệ", hãy suy nghĩ về đoạn code trước mắt bạn. Cái gì đang bị lặp lại? Nó có cần thiết phải như vậy không? Nếu bạn đóng gói đoạn code lặp lại đó vào một phương thức (method), danh sách tham số (parameter list) trông có hợp lý không? Đoạn code mới có tự giải thích được chính nó và đóng gói được "ý niệm" (idea) một cách rõ ràng không?

Chín phần mười trường hợp, bạn có thể nhìn vào danh sách tham số của một hàm, và nếu nó trông rối rắm khó hiểu, thì rất có thể đó là một cách áp dụng DRY tồi.

Nếu bạn cảm thấy việc làm cho code DRY trở nên khó khăn, có thể bạn đang làm cho mọi thứ phức tạp hơn; hãy cân nhắc dừng lại.

Hãy DRY một cách cẩn thận, **nhưng thực hành nó thường xuyên sẽ cải thiện khả năng đánh giá của bạn**. Tôi khuyến khích các đồng nghiệp của mình "cứ thử xem" và sử dụng source control (quản lý mã nguồn) để quay lại trạng thái an toàn nếu nó sai.

<u>**Tự mình thử nghiệm những điều này sẽ dạy bạn nhiều bài học hơn là chỉ ngồi thảo luận về nó**</u>, và source control kết hợp với các bài test tự động tốt (automated tests) sẽ mang đến cho bạn một môi trường hoàn hảo để thử nghiệm và học hỏi.

### Trích xuất (Extract) các giá trị "Ma thuật" (Magic values).

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

### Giúp cho public methods/functions dễ dàng để đọc lướt (scan)

Đoạn mã code của bạn có đang chứa những public methods hoặc functions quá dài dòng không?

Hãy bao bọc các thao tác trên vào trong những methods/functions cấu trúc private thông qua công cụ refactor extract method (`command+option+m`).

Đoạn code bên dưới chứa đựng một vài thủ tục nhiễu loạn nhàm chán xoay quanh quá trình khởi tạo chuỗi JSON string rồi gộp quy đổi nó thành kiểu `io.Reader` nhằm chuẩn bị được gửi đi thông qua một HTTP request dạng `POST`.

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

Đầu tiên, sử dụng chức năng refactor gộp biến (inline variable) (`command+option+n`) để gộp `payload` vào quá trình tạo buffer.

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

Một sự phản đối chung đối với việc refactoring kiểu này, vốn yêu thích các functions và methods nhỏ bé được cấu thành từ những cái khác, đó là nó có thể gây khó khăn cho việc hiểu cách code hoạt động. Lời đáp trả thẳng thắn của tôi cho việc này là:

> Bạn đã học cách điều hướng codebases bằng công cụ của mình một cách hiệu quả chưa?

Một cách rất có chủ ý, với tư cách là người _viết_ `CreateWidget`, tôi không muốn quá trình tạo ra một chuỗi string cụ thể trở thành một nhân vật thiết yếu trong mạch truyện của method này. Nó làm phân tâm người đọc, và là một sự nhiễu loạn không liên quan 99% thời gian.

Tuy nhiên, nếu ai đó _thực sự_ quan tâm, bạn chỉ cần nhấn `command+b` (hoặc phím tắt "điều hướng đến ký hiệu" (navigate to symbol) của bạn) trên `createWidgetPayload`... và đọc nó. Nhấn `command+left-arrow` để quay lại.

### Đưa quá trình tạo giá trị về thời điểm khởi tạo (construction time).

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

Một kỹ thuật refactoring bạn có thể áp dụng ở đây là, nếu một giá trị đang được tạo ra mà **không phụ thuộc vào các tham số truyền vào của method**, thì bạn có thể thiết lập một _trường (field)_ trong type của bạn và tính toán nó tại hàm khởi tạo (constructor function).

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

### Cố gắng loại bỏ các comments (bình luận).

> Một nguyên tắc nền tảng mà chúng tôi tuân theo là bất cứ khi nào chúng tôi cảm thấy cần phải comment một điều gì đó, thay vào đó chúng tôi sẽ viết một method.

-- Martin Fowler

Một lần nữa, refactor trích xuất method có thể là người bạn của bạn ở đây.

## Ngoại lệ của quy tắc

Có những sự cải tiến cho code yêu cầu sự thay đổi trong các bài tests của bạn, mà tôi vẫn sẽ rất sẵn lòng xếp vào nhóm "refactoring", mặc dù nó phá vỡ quy tắc.

Một ví dụ đơn giản sẽ là việc đổi tên một public symbol (ví dụ: một method, type, hoặc function) bằng `shift+F6`. Điều này tất nhiên sẽ thay đổi cả code production và test.

Tuy nhiên, vì nó là một sự thay đổi **tự động và an toàn (automated and safe)**, rủi ro rơi vào vòng xoáy của việc phá vỡ code production và test mà rất nhiều người gặp phải với các loại cập nhật *thiết kế* khác là rất nhỏ.

Vì lý do đó, bất kỳ thay đổi nào mà bạn có thể thực hiện một cách an toàn bằng IDE/editor của mình, tôi vẫn sẽ vui vẻ gọi là refactoring.

## Sử dụng các công cụ của bạn để giúp bạn luyện tập refactoring.

- Bạn nên chạy các unit tests của mình mỗi khi bạn thực hiện một trong những thay đổi nhỏ này. Chúng ta đầu tư thời gian vào việc làm cho code có thể unit-test được, và vòng lặp phản hồi (feedback loop) kéo dài vài phần nghìn giây là một trong những lợi ích lớn nhất; hãy sử dụng nó!
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

Chúng ta nên luôn hướng tới việc bàn giao hệ thống code trong ở một trạng thái *mẫu mực (exemplary)*.

Một cú refactoring tốt sẽ dẫn lối tới hệ thống code dễ dàng đọc hiểu hơn. Sự thấu hiểu về code có nghĩa là các sơ đồ thiết kế (designs) trở nên dễ dàng để phát hiện và nắm bắt hơn. Cực kỳ khó khăn để có thể nhìn ra được những designs ngon nghẻ nằm trong các hệ thống chứa đầy những function khổng lồ béo ú, code trùng lặp thừa mứa bừa bãi không cần thiết, đi kèm chuỗi lồng nhau sâu hoắm (deep nesting), vân vân. **Việc refactor tuy mang quy mô nhỏ bé nhưng xuất hiện xuất hiện thường xuyên là yếu tố cực thiết yếu cho một bản design tốt hơn.**

