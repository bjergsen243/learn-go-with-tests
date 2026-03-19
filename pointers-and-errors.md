# Pointers & errors

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/pointers)**

Chúng ta đã học về structs ở phần trước, cho phép nhóm các giá trị liên quan đến một khái niệm.

Đôi khi bạn có thể muốn dùng structs để quản lý state, cung cấp methods để người dùng thay đổi state theo cách bạn có thể kiểm soát.

**Fintech yêu thích Go** và... Bitcoin nữa! Vậy hãy xây dựng một hệ thống ngân hàng tuyệt vời.

Hãy tạo struct `Wallet` cho phép chúng ta nạp `Bitcoin`.

## Viết test trước tiên

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(10)

	got := wallet.Balance()
	want := 10

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

Trong [ví dụ trước](./structs-methods-and-interfaces.md) chúng ta truy cập các trường trực tiếp bằng tên trường, nhưng trong _ví trực tuyến bảo mật cao_ của chúng ta, chúng ta không muốn lộ state nội bộ ra bên ngoài. Chúng ta muốn kiểm soát truy cập qua methods.

## Thử chạy test

`./wallet_test.go:7:12: undefined: Wallet`

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Compiler không biết `Wallet` là gì nên hãy khai báo nó.

```go
type Wallet struct{}
```

Bây giờ hãy thử chạy lại test:

```
./wallet_test.go:9:8: wallet.Deposit undefined (type Wallet has no field or method Deposit)
./wallet_test.go:11:15: wallet.Balance undefined (type Wallet has no field or method Balance)
```

Chúng ta cần định nghĩa các methods này.

Nhớ chỉ làm đủ để chạy test. Chúng ta cần đảm bảo test fail đúng cách với thông báo lỗi rõ ràng.

```go
func (w Wallet) Deposit(amount int) {

}

func (w Wallet) Balance() int {
	return 0
}
```

Nếu cú pháp này chưa quen, hãy quay lại đọc phần structs.

Test giờ sẽ biên dịch và chạy

`wallet_test.go:15: got 0 want 10`

## Viết đủ code để test chạy thành công

Chúng ta cần một loại biến _balance_ trong struct để lưu state

```go
type Wallet struct {
	balance int
}
```

Trong Go, nếu một symbol (biến, kiểu, hàm,...) bắt đầu bằng chữ cái viết thường, thì nó là private _bên ngoài package định nghĩa nó_.

Trong trường hợp này, chúng ta muốn các methods có thể thao tác giá trị này, nhưng không ai khác được phép.

Nhớ rằng chúng ta có thể truy cập trường `balance` nội bộ của struct qua biến "receiver".

```go
func (w Wallet) Deposit(amount int) {
	w.balance += amount
}

func (w Wallet) Balance() int {
	return w.balance
}
```

Với sự nghiệp trong fintech được đảm bảo, hãy chạy test suite và tận hưởng test pass

`wallet_test.go:15: got 0 want 10`

### Không đúng rồi

À, khó hiểu quá, code trông có vẻ đúng. Chúng ta thêm số tiền mới vào số dư rồi method balance sẽ trả về trạng thái hiện tại của nó.

In Go, **khi bạn gọi function hoặc method, các đối số được** _**sao chép**_.

Khi gọi `func (w Wallet) Deposit(amount int)`, `w` là bản sao của bất cứ thứ gì chúng ta gọi method từ đó.

Không cần quá học thuật, khi bạn tạo một giá trị — như một wallet, nó được lưu ở đâu đó trong bộ nhớ. Bạn có thể tìm _địa chỉ_ của bộ nhớ đó với `&myVal`.

Hãy thử nghiệm bằng cách thêm một vài lệnh in vào code:

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(10)

	got := wallet.Balance()

	fmt.Printf("address of balance in test is %p \n", &wallet.balance)

	want := 10

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

```go
func (w Wallet) Deposit(amount int) {
	fmt.Printf("address of balance in Deposit is %p \n", &w.balance)
	w.balance += amount
}
```

`%p` in địa chỉ bộ nhớ ở hệ hexadecimal với tiền tố `0x` và `\n` in dòng mới. Chú ý rằng chúng ta lấy con trỏ (địa chỉ bộ nhớ) của thứ gì đó bằng cách đặt ký tự `&` ở đầu symbol.

