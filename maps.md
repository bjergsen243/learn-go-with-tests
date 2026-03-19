# Maps

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/maps)**

Trong [arrays & slices](arrays-and-slices.md), bạn thấy cách lưu trữ các giá trị theo thứ tự. Giờ chúng ta sẽ xem xét cách lưu trữ các mục theo `key` và tra cứu nhanh.

Maps cho phép bạn lưu trữ các mục theo cách tương tự như từ điển. Bạn có thể nghĩ `key` là từ và `value` là định nghĩa. Và cách nào tốt hơn để học về Maps hơn là xây dựng từ điển của riêng chúng ta?

Đầu tiên, giả sử chúng ta đã có một số từ với định nghĩa trong từ điển, nếu tìm kiếm một từ, nó sẽ trả về định nghĩa của từ đó.

## Viết test trước tiên

Trong `dictionary_test.go`

```go
package main

import "testing"

func TestSearch(t *testing.T) {
	dictionary := map[string]string{"test": "this is just a test"}

	got := Search(dictionary, "test")
	want := "this is just a test"

	if got != want {
		t.Errorf("got %q want %q given, %q", got, want, "test")
	}
}
```

Khai báo Map khá giống với array. Ngoại trừ, nó bắt đầu bằng từ khóa `map` và yêu cầu hai kiểu. Kiểu đầu tiên là kiểu key, được viết trong `[]`. Kiểu thứ hai là kiểu value, ngay sau `[]`.

