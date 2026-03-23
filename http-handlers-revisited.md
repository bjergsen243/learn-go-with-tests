# Xem lại HTTP Handler

**[Bạn có thể tìm thấy toàn bộ code tại đây](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/http-handlers-revisited)**

Cuốn sách này đã có một chương về [test HTTP handler](http-server.md), nhưng chương này sẽ thảo luận rộng hơn về cách thiết kế chúng sao cho dễ test.

Chúng ta sẽ xem một ví dụ thực tế và cách cải thiện thiết kế bằng các nguyên tắc như Single Responsibility Principle (SRP) và Separation of Concerns. Các nguyên tắc này được hiện thực hóa thông qua [interface](structs-methods-and-interfaces.md) và [dependency injection](dependency-injection.md). Nhờ đó, việc test handler khá đơn giản.

![Minh họa một câu hỏi phổ biến trong cộng đồng Go](amazing-art.png)

Test HTTP handler dường như là một câu hỏi lặp đi lặp lại trong cộng đồng Go, và tôi nghĩ nó chỉ ra một vấn đề rộng hơn là mọi người đang hiểu sai cách thiết kế chúng.

Thường thì khó khăn trong việc test bắt nguồn từ thiết kế code, không phải từ bản thân test. Như tôi đã nhấn mạnh nhiều lần trong cuốn sách này:

> Nếu các bài test khiến bạn thấy đau đớn, hãy lắng nghe tín hiệu đó và suy nghĩ về thiết kế code của bạn.

## Một ví dụ

