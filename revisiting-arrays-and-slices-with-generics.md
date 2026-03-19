# Xem lại mảng và lát cắt với generics

**[Mã nguồn của chương này là sự tiếp nối từ chương Mảng và lát cắt, tìm thấy tại đây](https://github.com/quii/learn-go-with-tests/tree/main/arrays)**

Hãy xem lại cả `SumAll` và `SumAllTails` mà chúng ta đã viết trong chương [mảng và lát cắt](arrays-and-slices.md). Nếu bạn không có phiên bản của mình, vui lòng sao chép mã nguồn từ chương [mảng và lát cắt](arrays-and-slices.md) cùng với các bản kiểm thử.

```go
// Sum tính tổng của một lát cắt số.
func Sum(numbers []int) int {
	var sum int
	for _, number := range numbers {
		sum += number
	}
	return sum
}

// SumAllTails tính tổng của tất cả trừ số đầu tiên cho mỗi lát cắt được cung cấp.
func SumAllTails(numbersToSum ...[]int) []int {
	var sums []int
	for _, numbers := range numbersToSum {
		if len(numbers) == 0 {
			sums = append(sums, 0)
		} else {
			tail := numbers[1:]
			sums = append(sums, Sum(tail))
		}
	}

	return sums
}
```

Bạn có thấy một mẫu hình lặp lại không?

- Tạo một loại giá trị kết quả "ban đầu".
- Lặp qua tập hợp, áp dụng một loại thao tác (hoặc hàm) nào đó cho kết quả và mục tiếp theo trong lát cắt, thiết lập một giá trị mới cho kết quả.
- Trả về kết quả.

Ý tưởng này thường được thảo luận trong các cộng đồng lập trình chức năng (functional programming), thường được gọi là 'reduce' hoặc [fold](https://en.wikipedia.org/wiki/Fold_(higher-order_function)).

> Trong lập trình chức năng, fold (còn được gọi là reduce, accumulate, aggregate, compress, hoặc inject) đề cập đến một họ các hàm bậc cao (higher-order functions) phân tích một cấu trúc dữ liệu đệ quy và thông qua việc sử dụng một thao tác kết hợp nhất định, kết hợp lại kết quả của việc xử lý đệ quy các thành phần cấu tạo của nó, xây dựng nên một giá trị trả về. Thông thường, một fold được cung cấp một hàm kết hợp, một nút trên cùng của một cấu trúc dữ liệu và có thể là một số giá trị mặc định được sử dụng trong các điều kiện nhất định. Sau đó, fold tiến hành kết hợp các phần tử của phân cấp cấu trúc dữ liệu, sử dụng hàm theo một cách có hệ thống.

Go luôn có các hàm bậc cao, và kể từ phiên bản 1.18, nó cũng có [generics](./generics.md), vì vậy hiện nay chúng ta có thể định nghĩa một số hàm này đã được thảo luận rộng rãi trong lĩnh vực của chúng ta. Không có lý do gì để trốn tránh, đây là một sự trừu tượng hóa rất phổ biến bên ngoài hệ sinh thái Go và việc hiểu nó sẽ mang lại lợi ích.

Bây giờ, tôi biết một số bạn có lẽ đang cảm thấy e ngại về điều này.

> Go vốn dĩ nên đơn giản

**Đừng nhầm lẫn giữa sự dễ dàng với sự đơn giản**. Việc sử dụng các vòng lặp và sao chép-dán mã nguồn là dễ dàng, nhưng nó không nhất thiết là đơn giản. Để biết thêm về đơn giản so với dễ dàng, hãy xem bài nói chuyện kiệt tác của Rich Hickey - [Simple Made Easy](https://www.youtube.com/watch?v=SxdOUGdseq4).

**Đừng nhầm lẫn giữa sự xa lạ với sự phức tạp**. Fold/reduce ban đầu nghe có vẻ đáng sợ và mang tính khoa học máy tính nhưng thực chất nó chỉ là một sự trừu tượng hóa cho một thao tác rất phổ biến: Lấy một tập hợp và kết hợp nó thành một mục duy nhất. Khi bạn lùi lại một bước, bạn sẽ nhận ra mình có lẽ thực thực hiện việc này *rất nhiều*.

## Tái cấu trúc bằng generic

Một sai lầm mà mọi người thường mắc phải với các tính năng ngôn ngữ mới sáng loáng là họ bắt đầu sử dụng chúng mà không có một trường hợp sử dụng cụ thể. Họ dựa vào phỏng đoán và sự võ đoán để dẫn dắt nỗ lực của mình.

May mắn thay, chúng ta đã viết các hàm "hữu ích" của mình và có các bản kiểm thử bao quanh chúng, vì vậy bây giờ chúng ta tự do thử nghiệm các ý tưởng trong giai đoạn tái cấu trúc của TDD và biết rằng bất cứ điều gì chúng ta đang thử đều có sự xác nhận giá trị của nó thông qua các bản kiểm thử đơn vị của chúng ta.

Sử dụng generics như một công cụ để đơn giản hóa mã nguồn thông qua bước tái cấu trúc có nhiều khả năng dẫn dắt bạn đến những cải tiến hữu ích, thay vì những sự trừu tượng hóa vội vàng.

Chúng ta an tâm để thử nghiệm mọi thứ, chạy lại các bản kiểm thử, nếu chúng ta thích thay đổi đó, chúng ta có thể commit. Nếu không, chỉ cần hoàn tác thay đổi. Sự tự do thử nghiệm này là một trong những giá trị thực sự to lớn của TDD.

Bạn nên làm quen với cú pháp generics [từ chương trước](generics.md), hãy thử viết hàm `Reduce` của riêng bạn và sử dụng nó bên trong `Sum` và `SumAllTails`.

### Gợi ý

Nếu bạn nghĩ về các đối số cho hàm của mình trước tiên, nó sẽ cung cấp cho bạn một tập hợp rất nhỏ các giải pháp hợp lệ:
  - Mảng bạn muốn reduce
  - Một loại hàm kết hợp nào đó

"Reduce" là một mẫu hình được ghi chép cực kỳ đầy đủ, không cần phải phát minh lại bánh xe. [Đọc wiki, đặc biệt là phần danh sách (lists)](https://en.wikipedia.org/wiki/Fold_(higher-order_function)), nó sẽ gợi ý cho bạn một đối số khác mà bạn sẽ cần.

> Trong thực tế, việc có một giá trị ban đầu là thuận tiện và tự nhiên.

### Lần viết `Reduce` đầu tiên của tôi

```go
func Reduce[A any](collection []A, f func(A, A) A, initialValue A) A {
	var result = initialValue
	for _, x := range collection {
		result = f(result, x)
	}
	return result
}
```

Reduce nắm bắt được *bản chất* của mẫu hình, nó là một hàm nhận một tập hợp (collection), một hàm tích lũy (accumulating function), một giá trị ban đầu, và trả về một giá trị duy nhất. Không có những sự phân tâm lộn xộn xung quanh các kiểu dữ liệu cụ thể.

Nếu bạn hiểu cú pháp generics, bạn sẽ không gặp khó khăn gì khi hiểu hàm này làm gì. Bằng cách sử dụng thuật ngữ đã được công nhận `Reduce`, các lập trình viên từ các ngôn ngữ khác cũng hiểu được ý định đó.

### Cách sử dụng

```go
// Sum tính tổng của một lát cắt số.
func Sum(numbers []int) int {
	add := func(acc, x int) int { return acc + x }
	return Reduce(numbers, add, 0)
}

// SumAllTails tính tổng của tất cả trừ số đầu tiên cho mỗi lát cắt được cung cấp.
func SumAllTails(numbers ...[]int) []int {
	sumTail := func(acc, x []int) []int {
		if len(x) == 0 {
			return append(acc, 0)
		} else {
			tail := x[1:]
			return append(acc, Sum(tail))
		}
	}

	return Reduce(numbers, sumTail, []int{})
}
```

`Sum` và `SumAllTails` giờ đây mô tả hành vi tính toán của chúng dưới dạng các hàm được khai báo tương ứng ở các dòng đầu tiên của chúng. Hành động chạy tính toán trên tập hợp được trừu tượng hóa trong `Reduce`.

## Các ứng dụng khác của reduce

Sử dụng các bản kiểm thử, chúng ta có thể chơi đùa với hàm reduce của mình để xem mức độ tái sử dụng của nó như thế nào. Tôi đã sao chép các hàm khẳng định generic (generic assertion functions) của chúng ta từ chương trước.

```go
func TestReduce(t *testing.T) {
	t.Run("multiplication of all elements", func(t *testing.T) {
		multiply := func(x, y int) int {
			return x * y
		}

		AssertEqual(t, Reduce([]int{1, 2, 3}, multiply, 1), 6)
	})

	t.Run("concatenate strings", func(t *testing.T) {
		concatenate := func(x, y string) string {
			return x + y
		}

		AssertEqual(t, Reduce([]string{"a", "b", "c"}, concatenate, ""), "abc")
	})
}
```

### Giá trị zero (Zero value)

Trong ví dụ về phép nhân, chúng ta thấy lý do của việc có một giá trị mặc định làm đối số cho `Reduce`. Nếu chúng ta dựa vào giá trị mặc định của Go là 0 cho kiểu `int`, chúng ta sẽ nhân giá trị ban đầu của mình với 0, và sau đó là những giá trị tiếp theo, vì vậy bạn sẽ chỉ luôn nhận được 0. Bằng cách đặt nó là 1, phần tử đầu tiên trong lát cắt sẽ giữ nguyên, và phần còn lại sẽ nhân với các phần tử tiếp theo.

Nếu bạn muốn nghe có vẻ thông thái với những người bạn mọt sách của mình, bạn sẽ gọi đây là [Phần tử đơn vị (The Identity Element)](https://en.wikipedia.org/wiki/Identity_element).

> Trong toán học, một phần tử đơn vị, hoặc phần tử trung hòa, của một phép toán nhị phân hoạt động trên một tập hợp là một phần tử của tập hợp mà giữ nguyên không đổi mọi phần tử của tập hợp khi phép toán được áp dụng.

Trong phép cộng, phần tử đơn vị là 0.

`1 + 0 = 1`

Với phép nhân, nó là 1.

`1 * 1 = 1`

## Nếu chúng ta muốn reduce thành một kiểu khác với `A` thì sao?

Giả sử chúng ta có một danh sách các giao dịch `Transaction` và chúng ta muốn một hàm nhận chúng, cộng với một cái tên để tìm ra số dư ngân hàng của họ.

Hãy tuân theo quy trình TDD.

## Viết test trước tiên

```go
func TestBadBank(t *testing.T) {
	transactions := []Transaction{
		{
			From: "Chris",
			To:   "Riya",
			Sum:  100,
		},
		{
			From: "Adil",
			To:   "Chris",
			Sum:  25,
		},
	}

	AssertEqual(t, BalanceFor(transactions, "Riya"), 100)
	AssertEqual(t, BalanceFor(transactions, "Chris"), -75)
	AssertEqual(t, BalanceFor(transactions, "Adil"), -25)
}
```

## Thử chạy test

```
# github.com/quii/learn-go-with-tests/arrays/v8 [github.com/quii/learn-go-with-tests/arrays/v8.test]
./bad_bank_test.go:6:20: undefined: Transaction
./bad_bank_test.go:18:14: undefined: BalanceFor
```

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Chúng ta chưa có các kiểu hoặc hàm của mình, hãy thêm chúng để bản kiểm thử có thể chạy được.

```go
type Transaction struct {
	From string
	To   string
	Sum  float64
}

func BalanceFor(transactions []Transaction, name string) float64 {
	return 0.0
}
```

Khi bạn chạy bản kiểm thử, bạn sẽ thấy kết quả như sau:

```
=== RUN   TestBadBank
    bad_bank_test.go:19: got 0, want 100
    bad_bank_test.go:20: got 0, want -75
    bad_bank_test.go:21: got 0, want -25
--- FAIL: TestBadBank (0.00s)
```

## Viết đủ mã nguồn để bản kiểm thử vượt qua

Hãy viết mã nguồn như thể chúng ta chưa có hàm `Reduce` trước tiên.

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	var balance float64
	for _, t := range transactions {
		if t.From == name {
			balance -= t.Sum
		}
		if t.To == name {
			balance += t.Sum
		}
	}
	return balance
}
```

## Tái cấu trúc

Tại thời điểm này, hãy có kỷ luật về quản lý mã nguồn (source control) và commit công việc của bạn. Chúng ta có phần mềm đang hoạt động, sẵn sàng để thách thức Monzo, Barclays, v.v.

Bây giờ công việc của chúng ta đã được commit, chúng ta có thể thoải mái thử nghiệm và thử một số ý tưởng khác nhau trong giai đoạn tái cấu trúc. Công bằng mà nói, mã nguồn chúng ta đang có không hẳn là tệ, nhưng vì mục đích của bài tập này, tôi muốn trình bày cùng một đoạn mã nguồn đó sử dụng `Reduce`.

```go
func BalanceFor(transactions []Transaction, name string) float64 {
	adjustBalance := func(currentBalance float64, t Transaction) float64 {
		if t.From == name {
			return currentBalance - t.Sum
		}
		if t.To == name {
			return currentBalance + t.Sum
		}
		return currentBalance
	}
	return Reduce(transactions, adjustBalance, 0.0)
}
```

Nhưng đoạn mã này sẽ không biên dịch được.

```
./bad_bank.go:19:35: type func(acc float64, t Transaction) float64 of adjustBalance does not match inferred type func(Transaction, Transaction) Transaction for func(A, A) A
```

Lý do là chúng ta đang cố gắng reduce thành một kiểu dữ liệu *khác* với kiểu dữ liệu của tập hợp. Điều này nghe có vẻ đáng sợ, nhưng thực ra chỉ yêu cầu chúng ta điều chỉnh chữ ký kiểu (type signature) của `Reduce` để làm cho nó hoạt động. Chúng ta sẽ không phải thay đổi thân hàm, và không phải thay đổi bất kỳ người gọi hiện tại nào của mình.

```go
func Reduce[A, B any](collection []A, f func(B, A) B, initialValue B) B {
	var result = initialValue
	for _, x := range collection {
		result = f(result, x)
	}
	return result
}
```

Chúng ta đã thêm một ràng buộc kiểu thứ hai cho phép chúng ta nới lỏng các ràng buộc đối với `Reduce`. Điều này cho phép mọi người có thể `Reduce` từ một tập hợp kiểu `A` thành một kiểu `B`. Trong trường hợp của chúng ta, từ `Transaction` thành `float64`.

Điều này làm cho `Reduce` có mục đích chung và có khả năng tái sử dụng cao hơn, và vẫn an toàn kiểu. Nếu bạn thử chạy lại các bản kiểm thử, chúng sẽ biên dịch và vượt qua.

## Mở rộng ngân hàng

Cho vui, tôi muốn cải thiện tính tiện dụng của mã nguồn ngân hàng. Tôi đã lược bỏ quy trình TDD để cho ngắn gọn.

```go
func TestBadBank(t *testing.T) {
	var (
		riya  = Account{Name: "Riya", Balance: 100}
		chris = Account{Name: "Chris", Balance: 75}
		adil  = Account{Name: "Adil", Balance: 200}

		transactions = []Transaction{
			NewTransaction(chris, riya, 100),
			NewTransaction(adil, chris, 25),
		}
	)

	newBalanceFor := func(account Account) float64 {
		return NewBalanceFor(account, transactions).Balance
	}

	AssertEqual(t, newBalanceFor(riya), 200)
	AssertEqual(t, newBalanceFor(chris), 0)
	AssertEqual(t, newBalanceFor(adil), 175)
}
```

Và đây là mã nguồn đã được cập nhật:

```go
package main

type Transaction struct {
	From string
	To   string
	Sum  float64
}

func NewTransaction(from, to Account, sum float64) Transaction {
	return Transaction{From: from.Name, To: to.Name, Sum: sum}
}

type Account struct {
	Name    string
	Balance float64
}

func NewBalanceFor(account Account, transactions []Transaction) Account {
	return Reduce(
		transactions,
		applyTransaction,
		account,
	)
}

func applyTransaction(a Account, transaction Transaction) Account {
	if transaction.From == a.Name {
		a.Balance -= transaction.Sum
	}
	if transaction.To == a.Name {
		a.Balance += transaction.Sum
	}
	return a
}
```

Tôi cảm thấy điều này thực sự cho thấy sức mạnh của việc sử dụng các khái niệm như `Reduce`. Hàm `NewBalanceFor` mang lại cảm giác *tuyên ngôn (declarative)* hơn, mô tả *những gì* xảy ra, thay vì *như thế nào*. Thông thường khi chúng ta đọc mã nguồn, chúng ta lướt qua rất nhiều file, và chúng ta đang cố gắng hiểu *những gì* đang xảy ra hơn là *như thế nào*, và phong cách mã nguồn này tạo điều kiện thuận lợi cho việc đó.

Nếu tôi muốn tìm hiểu sâu vào chi tiết, tôi có thể, và tôi có thể thấy *logic nghiệp vụ* của `applyTransaction` mà không cần lo lắng về các vòng lặp và thay đổi trạng thái (mutating state); `Reduce` thay mặt thực hiện việc đó một cách riêng biệt.

### Fold/reduce khá phổ biến

Khả năng là vô tận với `Reduce` (hoặc `Fold`). Nó là một mẫu hình phổ biến có lý do của nó, nó không chỉ dành cho số học hay nối chuỗi. Hãy thử một vài ứng dụng khác:

- Tại sao không trộn một số `color.RGBA` thành một màu duy nhất?
- Tổng hợp số phiếu bầu trong một cuộc thăm dò, hoặc các mặt hàng trong giỏ hàng.
- Bất cứ thứ gì liên quan đến việc xử lý một danh sách.

## Find

Bây giờ Go đã có generics, kết hợp chúng với các hàm bậc cao, chúng ta có thể giảm bớt rất nhiều mã nguồn lặp đi lặp lại trong các dự án của mình, giúp hệ thống của chúng ta dễ hiểu và dễ quản lý hơn.

Bạn không còn cần phải viết các hàm `Find` cụ thể cho từng loại tập hợp mà bạn muốn tìm kiếm, thay vào đó hãy tái sử dụng hoặc viết một hàm `Find`. Nếu bạn đã hiểu hàm `Reduce` ở trên, việc viết một hàm `Find` sẽ trở nên cực kỳ đơn giản.

Đây là một bản kiểm thử:

```go
func TestFind(t *testing.T) {
	t.Run("find first even number", func(t *testing.T) {
		numbers := []int{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

		firstEvenNumber, found := Find(numbers, func(x int) bool {
			return x%2 == 0
		})
		AssertTrue(t, found)
		AssertEqual(t, firstEvenNumber, 2)
	})
}
```

Và đây là triển khai:

```go
func Find[A any](items []A, predicate func(A) bool) (value A, found bool) {
	for _, v := range items {
		if predicate(v) {
			return v, true
		}
	}
	return
}
```

Một lần nữa, vì nó nhận một kiểu dữ liệu generic, chúng ta có thể tái sử dụng nó theo nhiều cách.

```go
type Person struct {
	Name string
}

t.Run("Find the best programmer", func(t *testing.T) {
	people := []Person{
		Person{Name: "Kent Beck"},
		Person{Name: "Martin Fowler"},
		Person{Name: "Chris James"},
	}

	king, found := Find(people, func(p Person) bool {
		return strings.Contains(p.Name, "Chris")
	})

	AssertTrue(t, found)
	AssertEqual(t, king, Person{Name: "Chris James"})
})
```

Như bạn thấy, mã nguồn này thật hoàn hảo.

## Tổng kết

Khi được thực hiện một cách tinh tế, các hàm bậc cao như thế này sẽ làm cho mã nguồn của bạn đơn giản hơn để đọc và bảo trì, nhưng hãy nhớ quy tắc chung:

Sử dụng quy trình TDD để dẫn dắt các hành vi thực tế, cụ thể mà bạn thực sự cần, trong giai đoạn tái cấu trúc, sau đó bạn *có thể* khám phá ra một số sự trừu tượng hữu ích để giúp làm sạch mã nguồn.

Thực hành kết hợp TDD với các thói quen quản lý mã nguồn tốt. Commit công việc của bạn khi bản kiểm thử của bạn vượt qua, *trước khi* thử tái cấu trúc. Bằng cách này, nếu bạn làm mọi thứ rối tung lên, bạn có thể dễ dàng quay lại trạng thái hoạt động của mình.

### Tên gọi rất quan trọng

Hãy cố gắng nghiên cứu bên ngoài hệ sinh thái Go, để bạn không phát minh lại các mẫu hình vốn đã tồn tại với một cái tên đã được thiết lập sẵn.

Viết một hàm nhận một tập hợp các đối tượng kiểu `A` và chuyển đổi chúng sang kiểu `B`? Đừng gọi nó là `Convert`, đó là [`Map`](https://en.wikipedia.org/wiki/Map_(higher-order_function)). Sử dụng cái tên "thực thụ" cho các hạng mục này sẽ giảm bớt gánh nặng nhận thức cho người khác và giúp việc tìm kiếm trên các công cụ tìm kiếm để tìm hiểu thêm trở nên dễ dàng hơn.

### Điều này có cảm thấy không giống phong cách Go (idiomatic)?

Hãy cố gắng giữ một tâm trí cởi mở.

Mặc dù các phong cách Go sẽ không và không nên thay đổi một cách *triệt để* do generics được phát hành, nhưng các phong cách này *sẽ* thay đổi - do ngôn ngữ thay đổi! Đây không phải là một quan điểm gây tranh cãi.

Việc nói rằng:

> Điều này không giống phong cách Go

Mà không có thêm bất kỳ chi tiết nào, thì không phải là một điều hữu ích hay có thể thực hiện được. Đặc biệt là khi thảo luận về các tính năng ngôn ngữ mới.

Hãy thảo luận với các đồng nghiệp của bạn về các mẫu hình và phong cách mã nguồn dựa trên các giá trị của chúng thay vì giáo điều. Miễn là bạn có các bản kiểm thử được thiết kế tốt, bạn sẽ luôn có thể tái cấu trúc và thay đổi mọi thứ khi bạn hiểu điều gì hoạt động tốt cho bạn và nhóm của mình.

### Tài liệu tham khảo

Fold là một khái niệm rất cơ bản trong khoa học máy tính. Đây là một số tài liệu thú vị nếu bạn muốn tìm hiểu sâu hơn về nó:
- [Wikipedia: Fold](https://en.wikipedia.org/wiki/Fold)
- [A tutorial on the universality and expressiveness of fold](http://www.cs.nott.ac.uk/~pszgmh/fold.pdf)