Bây giờ chạy lại test:

```text
address of balance in Deposit is 0xc420012268
address of balance in test is 0xc420012260
```

Bạn có thể thấy địa chỉ của hai balance khác nhau. Vì vậy khi chúng ta thay đổi giá trị balance trong code, chúng ta đang làm việc trên _bản sao_ của giá trị từ test. Do đó balance trong test không thay đổi.

Chúng ta có thể sửa bằng _pointers_. [Pointers](https://gobyexample.com/pointers) cho phép chúng ta _trỏ_ đến một số giá trị và sau đó thay đổi chúng. Thay vì sao chép toàn bộ Wallet, chúng ta lấy con trỏ đến wallet đó để có thể thay đổi các giá trị gốc trong đó.

```go
func (w *Wallet) Deposit(amount int) {
	w.balance += amount
}

func (w *Wallet) Balance() int {
	return w.balance
}
```

Sự khác biệt là kiểu receiver là `*Wallet` thay vì `Wallet`, có thể đọc là "con trỏ đến wallet".

Thử chạy lại test và chúng sẽ pass.

Bây giờ bạn có thể thắc mắc tại sao chúng pass? Chúng ta không dereference pointer trong hàm, như:

```go
func (w *Wallet) Balance() int {
	return (*w).balance
}
```

và dường như truy cập trực tiếp vào đối tượng. Thực ra, code trên dùng `(*w)` hoàn toàn hợp lệ. Tuy nhiên, người tạo ra Go cho rằng ký hiệu này phiền toái, nên ngôn ngữ cho phép chúng ta viết `w.balance` mà không cần dereference tường minh. Các con trỏ đến struct thậm chí có tên riêng: _struct pointers_ và chúng được [dereference tự động](https://golang.org/ref/spec#Method_values).

Về mặt kỹ thuật, bạn không cần đổi `Balance` để dùng pointer receiver vì sao chép balance là ổn. Tuy nhiên, theo quy ước, nên giữ kiểu receiver của methods nhất quán.

## Refactor

Chúng ta nói sẽ tạo Bitcoin wallet nhưng chưa đề cập đến chúng cho đến nay. Chúng ta đang dùng `int` vì chúng là kiểu tốt để đếm!

Có vẻ hơi quá khi tạo `struct` cho bitcoin. `int` ổn về cách hoạt động của nó, nhưng không mô tả được ý nghĩa.

Go cho phép bạn tạo các kiểu mới từ các kiểu đã có.

Cú pháp là `type MyName OriginalType`

```go
type Bitcoin int

type Wallet struct {
	balance Bitcoin
}

func (w *Wallet) Deposit(amount Bitcoin) {
	w.balance += amount
}

func (w *Wallet) Balance() Bitcoin {
	return w.balance
}
```

```go
func TestWallet(t *testing.T) {

	wallet := Wallet{}

	wallet.Deposit(Bitcoin(10))

	got := wallet.Balance()

	want := Bitcoin(10)

	if got != want {
		t.Errorf("got %d want %d", got, want)
	}
}
```

Để tạo `Bitcoin`, chỉ cần dùng cú pháp `Bitcoin(999)`.

Bằng cách này, chúng ta tạo kiểu mới và có thể khai báo _methods_ trên chúng. Điều này có thể rất hữu ích khi bạn muốn thêm một số chức năng miền cụ thể lên các kiểu đã có.

Hãy implement [Stringer](https://golang.org/pkg/fmt/#Stringer) trên Bitcoin:

```go
type Stringer interface {
	String() string
}
```

Interface này được định nghĩa trong package `fmt` và cho phép bạn định nghĩa cách kiểu của mình được in khi dùng với chuỗi format `%s` trong các lệnh print.

```go
func (b Bitcoin) String() string {
	return fmt.Sprintf("%d BTC", b)
}
```

Như bạn thấy, cú pháp tạo method trên khai báo kiểu giống với struct.

Tiếp theo chúng ta cần cập nhật các format string trong test để dùng `String()`:

```go
	if got != want {
		t.Errorf("got %s want %s", got, want)
	}
```

Để thấy điều này hoạt động, hãy cố tình làm test fail:

`wallet_test.go:18: got 10 BTC want 20 BTC`

Điều này làm rõ hơn những gì đang xảy ra trong test.

Yêu cầu tiếp theo là hàm `Withdraw`.

## Viết test trước tiên

Về cơ bản ngược lại với `Deposit()`

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}

		wallet.Deposit(Bitcoin(10))

		got := wallet.Balance()

		want := Bitcoin(10)

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	})

	t.Run("withdraw", func(t *testing.T) {
		wallet := Wallet{balance: Bitcoin(20)}

		wallet.Withdraw(Bitcoin(10))

		got := wallet.Balance()

		want := Bitcoin(10)

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	})
}
```

## Thử chạy test

`./wallet_test.go:26:9: wallet.Withdraw undefined (type Wallet has no field or method Withdraw)`

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func (w *Wallet) Withdraw(amount Bitcoin) {

}
```

