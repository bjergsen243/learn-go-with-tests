# Đọc file (Reading files)

- **[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/reading-files)**
- [Đây là video tôi đang giải quyết bài toán và trả lời các câu hỏi từ Twitch stream](https://www.youtube.com/watch?v=nXts4dEJnkU)

Trong chương này, chúng ta sẽ học cách đọc một số file, trích xuất dữ liệu từ chúng và thực hiện điều gì đó hữu ích.

Giả sử bạn đang làm việc với một người bạn để tạo một phần mềm blog. Ý tưởng là tác giả sẽ viết các bài đăng của họ bằng markdown, với một số siêu dữ liệu (metadata) ở đầu file. Khi khởi động, web server sẽ đọc một thư mục để tạo ra các `Post`, và sau đó một hàm `NewHandler` riêng biệt sẽ sử dụng các `Post` đó làm nguồn dữ liệu cho web server của blog.

Chúng ta được yêu cầu tạo package chuyển đổi một thư mục chứa các file bài đăng blog thành một tập hợp các `Post`.

### Dữ liệu mẫu

hello world.md
```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

The body of posts starts after the `---`
```

### Dữ liệu mong chờ

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

## Phát triển theo hướng kiểm thử, lặp lại (Iterative, TDD)

Chúng ta sẽ thực hiện theo phương pháp lặp lại, luôn thực hiện các bước đơn giản, an toàn hướng tới mục tiêu của mình.

Điều này đòi hỏi chúng ta phải chia nhỏ công việc, nhưng chúng ta nên cẩn thận để không rơi vào cái bẫy thực hiện phương pháp ["từ dưới lên" (bottom up)](https://en.wikipedia.org/wiki/Top-down_and_bottom-up_design).

Chúng ta không nên tin vào trí tưởng tượng quá mức của mình khi bắt đầu công việc. Chúng ta có thể bị cám dỗ tạo ra một loại trừu tượng nào đó mà chỉ được kiểm chứng sau khi chúng ta gắn kết mọi thứ lại với nhau, chẳng hạn như một loại `BlogPostFileParser` nào đó.

Điều này *không* mang tính lặp lại và đang bỏ lỡ các vòng lặp phản hồi chặt chẽ mà TDD mang lại cho chúng ta.

Kent Beck nói:

> Sự lạc quan là một mối nguy hiểm nghề nghiệp của lập trình viên. Phản hồi chính là phương pháp điều trị.

Thay vào đó, cách tiếp cận của chúng ta nên cố gắng đạt được việc mang lại giá trị *thực* cho người tiêu dùng càng nhanh càng tốt (thường được gọi là "happy path"). Một khi chúng ta đã mang lại một phần nhỏ giá trị cho người tiêu dùng từ đầu đến cuối (end-to-end), việc lặp lại tiếp theo cho các yêu cầu còn lại thường sẽ đơn giản.

## Suy nghĩ về loại test mà chúng ta muốn thấy

Hãy tự nhắc nhở bản thân về tư duy và mục tiêu của chúng ta khi bắt đầu:

- **Viết bản kiểm thử mà chúng ta muốn thấy**. Hãy nghĩ về cách chúng ta muốn sử dụng mã nguồn mà mình chuẩn bị viết từ quan điểm của người tiêu dùng.
- Tập trung vào *cái gì* và *tại sao*, nhưng đừng bị phân tâm bởi *làm thế nào*.

Package của chúng ta cần cung cấp một hàm có thể trỏ vào một thư mục và trả về cho chúng ta một số bài đăng.

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS("some-folder")
```

Để viết một bản kiểm thử xung quanh việc này, chúng ta cần một loại thư mục kiểm thử với một số bài đăng mẫu trong đó. *Cũng không có gì quá sai trái với điều này*, nhưng bạn đang thực hiện một số đánh đổi:

- đối với mỗi bản kiểm thử, bạn có thể cần tạo các file mới để kiểm thử một hành vi cụ thể
- một số hành vi sẽ khó kiểm thử, chẳng hạn như lỗi khi tải file
- các bản kiểm thử sẽ chạy chậm hơn một chút vì chúng cần truy cập vào hệ thống file

Chúng ta cũng đang ràng buộc bản thân một cách không cần thiết vào một triển khai cụ thể của hệ thống file.

### Trừu tượng hóa hệ thống file được giới thiệu trong Go 1.16

Go 1.16 đã giới thiệu một sự trừu tượng hóa cho hệ thống file; package [io/fs](https://golang.org/pkg/io/fs/).

> Package fs định nghĩa các interface cơ bản cho một hệ thống file. Một hệ thống file có thể được cung cấp bởi hệ điều hành máy chủ nhưng cũng có thể được cung cấp bởi các package khác.

Điều này cho phép chúng ta nới lỏng sự ràng buộc của mình vào một hệ thống file cụ thể, từ đó cho phép chúng ta truyền (inject) các triển khai khác nhau tùy theo nhu cầu của mình.

> [Về phía nhà cung cấp của interface, kiểu embed.FS mới triển khai fs.FS, tương tự như zip.Reader. Hàm os.DirFS mới cung cấp một triển khai của fs.FS được hỗ trợ bởi một cây các file hệ điều hành.](https://golang.org/doc/go1.16#fs)

Nếu chúng ta sử dụng interface này, người dùng package của chúng ta có một số tùy chọn được tích hợp sẵn trong thư viện tiêu chuẩn để sử dụng. Học cách tận dụng các interface được định nghĩa trong thư viện tiêu chuẩn của Go (ví dụ: `io.fs`, [`io.Reader`](https://golang.org/pkg/io/#Reader), [`io.Writer`](https://golang.org/pkg/io/#Writer)), là điều quan trọng để viết các package có độ ràng buộc thấp (loosely coupled). Các package này sau đó có thể được tái sử dụng trong các ngữ cảnh khác với những gì bạn tưởng tượng ban đầu, với sự phiền hà tối thiểu từ người tiêu dùng của bạn.

Trong trường hợp của chúng ta, có lẽ người tiêu dùng muốn các bài đăng được nhúng vào file nhị phân Go thay vì các file trong một hệ thống file "thật"? Dù thế nào đi nữa, *mã nguồn của chúng ta không cần phải quan tâm*.

Đối với các bản kiểm thử của chúng ta, package [testing/fstest](https://golang.org/pkg/testing/fstest/) cung cấp cho chúng ta một triển khai của [io/FS](https://golang.org/pkg/io/fs/#FS) để sử dụng, tương tự như các công cụ mà chúng ta đã quen thuộc trong [net/http/httptest](https://golang.org/pkg/net/http/httptest/).

Với những thông tin này, cách tiếp cận sau đây có vẻ tốt hơn:

```go
var posts []blogposts.Post
posts = blogposts.NewPostsFromFS(someFS)
```

## Viết test trước tiên

Chúng ta nên giữ phạm vi nhỏ gọn và hữu ích nhất có thể. Nếu chúng ta chứng minh được rằng mình có thể đọc tất cả các file trong một thư mục, đó sẽ là một khởi đầu tốt. Điều này sẽ cho chúng ta sự tự tin vào phần mềm mình đang viết. Chúng ta có thể kiểm tra xem số lượng `[]Post` được trả về có giống với số lượng file trong hệ thống file giả của chúng ta hay không.

Tạo một dự án mới để thực hành chương này.

- `mkdir blogposts`
- `cd blogposts`
- `go mod init github.com/{your-name}/blogposts`
- `touch blogposts_test.go`

```go
package blogposts_test

import (
	"testing"
	"testing/fstest"
)

func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts := blogposts.NewPostsFromFS(fs)

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}
```

Lưu ý rằng package của bản kiểm thử là `blogposts_test`. Hãy nhớ rằng, khi TDD được thực hành tốt, chúng ta thực hiện theo phương pháp *hướng người tiêu dùng* (consumer-driven): chúng ta không muốn kiểm thử các chi tiết nội bộ vì *người tiêu dùng* không quan tâm đến chúng. Bằng cách thêm `_test` vào tên package dự định của mình, chúng ta chỉ truy cập vào các thành viên được xuất (exported) từ package của mình - giống như một người dùng thực sự của package.

Chúng ta đã import [`testing/fstest`](https://golang.org/pkg/testing/fstest/) cung cấp quyền truy cập vào kiểu [`fstest.MapFS`](https://golang.org/pkg/testing/fstest/#MapFS). Hệ thống file giả của chúng ta sẽ truyền `fstest.MapFS` vào package của chúng ta.

> MapFS là một hệ thống file đơn giản trong bộ nhớ để sử dụng trong các bản kiểm thử, được đại diện dưới dạng một map từ tên đường dẫn (các đối số cho Open) đến thông tin về các file hoặc thư mục mà chúng đại diện.

Điều này có vẻ đơn giản hơn việc duy trì một thư mục chứa các file kiểm thử, và nó sẽ thực thi nhanh hơn.

Cuối cùng, chúng ta đã hệ thống hóa việc sử dụng API của mình từ góc nhìn của người tiêu dùng, sau đó kiểm tra xem nó có tạo ra số lượng bài đăng chính xác hay không.

## Thử chạy test

```
./blogpost_test.go:15:12: undefined: blogposts
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Package không tồn tại. Tạo một file mới `blogposts.go` và đặt `package blogposts` vào bên trong. Sau đó, bạn sẽ cần import package đó vào các bản kiểm thử của mình. Đối với tôi, các bản import bây giờ trông như thế này:

```go
import (
	blogposts "github.com/quii/learn-go-with-tests/reading-files"
	"testing"
	"testing/fstest"
)
```

Bây giờ các bản kiểm thử sẽ không biên dịch được vì package mới của chúng ta không có hàm `NewPostsFromFS` trả về một loại tập hợp nào đó.

```
./blogpost_test.go:16:12: undefined: blogposts.NewPostsFromFS
```

Điều này buộc chúng ta phải tạo bộ khung cho hàm của mình để bản kiểm thử có thể chạy. Hãy nhớ đừng quá suy nghĩ về mã nguồn tại thời điểm này; chúng ta chỉ đang cố gắng có một bản kiểm thử đang chạy, và để đảm bảo nó thất bại như chúng ta mong đợi. Nếu chúng ta bỏ qua bước này, chúng ta có thể bỏ qua các giả định và viết một bản kiểm thử không hữu ích.

```go
package blogposts

import "testing/fstest"

type Post struct {
}

func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return nil
}
```

Bản kiểm thử bây giờ sẽ thất bại một cách chính xác:

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:48: got 0 posts, wanted 2 posts
```

## Viết đủ code để test chạy thành công

Chúng ta *có thể* sử dụng ["slime"](https://deniseyu.github.io/leveling-up-tdd/) để làm cho nó vượt qua:

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	return []Post{{}, {}}
}
```

Nhưng, như Denise Yu đã viết:

> Sliming rất hữu ích để tạo "bộ khung" cho đối tượng của bạn. Thiết kế một interface và thực thi logic là hai mối quan tâm khác nhau, và việc kiểm thử slime một cách chiến thuật cho phép bạn tập trung vào từng thứ một.

Chúng ta đã có cấu trúc của mình. Vậy, chúng ta làm gì thay thế?

Vì chúng ta đã thu hẹp phạm vi, tất cả những gì chúng ta cần làm là đọc thư mục và tạo một bài đăng cho mỗi file mà chúng ta gặp. Chúng ta chưa phải lo lắng về việc mở các file và phân tích chúng.

```go
func NewPostsFromFS(fileSystem fstest.MapFS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

[`fs.ReadDir`](https://golang.org/pkg/io/fs/#ReadDir) đọc một thư mục bên trong một `fs.FS` nhất định và trả về [`[]DirEntry`](https://golang.org/pkg/io/fs/#DirEntry).

Ngay lập tức, cái nhìn lý tưởng hóa của chúng ta về thế giới đã bị cản trở vì các lỗi có thể xảy ra, nhưng hãy nhớ bây giờ trọng tâm của chúng ta là *làm cho bản kiểm thử vượt qua*, không phải thay đổi thiết kế, vì vậy chúng ta sẽ tạm bỏ qua lỗi lúc này.

Phần còn lại của mã nguồn khá đơn giản: lặp lại qua các mục nhập (entries), tạo một `Post` cho mỗi mục và trả về slice.

## Refactor

Mặc dù các bản kiểm thử của chúng ta đang vượt qua, chúng ta không thể sử dụng package mới này bên ngoài ngữ cảnh này, bởi vì nó bị ràng buộc vào một triển khai cụ thể `fstest.MapFS`. Nhưng, nó không nhất thiết phải như vậy. Hãy thay đổi đối số cho hàm `NewPostsFromFS` của chúng ta để chấp nhận interface từ thư viện tiêu chuẩn.

```go
func NewPostsFromFS(fileSystem fs.FS) []Post {
	dir, _ := fs.ReadDir(fileSystem, ".")
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts
}
```

Chạy lại các bản kiểm thử: mọi thứ sẽ hoạt động.

### Xử lý lỗi

Chúng ta đã tạm gác phần xử lý lỗi trước đó khi tập trung vào việc làm cho happy-path hoạt động. Trước khi tiếp tục lặp lại các chức năng, chúng ta nên thừa nhận rằng các lỗi có thể xảy ra khi làm việc với các file. Ngoài việc đọc thư mục, chúng ta có thể gặp sự cố khi mở từng file riêng lẻ. Hãy thay đổi API của chúng ta (thông qua các bản kiểm thử trước, đương nhiên) để nó có thể trả về một `error`.

```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("hi")},
		"hello-world2.md": {Data: []byte("hola")},
	}

	posts, err := blogposts.NewPostsFromFS(fs)

	if err != nil {
		t.Fatal(err)
	}

	if len(posts) != len(fs) {
		t.Errorf("got %d posts, wanted %d posts", len(posts), len(fs))
	}
}
```

Chạy bản kiểm thử: nó sẽ phàn nàn về số lượng giá trị trả về sai. Việc sửa mã nguồn khá đơn giản.

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for range dir {
		posts = append(posts, Post{})
	}
	return posts, nil
}
```

