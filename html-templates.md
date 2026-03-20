# HTML Templates

**[Bạn có thể tìm thấy toàn bộ mã nguồn tại đây](https://github.com/quii/learn-go-with-tests/tree/main/blogrenderer)**

Chúng ta đang sống trong một thế giới mà mọi người đều muốn xây dựng các ứng dụng web với các framework frontend mới nhất của tháng, được xây dựng trên hàng gigabyte JavaScript đã qua chuyển đổi (transpiled), hoạt động với một hệ thống build phức tạp; [nhưng có lẽ điều đó không phải lúc nào cũng cần thiết](https://quii.dev/The_Web_I_Want).

Tôi muốn nói rằng hầu hết các nhà phát triển Go đều coi trọng một chuỗi công cụ (toolchain) đơn giản, ổn định và nhanh chóng nhưng thế giới frontend thường xuyên thất bại trong việc mang lại điều này.

Nhiều trang web không cần phải là một [SPA](https://en.wikipedia.org/wiki/Single-page_application). **HTML và CSS là những cách tuyệt vời để phân phối nội dung** và bạn có thể sử dụng Go để tạo một trang web phân phối HTML.

Nếu bạn vẫn muốn có một số yếu tố động, bạn vẫn có thể thêm vào một chút JavaScript phía client, hoặc bạn thậm chí có thể muốn thử nghiệm với [Hotwire](https://hotwired.dev) cho phép bạn mang lại trải nghiệm động với cách tiếp cận phía server.

Bạn có thể tạo HTML trong Go bằng cách sử dụng công phu [`fmt.Fprintf`](https://pkg.go.dev/fmt#Fprintf), nhưng trong chương này, bạn sẽ học được rằng thư viện tiêu chuẩn của Go có một số công cụ để tạo HTML một cách đơn giản và dễ bảo trì hơn. Bạn cũng sẽ học được những cách kiểm thử loại mã nguồn này hiệu quả hơn mà bạn có thể chưa từng thấy trước đây.

## Những gì chúng ta sẽ xây dựng

Trong chương [Đọc file](/reading-files.md), chúng ta đã viết một số mã nguồn nhận vào một [`fs.FS`](https://pkg.go.dev/io/fs) (một hệ thống file) và trả về một slice các `Post` cho mỗi file markdown mà nó gặp.

```go
posts, err := blogposts.NewPostsFromFS(os.DirFS("posts"))
```

Đây là cách chúng ta định nghĩa `Post`

```go
type Post struct {
	Title, Description, Body string
	Tags                     []string
}
```

Đây là một ví dụ về một trong các file markdown có thể được phân tích.

```markdown
Title: Welcome to my blog
Description: Introduction to my blog
Tags: cooking, family, live-laugh-love
---
# First recipe!
Welcome to my **amazing recipe blog**. I am going to write about my family recipes, and make sure I write a long, irrelevant and boring story about my family before you get to the actual instructions.
```

Nếu tiếp tục hành trình viết phần mềm blog, chúng ta sẽ lấy dữ liệu này và tạo HTML từ đó để web server trả về khi phản hồi các yêu cầu HTTP.

Đối với blog của mình, chúng ta muốn tạo hai loại trang:

1. **Xem bài đăng**. Hiển thị một bài đăng cụ thể. Trường `Body` trong `Post` là một chuỗi chứa markdown, vì vậy nó nên được chuyển đổi sang HTML.
2. **Trang chủ (Index)**. Liệt kê tất cả các bài đăng, với các siêu liên kết để xem bài đăng cụ thể.

Chúng ta cũng muốn có một giao diện nhất quán trên toàn bộ trang web của mình, vì vậy đối với mỗi trang, chúng ta sẽ có các thành phần HTML thông thường như `<html>` và `<head>` chứa các liên kết đến các bản định kiểu (stylesheets) CSS và bất kỳ thứ gì khác mà chúng ta muốn.

Khi bạn xây dựng phần mềm blog, bạn có một vài tùy chọn về cách tiếp cận cách xây dựng và gửi HTML đến trình duyệt của người dùng.

Chúng ta sẽ thiết kế mã nguồn của mình để nó chấp nhận một `io.Writer`. Điều này có nghĩa là người gọi mã nguồn của chúng ta có sự linh hoạt để:

- Ghi chúng vào một [os.File](https://pkg.go.dev/os#File), để chúng có thể được phục vụ tĩnh
- Ghi trực tiếp HTML ra một [`http.ResponseWriter`](https://pkg.go.dev/net/http#ResponseWriter)
- Hoặc chỉ cần ghi chúng vào bất cứ thứ gì! Miễn là nó triển khai `io.Writer`, người dùng có thể tạo một số HTML từ một `Post`

## Viết test trước tiên

Như mọi khi, điều quan trọng là phải suy nghĩ về các yêu cầu trước khi dấn thân quá nhanh. Làm thế nào chúng ta có thể lấy tập hợp các yêu cầu có vẻ lớn như thế này và chia nó thành các bước nhỏ, có thể đạt được mà chúng ta có thể tập trung vào?

Theo quan điểm của tôi, việc thực sự xem nội dung có ưu tiên cao hơn một trang chủ. Chúng ta có thể khởi chạy sản phẩm này và chia sẻ các liên kết trực tiếp đến nội dung tuyệt vời của mình. Một trang chủ mà không thể liên kết đến nội dung thực tế thì không hữu ích.

Tuy nhiên, việc hiển thị một bài đăng như mô tả ở trên vẫn có cảm giác là quá lớn. Toàn bộ khung HTML, chuyển đổi markdown nội dung thành HTML, liệt kê các thẻ, v.v.

Tại giai đoạn này, tôi không quá bận tâm đến markup cụ thể, và một bước đầu tiên dễ dàng sẽ là chỉ cần kiểm tra xem chúng ta có thể hiển thị tiêu đề của bài đăng dưới dạng một thẻ `<h1>` hay không. Điều này mang lại *cảm giác* là bước đi đầu tiên nhỏ nhất có thể giúp chúng ta tiến về phía trước một chút.

```go
package blogrenderer_test

import (
	"bytes"
	"github.com/quii/learn-go-with-tests/blogrenderer"
	"testing"
)

func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>`
		if got != want {
			t.Errorf("got '%s' want '%s'", got, want)
		}
	})
}
```

Quyết định của chúng ta chấp nhận một `io.Writer` cũng làm cho việc kiểm thử trở nên đơn giản, trong trường hợp này chúng ta đang ghi vào một [`bytes.Buffer`](https://pkg.go.dev/bytes#Buffer) mà sau đó chúng ta có thể kiểm tra nội dung của nó.

## Thử chạy test

Nếu bạn đã đọc các chương trước của cuốn sách này, bây giờ bạn hẳn đã thực hành thuần thục điều này. Bạn sẽ không thể chạy bản kiểm thử vì chúng ta chưa định nghĩa package hoặc hàm `Render`. Hãy thử tự mình làm theo các thông báo của trình biên dịch và đưa mã nguồn về trạng thái có thể chạy được bản kiểm thử và thấy rằng nó thất bại với một thông báo rõ ràng.

Điều thực sự quan trọng là bạn phải thực hành việc các bản kiểm thử của mình thất bại, bạn sẽ cảm ơn bản thân khi vô tình làm một bản kiểm thử thất bại sau 6 tháng mà bạn đã bỏ công sức *ngay bây giờ* để kiểm tra xem nó có thất bại với một thông báo rõ ràng hay không.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Đây là lượng mã nguồn tối thiểu để bản kiểm thử có thể chạy được

```go
package blogrenderer

// nếu bạn đang tiếp tục từ chương đọc file, bạn không nên định nghĩa lại cái này
type Post struct {
	Title, Description, Body string
	Tags                     []string
}

func Render(w io.Writer, p Post) error {
	return nil
}
```

Bản kiểm thử sẽ phàn nàn rằng một chuỗi rỗng không bằng những gì chúng ta muốn.

## Viết đủ code để test chạy thành công

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1>", p.Title)
	return err
}
```

Hãy nhớ rằng, phát triển phần mềm chủ yếu là một hoạt động học tập. Để khám phá và học hỏi khi làm việc, chúng ta cần làm việc theo cách mang lại cho mình những vòng lặp phản hồi thường xuyên, chất lượng cao, và cách dễ nhất để làm điều đó là làm việc theo từng bước nhỏ.

Vì vậy, chúng ta không cần lo lắng về việc sử dụng bất kỳ thư viện templating nào ngay bây giờ. Bạn có thể tạo HTML chỉ với templating chuỗi "thông thường" một cách hoàn toàn tốt, và bằng cách bỏ qua phần template, chúng ta có thể xác thực một phần nhỏ hành vi hữu ích và chúng ta đã thực hiện một phần nhỏ công việc thiết kế cho API của package.

## Refactor

Chưa có gì nhiều để tái cấu trúc, vì vậy hãy chuyển sang lần lặp lại tiếp theo

## Viết test trước tiên

Bây giờ chúng ta đã có một phiên bản cơ bản hoạt động, chúng ta có thể lặp lại bản kiểm thử để mở rộng chức năng. Trong trường hợp này, hiển thị thêm thông tin từ `Post`.

```go
	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}
		err := blogrenderer.Render(&buf, aPost)

		if err != nil {
			t.Fatal(err)
		}

		got := buf.String()
		want := `<h1>hello world</h1>
<p>This is a description</p>
Tags: <ul><li>go</li><li>tdd</li></ul>`

		if got != want {
			t.Errorf("got '%s' want '%s'", got, want)
		}
	})
```

Lưu ý rằng khi viết điều này, bạn sẽ *cảm thấy* kỳ cục. Nhìn thấy tất cả các markup đó trong bản kiểm thử cảm thấy không ổn, và chúng ta còn chưa đưa phần nội dung vào, hoặc HTML thực tế mà chúng ta muốn với tất cả nội dung của `<head>` và bất kỳ thành phần trang web nào chúng ta cần.

Tuy nhiên, hãy chấp nhận sự đau đớn đó *lúc này*.

## Thử chạy test

Nó sẽ thất bại, phàn nàn rằng nó không có chuỗi mà chúng ta mong đợi, vì chúng ta không hiển thị mô tả và các thẻ.

## Viết đủ code để test chạy thành công

Hãy thử tự mình làm điều này thay vì sao chép mã nguồn. Những gì bạn sẽ thấy là làm cho bản kiểm thử này vượt qua *là một chút khó chịu*! Khi tôi thử, lần thử đầu tiên của tôi đã gặp lỗi này

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:32: got '<h1>hello world</h1><p>This is a description</p><ul><li>go</li><li>tdd</li></ul>' want '<h1>hello world</h1>
        <p>This is a description</p>
        Tags: <ul><li>go</li><li></li></ul>'
```

Các dòng mới! Ai quan tâm chứ? Bản kiểm thử của chúng ta có quan tâm, vì nó khớp chính xác trên một giá trị chuỗi. Có nên như vậy không? Tôi đã tạm thời xóa các dòng mới chỉ để đưa bản kiểm thử vượt qua.

```go
func Render(w io.Writer, p Post) error {
	_, err := fmt.Fprintf(w, "<h1>%s</h1><p>%s</p>", p.Title, p.Description)
	if err != nil {
		return err
	}

	_, err = fmt.Fprint(w, "Tags: <ul>")
	if err != nil {
		return err
	}

	for _, tag := range p.Tags {
		_, err = fmt.Fprintf(w, "<li>%s</li>", tag)
		if err != nil {
			return err
		}
	}

	_, err = fmt.Fprint(w, "</ul>")
	if err != nil {
		return err
	}

	return nil
}
```

**Yikes**. Không phải là đoạn mã nguồn đẹp nhất mà tôi từng viết, và chúng ta mới chỉ đang ở giai đoạn triển khai markup rất sớm. Chúng ta sẽ cần thêm rất nhiều nội dung và các thứ khác trên trang web của mình, chúng ta nhanh chóng nhận ra rằng cách tiếp cận này là không phù hợp.

Tuy nhiên, quan trọng nhất là chúng ta có một bản kiểm thử đã vượt qua; chúng ta có phần mềm đang hoạt động.

## Refactor

Với tấm lưới an toàn của một bản kiểm thử đã vượt qua cho mã nguồn đang hoạt động, bây giờ chúng ta có thể nghĩ về việc thay đổi cách tiếp cận triển khai của mình ở giai đoạn tái cấu trúc.

### Giới thiệu về template

Go có hai package templating [text/template](https://pkg.go.dev/text/template) và [html/template](https://pkg.go.dev/html/template) và chúng chia sẻ cùng một interface. Những gì cả hai làm là cho phép bạn kết hợp một mẫu (template) và một số dữ liệu để tạo ra một chuỗi.

Sự khác biệt với phiên bản HTML là gì?

> Package template (html/template) triển khai các mẫu dựa trên dữ liệu để tạo đầu ra là HTML an toàn trước các cuộc tấn công tiêm mã (code injection). Nó cung cấp cùng một interface như package text/template và nên được sử dụng thay vì text/template bất cứ khi nào đầu ra là HTML.

Ngôn ngữ templating rất giống với [Mustache](https://mustache.github.io) và cho phép bạn tạo nội dung động một cách rất sạch sẽ với sự tách biệt rõ ràng các mối quan tâm (separation of concerns). So với các ngôn ngữ templating khác mà bạn có thể đã sử dụng, nó bị hạn chế hoặc "logic-less" (không logic) như Mustache thường nói. Đây là một quyết định thiết kế quan trọng **và có chủ đích**.

Mặc dù ở đây chúng ta tập trung vào việc tạo HTML, nhưng nếu dự án của bạn đang thực hiện các việc nối chuỗi phức tạp, bạn có thể muốn sử dụng `text/template` để làm sạch mã nguồn của mình.

### Quay lại với mã nguồn

Đây là một mẫu cho blog của chúng ta:

`<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`

Chúng ta định nghĩa chuỗi này ở đâu? Có một vài lựa chọn, nhưng để thực hiện các bước nhỏ, hãy bắt đầu với một chuỗi bình thường cũ

```go
package blogrenderer

import (
	"html/template"
	"io"
)

const (
	postTemplate = `<h1>{{.Title}}</h1><p>{{.Description}}</p>Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>`
)

func Render(w io.Writer, p Post) error {
	templ, err := template.New("blog").Parse(postTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

Chúng ta tạo một mẫu mới với một cái tên, và sau đó phân tích (parse) chuỗi mẫu của mình. Sau đó, chúng ta có thể sử dụng phương thức `Execute` trên đó, truyền vào dữ liệu của mình, trong trường hợp này là `Post`.

Mẫu sẽ thay thế những thứ như `{{.Description}}` bằng nội dung của `p.Description`. Các mẫu cũng cung cấp cho bạn một số nguyên hàm lập trình như `range` để lặp qua các giá trị, và `if`. Bạn có thể tìm thêm chi tiết trong [tài liệu text/template](https://pkg.go.dev/text/template).

*Đây nên là một lần tái cấu trúc thuần túy (pure refactor).* Chúng ta không cần thay đổi các bản kiểm thử và chúng nên tiếp tục vượt qua. Quan trọng hơn, mã nguồn của chúng ta dễ đọc hơn và ít phải đối mặt với các vấn đề xử lý lỗi khó chịu hơn.

Mọi người thường xuyên phàn nàn về sự rườm rà của việc xử lý lỗi trong Go, nhưng bạn có thể thấy rằng mình có thể tìm ra những cách tốt hơn để viết mã nguồn sao cho ban đầu nó ít gây ra lỗi hơn, như ở đây.

### Tái cấu trúc thêm nữa

Việc sử dụng `html/template` chắc chắn đã là một sự cải tiến, nhưng việc để nó dưới dạng hằng số chuỗi trong mã nguồn của chúng ta không phải là điều tuyệt vời:

- Nó vẫn khá khó đọc.
- Nó không thân thiện với IDE/trình soạn thảo mã nguồn. Không có cú pháp tô màu (syntax highlighting), khả năng định dạng lại, tái cấu trúc, v.v.
- Nó trông giống HTML, nhưng bạn không thực sự có thể làm việc với nó như một file HTML "thông thường"

Điều chúng ta muốn làm là để các mẫu của mình nằm trong các file riêng biệt để chúng ta có thể tổ chức chúng tốt hơn và làm việc với chúng như thể chúng là các file HTML.

Tạo một thư mục tên là "templates" và bên trong đó tạo một file tên là `blog.gohtml`, dán mẫu của chúng ta vào file đó.

Bây giờ hãy thay đổi mã nguồn của chúng ta để nhúng (embed) các hệ thống file bằng cách sử dụng [chức năng nhúng được bao gồm trong go 1.16](https://pkg.go.dev/embed).

```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	if err := templ.Execute(w, p); err != nil {
		return err
	}

	return nil
}
```

Bằng cách nhúng một "hệ thống file" vào mã nguồn của mình, chúng ta có thể tải nhiều mẫu và kết hợp chúng một cách tự do. Điều này sẽ trở nên hữu ích khi chúng ta muốn chia sẻ logic hiển thị giữa các mẫu khác nhau, chẳng hạn như phần header cho đầu trang HTML và phần footer.

### Nhúng (Embed)?

Nhúng đã được đề cập nhẹ nhàng trong chương [Đọc file](reading-files.md). [Tài liệu từ thư viện tiêu chuẩn giải thích](https://pkg.go.dev/embed)

> Package embed cung cấp quyền truy cập vào các file được nhúng trong chương trình Go đang chạy.
>
> Các file nguồn Go import "embed" có thể sử dụng chỉ thị //go:embed để khởi tạo các biến kiểu string, []byte, hoặc FS với nội dung của các file được đọc từ thư mục của package đó hoặc các thư mục con tại thời điểm biên dịch.

Tại sao chúng ta lại muốn sử dụng điều này? Một giải pháp thay thế là chúng ta *có thể* tải các mẫu của mình từ một hệ thống file "thông thường". Tuy nhiên, điều này có nghĩa là chúng ta phải đảm bảo rằng các mẫu nằm đúng đường dẫn file ở bất cứ nơi nào chúng ta muốn sử dụng phần mềm này. Trong công việc của mình, bạn có thể có nhiều môi trường khác nhau như phát triển (development), thử nghiệm (staging) và thực tế (live). Để điều này hoạt động, bạn cần đảm bảo rằng các mẫu của mình được sao chép đến đúng nơi.

Với embed, các file được bao gồm trong chương trình Go của bạn khi bạn xây dựng (build) nó. Điều này có nghĩa là một khi bạn đã xây dựng chương trình của mình (việc này bạn chỉ nên làm một lần), các file luôn có sẵn cho bạn.

Điều tiện lợi là bạn không chỉ có thể nhúng các file riêng lẻ mà còn cả hệ thống file; và hệ thống file đó triển khai [io/fs](https://pkg.go.dev/io/fs), điều này có nghĩa là mã nguồn của bạn không cần quan tâm nó đang làm việc với loại hệ thống file nào.

Tuy nhiên, nếu bạn muốn sử dụng các mẫu khác nhau tùy thuộc vào cấu hình, bạn có thể muốn tiếp tục tải các mẫu từ ổ đĩa theo cách thông thường hơn.

## Tiếp theo: Làm cho mẫu trở nên "đẹp"

Chúng ta không thực sự muốn mẫu của mình được định nghĩa dưới dạng một chuỗi dài một dòng. Chúng ta muốn có thể giãn cách nó ra để làm cho nó dễ đọc và dễ làm việc hơn, đại loại như thế này:

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

Nhưng nếu chúng ta làm điều này, bản kiểm thử của chúng ta sẽ thất bại. Điều này là do bản kiểm thử của chúng ta đang mong đợi một chuỗi rất cụ thể được trả về.

Nhưng thực sự, chúng ta không quan tâm đến khoảng trắng. Việc duy trì bản kiểm thử này sẽ trở thành một cơn ác mộng nếu chúng ta liên tục phải cập nhật chuỗi xác nhận một cách tỉ mỉ mỗi khi chúng ta thực hiện các thay đổi nhỏ đối với markup. Khi mẫu phát triển lên, những loại chỉnh sửa này trở nên khó quản lý hơn và chi phí công việc sẽ vượt quá tầm kiểm soát.

## Giới thiệu Approval Tests

[Go Approval Tests](https://github.com/approvals/go-approval-tests)

> ApprovalTests cho phép dễ dàng kiểm thử các đối tượng lớn, chuỗi và bất kỳ thứ gì khác có thể được lưu vào một file (hình ảnh, âm thanh, CSV, v.v.)

Ý tưởng tương tự như các file "vàng" (golden files), hoặc kiểm thử snapshot (snapshot testing). Thay vì duy trì các chuỗi một cách vụng về trong một file kiểm thử, công cụ phê duyệt (approval tool) có thể so sánh đầu ra cho bạn với một file "đã phê duyệt" (approved file) mà bạn đã tạo. Sau đó, bạn chỉ cần sao chép phiên bản mới qua nếu bạn phê duyệt nó. Chạy lại bản kiểm thử và bạn sẽ quay lại trạng thái xanh (vượt qua).

Thêm một phụ thuộc vào `"github.com/approvals/go-approval-tests"` vào dự án của bạn và chỉnh sửa bản kiểm thử thành như sau:

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := blogrenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

Lần đầu tiên bạn chạy nó, nó sẽ thất bại vì chúng ta chưa phê duyệt bất cứ thứ gì

```
=== RUN   TestRender
=== RUN   TestRender/it_converts_a_single_post_into_HTML
    renderer_test.go:29: Failed Approval: received does not match approved.
```

Nó sẽ tạo ra hai file, trông giống như sau:

- `renderer_test.TestRender.it_converts_a_single_post_into_HTML.received.txt`
- `renderer_test.TestRender.it_converts_a_single_post_into_HTML.approved.txt`

File "received" chứa phiên bản mới, chưa được phê duyệt của đầu ra. Sao chép nội dung đó vào file "approved" đang trống và chạy lại bản kiểm thử.

Bằng cách sao chép phiên bản mới, bạn đã "phê duyệt" thay đổi, và bản kiểm thử giờ đây đã vượt qua.

Để thấy quy trình làm việc thực tế, hãy chỉnh sửa mẫu theo cách chúng ta đã thảo luận để làm nó dễ đọc hơn (nhưng về mặt ngữ nghĩa, nó vẫn như cũ).

```handlebars
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
```

Chạy lại bản kiểm thử. Một file "received" mới sẽ được tạo ra vì đầu ra mã nguồn của chúng ta khác với phiên bản đã phê duyệt. Hãy xem qua chúng, và nếu bạn hài lòng với những thay đổi, chỉ cần sao chép phiên bản mới qua và chạy lại bản kiểm thử. Hãy nhớ commit các file đã phê duyệt vào hệ thống quản lý mã nguồn.

Cách tiếp cận này làm cho việc quản lý các thay đổi đối với những thứ lớn và lộn xộn như HTML trở nên đơn giản hơn nhiều. Bạn có thể sử dụng một công cụ diff để xem và quản lý những sự khác biệt, và nó giữ cho mã kiểm thử của bạn sạch sẽ hơn.

![Sử dụng công cụ diff để quản lý các thay đổi](https://i.imgur.com/0MoNdva.png)

Đây thực sự là một cách sử dụng khá nhỏ đối với approval tests, một công cụ cực kỳ hữu ích trong kho vũ khí kiểm thử của bạn. [Emily Bache](https://twitter.com/emilybache) có một [video thú vị nơi cô ấy sử dụng approval tests để thêm một bộ kiểm thử cực kỳ sâu rộng vào một mã nguồn phức tạp không có bản kiểm thử nào](https://www.youtube.com/watch?v=zyM2Ep28ED8). "Combinatorial Testing" chắc chắn là thứ đáng để tìm hiểu.

Bây giờ chúng ta đã thực hiện thay đổi này, chúng ta vẫn được hưởng lợi từ việc mã nguồn được kiểm thử tốt, nhưng các bản kiểm thử sẽ không gây cản trở quá nhiều khi chúng ta đang chỉnh sửa markup.

### Chúng ta có còn đang làm TDD không?

Một tác dụng phụ thú vị của cách tiếp cận này là nó đưa chúng ta rời xa TDD. Tất nhiên bạn *có thể* chỉnh sửa thủ công các file đã phê duyệt về trạng thái bạn muốn, chạy các bản kiểm thử và sau đó sửa các mẫu sao cho chúng xuất ra những gì bạn đã định nghĩa.

Nhưng điều đó thật ngớ ngẩn! TDD là một phương pháp để thực hiện công việc, cụ thể là thiết kế; nhưng điều đó không có nghĩa là chúng ta phải sử dụng nó cho **mọi thứ** một cách giáo điều.

Điều quan trọng là, chúng ta đã làm đúng việc và sử dụng TDD như một **công cụ thiết kế** để thiết kế API cho package của mình. Đối với các thay đổi về mẫu, quy trình của chúng ta có thể là:

- Thực hiện một thay đổi nhỏ đối với mẫu
- Chạy bản kiểm thử phê duyệt (approval test)
- Quan sát đầu ra để kiểm tra xem nó trông có đúng không
- Thực hiện phê duyệt
- Lặp lại

Chúng ta vẫn không nên từ bỏ giá trị của việc làm việc theo từng bước nhỏ có thể đạt được. Hãy cố gắng tìm cách làm cho các thay đổi nhỏ và tiếp tục chạy lại các bản kiểm thử để nhận được phản hồi thực tế về những gì bạn đang làm.

Nếu chúng ta bắt đầu làm những việc như thay đổi mã nguồn *xung quanh* các mẫu, thì tất nhiên điều đó có thể đảm bảo việc quay lại phương pháp làm việc TDD của chúng ta.

## Mở rộng markup

Hầu hết các trang web đều có markup HTML phong phú hơn những gì chúng ta đang có lúc này. Chẳng hạn, một phần tử `html`, cùng với một `head`, có lẽ cả một số `nav` nữa. Thường cũng có ý tưởng về một phần footer (chân trang).

Nếu trang web của chúng ta sẽ có nhiều trang khác nhau, chúng ta muốn định nghĩa những thứ này ở một nơi duy nhất để giữ cho trang web của mình trông nhất quán. Go template hỗ trợ chúng ta định nghĩa các phần (sections) mà sau đó chúng ta có thể nhập vào các mẫu khác.

Chỉnh sửa mẫu hiện có của chúng ta để nhập mẫu trên cùng (top) và dưới cùng (bottom)

```handlebars
{{template "top" .}}
<h1>{{.Title}}</h1>

<p>{{.Description}}</p>

Tags: <ul>{{range .Tags}}<li>{{.}}</li>{{end}}</ul>
{{template "bottom" .}}
```

Sau đó tạo `top.gohtml` với nội dung sau:

```handlebars
{{define "top"}}
<!DOCTYPE html>
<html lang="en">
<head>
    <title>My amazing blog!</title>
    <meta charset="UTF-8"/>
    <meta name="description" content="Wow, like and subscribe, it really helps the channel guys" lang="en"/>
</head>
<body>
<nav role="navigation">
    <div>
        <h1>Budding Gopher's blog</h1>
        <ul>
            <li><a href="/">home</a></li>
            <li><a href="about">about</a></li>
            <li><a href="archive">archive</a></li>
        </ul>
    </div>
</nav>
<main>
{{end}}
```

Và `bottom.gohtml`

```handlebars
{{define "bottom"}}
</main>
<footer>
    <ul>
        <li><a href="https://twitter.com/quii">Twitter</a></li>
        <li><a href="https://github.com/quii">GitHub</a></li>
    </ul>
</footer>
</body>
</html>
{{end}}
```

(Đương nhiên, hãy thoải mái đặt bất cứ markup nào bạn thích!)

Bây giờ chúng ta cần chỉ định một mẫu cụ thể để chạy. Trong trình hiển thị blog, hãy thay đổi lệnh `Execute` thành `ExecuteTemplate`

```go
if err := templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
	return err
}
```

Chạy lại bản kiểm thử của bạn. Một file "received" mới nên được tạo ra và bản kiểm thử sẽ thất bại. Hãy kiểm tra lại và nếu bạn hài lòng, hãy phê duyệt nó bằng cách sao chép nó qua phiên bản cũ. Chạy lại bản kiểm thử một lần nữa và nó sẽ vượt qua.

## Lý do để tò mò với Benchmarking

Trước khi tiếp tục, hãy xem xét mã nguồn của chúng ta đang làm gì.

```go
func Render(w io.Writer, p Post) error {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return err
	}

	if err := templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
		return err
	}

	return nil
}
```

- Phân tích (Parse) các mẫu.
- Sử dụng mẫu để hiển thị một bài đăng ra một `io.Writer`.

Mặc dù tác động hiệu năng của việc phân tích lại các mẫu cho mỗi bài đăng trong hầu hết các trường hợp sẽ khá không đáng kể, nhưng nỗ lực để *không* làm điều này cũng rất ít và sẽ giúp mã nguồn gọn gàng hơn một chút.

Để thấy tác động của việc không thực hiện phân tích này lặp đi lặp lại, chúng ta có thể sử dụng công cụ benchmarking để xem hàm của chúng ta nhanh như thế nào.

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		blogrenderer.Render(io.Discard, aPost)
	}
}
```

Trên máy tính của tôi, đây là kết quả:

```
BenchmarkRender-8 22124 53812 ns/op
```

Để ngăn chúng ta phải phân tích lại các mẫu lặp đi lặp lại, chúng ta sẽ tạo một kiểu dữ liệu giữ mẫu đã được phân tích, và kiểu đó sẽ có một phương thức để thực hiện việc hiển thị.

```go
type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {

	if err := r.templ.ExecuteTemplate(w, "blog.gohtml", p); err != nil {
		return err
	}

	return nil
}
```

Điều này có làm thay đổi interface mã nguồn của chúng ta, vì vậy chúng ta cần cập nhật bản kiểm thử:

```go
func TestRender(t *testing.T) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		t.Fatal(err)
	}

	t.Run("it converts a single post into HTML", func(t *testing.T) {
		buf := bytes.Buffer{}

		if err := postRenderer.Render(&buf, aPost); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
}
```

Và bản benchmark của chúng ta:

```go
func BenchmarkRender(b *testing.B) {
	var (
		aPost = blogrenderer.Post{
			Title:       "hello world",
			Body:        "This is a post",
			Description: "This is a description",
			Tags:        []string{"go", "tdd"},
		}
	)

	postRenderer, err := blogrenderer.NewPostRenderer()

	if err != nil {
		b.Fatal(err)
	}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		postRenderer.Render(io.Discard, aPost)
	}
}
```

Bản kiểm thử nên tiếp tục vượt qua. Thế còn bản benchmark của chúng ta thì sao?

`BenchmarkRender-8 362124 3131 ns/op`. Các giá trị cũ là `53812 ns/op`, vì vậy đây là một sự cải thiện đáng kể! Khi chúng ta thêm các phương thức khác để hiển thị, ví dụ như một trang Index, nó sẽ làm đơn giản hóa mã nguồn vì chúng ta không cần lặp lại việc phân tích mẫu.

## Quay lại với công việc thực tế

Về việc hiển thị các bài đăng, phần quan trọng còn lại là thực sự hiển thị trường `Body`. Nếu bạn còn nhớ, đó nên là markdown mà tác giả đã viết, vì vậy nó cần được chuyển đổi sang HTML.

Chúng tôi sẽ để phần này làm bài tập cho bạn, người đọc. Bạn nên có thể tìm thấy một thư viện Go để làm việc này cho bạn. Hãy sử dụng approval test để xác thực những gì bạn đang làm.

### Về việc kiểm thử các thư viện bên thứ ba (3rd-party libraries)

**Ghi chú**. Cẩn thận đừng lo lắng quá nhiều về việc kiểm thử rõ ràng cách một thư viện bên thứ ba hoạt động trong các bản kiểm thử đơn vị (unit tests).

Viết các bản kiểm thử cho mã nguồn mà bạn không kiểm soát là lãng phí và thêm gánh nặng bảo trì. Đôi khi bạn có thể muốn sử dụng [dependency injection](./dependency-injection.md) để kiểm soát một phụ thuộc và giả lập (mock) hành vi của nó cho một bản kiểm thử.

Tuy nhiên, trong trường hợp này, tôi xem việc chuyển đổi markdown thành HTML là chi tiết triển khai của quá trình hiển thị, và các approval tests của chúng ta sẽ mang lại đủ sự tin tưởng.

### Hiển thị trang chủ (Render index)

Phần chức năng tiếp theo mà chúng ta sẽ thực hiện là hiển thị một trang Index, liệt kê các bài đăng dưới dạng một danh sách có thứ tự HTML.

Chúng ta đang mở rộng API của mình, vì vậy hãy đội lại chiếc mũ TDD.

## Viết test trước tiên

Nhìn bề ngoài thì một trang chủ có vẻ đơn giản, nhưng việc viết bản kiểm thử vẫn thúc đẩy chúng ta đưa ra một số quyết định thiết kế.

```go
t.Run("it renders an index of posts", func(t *testing.T) {
	buf := bytes.Buffer{}
	posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

	if err := postRenderer.RenderIndex(&buf, posts); err != nil {
		t.Fatal(err)
	}

	got := buf.String()
	want := `<ol><li><a href="/post/hello-world">Hello World</a></li><li><a href="/post/hello-world-2">Hello World 2</a></li></ol>`

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
})
```

1. Chúng ta đang sử dụng trường tiêu đề (title) của `Post` làm một phần đường dẫn của URL, nhưng chúng ta thực sự không muốn có khoảng trắng trong URL vì vậy chúng ta đang thay thế chúng bằng các dấu gạch nối.
2. Chúng ta đã thêm một phương thức `RenderIndex` vào `PostRenderer` cũng nhận vào một `io.Writer` và một slice các `Post`.

Nếu chúng ta bám vào phương pháp test-after, approval tests ở đây, chúng ta sẽ không trả lời được những câu hỏi này trong một môi trường được kiểm soát. **Các bản kiểm thử cho chúng ta không gian để suy nghĩ**.

## Thử chạy test

```
./renderer_test.go:41:13: undefined: blogrenderer.RenderIndex
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return nil
}
```

Ở trên nên nhận kết quả thất bại bản kiểm thử sau:

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
--- FAIL: TestRender (0.00s)
```

## Viết đủ code để test chạy thành công

Mặc dù điều này *có cảm giác* là nó nên dễ dàng, nhưng nó hơi khó xử một chút. Tôi đã làm nó trong nhiều bước.

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

Tôi chưa muốn bận tâm đến các file mẫu riêng biệt ngay từ đầu, tôi chỉ muốn làm cho nó hoạt động. Tôi xem việc phân tích mẫu ngay từ đầu và tách biệt nó là việc tái cấu trúc mà tôi có thể thực hiện sau này.

Cái này chưa vượt qua, nhưng nó đã khá gần.

```
=== RUN   TestRender
=== RUN   TestRender/it_renders_an_index_of_posts
    renderer_test.go:49: got "<ol><li><a href=\"/post/Hello%20World\">Hello World</a></li><li><a href=\"/post/Hello%20World%202\">Hello World 2</a></li></ol>" want "<ol><li><a href=\"/post/hello-world\">Hello World</a></li><li><a href=\"/post/hello-world-2\">Hello World 2</a></li></ol>"
--- FAIL: TestRender (0.00s)
    --- FAIL: TestRender/it_renders_an_index_of_posts (0.00s)
```

Bạn có thể thấy mã nguồn templating đang thoát (escape) các khoảng trắng trong các thuộc tính `href`. Chúng ta cần một cách để thực hiện thay thế (replace) chuỗi khoảng trắng bằng dấu gạch nối. Chúng ta không thể chỉ lặp qua `[]Post` và thay thế chúng trong bộ nhớ vì chúng ta vẫn muốn các khoảng trắng được hiển thị cho người dùng trong các thẻ link (anchors).

Chúng ta có một vài lựa chọn. Cách đầu tiên chúng ta sẽ khám phá là truyền một hàm vào mẫu của mình.

### Truyền các hàm vào template

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{sanitiseTitle .Title}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Funcs(template.FuncMap{
		"sanitiseTitle": func(title string) string {
			return strings.ToLower(strings.Replace(title, " ", "-", -1))
		},
	}).Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

*Trước khi bạn phân tích một mẫu*, bạn có thể thêm một `template.FuncMap` vào mẫu của mình, cho phép bạn định nghĩa các hàm có thể được gọi bên trong mẫu của mình. Trong trường hợp này, chúng ta đã tạo một hàm `sanitiseTitle`, sau đó chúng ta gọi nó bên trong mẫu của mình bằng `{{sanitiseTitle .Title}}`.

Đây là một tính năng mạnh mẽ, việc có thể gửi các hàm vào mẫu sẽ cho phép bạn làm một số điều rất thú vị, nhưng, bạn có nên làm thế không? Quay lại các nguyên tắc của Mustache và các logic-less template, tại sao họ lại ủng hộ logic-less? **Có vấn đề gì với logic trong mẫu?**

Như chúng ta đã chỉ ra, để kiểm thử các mẫu của mình, *chúng ta đã phải giới thiệu một loại kiểm thử hoàn toàn khác*.

Hãy tưởng tượng bạn đưa một hàm vào một mẫu có một vài hoán vị hành vi và các trường hợp biên, **bạn sẽ kiểm thử nó như thế nào**? Với thiết kế hiện tại này, phương tiện kiểm thử logic duy nhất của bạn là bằng cách *hiển thị HTML và so sánh chuỗi*. Đây không phải là một cách dễ dàng hay lành mạnh để kiểm thử logic, và chắc chắn không phải là những gì bạn muốn cho logic nghiệp vụ *quan trọng*.

Mặc dù kỹ thuật approval tests đã làm giảm chi phí duy trì các bản kiểm thử này, chúng vẫn đắt đỏ để duy trì hơn hầu hết các bản kiểm thử đơn vị mà bạn sẽ viết. Chúng vẫn nhạy cảm với bất kỳ thay đổi markup nhỏ nào bạn thực hiện, chỉ là chúng ta đã làm cho việc quản lý dễ dàng hơn. Chúng ta vẫn nên cố gắng kiến trúc mã nguồn của mình sao cho không phải viết nhiều bản kiểm thử xung quanh các mẫu, và cố gắng tách biệt các mối quan tâm để bất kỳ logic nào không cần thiết phải nằm trong mã nguồn hiển thị đều được tách biệt một cách hợp lý.

Những gì các engine templating chịu ảnh hưởng từ Mustache mang lại cho bạn là một sự ràng buộc hữu ích, đừng cố gắng lách qua nó quá thường xuyên; **đừng đi ngược lại xu thế**. Thay vào đó, hãy đón nhận ý tưởng về [view models](https://stackoverflow.com/a/11074506/3193), nơi bạn xây dựng các kiểu dữ liệu cụ thể chứa dữ liệu bạn cần hiển thị, theo cách thuận tiện cho ngôn ngữ templating.

Bằng cách này, bất kỳ logic nghiệp vụ quan trọng nào bạn sử dụng để tạo túi dữ liệu đó đều có thể được kiểm thử đơn vị một cách riêng biệt, tránh xa thế giới lộn xộn của HTML và templating.

### Tách biệt các mối quan tâm (Separating concerns)

Vậy chúng ta có thể làm gì thay thế?

#### Thêm một phương thức vào `Post` và sau đó gọi nó trong mẫu

Chúng ta có thể gọi các phương thức trong mã nguồn templating trên các kiểu dữ liệu chúng ta gửi đi, vì vậy chúng ta có thể thêm một phương thức `SanitisedTitle` vào `Post`. Điều này sẽ đơn giản hóa mẫu và chúng ta có thể dễ dàng kiểm thử đơn vị logic này một cách riêng biệt nếu muốn. Đây có lẽ là giải pháp dễ dàng nhất, mặc dù không hẳn là đơn giản nhất.

Một mặt trái của cách tiếp cận này là đây vẫn là logic của *view*. Nó không thú vị đối với phần còn lại của hệ thống nhưng giờ đây nó trở thành một phần API cho một core domain object. Loại cách tiếp cận này theo thời gian có thể dẫn đến việc bạn tạo ra các [God Objects](https://en.wikipedia.org/wiki/God_object).

#### Tạo một view model chuyên dụng, chẳng hạn như `PostViewModel` với chính xác dữ liệu chúng ta cần

Thay vì mã nguồn hiển thị của chúng ta bị ràng buộc vào domain object, `Post`, thay vào đó nó nhận một view model.

```go
type PostViewModel struct {
	Title, SanitisedTitle, Description, Body string
	Tags                                     []string
}
```

Người gọi mã nguồn của chúng ta sẽ phải ánh xạ (map) từ `[]Post` sang `[]PostView`, tạo ra `SanitisedTitle`. Một cách để giữ cho việc này sạch sẽ là có một `func NewPostView(p Post) PostView` để đóng gói việc ánh xạ.

Điều này sẽ giữ cho mã nguồn hiển thị của chúng ta không có logic (logic-less) và có lẽ là sự tách biệt các mối quan tâm nghiêm ngặt nhất mà chúng ta có thể thực hiện, nhưng sự đánh đổi là một quy trình rắc rối hơn một chút để hiển thị bài đăng của mình.

Cả hai tùy chọn đều ổn, trong trường hợp này tôi đang bị cám dỗ chọn phương án đầu tiên. Khi bạn phát triển hệ thống, bạn nên thận trọng khi thêm ngày càng nhiều phương thức ad-hoc chỉ để làm mượt quá trình hiển thị; các view model chuyên dụng trở nên hữu ích hơn khi việc chuyển đổi giữa domain object và view trở nên phức tạp hơn.

Vì vậy chúng ta có thể thêm phương thức của mình vào `Post`:

```go
func (p Post) SanitisedTitle() string {
	return strings.ToLower(strings.Replace(p.Title, " ", "-", -1))
}
```

Và sau đó chúng ta có thể quay lại một thế giới đơn giản hơn trong mã nguồn hiển thị của mình:

```go
func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	indexTemplate := `<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>`

	templ, err := template.New("index").Parse(indexTemplate)
	if err != nil {
		return err
	}

	if err := templ.Execute(w, posts); err != nil {
		return err
	}

	return nil
}
```

## Refactor

Cuối cùng thì bản kiểm thử cũng nên vượt qua. Bây giờ chúng ta có thể di chuyển mẫu của mình vào một file (`templates/index.gohtml`) và tải nó một lần duy nhất khi chúng ta xây dựng renderer của mình.

```go
package blogrenderer

import (
	"embed"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ *template.Template
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	return &PostRenderer{templ: templ}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", p)
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}
```

Bằng cách phân tích nhiều hơn một mẫu vào `templ`, bây giờ chúng ta phải gọi `ExecuteTemplate` và chỉ định mẫu *nào* chúng ta muốn hiển thị cho phù hợp, nhưng hy vọng bạn sẽ đồng ý rằng mã nguồn mà chúng ta đạt được trông thật tuyệt vời.

Có một rủi ro *nhẹ* nếu ai đó đổi tên một trong các file mẫu, nó sẽ gây ra lỗi, nhưng các bản kiểm thử đơn vị chạy nhanh của chúng ta sẽ phát hiện ra điều này một cách nhanh chóng.

Bây giờ chúng ta đã hài lòng với thiết kế API cho package của mình và có một số hành vi cơ bản được dẫn dắt bởi TDD, hãy thay đổi bản kiểm thử của chúng ta để sử dụng approvals.

```go
	t.Run("it renders an index of posts", func(t *testing.T) {
		buf := bytes.Buffer{}
		posts := []blogrenderer.Post{{Title: "Hello World"}, {Title: "Hello World 2"}}

		if err := postRenderer.RenderIndex(&buf, posts); err != nil {
			t.Fatal(err)
		}

		approvals.VerifyString(t, buf.String())
	})
```

Hãy nhớ chạy bản kiểm thử để thấy nó thất bại, và sau đó phê duyệt thay đổi.

Cuối cùng, chúng ta có thể thêm khung trang (page furniture) vào trang chủ (index page):

```handlebars
{{template "top" .}}
<ol>{{range .}}<li><a href="/post/{{.SanitisedTitle}}">{{.Title}}</a></li>{{end}}</ol>
{{template "bottom" .}}
```

Chạy lại bản kiểm thử, phê duyệt thay đổi và chúng ta đã hoàn tất phần trang chủ!

## Hiển thị thân bài markdown

Tôi đã khuyến khích bạn tự mình thử sức, đây là cách tiếp cận mà tôi đã kết thúc việc triển khai.

```go
package blogrenderer

import (
	"embed"
	"github.com/gomarkdown/markdown"
	"github.com/gomarkdown/markdown/parser"
	"html/template"
	"io"
)

var (
	//go:embed "templates/*"
	postTemplates embed.FS
)

type PostRenderer struct {
	templ    *template.Template
	mdParser *parser.Parser
}

func NewPostRenderer() (*PostRenderer, error) {
	templ, err := template.ParseFS(postTemplates, "templates/*.gohtml")
	if err != nil {
		return nil, err
	}

	extensions := parser.CommonExtensions | parser.AutoHeadingIDs
	parser := parser.NewWithExtensions(extensions)

	return &PostRenderer{templ: templ, mdParser: parser}, nil
}

func (r *PostRenderer) Render(w io.Writer, p Post) error {
	return r.templ.ExecuteTemplate(w, "blog.gohtml", newPostVM(p, r))
}

func (r *PostRenderer) RenderIndex(w io.Writer, posts []Post) error {
	return r.templ.ExecuteTemplate(w, "index.gohtml", posts)
}

type postViewModel struct {
	Post
	HTMLBody template.HTML
}

func newPostVM(p Post, r *PostRenderer) postViewModel {
	vm := postViewModel{Post: p}
	vm.HTMLBody = template.HTML(markdown.ToHTML([]byte(p.Body), r.mdParser, nil))
	return vm
}
```

Tôi đã sử dụng thư viện [gomarkdown](https://github.com/gomarkdown/markdown) xuất sắc hoạt động chính xác theo những gì tôi hy vọng.

Nếu bạn tự mình thử làm điều này, bạn có thể thấy rằng phần hiển thị thân bài của mình đã bị thoát (escaped) HTML. Đây là một tính năng bảo mật của package html/template của Go để ngăn chặn HTML bên thứ ba độc hại bị xuất ra.

Để giải quyết vấn đề này, trong kiểu dữ liệu bạn gửi để hiển thị, bạn cần bọc đoạn HTML tin cậy của mình trong [template.HTML](https://pkg.go.dev/html/template#HTML).

> HTML đóng gói một đoạn tài liệu HTML an toàn đã biết. Nó không nên được sử dụng cho HTML từ bên thứ ba, hoặc HTML với các thẻ hoặc nhận xét chưa đóng. Đầu ra của một sanitizer HTML lành mạnh và một mẫu được thoát bởi package này sẽ tốt để sử dụng với HTML.
>
> Việc sử dụng kiểu dữ liệu này có rủi ro bảo mật: nội dung đóng gói nên đến từ một nguồn đáng tin cậy, vì nó sẽ được bao gồm nguyên văn trong đầu ra của mẫu.

Vì vậy, tôi đã tạo một view model **không được xuất bản** (unexported) (`postViewModel`), vì tôi vẫn coi đây là chi tiết triển khai nội bộ của việc hiển thị. Tôi không cần phải kiểm thử việc này riêng biệt và tôi không muốn nó làm vấy bẩn API của mình.

Tôi xây dựng một cái khi hiển thị để có thể phân tích `Body` thành `HTMLBody` và sau đó tôi sử dụng trường đó trong mẫu để hiển thị HTML.

## Tổng kết

Nếu bạn kết hợp các bài học về chương [đọc file](reading-files.md) và chương này, bạn có thể thoải mái tạo ra một trình tạo trang web tĩnh đơn giản, được kiểm thử tốt và tự mình khởi chạy một blog của riêng mình. Tìm thêm một số hướng dẫn về CSS và bạn có thể làm cho nó trông đẹp mắt nữa.

Cách tiếp cận này mở rộng ra ngoài các blog. Lấy dữ liệu từ bất kỳ nguồn nào, có thể là cơ sở dữ liệu, một API hoặc một hệ thống file và chuyển đổi nó thành HTML và trả về từ một server là một kỹ thuật đơn giản kéo dài hàng thập kỷ. Mọi người thích than phiền về sự phức tạp của phát triển web hiện đại nhưng bạn có chắc mình không chỉ đang tự gây ra sự phức tạp cho chính mình không?

Go là tuyệt vời cho phát triển web, đặc biệt khi bạn suy nghĩ rõ ràng về yêu cầu thực sự của trang web mà bạn đang xây dựng. Việc tạo HTML phía server thường là một cách tiếp cận tốt hơn, đơn giản hơn và hiệu năng cao hơn so với việc tạo một "ứng dụng web" với các công nghệ như React.

### Những gì chúng ta đã học

- Cách tạo và hiển thị các mẫu HTML (HTML templates).
- Cách kết hợp các mẫu với nhau và áp dụng nguyên tắc [DRY](https://en.wikipedia.org/wiki/Don't_repeat_yourself) cho các markup liên quan, giúp chúng ta duy trì giao diện nhất quán.
- Cách truyền các hàm vào mẫu, và tại sao bạn nên suy nghĩ kỹ trước khi làm điều đó.
- Cách viết "Approval Tests", giúp chúng ta kiểm thử các đầu ra lớn và lộn xộn của những thứ như trình hiển thị mẫu (template renderers).

### Về các mẫu không logic (logic-less templates)

Như mọi khi, tất cả đều xoay quanh **tách biệt các mối quan tâm (separation of concerns)**. Điều quan trọng là chúng ta cân nhắc trách nhiệm của các phần khác nhau trong hệ thống. Quá thường xuyên, mọi người để rò rỉ logic nghiệp vụ quan trọng vào các mẫu, trộn lẫn các mối quan tâm và khiến hệ thống trở nên khó hiểu, khó bảo trì và khó kiểm thử.

### Không chỉ dành cho HTML

Hãy nhớ rằng Go có `text/template` để tạo ra các loại dữ liệu khác từ một mẫu. Nếu bạn thấy mình cần chuyển đổi dữ liệu thành một loại đầu ra có cấu trúc nào đó, các kỹ thuật được trình bày trong chương này có thể hữu ích.

### Tài liệu tham khảo và nội dung bổ sung

- [Khóa học 'Learn Web Development with Go' của John Calhoun](https://www.calhoun.io/intro-to-templates-p1-contextual-encoding/) có một số bài viết xuất sắc về templating.
- [Hotwire](https://hotwired.dev) - Bạn có thể sử dụng các kỹ thuật này để tạo các ứng dụng web Hotwire. Nó được xây dựng bởi Basecamp, chủ yếu là một công ty sử dụng Ruby on Rails, nhưng vì nó hoạt động phía server, chúng ta có thể sử dụng nó với Go.