`wallet_test.go:33: got 20 BTC want 10 BTC`

## Viết đủ code để test chạy thành công

```go
func (w *Wallet) Withdraw(amount Bitcoin) {
	w.balance -= amount
}
```

## Refactor

Có một số sự trùng lặp trong test, hãy refactor chúng.

```go
func TestWallet(t *testing.T) {

	assertBalance := func(t testing.TB, wallet Wallet, want Bitcoin) {
		t.Helper()
		got := wallet.Balance()

		if got != want {
			t.Errorf("got %s want %s", got, want)
		}
	}

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw", func(t *testing.T) {
		wallet := Wallet{balance: Bitcoin(20)}
		wallet.Withdraw(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

}
```

Điều gì sẽ xảy ra nếu bạn cố `Withdraw` nhiều hơn số tiền còn lại trong tài khoản? Hiện tại, yêu cầu là giả định không có tiện ích thấu chi.

Làm sao chúng ta báo hiệu vấn đề khi dùng `Withdraw`?

Trong Go, nếu bạn muốn chỉ ra lỗi, thông lệ là hàm của bạn trả về `err` để người gọi kiểm tra và xử lý.

Hãy thử trong test.

## Viết test trước tiên

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertBalance(t, wallet, startingBalance)

	if err == nil {
		t.Error("wanted an error but didn't get one")
	}
})
```

Chúng ta muốn `Withdraw` trả về lỗi _nếu_ bạn cố rút nhiều hơn số tiền có và balance phải giữ nguyên.

Sau đó chúng ta kiểm tra lỗi đã được trả về bằng cách fail test nếu nó là `nil`.

`nil` tương đương với `null` trong các ngôn ngữ khác. Errors có thể là `nil` vì kiểu trả về của `Withdraw` sẽ là `error`, là một interface. Nếu bạn thấy hàm nhận đối số hoặc trả về giá trị là interfaces, chúng có thể là nil.

Giống `null`, nếu bạn truy cập vào giá trị `nil`, nó sẽ gây ra **runtime panic**. Điều này tệ! Bạn phải đảm bảo kiểm tra nil.

## Thử chạy test

`./wallet_test.go:31:25: wallet.Withdraw(Bitcoin(100)) used as value`

Cách diễn đạt có thể hơi khó hiểu, nhưng ý định trước đây với `Withdraw` là chỉ gọi nó, không bao giờ trả về giá trị. Để biên dịch được, chúng ta sẽ cần thay đổi nó để có kiểu trả về.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {
	w.balance -= amount
	return nil
}
```

Một lần nữa, điều quan trọng là chỉ viết đủ code để thỏa mãn compiler. Chúng ta sửa method `Withdraw` để trả về `error` và hiện tại phải trả về _thứ gì đó_ nên hãy trả về `nil`.

## Viết đủ code để test chạy thành công

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return errors.New("oh no")
	}

	w.balance -= amount
	return nil
}
```

Nhớ import `errors` vào code.

`errors.New` tạo `error` mới với thông báo bạn chọn.

## Refactor

Hãy tạo test helper nhanh để kiểm tra lỗi và cải thiện khả năng đọc của test:

```go
assertError := func(t testing.TB, err error) {
	t.Helper()
	if err == nil {
		t.Error("wanted an error but didn't get one")
	}
}
```

Và trong test:

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertError(t, err)
	assertBalance(t, wallet, startingBalance)
})
```