Điều này sẽ làm cho bản kiểm thử vượt qua. Người thực hành TDD trong bạn có thể thấy khó chịu vì chúng ta không thấy một bản kiểm thử thất bại trước khi viết mã nguồn để lan truyền lỗi từ `fs.ReadDir`. Để làm điều này "đúng đắn", chúng ta sẽ cần một bản kiểm thử mới nơi chúng ta truyền vào một `fs.FS` giả (test-double) bị lỗi để làm cho `fs.ReadDir` trả về một `error`.

```go
type StubFailingFS struct {
}

func (s StubFailingFS) Open(name string) (fs.File, error) {
	return nil, errors.New("oh no, i always fail")
}
```
```go
// sau đó
_, err := blogposts.NewPostsFromFS(StubFailingFS{})
```

Điều này sẽ mang lại cho bạn sự tự tin vào cách tiếp cận của chúng ta. Interface mà chúng ta đang sử dụng chỉ có một phương thức, điều này làm cho việc tạo các test-double để kiểm thử các kịch bản khác nhau trở nên dễ dàng.

Trong một số trường hợp, kiểm thử việc xử lý lỗi là điều thực tế nên làm, nhưng trong trường hợp của chúng ta, chúng ta không làm bất cứ điều gì *thú vị* với lỗi đó, chúng ta chỉ lan truyền nó, vì vậy việc viết một bản kiểm thử mới là không đáng công sức.