[Santosh Kumar đã tweet cho tôi](https://twitter.com/sntshk/status/1255559003339284481):

> Làm cách nào để tôi test một http handler có phụ thuộc vào mongodb?

Đây là đoạn mã:

```go
func Registration(w http.ResponseWriter, r *http.Request) {
	var res model.ResponseResult
	var user model.User
	w.Header().Set("Content-Type", "application/json")

	jsonDecoder := json.NewDecoder(r.Body)
	jsonDecoder.DisallowUnknownFields()
	defer r.Body.Close()

	// kiểm tra xem có thân bài json hợp lệ hay không
	if err := jsonDecoder.Decode(&user); err != nil {
		res.Error = err.Error()
		// trả về mã trạng thái 400
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// Kết nối tới mongodb
	client, _ := mongo.NewClient(options.Client().ApplyURI("mongodb://127.0.0.1:27017"))
	ctx, _ := context.WithTimeout(context.Background(), 10*time.Second)
	err := client.Connect(ctx)
	if err != nil {
		panic(err)
	}
	defer client.Disconnect(ctx)
	// Kiểm tra xem tên người dùng đã tồn tại trong kho lưu trữ dữ liệu người dùng chưa, nếu có, trả về 400
	// ngược lại thì thêm người dùng ngay lập tức
	collection := client.Database("test").Collection("users")
	filter := bson.D{{"username", user.Username}}
	var foundUser model.User
	err = collection.FindOne(context.TODO(), filter).Decode(&foundUser)
	if foundUser.Username == user.Username {
		res.Error = UserExists
		// trả về mã trạng thái 400
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	pass, err := bcrypt.GenerateFromPassword([]byte(user.Password), bcrypt.DefaultCost)
	if err != nil {
		res.Error = err.Error()
		// trả về mã trạng thái 400
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}
	user.Password = string(pass)

	insertResult, err := collection.InsertOne(context.TODO(), user)
	if err != nil {
		res.Error = err.Error()
		// trả về mã trạng thái 400
		w.WriteHeader(http.StatusBadRequest)
		json.NewEncoder(w).Encode(res)
		return
	}

	// trả về 200
	w.WriteHeader(http.StatusOK)
	res.Result = fmt.Sprintf("%s: %s", UserCreated, insertResult.InsertedID)
	json.NewEncoder(w).Encode(res)
	return
}
```

Hãy liệt kê tất cả những việc mà một hàm duy nhất này phải làm:

1. Viết các phản hồi HTTP, gửi các header, mã trạng thái, v.v.
2. Giải mã request body thành một `User`.
3. Kết nối với cơ sở dữ liệu (và tất cả các chi tiết xung quanh việc đó).
4. Truy vấn cơ sở dữ liệu và áp dụng một số logic nghiệp vụ tùy thuộc vào kết quả.
5. Tạo mật khẩu.
6. Thêm một bản ghi.

Việc này là quá nhiều.

## HTTP Handler là gì và nó nên làm gì?

Tạm quên các chi tiết cụ thể của Go trong giây lát, bất kể tôi đã làm việc với ngôn ngữ nào, điều luôn giúp ích cho tôi là suy nghĩ về [separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) và [single responsibility principle](https://en.wikipedia.org/wiki/Single-responsibility_principle).

Việc áp dụng các nguyên tắc này có thể khá khó khăn tùy thuộc vào vấn đề bạn đang giải quyết. Chính xác thì một trách nhiệm _là_ cái gì?

Ranh giới có thể bị mờ nhạt tùy thuộc vào mức độ trừu tượng bạn đang suy nghĩ và đôi khi phỏng đoán đầu tiên của bạn có thể không đúng.

Rất may, với các HTTP handler, tôi cảm thấy mình có ý tưởng khá rõ ràng về những gì chúng nên làm, bất kể tôi đã làm việc trong dự án nào:

1. Chấp nhận một HTTP request, parse và validate nó.
2. Gọi một `ServiceThing` nào đó để thực hiện `ImportantBusinessLogic` với dữ liệu tôi có được từ bước 1.
3. Gửi một phản hồi `HTTP` phù hợp tùy thuộc vào những gì `ServiceThing` trả về.

Tôi không nói rằng mọi HTTP handler _từng tồn tại_ đều nên có hình dạng đại loại như thế này, nhưng trong 99 trên 100 trường hợp, đó dường như là kịch bản đối với tôi.

Khi bạn phân tách các mối quan tâm này:

 - Việc test handler trở nên dễ dàng và tập trung vào một số ít các mối quan tâm.
 - Quan trọng là việc test `ImportantBusinessLogic` không còn phải bận tâm đến `HTTP` nữa, bạn có thể test logic nghiệp vụ một cách sạch sẽ.
 - Bạn có thể sử dụng `ImportantBusinessLogic` trong các ngữ cảnh khác mà không phải sửa đổi nó.
 - Nếu `ImportantBusinessLogic` thay đổi những gì nó làm, miễn là interface vẫn giữ nguyên, bạn không phải thay đổi các handler của mình.

## Các Handler của Go

[`http.HandlerFunc`](https://golang.org/pkg/net/http/#HandlerFunc)

> Kiểu HandlerFunc là một adapter cho phép sử dụng các hàm thông thường như các HTTP handler.

`type HandlerFunc func(ResponseWriter, *Request)`

Bạn đọc hãy hít một hơi thật sâu và nhìn vào đoạn mã trên. Bạn nhận thấy điều gì?

**Nó là một hàm nhận vào một số đối số**

Không có phép màu nào từ framework, không có annotation, không có magic bean, không có gì cả.

Nó chỉ là một hàm, _và chúng ta biết cách test các hàm_.

Nó hoàn toàn phù hợp với những bình luận ở trên:

- Nó nhận vào một [`http.Request`](https://golang.org/pkg/net/http/#Request), thứ chỉ là một gói dữ liệu để chúng ta kiểm tra, phân tích và xác thực.
- > [`http.ResponseWriter` interface được sử dụng bởi một HTTP handler để xây dựng một phản hồi HTTP.](https://golang.org/pkg/net/http/#ResponseWriter)

### Ví dụ test siêu cơ bản

```go
func Teapot(res http.ResponseWriter, req *http.Request) {
	res.WriteHeader(http.StatusTeapot)
}

func TestTeapotHandler(t *testing.T) {
	req := httptest.NewRequest(http.MethodGet, "/", nil)
	res := httptest.NewRecorder()

	Teapot(res, req)

	if res.Code != http.StatusTeapot {
		t.Errorf("got status %d but wanted %d", res.Code, http.StatusTeapot)
	}
}
```

Để test một hàm, chúng ta _gọi_ nó.

Đối với bài test của chúng ta, chúng ta truyền một `httptest.ResponseRecorder` làm đối số `http.ResponseWriter`, và hàm của chúng ta sẽ sử dụng nó để viết phản hồi `HTTP`. Recorder sẽ ghi lại (hoặc _spy_) những gì đã được gửi, và sau đó chúng ta có thể thực hiện các assertion của mình.

## Gọi một `ServiceThing` trong handler của chúng ta

Một lời phàn nàn phổ biến về các hướng dẫn TDD là chúng luôn "quá đơn giản" và không "đủ thực tế". Câu trả lời của tôi cho điều đó là:

> Sẽ thật tuyệt nếu tất cả code của bạn đều dễ đọc và dễ test như các ví dụ bạn đề cập đúng không?

Đây là một trong những thách thức lớn nhất mà chúng ta phải đối mặt nhưng cần tiếp tục nỗ lực. Điều đó _là khả thi_ (mặc dù không nhất thiết là dễ dàng) để thiết kế code sao cho nó có thể dễ đọc và dễ test nếu chúng ta thực hành và áp dụng các nguyên tắc kỹ thuật phần mềm tốt.

Tóm tắt lại những gì handler từ trước đó làm:

1. Viết các phản hồi HTTP, gửi các header, mã trạng thái, v.v.
2. Giải mã thân bài yêu cầu thành một `User`.
3. Kết nối với cơ sở dữ liệu (và tất cả các chi tiết xung quanh việc đó).
4. Truy vấn cơ sở dữ liệu và áp dụng một số logic nghiệp vụ tùy thuộc vào kết quả.
5. Tạo mật khẩu.
6. Thêm một bản ghi.

Lấy ý tưởng về việc phân tách các mối quan tâm lý tưởng hơn, tôi muốn nó giống như sau hơn:

1. Giải mã thân bài yêu cầu thành một `User`.
2. Gọi một `UserService.Register(user)` (đây chính là `ServiceThing` của chúng ta).
3. Nếu có lỗi, hãy xử lý nó (ví dụ ban đầu luôn gửi lỗi `400 BadRequest`, tôi nghĩ điều đó là không đúng), tôi sẽ chỉ sử dụng một trình xử lý bao quát cho mọi lỗi là `500 Internal Server Error` _vào lúc này_. Tôi phải nhấn mạnh rằng việc trả về `500` cho mọi lỗi sẽ tạo ra một API rất tệ! Sau này chúng ta có thể làm cho việc xử lý lỗi tinh vi hơn, có lẽ với [error types](error-types.md).
4. Nếu không có lỗi, trả về `201 Created` với ID trong thân bài phản hồi (một lần nữa là vì sự ngắn gọn/lười biếng).

Vì mục đích ngắn gọn, tôi sẽ không đi sâu vào quy trình TDD thông thường, hãy kiểm tra tất cả các chương khác để xem ví dụ.

### Thiết kế mới

```go
type UserService interface {
	Register(user User) (insertedID string, err error)
}

type UserServer struct {
	service UserService
}

func NewUserServer(service UserService) *UserServer {
	return &UserServer{service: service}
}

func (u *UserServer) RegisterUser(w http.ResponseWriter, r *http.Request) {
	defer r.Body.Close()

	// phân tích và xác thực yêu cầu
	var newUser User
	err := json.NewDecoder(r.Body).Decode(&newUser)

	if err != nil {
		http.Error(w, fmt.Sprintf("could not decode user payload: %v", err), http.StatusBadRequest)
		return
	}

	// gọi một service thing để đảm nhận phần việc khó khăn
	insertedID, err := u.service.Register(newUser)

	// tùy thuộc vào những gì chúng ta nhận lại, phản hồi tương ứng
	if err != nil {		//todo: xử lý các loại lỗi khác nhau theo các cách khác nhau
		http.Error(w, fmt.Sprintf("problem registering new user: %v", err), http.StatusInternalServerError)
		return
	}

	w.WriteHeader(http.StatusCreated)
	fmt.Fprint(w, insertedID)
}
```

Phương thức `RegisterUser` của chúng ta khớp với hình dạng của `http.HandlerFunc` nên chúng ta có thể bắt đầu sử dụng nó. Chúng ta gắn nó như một phương thức trên một kiểu dữ liệu mới `UserServer`, kiểu này chứa một phụ thuộc vào `UserService` được ghi nhận dưới dạng một interface.

Các interface là một cách tuyệt vời để đảm bảo các mối quan tâm về `HTTP` của chúng ta được tách rời khỏi bất kỳ triển khai cụ thể nào; chúng ta chỉ cần gọi phương thức trên phụ thuộc đó, và chúng ta không cần quan tâm _làm thế nào_ một người dùng được đăng ký.

Nếu bạn muốn khám phá phương pháp này chi tiết hơn theo quy trình TDD, hãy đọc chương [Dependency Injection](dependency-injection.md) và chương [HTTP Server trong phần "Xây dựng một ứng dụng"](http-server.md).

Bây giờ chúng ta đã tách rời khỏi bất kỳ chi tiết triển khai cụ thể nào xung quanh việc đăng ký, việc viết code cho handler trở nên đơn giản và tuân theo các trách nhiệm được mô tả trước đó.

### Các bài test!

Sự đơn giản này được phản ánh trong các bài test của chúng ta.

```go
type MockUserService struct {
	RegisterFunc    func(user User) (string, error)
	UsersRegistered []User
}

func (m *MockUserService) Register(user User) (insertedID string, err error) {
	m.UsersRegistered = append(m.UsersRegistered, user)
	return m.RegisterFunc(user)
}

func TestRegisterUser(t *testing.T) {
	t.Run("can register valid users", func(t *testing.T) {
		user := User{Name: "CJ"}
		expectedInsertedID := "whatever"

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return expectedInsertedID, nil
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusCreated)

		if res.Body.String() != expectedInsertedID {
			t.Errorf("expected body of %q but got %q", res.Body.String(), expectedInsertedID)
		}

		if len(service.UsersRegistered) != 1 {
			t.Fatalf("expected 1 user added but got %d", len(service.UsersRegistered))
		}

		if !reflect.DeepEqual(service.UsersRegistered[0], user) {
			t.Errorf("the user registered %+v was not what was expected %+v", service.UsersRegistered[0], user)
		}
	})

	t.Run("returns 400 bad request if body is not valid user JSON", func(t *testing.T) {
		server := NewUserServer(nil)

		req := httptest.NewRequest(http.MethodGet, "/", strings.NewReader("trouble will find me"))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusBadRequest)
	})

	t.Run("returns a 500 internal server error if the service fails", func(t *testing.T) {
		user := User{Name: "CJ"}

		service := &MockUserService{
			RegisterFunc: func(user User) (string, error) {
				return "", errors.New("couldn't add new user")
			},
		}
		server := NewUserServer(service)

		req := httptest.NewRequest(http.MethodGet, "/", userToJSON(user))
		res := httptest.NewRecorder()

		server.RegisterUser(res, req)

		assertStatus(t, res.Code, http.StatusInternalServerError)
	})
}
```

Giờ handler không còn gắn chặt với một triển khai lưu trữ cụ thể, việc viết `MockUserService` để test nhanh các trách nhiệm cụ thể trở nên rất đơn giản.

### Thế còn code cơ sở dữ liệu thì sao? Bạn đang gian lận!

Điều này hoàn toàn là có tính toán. Chúng ta không muốn các HTTP handler bận tâm đến logic nghiệp vụ, cơ sở dữ liệu, các kết nối, v.v.

Bằng cách này, chúng ta đã giải phóng handler khỏi những chi tiết lộn xộn, chúng ta _cũng_ làm cho việc test lớp lưu trữ và logic nghiệp vụ trở nên dễ dàng hơn vì nó cũng không còn bị bó buộc vào các chi tiết HTTP không liên quan nữa.

Tất cả những gì chúng ta cần làm bây giờ là triển khai `UserService` bằng bất kỳ cơ sở dữ liệu nào chúng ta muốn sử dụng:

```go
type MongoUserService struct {
}

func NewMongoUserService() *MongoUserService {
	//todo: truyền DB URL làm đối số cho hàm này
	//todo: kết nối với db, tạo một pool kết nối
	return &MongoUserService{}
}

func (m MongoUserService) Register(user User) (insertedID string, err error) {
	// sử dung m.mongoConnection để thực hiện truy vấn
	panic("implement me")
}
```

Chúng ta có thể test phần này một cách riêng biệt và một khi chúng ta thấy hài lòng, trong `main` chúng ta có thể kết nối hai đơn vị này lại với nhau cho ứng dụng đang hoạt động của mình.

```go
func main() {
	mongoService := NewMongoUserService()
	server := NewUserServer(mongoService)
	http.ListenAndServe(":8000", http.HandlerFunc(server.RegisterUser))
}
```

### Một thiết kế mạnh mẽ và dễ mở rộng hơn với ít nỗ lực

Những nguyên tắc này không chỉ làm cho cuộc sống của chúng ta dễ dàng hơn trong ngắn hạn mà còn giúp hệ thống dễ dàng mở rộng trong tương lai.

Sẽ không có gì ngạc nhiên khi trong các lần lặp tiếp theo của hệ thống này, chúng ta muốn gửi email xác nhận đăng ký cho người dùng.

Với thiết kế cũ, chúng ta sẽ phải thay đổi handler _với_ các bài test xung quanh. Đây thường là cách các phần code trở nên không thể bảo trì, ngày càng có nhiều chức năng len lỏi vào vì nó vốn đã được _thiết kế_ theo cách đó; dành cho "HTTP handler" để xử lý... mọi thứ!

Bằng cách tách biệt các mối quan tâm bằng một interface, chúng ta không phải chỉnh sửa handler _chút nào_ vì nó không quan tâm đến logic nghiệp vụ xung quanh việc đăng ký.

## Tổng kết

Test HTTP handler của Go không hề khó khăn, nhưng thiết kế phần mềm tốt thì có thể!

Mọi người mắc sai lầm khi nghĩ rằng các HTTP handler là đặc biệt và vứt bỏ các thực hành kỹ thuật phần mềm tốt khi viết chúng, điều này sau đó làm cho việc test chúng trở nên thách thức.

Nhắc lại một lần nữa; **các http handler của Go chỉ là các hàm**. Nếu bạn viết chúng giống như các hàm khác, với các trách nhiệm rõ ràng và separation of concerns tốt, bạn sẽ không gặp khó khăn gì khi test chúng, và code của bạn sẽ lành mạnh hơn vì điều đó.
