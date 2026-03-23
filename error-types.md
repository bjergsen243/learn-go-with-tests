# Các kiểu lỗi

**[Bạn có thể tìm thấy toàn bộ mã nguồn tại đây](https://github.com/quii/learn-go-with-tests/tree/main/q-and-a/error-types)**

**Việc tạo các kiểu dữ liệu riêng cho lỗi có thể là một cách trang nhã để dọn dẹp mã nguồn, giúp mã của bạn dễ sử dụng và dễ kiểm thử hơn.**

Pedro trên Gopher Slack có hỏi:

> Nếu tôi tạo một lỗi như `fmt.Errorf("%s must be foo, got %s", bar, baz)`, có cách nào để kiểm tra tính bằng nhau mà không cần so sánh giá trị chuỗi không?

Hãy tạo một hàm giả định để giúp khám phá ý tưởng này.

```go
// DumbGetter sẽ lấy nội dung thân bài (body) dạng chuỗi của url nếu nhận được mã 200
func DumbGetter(url string) (string, error) {
	res, err := http.Get(url)

	if err != nil {
		return "", fmt.Errorf("problem fetching from %s, %v", url, err)
	}

	if res.StatusCode != http.StatusOK {
		return "", fmt.Errorf("did not get 200 from %s, got %d", url, res.StatusCode)
	}

	defer res.Body.Close()
	body, _ := io.ReadAll(res.Body) // bỏ qua err cho ngắn gọn

	return string(body), nil
}
```

Việc viết một hàm có thể thất bại vì nhiều lý do khác nhau là điều không hiếm gặp, và chúng ta muốn đảm bảo mình xử lý đúng từng kịch bản.

Như Pedro đã nói, chúng ta _có thể_ viết một bài kiểm thử cho lỗi trạng thái (status error) như sau.

```go
t.Run("khi bạn không nhận được mã 200, bạn sẽ nhận được một lỗi trạng thái", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("mong đợi một lỗi")
	}

	want := fmt.Sprintf("did not get 200 from %s, got %d", svr.URL, http.StatusTeapot)
	got := err.Error()

	if got != want {
		t.Errorf(`got "%v", want "%v"`, got, want)
	}
})
```

Bài kiểm thử này tạo ra một máy chủ luôn trả về `StatusTeapot`, sau đó chúng ta sử dụng URL của nó làm đối số cho `DumbGetter` để xem nó có xử lý các phản hồi không phải `200` một cách chính xác hay không.

## Những vấn đề của cách kiểm thử này

Cuốn sách này luôn nhấn mạnh việc _lắng nghe test_ và bài test này cho cảm giác không ổn:

- Chúng ta đang xây dựng cùng một chuỗi ký tự như mã nguồn thực tế (production code) để kiểm thử nó.
- Nó gây khó chịu khi đọc và viết.
- Liệu chuỗi thông điệp lỗi chính xác có phải là điều chúng ta _thực sự quan tâm_ không?

Điều này nói lên gì? Trải nghiệm viết test phản ánh trải nghiệm của người dùng code của bạn.

Người dùng code sẽ xử lý lỗi như thế nào? Tốt nhất họ chỉ có thể so sánh chuỗi lỗi — rất dễ sai và khó bảo trì.

## Những gì chúng ta nên làm

Với TDD, chúng ta có lợi thế là có thể tư duy theo kiểu:

> _Tôi_ muốn sử dụng mã này như thế nào?

Những gì chúng ta có thể làm cho `DumbGetter` là cung cấp một cách để người dùng sử dụng hệ thống kiểu (type system) để hiểu loại lỗi nào đã xảy ra.

Điều gì sẽ xảy ra nếu `DumbGetter` có thể trả về cho chúng ta thứ gì đó như:

```go
type BadStatusError struct {
	URL    string
	Status int
}
```

Thay vì một chuỗi ký tự mang tính "ma thuật", chúng ta có _dữ liệu_ thực sự để làm việc.

Hãy thay đổi bài kiểm thử hiện tại của chúng ta để phản ánh nhu cầu này:

```go
t.Run("khi bạn không nhận được mã 200, bạn sẽ nhận được một lỗi trạng thái", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("mong đợi một lỗi")
	}

	got, isStatusErr := err.(BadStatusError)

	if !isStatusErr {
		t.Fatalf("không phải là BadStatusError, nhận được %T", err)
	}

	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

Chúng ta sẽ phải làm cho `BadStatusError` triển khai (implement) error interface.

```go
func (b BadStatusError) Error() string {
	return fmt.Sprintf("did not get 200 from %s, got %d", b.URL, b.Status)
}
```

### Bài kiểm thử làm gì?

Thay vì kiểm tra chuỗi ký tự chính xác của lỗi, chúng ta đang thực hiện một [type assertion](https://tour.golang.org/methods/15) trên lỗi để xem nó có phải là một `BadStatusError` hay không. Điều này thể hiện mong muốn của chúng ta về _loại_ lỗi một cách rõ ràng hơn. Giả sử việc kiểm tra (assertion) thành công, chúng ta có thể kiểm tra các thuộc tính của lỗi xem chúng có chính xác không.

Khi chúng ta chạy bài kiểm thử, nó sẽ cho chúng ta biết rằng chúng ta đã không trả về đúng loại lỗi:

```
--- FAIL: TestDumbGetter (0.00s)
    --- FAIL: TestDumbGetter/when_you_dont_get_a_200_you_get_a_status_error (0.00s)
    	error-types_test.go:56: was not a BadStatusError, got *errors.errorString
```

Hãy sửa `DumbGetter` bằng cách cập nhật mã xử lý lỗi để sử dụng kiểu dữ liệu của chúng ta:

```go
if res.StatusCode != http.StatusOK {
	return "", BadStatusError{URL: url, Status: res.StatusCode}
}
```

Sự thay đổi này đã mang lại một số _hiệu ứng tích cực thực sự_:

- Hàm `DumbGetter` của chúng ta đã trở nên đơn giản hơn, nó không còn bận tâm đến những chi tiết phức tạp của một chuỗi lỗi nữa, nó chỉ tạo ra một `BadStatusError`.
- Các bài kiểm thử của chúng ta giờ đây phản ánh (và làm tài liệu) những gì người dùng mã của chúng ta _có thể_ làm nếu họ quyết định muốn xử lý lỗi tinh vi hơn là chỉ ghi nhật ký (logging). Chỉ cần thực hiện một type assertion và sau đó bạn có thể dễ dàng truy cập vào các thuộc tính của lỗi.
- Nó vẫn "chỉ" là một `error`, vì vậy nếu họ muốn, họ có thể chuyển nó lên trên ngăn xếp cuộc gọi (call stack) hoặc ghi nhật ký nó như bất kỳ `error` nào khác.

## Tổng kết

Nếu bạn thấy mình đang kiểm thử nhiều điều kiện lỗi khác nhau, đừng rơi vào cái bẫy so sánh các thông điệp lỗi.

Việc này dẫn đến các bài kiểm thử dễ bị hỏng (flaky) và khó đọc/viết, đồng thời nó cũng phản ánh những khó khăn mà người dùng mã của bạn sẽ gặp phải nếu họ cũng cần bắt đầu thực hiện những việc khác nhau tùy thuộc vào loại lỗi đã xảy ra.

Hãy luôn đảm bảo các bài kiểm thử phản ánh cách _bạn_ muốn sử dụng mã của mình, vì vậy về khía cạnh này, hãy cân nhắc việc tạo các kiểu lỗi để đóng gói các loại lỗi của bạn. Điều này giúp việc xử lý các loại lỗi khác nhau trở nên dễ dàng hơn cho người dùng mã của bạn, đồng thời giúp mã xử lý lỗi của bạn đơn giản và dễ đọc hơn.

## Phụ lục

Kể từ Go 1.13, có những cách mới để làm việc với các lỗi trong thư viện chuẩn, điều này đã được đề cập trong [Go Blog](https://blog.golang.org/go1.13-errors)

```go
t.Run("khi bạn không nhận được mã 200, bạn sẽ nhận được một lỗi trạng thái", func(t *testing.T) {

	svr := httptest.NewServer(http.HandlerFunc(func(res http.ResponseWriter, req *http.Request) {
		res.WriteHeader(http.StatusTeapot)
	}))
	defer svr.Close()

	_, err := DumbGetter(svr.URL)

	if err == nil {
		t.Fatal("mong đợi một lỗi")
	}

	var got BadStatusError
	isBadStatusError := errors.As(err, &got)
	want := BadStatusError{URL: svr.URL, Status: http.StatusTeapot}

	if !isBadStatusError {
		t.Fatalf("không phải là BadStatusError, nhận được %T", err)
	}

	if got != want {
		t.Errorf("got %v, want %v", got, want)
	}
})
```

Trong trường hợp này, chúng ta đang sử dụng [`errors.As`](https://pkg.go.dev/errors#example-As) để cố gắng trích xuất lỗi của mình vào kiểu dữ liệu tùy chỉnh. Nó trả về một giá trị `bool` để biểu thị sự thành công và trích xuất lỗi vào biến `got` cho chúng ta.