Về logic, các lần lặp lại tiếp theo của chúng ta sẽ xoay quanh việc mở rộng kiểu `Post` để nó có một số dữ liệu hữu ích.

## Viết test trước tiên

Chúng ta sẽ bắt đầu với dòng đầu tiên trong sơ đồ bài đăng blog dự kiến, trường tiêu đề (title).

Chúng ta cần thay đổi nội dung của các file kiểm thử sao cho chúng khớp với những gì đã chỉ định, sau đó chúng ta có thể thực hiện một xác nhận (assertion) rằng nó được phân tích chính xác.

```go
func TestNewBlogPosts(t *testing.T) {
	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte("Title: Post 1")},
		"hello-world2.md": {Data: []byte("Title: Post 2")},
	}

	// phần còn lại của mã kiểm thử được lược bỏ cho ngắn gọn
	got := posts[0]
	want := blogposts.Post{Title: "Post 1"}

	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

## Thử chạy test
```
./blogpost_test.go:58:26: unknown field 'Title' in struct literal of type blogposts.Post
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Thêm trường mới vào kiểu `Post` của chúng ta để bản kiểm thử có thể chạy

```go
type Post struct {
	Title string
}
```

Chạy lại bản kiểm thử, và bạn sẽ nhận được một kết quả thất bại rõ ràng