Hy vọng khi trả về lỗi "oh no", bạn đang suy nghĩ rằng chúng ta _có thể_ cần lặp lại điều đó vì nó không có vẻ hữu ích.

Giả sử lỗi cuối cùng được trả về cho người dùng, hãy cập nhật test để assert về một thông báo lỗi cụ thể thay vì chỉ sự tồn tại của lỗi.

## Viết test trước tiên

Cập nhật helper với `string` để so sánh.

```go
assertError := func(t testing.TB, got error, want string) {
	t.Helper()

	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got.Error() != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

Như bạn thấy, `Error`s có thể được chuyển thành string bằng method `.Error()`, chúng ta dùng để so sánh với string mong đợi. Chúng ta cũng đảm bảo lỗi không phải `nil` để tránh gọi `.Error()` trên `nil`.

Và cập nhật caller:

```go
t.Run("withdraw insufficient funds", func(t *testing.T) {
	startingBalance := Bitcoin(20)
	wallet := Wallet{startingBalance}
	err := wallet.Withdraw(Bitcoin(100))

	assertError(t, err, "cannot withdraw, insufficient funds")
	assertBalance(t, wallet, startingBalance)
})
```

Chúng ta đã giới thiệu `t.Fatal` sẽ dừng test nếu được gọi. Điều này vì chúng ta không muốn tiếp tục assert về lỗi trả về nếu không có. Không có điều này, test sẽ tiếp tục bước tiếp theo và panic vì pointer nil.

## Thử chạy test

`wallet_test.go:61: got err 'oh no' want 'cannot withdraw, insufficient funds'`

## Viết đủ code để test chạy thành công

```go
func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return errors.New("cannot withdraw, insufficient funds")
	}

	w.balance -= amount
	return nil
}
```

## Refactor

Chúng ta có sự trùng lặp thông báo lỗi trong cả code test và code `Withdraw`.

Sẽ rất phiền nếu test fail chỉ vì ai đó muốn đổi từ ngữ lỗi và nó quá chi tiết cho test của chúng ta. Chúng ta không _thực sự_ quan tâm đến từ ngữ chính xác, chỉ cần một loại lỗi có ý nghĩa về rút tiền được trả về trong điều kiện nhất định.

Trong Go, errors là values, nên chúng ta có thể refactor nó thành biến và có một nguồn sự thật duy nhất.

```go
var ErrInsufficientFunds = errors.New("cannot withdraw, insufficient funds")

func (w *Wallet) Withdraw(amount Bitcoin) error {

	if amount > w.balance {
		return ErrInsufficientFunds
	}

	w.balance -= amount
	return nil
}
```

Từ khóa `var` cho phép định nghĩa giá trị global cho package.

Đây là thay đổi tích cực vì bây giờ hàm `Withdraw` của chúng ta trông rõ ràng hơn.

Tiếp theo chúng ta có thể refactor code test để dùng giá trị này thay vì các chuỗi cụ thể.

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw with funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		wallet.Withdraw(Bitcoin(10))
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw insufficient funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(100))

		assertError(t, err, ErrInsufficientFunds)
		assertBalance(t, wallet, Bitcoin(20))
	})
}

func assertBalance(t testing.TB, wallet Wallet, want Bitcoin) {
	t.Helper()
	got := wallet.Balance()

	if got != want {
		t.Errorf("got %q want %q", got, want)
	}
}

func assertError(t testing.TB, got, want error) {
	t.Helper()
	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got != want {
		t.Errorf("got %q, want %q", got, want)
	}
}
```

Và giờ test dễ theo dõi hơn.

Tôi đã đưa các helpers ra ngoài hàm test chính để khi ai đó mở file, họ có thể đọc assertions của chúng ta trước, chứ không phải một số helpers.

Một tính chất hữu ích khác của tests là chúng giúp chúng ta hiểu _cách dùng thực tế_ của code. Chúng ta có thể thấy ở đây rằng developer có thể đơn giản gọi code của chúng ta và thực hiện so sánh bằng `ErrInsufficientFunds` rồi xử lý tương ứng.

### Errors chưa được kiểm tra