Kiểu key là đặc biệt. Nó chỉ có thể là kiểu comparable vì nếu không có khả năng so sánh 2 keys bằng nhau, chúng ta không có cách nào đảm bảo chúng ta đang lấy đúng value. Các kiểu comparable được giải thích kỹ trong [language spec](https://golang.org/ref/spec#Comparison_operators).

Kiểu value, mặt khác, có thể là bất kỳ kiểu nào. Thậm chí có thể là một map khác.

Mọi thứ khác trong test này đã quen thuộc.

## Thử chạy test

Bằng cách chạy `go test`, compiler sẽ thất bại với `./dictionary_test.go:8:9: undefined: Search`.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Trong `dictionary.go`

```go
package main

func Search(dictionary map[string]string, word string) string {
	return ""
}
```

Test của bạn giờ sẽ thất bại với *thông báo lỗi rõ ràng*

`dictionary_test.go:12: got '' want 'this is just a test' given, 'test'`.

## Viết đủ code để test chạy thành công

```go
func Search(dictionary map[string]string, word string) string {
	return dictionary[word]
}
```

Lấy giá trị từ Map giống như lấy từ Array `map[key]`.

## Refactor

```go
func TestSearch(t *testing.T) {
	dictionary := map[string]string{"test": "this is just a test"}

	got := Search(dictionary, "test")
	want := "this is just a test"

	assertStrings(t, got, want)
}

func assertStrings(t testing.TB, got, want string) {
	t.Helper()

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}
```

Tôi quyết định tạo helper `assertStrings` để khái quát hóa implementation hơn.

### Dùng kiểu tùy chỉnh

Chúng ta có thể cải thiện cách dùng từ điển bằng cách tạo kiểu mới xung quanh map và biến `Search` thành method.

Trong `dictionary_test.go`:

```go
func TestSearch(t *testing.T) {
	dictionary := Dictionary{"test": "this is just a test"}

	got := dictionary.Search("test")
	want := "this is just a test"

	assertStrings(t, got, want)
}
```

Chúng ta bắt đầu dùng kiểu `Dictionary`, mà chúng ta chưa định nghĩa. Sau đó gọi `Search` trên instance `Dictionary`.

Chúng ta không cần thay đổi `assertStrings`.

Trong `dictionary.go`:

```go
type Dictionary map[string]string

func (d Dictionary) Search(word string) string {
	return d[word]
}
```

Ở đây chúng ta tạo kiểu `Dictionary` hoạt động như wrapper mỏng quanh `map`. Với kiểu tùy chỉnh được định nghĩa, chúng ta có thể tạo method `Search`.

## Viết test trước tiên

Tìm kiếm cơ bản rất dễ implement, nhưng điều gì sẽ xảy ra nếu chúng ta cung cấp từ không có trong từ điển?

Thực ra chúng ta không nhận được gì. Điều này tốt vì chương trình có thể tiếp tục chạy, nhưng có cách tiếp cận tốt hơn. Hàm có thể báo cáo từ không có trong từ điển. Như vậy người dùng không phải tự hỏi từ đó không tồn tại hay chỉ là không có định nghĩa (điều này không có vẻ hữu ích cho từ điển, nhưng là tình huống có thể quan trọng trong các use case khác).

```go
func TestSearch(t *testing.T) {
	dictionary := Dictionary{"test": "this is just a test"}

	t.Run("known word", func(t *testing.T) {
		got, _ := dictionary.Search("test")
		want := "this is just a test"

		assertStrings(t, got, want)
	})

	t.Run("unknown word", func(t *testing.T) {
		_, err := dictionary.Search("unknown")
		want := "could not find the word you were looking for"

		if err == nil {
			t.Fatal("expected to get an error.")
		}

		assertStrings(t, err.Error(), want)
	})
}
```

Cách xử lý tình huống này trong Go là trả về đối số thứ hai kiểu `Error`.

Chú ý rằng như đã thấy trong [phần pointers và error](./pointers-and-errors.md), để assert thông báo lỗi, trước tiên chúng ta kiểm tra lỗi không phải `nil` rồi dùng method `.Error()` để lấy string mà chúng ta có thể truyền vào assertion.

## Thử chạy test

Code này không biên dịch được

```
./dictionary_test.go:18:10: assignment mismatch: 2 variables but 1 values
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func (d Dictionary) Search(word string) (string, error) {
	return d[word], nil
}
```

Test của bạn sẽ thất bại với thông báo lỗi rõ ràng hơn.

`dictionary_test.go:22: expected to get an error.`

## Viết đủ code để test chạy thành công

```go
func (d Dictionary) Search(word string) (string, error) {
	definition, ok := d[word]
	if !ok {
		return "", errors.New("could not find the word you were looking for")
	}

	return definition, nil
}
```

Để test pass, chúng ta dùng thuộc tính thú vị của map lookup. Nó có thể trả về 2 giá trị. Giá trị thứ hai là boolean báo hiệu key có được tìm thấy không.

Thuộc tính này cho phép chúng ta phân biệt giữa từ không tồn tại và từ không có định nghĩa.

## Refactor

```go
var ErrNotFound = errors.New("could not find the word you were looking for")

func (d Dictionary) Search(word string) (string, error) {
	definition, ok := d[word]
	if !ok {
		return "", ErrNotFound
	}

	return definition, nil
}
```

Chúng ta có thể loại bỏ magic error trong hàm `Search` bằng cách trích xuất nó thành biến. Điều này cũng cho phép test tốt hơn.

```go
t.Run("unknown word", func(t *testing.T) {
	_, got := dictionary.Search("unknown")
	if got == nil {
		t.Fatal("expected to get an error.")
	}
	assertError(t, got, ErrNotFound)
})
```
```go
func assertError(t testing.TB, got, want error) {
	t.Helper()

	if got != want {
		t.Errorf("got error %q want %q", got, want)
	}
}
```

Bằng cách tạo helper mới, chúng ta đơn giản hóa test và bắt đầu dùng biến `ErrNotFound` để test không fail nếu chúng ta thay đổi văn bản lỗi sau này.

## Viết test trước tiên

Chúng ta có cách tốt để tìm kiếm trong từ điển. Tuy nhiên, chúng ta không có cách nào thêm từ mới.

```go
func TestAdd(t *testing.T) {
	dictionary := Dictionary{}
	dictionary.Add("test", "this is just a test")

	want := "this is just a test"
	got, err := dictionary.Search("test")
	if err != nil {
		t.Fatal("should find added word:", err)
	}

	assertStrings(t, got, want)
}
```

Trong test này, chúng ta dùng hàm `Search` để validate từ điển dễ hơn.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Trong `dictionary.go`

```go
func (d Dictionary) Add(word, definition string) {
}
```

Test của bạn sẽ thất bại

```
dictionary_test.go:31: should find added word: could not find the word you were looking for
```

## Viết đủ code để test chạy thành công

```go
func (d Dictionary) Add(word, definition string) {
	d[word] = definition
}
```

Thêm vào map cũng giống như array. Chỉ cần chỉ định key và đặt nó bằng value.

### Pointers, copies, v.v.

Một thuộc tính thú vị của maps là bạn có thể sửa đổi chúng mà không cần truyền địa chỉ (ví dụ `&myMap`)

Điều này có thể khiến chúng _cảm thấy_ như "kiểu tham chiếu", [nhưng như Dave Cheney mô tả](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it), chúng không phải vậy.

> Giá trị map là pointer đến cấu trúc runtime.hmap.

Vì vậy khi bạn truyền map vào hàm/method, bạn đang sao chép nó, nhưng chỉ phần pointer, không phải cấu trúc dữ liệu bên dưới chứa data.

Điều đáng chú ý với maps là chúng có thể là giá trị `nil`. Map `nil` hoạt động như map rỗng khi đọc, nhưng cố ghi vào map `nil` sẽ gây ra runtime panic. Bạn có thể đọc thêm về maps [tại đây](https://blog.golang.org/go-maps-in-action).

Do đó, bạn không bao giờ nên khởi tạo biến map nil:

```go
var m map[string]string
```

Thay vào đó, có thể khởi tạo map rỗng hoặc dùng từ khóa `make` để tạo map:

```go
var dictionary = map[string]string{}

// HOẶC

var dictionary = make(map[string]string)
```

Cả hai cách đều tạo `hash map` rỗng và trỏ `dictionary` vào nó, đảm bảo bạn không bao giờ gặp runtime panic.

## Refactor

Không có nhiều thứ cần refactor trong implementation nhưng test có thể đơn giản hóa một chút.

```go
func TestAdd(t *testing.T) {
	dictionary := Dictionary{}
	word := "test"
	definition := "this is just a test"

	dictionary.Add(word, definition)

	assertDefinition(t, dictionary, word, definition)
}

func assertDefinition(t testing.TB, dictionary Dictionary, word, definition string) {
	t.Helper()

	got, err := dictionary.Search(word)
	if err != nil {
		t.Fatal("should find added word:", err)
	}
	assertStrings(t, got, definition)
}
```

Chúng ta tạo biến cho word và definition, và chuyển assertion definition sang hàm helper riêng.

`Add` của chúng ta trông tốt. Ngoại trừ, chúng ta không xem xét điều gì xảy ra khi value chúng ta cố thêm đã tồn tại!

Map không throw lỗi nếu value đã tồn tại. Thay vào đó, chúng sẽ ghi đè value bằng value mới được cung cấp. Điều này có thể tiện lợi trong thực tế, nhưng khiến tên hàm kém chính xác. `Add` không nên sửa đổi các giá trị đã có. Nó chỉ nên thêm từ mới vào từ điển.

## Viết test trước tiên

```go
func TestAdd(t *testing.T) {
	t.Run("new word", func(t *testing.T) {
		dictionary := Dictionary{}
		word := "test"
		definition := "this is just a test"

		err := dictionary.Add(word, definition)

		assertError(t, err, nil)
		assertDefinition(t, dictionary, word, definition)
	})

	t.Run("existing word", func(t *testing.T) {
		word := "test"
		definition := "this is just a test"
		dictionary := Dictionary{word: definition}
		err := dictionary.Add(word, "new test")

		assertError(t, err, ErrWordExists)
		assertDefinition(t, dictionary, word, definition)
	})
}
```

Trong test này, chúng ta sửa đổi `Add` để trả về lỗi, mà chúng ta validate dựa trên biến lỗi mới, `ErrWordExists`. Chúng ta cũng sửa test trước để kiểm tra lỗi `nil`.

## Thử chạy test

Compiler sẽ thất bại vì chúng ta không trả về value cho `Add`.

```
./dictionary_test.go:30:13: dictionary.Add(word, definition) used as value
./dictionary_test.go:41:13: dictionary.Add(word, "new test") used as value
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Trong `dictionary.go`

```go
var (
	ErrNotFound   = errors.New("could not find the word you were looking for")
	ErrWordExists = errors.New("cannot add word because it already exists")
)

func (d Dictionary) Add(word, definition string) error {
	d[word] = definition
	return nil
}
```

Bây giờ chúng ta nhận được thêm hai lỗi. Chúng ta vẫn đang sửa đổi value và trả về lỗi `nil`.

```
dictionary_test.go:43: got error '%!q(<nil>)' want 'cannot add word because it already exists'
dictionary_test.go:44: got 'new test' want 'this is just a test'
```

## Viết đủ code để test chạy thành công

```go
func (d Dictionary) Add(word, definition string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		d[word] = definition
	case nil:
		return ErrWordExists
	default:
		return err
	}

	return nil
}
```

Ở đây chúng ta dùng câu lệnh `switch` để match trên lỗi. Có `switch` như này cung cấp mạng lưới an toàn thêm, phòng trường hợp `Search` trả về lỗi khác ngoài `ErrNotFound`.

## Refactor

Chúng ta không có nhiều thứ cần refactor, nhưng khi error usage phát triển, chúng ta có thể thực hiện một vài sửa đổi.

```go
const (
	ErrNotFound   = DictionaryErr("could not find the word you were looking for")
	ErrWordExists = DictionaryErr("cannot add word because it already exists")
)

type DictionaryErr string

func (e DictionaryErr) Error() string {
	return string(e)
}
```

Chúng ta tạo errors là constants; điều này yêu cầu tạo kiểu `DictionaryErr` riêng implement interface `error`. Bạn có thể đọc thêm trong [bài viết xuất sắc của Dave Cheney](https://dave.cheney.net/2016/04/07/constant-errors). Nói ngắn gọn, nó làm errors có thể tái sử dụng và bất biến hơn.

Tiếp theo, hãy tạo hàm `Update` để sửa đổi định nghĩa của một từ.

## Viết test trước tiên

```go
func TestUpdate(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{word: definition}
	newDefinition := "new definition"

	dictionary.Update(word, newDefinition)

	assertDefinition(t, dictionary, word, newDefinition)
}
```

`Update` liên quan chặt chẽ đến `Add` và sẽ là implementation tiếp theo của chúng ta.

## Thử chạy test

```
./dictionary_test.go:53:2: dictionary.Update undefined (type Dictionary has no field or method Update)
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Chúng ta đã biết cách xử lý lỗi kiểu này. Cần định nghĩa hàm.

```go
func (d Dictionary) Update(word, definition string) {}
```

Với điều đó, chúng ta có thể thấy cần thay đổi định nghĩa của từ.

```
dictionary_test.go:55: got 'this is just a test' want 'new definition'
```

## Viết đủ code để test chạy thành công

Chúng ta đã thấy cách làm điều này khi sửa vấn đề với `Add`. Hãy implement tương tự như `Add`.

```go
func (d Dictionary) Update(word, definition string) {
	d[word] = definition
}
```

Không cần refactor vì đây là thay đổi đơn giản. Tuy nhiên, bây giờ chúng ta có vấn đề tương tự như `Add`. Nếu truyền vào từ mới, `Update` sẽ thêm nó vào từ điển.

## Viết test trước tiên

```go
t.Run("existing word", func(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{word: definition}
	newDefinition := "new definition"

	err := dictionary.Update(word, newDefinition)

	assertError(t, err, nil)
	assertDefinition(t, dictionary, word, newDefinition)
})

t.Run("new word", func(t *testing.T) {
	word := "test"
	definition := "this is just a test"
	dictionary := Dictionary{}

	err := dictionary.Update(word, definition)

	assertError(t, err, ErrWordDoesNotExist)
})
```

Chúng ta thêm loại lỗi mới khi từ không tồn tại. Chúng ta cũng sửa đổi `Update` để trả về giá trị `error`.

## Thử chạy test

```
./dictionary_test.go:53:16: dictionary.Update(word, newDefinition) used as value
./dictionary_test.go:64:16: dictionary.Update(word, definition) used as value
./dictionary_test.go:66:23: undefined: ErrWordDoesNotExist
```

Chúng ta nhận 3 lỗi lần này, nhưng chúng ta biết cách xử lý chúng.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
const (
	ErrNotFound         = DictionaryErr("could not find the word you were looking for")
	ErrWordExists       = DictionaryErr("cannot add word because it already exists")
	ErrWordDoesNotExist = DictionaryErr("cannot perform operation on word because it does not exist")
)

func (d Dictionary) Update(word, definition string) error {
	d[word] = definition
	return nil
}
```

Chúng ta thêm kiểu lỗi riêng và trả về lỗi `nil`.

Với những thay đổi này, chúng ta nhận lỗi rõ ràng:

```
dictionary_test.go:66: got error '%!q(<nil>)' want 'cannot update word because it does not exist'
```

## Viết đủ code để test chạy thành công

```go
func (d Dictionary) Update(word, definition string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		return ErrWordDoesNotExist
	case nil:
		d[word] = definition
	default:
		return err
	}

	return nil
}
```

Hàm này trông gần giống `Add` ngoại trừ chúng ta đổi khi nào cập nhật `dictionary` và khi nào trả về lỗi.

### Lưu ý về khai báo lỗi mới cho Update

Chúng ta có thể tái sử dụng `ErrNotFound` và không thêm lỗi mới. Tuy nhiên, thường tốt hơn là có lỗi chính xác khi update fail.

Có errors cụ thể cung cấp thêm thông tin về những gì đã xảy ra. Đây là ví dụ trong web app:

> Bạn có thể redirect người dùng khi gặp `ErrNotFound`, nhưng hiển thị thông báo lỗi khi gặp `ErrWordDoesNotExist`.

Tiếp theo, hãy tạo hàm `Delete` từ trong từ điển.

## Viết test trước tiên

```go
func TestDelete(t *testing.T) {
	word := "test"
	dictionary := Dictionary{word: "test definition"}

	dictionary.Delete(word)

	_, err := dictionary.Search(word)
	assertError(t, err, ErrNotFound)
}
```

Test của chúng ta tạo `Dictionary` với một từ rồi kiểm tra từ đó đã bị xóa.

## Thử chạy test

Bằng cách chạy `go test`:

```
./dictionary_test.go:74:6: dictionary.Delete undefined (type Dictionary has no field or method Delete)
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func (d Dictionary) Delete(word string) {

}
```

Sau khi thêm điều này, test cho biết chúng ta không xóa từ.

```
dictionary_test.go:78: got error '%!q(<nil>)' want 'could not find the word you were looking for'
```

## Viết đủ code để test chạy thành công

```go
func (d Dictionary) Delete(word string) {
	delete(d, word)
}
```

Go có hàm built-in `delete` hoạt động trên maps. Nó nhận hai đối số và không trả về gì. Đối số đầu tiên là map và đối số thứ hai là key cần xóa.

## Refactor

Không có nhiều thứ cần refactor, nhưng chúng ta có thể implement logic tương tự từ `Update` để xử lý trường hợp từ không tồn tại.

```go
func TestDelete(t *testing.T) {
	t.Run("existing word", func(t *testing.T) {
		word := "test"
		dictionary := Dictionary{word: "test definition"}

		err := dictionary.Delete(word)

		assertError(t, err, nil)

		_, err = dictionary.Search(word)

		assertError(t, err, ErrNotFound)
	})

	t.Run("non-existing word", func(t *testing.T) {
		word := "test"
		dictionary := Dictionary{}

		err := dictionary.Delete(word)

		assertError(t, err, ErrWordDoesNotExist)
	})
}
```

## Thử chạy test

Compiler sẽ thất bại vì chúng ta không trả về value cho `Delete`.

```
./dictionary_test.go:77:10: dictionary.Delete(word) (no value) used as value
./dictionary_test.go:90:10: dictionary.Delete(word) (no value) used as value
```

## Viết đủ code để test chạy thành công

```go
func (d Dictionary) Delete(word string) error {
	_, err := d.Search(word)

	switch err {
	case ErrNotFound:
		return ErrWordDoesNotExist
	case nil:
		delete(d, word)
	default:
		return err
	}

	return nil
}
```

Chúng ta dùng switch statement để match trên lỗi khi cố xóa từ không tồn tại.

## Tổng kết

Trong phần này, chúng ta đã học nhiều thứ. Chúng ta đã tạo API CRUD (Create, Read, Update và Delete) đầy đủ cho từ điển. Trong suốt quá trình, chúng ta học được cách:

* Tạo maps
* Tìm kiếm các mục trong maps
* Thêm mục mới vào maps
* Cập nhật mục trong maps
* Xóa mục khỏi map
* Tìm hiểu thêm về errors
  * Cách tạo errors là constants
  * Viết error wrappers