```
=== RUN   TestNewBlogPosts
=== RUN   TestNewBlogPosts/parses_the_post
    blogpost_test.go:61: got {Title:}, want {Title:Post 1}
```

## Viết đủ code để test chạy thành công

Chúng ta sẽ cần mở từng file và sau đó trích xuất tiêu đề

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f)
		if err != nil {
			return nil, err //todo: cần làm rõ, chúng ta nên hoàn toàn thất bại nếu một file bị lỗi? hay chỉ bỏ qua?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()

	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

Lưu ý rằng trọng tâm của chúng ta tại thời điểm này không phải là viết mã nguồn đẹp đẽ, mà chỉ là đạt được một điểm mà chúng ta có phần mềm hoạt động.

Mặc dù điều này mang lại cảm giác là một bước tiến nhỏ nhưng nó vẫn yêu cầu chúng ta viết một lượng mã nguồn đáng kể và đưa ra một số giả định về cách xử lý lỗi. Đây sẽ là thời điểm bạn nên nói chuyện với đồng nghiệp của mình và quyết định cách tiếp cận tốt nhất.

Cách tiếp cận lặp lại đã mang lại cho chúng ta những phản hồi nhanh chóng rằng hiểu biết của chúng ta về các yêu cầu là chưa đầy đủ.

`fs.FS` cung cấp cho chúng ta một cách để mở một file bên trong nó theo tên bằng phương thức `Open`. Từ đó, chúng ta đọc dữ liệu từ file và hiện tại, chúng ta không cần bất kỳ phân tích phức tạp nào, chỉ cần cắt bỏ phần văn bản `Title: ` bằng cách cắt (slicing) chuỗi.

## Refactor

Việc tách đoạn 'mã mở file' khỏi đoạn 'mã phân tích nội dung file' sẽ làm cho mã nguồn trở nên đơn giản hơn để hiểu và làm việc.

```go
func getPost(fileSystem fs.FS, f fs.DirEntry) (Post, error) {
	postFile, err := fileSystem.Open(f.Name())
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile fs.File) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

Khi bạn tách ra các hàm hoặc phương thức mới, hãy cẩn thận và suy nghĩ về các đối số. Bạn đang thiết kế ở đây, và bạn có một sự tự do để suy nghĩ sâu sắc về những gì là phù hợp vì bạn có các bản kiểm thử đã vượt qua. Hãy nghĩ về sự ràng buộc (coupling) và sự gắn kết (cohesion). Trong trường hợp này, bạn nên tự hỏi bản thân:

> `newPost` có phải bị ràng buộc vào một `fs.File` không? Chúng ta có sử dụng tất cả các phương thức và dữ liệu từ kiểu này không? Chúng ta *thực sự* cần cái gì?

Trong trường hợp của chúng ta, chúng ta chỉ sử dụng nó làm đối số cho `io.ReadAll`, nơi cần một `io.Reader`. Vì vậy, chúng ta nên nới lỏng sự ràng buộc trong hàm của mình và yêu cầu một `io.Reader`.

```go
func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

Bạn có thể đưa ra một lập luận tương tự cho hàm `getPost`, hàm này nhận vào một đối số `fs.DirEntry` nhưng chỉ gọi `Name()` để lấy tên file. Chúng ta không cần tất cả những thứ đó; hãy tách rời khỏi kiểu dữ liệu đó và truyền tên file qua dưới dạng một chuỗi. Đây là mã nguồn đã được tái cấu trúc hoàn chỉnh:

```go
func NewPostsFromFS(fileSystem fs.FS) ([]Post, error) {
	dir, err := fs.ReadDir(fileSystem, ".")
	if err != nil {
		return nil, err
	}
	var posts []Post
	for _, f := range dir {
		post, err := getPost(fileSystem, f.Name())
		if err != nil {
			return nil, err //todo: cần làm rõ, chúng ta nên hoàn toàn thất bại nếu một file bị lỗi? hay chỉ bỏ qua?
		}
		posts = append(posts, post)
	}
	return posts, nil
}

func getPost(fileSystem fs.FS, fileName string) (Post, error) {
	postFile, err := fileSystem.Open(fileName)
	if err != nil {
		return Post{}, err
	}
	defer postFile.Close()
	return newPost(postFile)
}

func newPost(postFile io.Reader) (Post, error) {
	postData, err := io.ReadAll(postFile)
	if err != nil {
		return Post{}, err
	}

	post := Post{Title: string(postData)[7:]}
	return post, nil
}
```

Từ giờ trở đi, hầu hết các nỗ lực của chúng ta có thể được gói gọn trong `newPost`. Mối quan tâm về việc mở và lặp lại qua các file đã hoàn tất, và bây giờ chúng ta có thể tập trung vào việc trích xuất dữ liệu cho kiểu `Post` của mình. Mặc dù về mặt kỹ thuật là không bắt buộc, các file là một cách tốt để nhóm các thứ liên quan lại với nhau một cách logic, vì vậy tôi đã di chuyển kiểu `Post` và `newPost` sang một file `post.go` mới.

### Test helper

Chúng ta cũng nên chăm sóc các bản kiểm thử của mình. Chúng ta sẽ thực hiện nhiều xác nhận trên `Post`, vì vậy chúng ta nên viết một số mã nguồn để trợ giúp việc đó

```go
func assertPost(t *testing.T, got blogposts.Post, want blogposts.Post) {
	t.Helper()
	if !reflect.DeepEqual(got, want) {
		t.Errorf("got %+v, want %+v", got, want)
	}
}
```

```go
assertPost(t, posts[0], blogposts.Post{Title: "Post 1"})
```

## Viết test trước tiên

Hãy mở rộng bản kiểm thử hơn nữa để trích xuất dòng tiếp theo từ file, phần mô tả (description). Cho đến khi làm cho nó vượt qua, mọi thứ lúc này sẽ mang lại cảm giác thoải mái và quen thuộc.

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1`
		secondBody = `Title: Post 2
Description: Description 2`
	)

	fs := fstest.MapFS{
		"hello world.md":  {Data: []byte(firstBody)},
		"hello-world2.md": {Data: []byte(secondBody)},
	}

	// phần còn lại của mã kiểm thử được lược bỏ cho ngắn gọn
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
	})

}
```

## Thử chạy test

```
./blogpost_test.go:47:58: unknown field 'Description' in struct literal of type blogposts.Post
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Thêm trường mới vào `Post`.

```go
type Post struct {
	Title       string
	Description string
}
```

Bản kiểm thử lúc này có thể biên dịch được, và thất bại.

```
=== RUN   TestNewBlogPosts
    blogpost_test.go:47: got {Title:Post 1
        Description: Description 1 Description:}, want {Title:Post 1 Description:Description 1}
```

## Viết đủ code để test chạy thành công

Thư viện tiêu chuẩn có một gói tiện lợi giúp bạn quét qua dữ liệu, từng dòng một: [`bufio.Scanner`](https://golang.org/pkg/bufio/#Scanner).

> Scanner cung cấp một interface thuận tiện để đọc dữ liệu như một file chứa các dòng văn bản được phân tách bằng dấu xuống dòng.

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	scanner.Scan()
	titleLine := scanner.Text()

	scanner.Scan()
	descriptionLine := scanner.Text()

	return Post{Title: titleLine[7:], Description: descriptionLine[13:]}, nil
}
```

Thật tiện lợi, nó cũng nhận một `io.Reader` để đọc qua (cảm ơn một lần nữa sự ràng buộc lỏng lẻo), chúng ta không cần thay đổi các đối số của hàm.

Gọi `Scan` để đọc một dòng, sau đó trích xuất dữ liệu bằng `Text`.

Hàm này có thể không bao giờ trả về `error`. Sẽ là rất cám dỗ nếu lúc này xóa nó khỏi các kiểu trả về, nhưng chúng ta biết rằng sau này mình sẽ phải xử lý các cấu trúc file không hợp lệ, vì vậy chúng ta có thể để nguyên như vậy.

## Refactor

Chúng ta có sự lặp lại xung quanh việc quét một dòng và sau đó đọc văn bản. Chúng ta biết mình sẽ thực hiện thao tác này ít nhất một lần nữa, đây là một lần tái cấu trúc đơn giản để DRY mã nguồn, hãy bắt đầu với điều đó.

```go
func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[7:]
	description := readLine()[13:]

	return Post{Title: title, Description: description}, nil
}
```

Điều này hầu như không tiết kiệm được dòng mã nào, nhưng đó hiếm khi là mục tiêu của việc tái cấu trúc. Những gì tôi đang cố gắng làm ở đây chỉ là tách biệt phần *cái gì* khỏi phần *làm thế nào* của việc đọc các dòng để làm cho mã nguồn mang tính mô tả (declarative) hơn một chút đối với người đọc.

Mặc dù các con số ma thuật 7 và 13 có thể hoàn thành công việc, nhưng chúng không thực sự mang tính mô tả.

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
)