Dù compiler Go giúp bạn rất nhiều, đôi khi vẫn có những thứ bạn bỏ lỡ và xử lý lỗi đôi khi có thể phức tạp.

Có một tình huống chúng ta chưa test. Để tìm nó, chạy lệnh sau trong terminal để cài `errcheck`, một trong nhiều linter có sẵn cho Go.

`go install github.com/kisielk/errcheck@latest`

Sau đó, trong thư mục có code của bạn, chạy `errcheck .`

Bạn sẽ thấy:

`wallet_test.go:17:18: wallet.Withdraw(Bitcoin(10))`

Điều này cho biết chúng ta chưa kiểm tra lỗi được trả về trên dòng code đó. Dòng code đó trong máy tôi tương ứng với kịch bản withdraw bình thường của chúng ta vì chúng ta chưa kiểm tra xem nếu `Withdraw` thành công thì lỗi có _không_ được trả về.

Đây là code test cuối cùng xử lý điều này.

```go
func TestWallet(t *testing.T) {

	t.Run("deposit", func(t *testing.T) {
		wallet := Wallet{}
		wallet.Deposit(Bitcoin(10))

		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw with funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(10))

		assertNoError(t, err)
		assertBalance(t, wallet, Bitcoin(10))
	})

	t.Run("withdraw insufficient funds", func(t *testing.T) {
		wallet := Wallet{Bitcoin(20)}
		err := wallet.Withdraw(Bitcoin(100))

		assertError(t, err, ErrInsufficientFunds)
		assertBalance(t, wallet, Bitcoin(20))
	})
}

func assertBalance(t testing.TB, wallet Wallet, want Bitcoin) {
	t.Helper()
	got := wallet.Balance()

	if got != want {
		t.Errorf("got %s want %s", got, want)
	}
}

func assertNoError(t testing.TB, got error) {
	t.Helper()
	if got != nil {
		t.Fatal("got an error but didn't want one")
	}
}

func assertError(t testing.TB, got error, want error) {
	t.Helper()
	if got == nil {
		t.Fatal("didn't get an error but wanted one")
	}

	if got != want {
		t.Errorf("got %s, want %s", got, want)
	}
}
```

## Tổng kết

### Pointers

* Go sao chép values khi bạn truyền chúng vào functions/methods, vì vậy nếu bạn viết function cần mutate state, bạn cần nó nhận pointer đến thứ bạn muốn thay đổi.
* Việc Go sao chép values hữu ích nhiều lần nhưng đôi khi bạn không muốn hệ thống sao chép thứ gì đó, trong trường hợp đó bạn cần truyền reference. Ví dụ bao gồm tham chiếu đến các cấu trúc dữ liệu rất lớn hoặc những thứ mà chỉ cần một instance \(như connection pool của database\).

### nil

* Pointers có thể là nil
* Khi hàm trả về pointer đến thứ gì đó, bạn cần đảm bảo kiểm tra nếu nó là nil hoặc bạn có thể gây ra runtime exception — compiler không giúp bạn ở đây.
* Hữu ích khi bạn muốn mô tả giá trị có thể bị thiếu

### Errors

* Errors là cách để báo hiệu thất bại khi gọi function/method.
* Bằng cách lắng nghe test của chúng ta, chúng ta kết luận rằng kiểm tra chuỗi trong error sẽ dẫn đến test không ổn định. Vì vậy chúng ta refactor implementation để dùng giá trị có ý nghĩa thay thế và điều này dẫn đến code dễ test hơn và kết luận điều này sẽ dễ hơn cho người dùng API của chúng ta.
* Đây không phải là hết câu chuyện về xử lý lỗi, bạn có thể làm những thứ phức tạp hơn nhưng đây chỉ là giới thiệu. Các phần sau sẽ đề cập thêm các chiến lược.
* [Đừng chỉ kiểm tra errors, hãy xử lý chúng một cách khéo léo](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)

### Tạo kiểu mới từ kiểu đã có

* Hữu ích để thêm ý nghĩa miền cụ thể hơn cho values
* Có thể cho phép implement interfaces

Pointers và errors là phần lớn trong việc viết Go mà bạn cần phải quen. Rất may, compiler thường _sẽ_ giúp bạn nếu bạn làm gì đó sai, chỉ cần dành thời gian đọc lỗi.