func newPost(postFile io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postFile)

	readLine := func() string {
		scanner.Scan()
		return scanner.Text()
	}

	title := readLine()[len(titleSeparator):]
	description := readLine()[len(descriptionSeparator):]

	return Post{Title: title, Description: description}, nil
}
```

Bây giờ khi tôi đang nhìn vào mã nguồn với tư duy tái cấu trúc sáng tạo, tôi muốn thử làm cho hàm `readLine` của mình đảm nhận việc loại bỏ thẻ. Cũng có một cách dễ đọc hơn để cắt bỏ tiền tố khỏi một chuỗi bằng hàm `strings.TrimPrefix`.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
	}, nil
}
```

Bạn có thể thích ý tưởng này hoặc không, nhưng tôi thì thích. Vấn đề là ở giai đoạn tái cấu trúc, chúng ta được tự do chơi đùa với các chi tiết bên trong, và bạn có thể tiếp tục chạy các bản kiểm thử để kiểm tra xem mọi thứ vẫn hoạt động chính xác hay không. Chúng ta luôn có thể quay lại các trạng thái trước đó nếu không hài lòng. Cách tiếp cận TDD mang lại cho chúng ta quyền được thường xuyên thử nghiệm các ý tưởng, nhờ đó có nhiều cơ hội hơn để viết được mã nguồn tuyệt vời.

Yêu cầu tiếp theo là trích xuất các thẻ (tags) của bài đăng. Nếu bạn đang theo dõi, tôi khuyên bạn nên tự mình triển khai nó trước khi đọc tiếp. Bây giờ bạn đã có một nhịp độ lặp lại tốt và cảm thấy tự tin để trích xuất dòng tiếp theo và phân tích dữ liệu.

Để ngắn gọn, tôi sẽ không đi qua các bước TDD, nhưng đây là bản kiểm thử với các thẻ được thêm vào.

```go
func TestNewBlogPosts(t *testing.T) {
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker`
	)

	// phần còn lại của mã kiểm thử được lược bỏ cho ngắn gọn
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
	})
}
```

Bạn sẽ chỉ đang tự lừa dối mình nếu chỉ sao chép và dán những gì tôi viết. Để đảm bảo tất cả chúng ta đều chung một quan điểm, đây là mã nguồn bao gồm việc trích xuất các thẻ.

```go
const (
	titleSeparator       = "Title: "
	descriptionSeparator = "Description: "
	tagsSeparator        = "Tags: "
)

func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
	}, nil
}
```

Hy vọng không có gì bất ngờ ở đây. Chúng ta đã có thể tái sử dụng `readMetaLine` để lấy dòng tiếp theo cho các thẻ và sau đó chia chúng bằng `strings.Split`.

Lần lặp lại cuối cùng trên happy-path của chúng ta là trích xuất phần nội dung (body).

Đây là lời nhắc nhở về định dạng file được đề xuất.

```markdown
Title: Hello, TDD world!
Description: First post on our wonderful blog
Tags: tdd, go
---
Hello world!

The body of posts starts after the `---`
```

Chúng ta đã đọc xong 3 dòng đầu tiên. Sau đó chúng ta cần đọc thêm một dòng nữa, bỏ qua nó và phần còn lại của file sẽ chứa nội dung bài đăng.

## Viết test trước tiên

Thay đổi dữ liệu kiểm thử để có dấu phân cách, và một phần nội dung với một vài dòng mới để kiểm tra xem chúng ta có lấy được tất cả nội dung không.

```go
	const (
		firstBody = `Title: Post 1
Description: Description 1
Tags: tdd, go
---
Hello
World`
		secondBody = `Title: Post 2
Description: Description 2
Tags: rust, borrow-checker
---
B
L
M`
	)
```

Thêm vào xác nhận của chúng ta giống như các phần khác

```go
	assertPost(t, posts[0], blogposts.Post{
		Title:       "Post 1",
		Description: "Description 1",
		Tags:        []string{"tdd", "go"},
		Body: `Hello
World`,
	})
```

## Thử chạy test

```
./blogpost_test.go:60:3: unknown field 'Body' in struct literal of type blogposts.Post
```

Như chúng ta mong đợi.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Thêm `Body` vào `Post` và bản kiểm thử sẽ thất bại.

```
=== RUN   TestNewBlogPosts
    blogposts_test.go:38: got {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:}, want {Title:Post 1 Description:Description 1 Tags:[tdd go] Body:Hello
        World}
```

## Viết đủ code để test chạy thành công

1. Quét dòng tiếp theo để bỏ qua dấu phân cách `---`.
2. Tiếp tục quét cho đến khi không còn gì để quét.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	title := readMetaLine(titleSeparator)
	description := readMetaLine(descriptionSeparator)
	tags := strings.Split(readMetaLine(tagsSeparator), ", ")

	scanner.Scan() // bỏ qua một dòng

	buf := bytes.Buffer{}
	for scanner.Scan() {
		fmt.Fprintln(&buf, scanner.Text())
	}
	body := strings.TrimSuffix(buf.String(), "\n")

	return Post{
		Title:       title,
		Description: description,
		Tags:        tags,
		Body:        body,
	}, nil
}
```

- `scanner.Scan()` trả về một giá trị `bool` cho biết liệu còn dữ liệu để quét hay không, vì vậy chúng ta có thể sử dụng giá trị đó với một vòng lặp `for` để tiếp tục đọc qua dữ liệu cho đến khi kết thúc.
- Sau mỗi bản `Scan()`, chúng ta ghi dữ liệu vào buffer bằng `fmt.Fprintln`. Chúng ta sử dụng phiên bản thêm một dòng mới vì scanner loại bỏ các dòng mới khỏi mỗi dòng, nhưng chúng ta cần duy trì chúng.
- Vì lý do trên, chúng ta cần cắt bỏ ký tự xuống dòng cuối cùng, để không gặp phải ký tự dư thừa ở cuối.

## Refactor

Việc đóng gói ý tưởng lấy phần dữ liệu còn lại vào một hàm sẽ giúp những người đọc trong tương lai nhanh chóng hiểu *cái gì* đang xảy ra trong `newPost`, mà không cần phải bận tâm đến các chi tiết thực thi cụ thể.

```go
func newPost(postBody io.Reader) (Post, error) {
	scanner := bufio.NewScanner(postBody)

	readMetaLine := func(tagName string) string {
		scanner.Scan()
		return strings.TrimPrefix(scanner.Text(), tagName)
	}

	return Post{
		Title:       readMetaLine(titleSeparator),
		Description: readMetaLine(descriptionSeparator),
		Tags:        strings.Split(readMetaLine(tagsSeparator), ", "),
		Body:        readBody(scanner),
	}, nil
}

func readBody(scanner *bufio.Scanner) string {
	scanner.Scan() // bỏ qua một dòng
	buf := bytes.Buffer{}
	for scanner.Scan() {
		fmt.Fprintln(&buf, scanner.Text())
	}
	return strings.TrimSuffix(buf.String(), "\n")
}
```

## Tiếp tục lặp lại

Chúng ta đã tạo ra "sợi dây thép" cho chức năng của mình, đi con đường ngắn nhất để đến được happy-path, nhưng rõ ràng vẫn còn một chặng đường trước khi nó sẵn sàng để đi vào sản xuất.

Chúng ta vẫn chưa xử lý:

- khi định dạng của file không đúng
- file không phải là `.md`
- điều gì sẽ xảy ra nếu thứ tự của các trường metadata khác nhau? Điều đó có nên được phép không? Chúng ta có nên xử lý nó không?

Tuy nhiên, quan trọng nhất là chúng ta có phần mềm hoạt động, và chúng ta đã định nghĩa interface của mình. Những điều trên chỉ là các lần lặp lại tiếp theo, thêm nhiều bản kiểm thử hơn để viết và thúc đẩy hành vi của chúng ta. Để hỗ trợ bất kỳ hành vi nào ở trên, chúng ta không cần phải thay đổi *thiết kế*, chỉ cần các chi tiết thực thi.

Việc tập trung vào mục tiêu có nghĩa là chúng ta đã đưa ra các quyết định quan trọng, và xác nhận chúng dựa trên hành vi mong muốn, thay vì bị sa lầy vào những vấn đề không ảnh hưởng đến thiết kế tổng thể.

## Tổng kết

`fs.FS`, và những thay đổi khác trong Go 1.16 mang lại cho chúng ta một số cách trang nhã để đọc dữ liệu từ hệ thống file và kiểm thử chúng một cách đơn giản.

Nếu bạn muốn thử nghiệm mã nguồn "thật":

- Tạo một thư mục `cmd` bên trong dự án, thêm một file `main.go`
- Thêm mã nguồn sau

```go
import (
	blogposts "github.com/quii/fstest-spike"
	"log"
	"os"
)

func main() {
	posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
	if err != nil {
		log.Fatal(err)
	}
	log.Println(posts)
}
```

- Thêm một số file markdown vào thư mục `posts` và chạy chương trình!

Lưu ý sự đối xứng giữa mã nguồn sản xuất

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

Và các bản kiểm thử

```go
posts, err := blogposts.NewPostsFromFS(fs)
```

Đây là khi phương pháp TDD hướng người tiêu dùng, từ trên xuống dưới (top-down) *cảm thấy đúng đắn*.

Người dùng package của chúng ta có thể nhìn vào các bản kiểm thử của chúng ta và nhanh chóng nắm bắt được nó phải làm gì và cách sử dụng nó như thế nào. Với tư cách là những người bảo trì, chúng ta có thể *tự tin rằng các bản kiểm thử của mình là hữu ích vì chúng được đứng từ quan điểm của người tiêu dùng*. Chúng ta không kiểm thử các chi tiết thực thi hoặc các chi tiết phụ khác, vì vậy chúng ta có thể tin tưởng một cách hợp lý rằng các bản kiểm thử của mình sẽ giúp ích, thay vì cản trở chúng ta khi tái cấu trúc.

Bằng cách dựa trên các thực hành kỹ thuật phần mềm tốt như [**Dependency Injection**](dependency-injection.md), mã nguồn của chúng ta trở nên đơn giản để kiểm thử và tái sử dụng.

Khi bạn tạo các package, ngay cả khi chúng chỉ là nội bộ trong dự án của bạn, hãy ưu tiên cách tiếp cận hướng người tiêu dùng từ trên xuống dưới. Điều này sẽ ngăn bạn tưởng tượng quá mức các thiết kế và tạo ra các sự trừu tượng có thể bạn thậm chí không cần, và sẽ giúp đảm bảo các bản kiểm thử bạn viết là hữu ích.

Cách tiếp cận lặp lại giữ cho mỗi bước đều nhỏ, và các phản hồi liên tục đã giúp chúng ta phát hiện ra những yêu cầu chưa rõ ràng có lẽ sớm hơn so với các phương pháp khác mang tính tùy hứng hơn.

### Ghi file?

Điều quan trọng cần lưu ý là các tính năng mới này chỉ có các hoạt động để *đọc* file. Nếu công việc của bạn cần thực hiện việc ghi, bạn sẽ cần tìm giải pháp khác. Hãy nhớ luôn suy nghĩ về những gì thư viện tiêu chuẩn hiện đang cung cấp, nếu bạn đang ghi dữ liệu, bạn có lẽ nên xem xét việc tận dụng các interface hiện có như `io.Writer` để giữ cho mã nguồn của mình được ràng buộc lỏng lẻo và có thể tái sử dụng.

### Đọc thêm

- Đây là lời giới thiệu nhẹ nhàng về `io/fs`. [Ben Congdon đã thực hiện một bài viết xuất sắc](https://benjamincongdon.me/blog/2021/01/21/A-Tour-of-Go-116s-iofs-package/), bài viết này đã giúp ích rất nhiều cho việc viết chương này.
- [Thảo luận về các interface hệ thống file](https://github.com/golang/go/issues/41190)
