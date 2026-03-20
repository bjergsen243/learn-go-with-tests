# Toán học (Mathematics)

**[Tất cả code của chương này được lưu tại đây](https://github.com/quii/learn-go-with-tests/tree/main/math)**

Dù máy tính hiện đại có sức mạnh thực hiện những phép tính khổng lồ với tốc độ ánh sáng, một nhà phát triển trung bình hiếm khi sử dụng toán học trong công việc. Nhưng hôm nay thì khác! Hôm nay chúng ta sẽ sử dụng toán học để giải quyết một vấn đề *thực tế*. Và không phải là toán học nhàm chán - chúng ta sẽ sử dụng lượng giác, vector và đủ thứ mà bạn từng nói rằng mình sẽ chẳng bao giờ phải dùng sau khi tốt nghiệp trung học.

## Vấn đề

Bạn muốn tạo một file SVG của một chiếc đồng hồ. Không phải đồng hồ kỹ thuật số - không, cái đó thì quá dễ - mà là một chiếc đồng hồ *kim* (analogue), có các kim chỉ giờ. Bạn không cần gì quá cầu kỳ, chỉ cần một hàm nhận vào một `Time` từ package `time` và xuất ra một SVG của đồng hồ với đầy đủ các kim - giờ, phút và giây - chỉ đúng hướng. Điều đó có thể khó đến mức nào?

Đầu tiên, chúng ta cần một SVG của đồng hồ để thử nghiệm. SVG là một định dạng hình ảnh tuyệt vời để thao tác bằng mã nguồn vì chúng được viết dưới dạng một chuỗi các hình khối, được mô tả bằng XML. Chiếc đồng hồ này:

![an svg of a clock](math/example_clock.svg)

được mô tả như sau:

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg"
     width="100%"
     height="100%"
     viewBox="0 0 300 300"
     version="2.0">

  <!-- bezel -->
  <circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/>

  <!-- hour hand -->
  <line x1="150" y1="150" x2="114.150000" y2="132.260000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>

  <!-- minute hand -->
  <line x1="150" y1="150" x2="101.290000" y2="99.730000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>

  <!-- second hand -->
  <line x1="150" y1="150" x2="77.190000" y2="202.900000"
        style="fill:none;stroke:#f00;stroke-width:3px;"/>
</svg>
```

Nó là một vòng tròn với ba đường kẻ, mỗi đường kẻ bắt đầu từ tâm vòng tròn (x=150, y=150) và kết thúc ở một khoảng cách nào đó.

Vậy những gì chúng ta sẽ làm là tái cấu trúc lại nội dung trên, nhưng thay đổi các đường kẻ sao cho chúng chỉ đúng hướng với một thời gian cụ thể.

## Một Acceptance Test

Trước khi đi quá sâu, hãy nghĩ về một acceptance test (kiểm thử chấp nhận).

Đợi đã, có thể bạn chưa biết acceptance test là gì. Hãy để tôi giải thích.

Để tôi hỏi bạn: chiến thắng trông như thế nào? Làm thế nào chúng ta biết mình đã hoàn thành công việc? TDD cung cấp một cách tốt để biết khi nào bạn đã xong: khi bản kiểm thử vượt qua. Đôi khi thật tốt - thực tế là hầu như luôn luôn tốt - khi viết một bản kiểm thử cho bạn biết khi nào bạn đã viết xong toàn bộ tính năng có thể sử dụng được. Không chỉ là một bản kiểm thử cho biết một hàm cụ thể đang hoạt động theo cách bạn mong đợi, mà là một bản kiểm thử cho biết toàn bộ thứ mà bạn đang cố gắng đạt được - 'tính năng' (feature) - đã hoàn thành.

Những bản kiểm thử này đôi khi được gọi là 'acceptance tests', đôi khi được gọi là 'feature tests'. Ý tưởng là bạn viết một bản kiểm thử ở cấp độ rất cao để mô tả những gì bạn đang cố gắng đạt được - ví dụ: một người dùng nhấp vào một nút trên trang web và họ thấy danh sách đầy đủ các Pokémon họ đã bắt được. Khi chúng ta đã viết bản kiểm thử đó, sau đó chúng ta có thể viết thêm các bản kiểm thử khác - unit tests - để xây dựng hướng tới một hệ thống hoạt động có thể vượt qua acceptance test. Vì vậy, đối với ví dụ của chúng ta, các bản kiểm thử này có thể là về việc hiển thị một trang web với một cái nút, kiểm thử các route handlers trên một web server, thực hiện truy vấn cơ sở dữ liệu, v.v. Tất cả những thứ này sẽ được thực hiện theo quy trình TDD, và tất cả chúng sẽ hướng tới việc làm cho bản acceptance test ban đầu vượt qua.

Giống như bức tranh *kinh điển* này của Nat Pryce và Steve Freeman:

![Outside-in feedback loops in TDD](TDD-outside-in.jpg)

Dù sao thì, hãy thử viết acceptance test đó - cái sẽ cho chúng ta biết khi nào chúng ta hoàn thành.

Chúng ta đã có một ví dụ về đồng hồ, vì vậy hãy nghĩ xem các thông số quan trọng sẽ là gì.

```xml
<line x1="150" y1="150" x2="114.150000" y2="132.260000"
        style="fill:none;stroke:#000;stroke-width:7px;"/>
```

Tâm của đồng hồ (các thuộc tính `x1` và `y1` cho đường kẻ này) là giống nhau cho mỗi kim đồng hồ. Các giá trị cần thay đổi cho mỗi kim - các tham số cho bất kỳ thứ gì xây dựng SVG - là các thuộc tính `x2` và `y2`. Chúng ta sẽ cần một X và một Y cho mỗi kim đồng hồ.

Tôi *có thể* nghĩ về nhiều tham số hơn - bán kính của mặt đồng hồ tròn, kích thước của SVG, màu sắc của các kim, hình dạng của chúng, v.v... nhưng tốt hơn là bắt đầu bằng cách giải quyết một vấn đề đơn giản, cụ thể với một giải pháp đơn giản, cụ thể, và sau đó bắt đầu thêm các tham số để làm cho nó trở nên tổng quát hơn.

Vì vậy, chúng ta sẽ thống nhất rằng:
- mỗi đồng hồ có tâm là (150, 150)
- kim giờ dài 50
- kim phút dài 80
- kim giây dài 90.

Một điều cần lưu ý về SVG: điểm gốc - điểm (0,0) - nằm ở góc *trên cùng bên trái*, không phải *dưới cùng bên trái* như chúng ta thường mong đợi. Việc ghi nhớ điều này rất quan trọng khi chúng ta tính toán các giá trị để đưa vào các đường kẻ của mình.

Cuối cùng, tôi chưa quyết định *cách* xây dựng SVG - chúng ta có thể sử dụng một template từ package [`text/template`][texttemplate], hoặc chúng ta có thể chỉ gửi các byte vào một `bytes.Buffer` hoặc một writer. Nhưng chúng ta biết mình sẽ cần những con số đó, vì vậy hãy tập trung vào việc kiểm thử thứ gì đó tạo ra chúng.

### Viết test trước tiên

Bản kiểm thử đầu tiên của tôi trông như thế này:

```go
package clockface_test

import (
	"projectpath/clockface"
	"testing"
	"time"
)

func TestSecondHandAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)

	want := clockface.Point{X: 150, Y: 150 - 90}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

Hãy nhớ cách SVG vẽ các tọa độ từ góc trên bên trái không? Để đặt kim giây ở vị trí nửa đêm, chúng ta mong đợi nó không di chuyển khỏi tâm đồng hồ trên trục X - vẫn là 150 - và trục Y là độ dài của kim 'lên trên' từ tâm; 150 trừ đi 90.

### Thử chạy test

Điều này đẩy ra các lỗi mong đợi xoay quanh các hàm và kiểu còn thiếu:

```
--- FAIL: TestSecondHandAtMidnight (0.00s)
./clockface_test.go:13:10: undefined: clockface.Point
./clockface_test.go:14:9: undefined: clockface.SecondHand
```

Vậy là cần một `Point` nơi đầu kim giây sẽ chỉ tới, và một hàm để lấy nó.

### Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Hãy triển khai các kiểu dữ liệu đó để mã nguồn có thể biên dịch:

```go
package clockface

import "time"

// A Point đại diện cho một tọa độ Descartes hai chiều
type Point struct {
	X float64
	Y float64
}

// SecondHand là vector đơn vị của kim giây của đồng hồ kim tại thời điểm `t`
// được đại diện dưới dạng một Point.
func SecondHand(t time.Time) Point {
	return Point{}
}
```

và bây giờ chúng ta nhận được:

```
--- FAIL: TestSecondHandAtMidnight (0.00s)
    clockface_test.go:17: Got {0 0}, wanted {150 60}
FAIL
exit status 1
FAIL	learn-go-with-tests/math/clockface	0.006s
```

### Viết đủ code để test chạy thành công

Khi nhận được lỗi mong đợi, chúng ta có thể điền giá trị trả về của `SecondHand`:

```go
// SecondHand là vector đơn vị của kim giây của đồng hồ kim tại thời điểm `t`
// được đại diện dưới dạng một Point.
func SecondHand(t time.Time) Point {
	return Point{150, 60}
}
```

Và đây là kết quả, test đã vượt qua.

```
PASS
ok  	    clockface	0.006s
```

### Refactor

Chưa cần phải tái cấu trúc - hầu như chưa có mã nguồn nào đáng kể!

### Lặp lại cho các yêu cầu mới

Chúng ta chắc chắn cần phải làm thêm một số việc ở đây chứ không chỉ trả về một chiếc đồng hồ luôn hiển thị nửa đêm cho mọi thời điểm...

### Viết test trước tiên

```go
func TestSecondHandAt30Seconds(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 30, 0, time.UTC)

	want := clockface.Point{X: 150, Y: 150 + 90}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

Cùng một ý tưởng, nhưng bây giờ kim giây đang chỉ *xuống dưới* nên chúng ta *cộng* thêm độ dài vào trục Y.

Điều này sẽ biên dịch được... nhưng làm cách nào để làm cho nó vượt qua?

## Thời gian suy ngẫm

Chúng ta sẽ giải quyết vấn đề này như thế nào?

Mỗi phút, kim giây đi qua cùng 60 trạng thái, chỉ theo 60 hướng khác nhau. Khi là 0 giây, nó chỉ lên trên cùng của mặt đồng hồ, khi là 30 giây, nó chỉ xuống dưới cùng mặt đồng hồ. Thật dễ dàng.

Vì vậy, nếu tôi muốn nghĩ về hướng mà kim giây đang chỉ vào lúc, chẳng hạn như 37 giây, tôi sẽ muốn góc giữa 12 giờ và 37/60 vòng tròn. Tính theo độ thì đây là `(360 / 60 ) * 37 = 222`, nhưng dễ hơn là chỉ cần nhớ rằng nó là `37/60` của một vòng quay hoàn chỉnh.

Nhưng góc chỉ là một nửa câu chuyện; chúng ta cần biết tọa độ X và Y mà đầu kim giây đang chỉ tới. Làm thế nào chúng ta có thể tính toán điều đó?

## Toán học

Hãy tưởng tượng một vòng tròn có bán kính bằng 1 được vẽ quanh gốc tọa độ `0, 0`.

![picture of the unit circle](math/images/unit_circle.png)

Đây được gọi là 'vòng tròn đơn vị' vì... chà, bán kính của nó là 1 đơn vị!

Chu vi của vòng tròn được tạo thành từ các điểm trên lưới - chính là các tọa độ. Các thành phần x và y của mỗi tọa độ này tạo thành một tam giác, trong đó cạnh huyền luôn bằng 1 (tức là bán kính của vòng tròn).

![picture of the unit circle with a point defined on the circumference](math/images/unit_circle_coords.png)

Bây giờ, lượng giác sẽ cho phép chúng ta tính toán độ dài của X và Y cho mỗi tam giác nếu chúng ta biết góc mà chúng tạo với gốc tọa độ. Tọa độ X sẽ là cos(a) và tọa độ Y sẽ là sin(a), với a là góc được tạo giữa đường thẳng và trục x (dương).

![picture of the unit circle with the x and y elements of a ray defined as cos(a) and sin(a) respectively, where a is the angle made by the ray with the x axis](math/images/unit_circle_params.png)

(Nếu bạn không tin điều này, [hãy xem Wikipedia...][circle])

Một điểm mấu chốt cuối cùng - vì chúng ta muốn đo góc từ hướng 12 giờ thay vì từ trục X (hướng 3 giờ), chúng ta cần tráo đổi các trục; bây giờ x = sin(a) và y = cos(a).

![unit circle ray defined from by angle from y axis](math/images/unit_circle_12_oclock.png)

Vậy bây giờ chúng ta đã biết cách lấy góc của kim giây (1/60 vòng tròn cho mỗi giây) và các tọa độ X và Y. Chúng ta sẽ cần các hàm cho cả `sin` và `cos`.

## `math`

May mắn thay, package `math` của Go có cả hai hàm này, nhưng có một vướng mắc nhỏ chúng ta cần lưu ý; nếu chúng ta nhìn vào mô tả của [`math.Cos`][mathcos]:

> Cos trả về cosine của đối số radian x.

Nó muốn góc phải theo đơn vị radian. Vậy radian là gì? Thay vì định nghĩa một vòng quay hoàn chỉnh của vòng tròn được tạo thành từ 360 độ, chúng ta định nghĩa một vòng quay hoàn chỉnh là 2π radian. Có những lý do chính đáng để làm điều này mà chúng ta sẽ không đi sâu vào.[^2]

Bây giờ chúng ta đã đọc xong, học xong và suy nghĩ xong, chúng ta có thể viết bản kiểm thử tiếp theo.

### Viết test trước tiên

Tất cả những kiến thức toán học này thật khó và gây bối rối. Tôi không tự tin rằng mình hiểu chuyện gì đang xảy ra - vì vậy hãy viết một bản kiểm thử! Chúng ta không cần giải quyết toàn bộ vấn đề trong một lần - hãy bắt đầu bằng việc tính toán góc chính xác, theo radian, của kim giây tại một thời điểm cụ thể.

Tôi sẽ *tạm đóng* (comment out) bản acceptance test mà mình đang làm trong khi thực hiện các bản kiểm thử này - tôi không muốn bị phân tâm bởi bản kiểm thử đó trong khi đang cố làm cho bản này vượt qua.

### Nhắc lại về các package

Hiện tại, các acceptance tests của chúng ta nằm trong package `clockface_test`. Các bản kiểm thử của chúng ta có thể nằm ngoài package `clockface` - miễn là tên của chúng kết thúc bằng `_test.go` thì chúng có thể được chạy.

Tôi sẽ viết các bản kiểm thử radian này *bên trong* package `clockface`; chúng có thể không bao giờ được xuất (export), và chúng có thể bị xóa (hoặc di dời) khi tôi đã hiểu rõ hơn về những gì đang xảy ra. Tôi sẽ đổi tên file acceptance test thành `clockface_acceptance_test.go`, để tôi có thể tạo một file *mới* gọi là `clockface_test` để kiểm thử kim giây theo radian.

```go
package clockface

import (
	"math"
	"testing"
	"time"
)

func TestSecondsInRadians(t *testing.T) {
	thirtySeconds := time.Date(312, time.October, 28, 0, 0, 30, 0, time.UTC)
	want := math.Pi
	got := secondsInRadians(thirtySeconds)

	if want != got {
		t.Fatalf("Wanted %v radians, but got %v", want, got)
	}
}
```

Ở đây chúng ta đang kiểm thử rằng 30 giây rưỡi nên đặt kim giây ở vị trí nửa vòng tròn. Và đây là lần đầu tiên chúng ta sử dụng package `math`! Nếu một vòng tròn hoàn chỉnh là 2π radian, chúng ta biết rằng nửa vòng tròn nên là π radian. `math.Pi` cung cấp cho chúng ta giá trị của π.

### Thử chạy test

```
./clockface_test.go:12:9: undefined: secondsInRadians
```

### Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func secondsInRadians(t time.Time) float64 {
	return 0
}
```

```
clockface_test.go:15: Wanted 3.141592653589793 radians, but got 0
```

### Viết đủ code để test chạy thành công

```go
func secondsInRadians(t time.Time) float64 {
	return math.Pi
}
```

```
PASS
ok  	clockface	0.011s
```

### Refactor

Chưa cần tái cấu trúc gì cả.

### Lặp lại cho các yêu cầu mới

Bây giờ chúng ta có thể mở rộng cuộc kiểm thử để bao gồm một vài kịch bản khác. Tôi sẽ đi nhanh một chút và trình bày một số mã kiểm thử đã được tái cấu trúc - hy vọng bạn có thể hiểu rõ cách tôi đạt được kết quả này.

```go
func TestSecondsInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(0, 0, 30), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(0, 0, 45), (math.Pi / 2) * 3},
		{simpleTime(0, 0, 7), (math.Pi / 30) * 7},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondsInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

Tôi đã thêm một vài hàm trợ giúp (helper functions) để việc viết bản kiểm thử dựa trên bảng này bớt nhàm chán hơn. `testName` chuyển đổi thời gian sang định dạng đồng hồ số (HH:MM:SS), và `simpleTime` tạo một `time.Time` chỉ sử dụng các phần mà chúng ta thực sự quan tâm (giờ, phút và giây).[^1] Đây là chúng:

```go
func simpleTime(hours, minutes, seconds int) time.Time {
	return time.Date(312, time.October, 28, hours, minutes, seconds, 0, time.UTC)
}

func testName(t time.Time) string {
	return t.Format("15:04:05")
}
```

Hai hàm này sẽ giúp việc viết các bản kiểm thử này (và các bản kiểm thử trong tương lai) dễ dàng hơn một chút để viết và bảo trì.

Điều này cho chúng ta một số output kiểm thử khá đẹp:

```
clockface_test.go:24: Wanted 0 radians, but got 3.141592653589793

clockface_test.go:24: Wanted 4.71238898038469 radians, but got 3.141592653589793
```

Đã đến lúc triển khai tất cả đống toán học mà chúng ta đã thảo luận ở trên:

```go
func secondsInRadians(t time.Time) float64 {
	return float64(t.Second()) * (math.Pi / 30)
}
```

Một giây là (2π / 60) radian... triệt tiêu con số 2 và chúng ta có π/30 radian. Nhân con số đó với số giây (dưới dạng `float64`) và bây giờ tất cả các bản kiểm thử sẽ vượt qua...

```
clockface_test.go:24: Wanted 3.141592653589793 radians, but got 3.1415926535897936
```

Đợi đã, cái gì thế này?

### Float thật kinh khủng

Phép toán dấu phẩy động (floating point arithmetic) [vốn nổi tiếng là không chính xác][floatingpoint]. Máy tính chỉ có thể thực sự xử lý các số nguyên, và ở một mức độ nào đó là các số hữu tỷ. Các số thập phân bắt đầu trở nên không chính xác, đặc biệt là khi chúng ta nhân chia chúng như trong hàm `secondsInRadians`. Bằng cách chia `math.Pi` cho 30 và sau đó nhân nó với 30, chúng ta đã kết thúc với *một con số không còn giống như `math.Pi` nữa*.

Có hai cách để giải quyết vấn đề này:

1. Sống chung với nó
2. Tái cấu trúc hàm bằng cách cấu trúc lại phương trình

Bây giờ phương án (1) nghe có vẻ không hấp dẫn lắm, nhưng nó thường là cách duy nhất để làm cho so sánh bằng của dấu phẩy động hoạt động. Việc không chính xác một chút xíu thực tế sẽ chẳng quan trọng cho mục đích vẽ một mặt đồng hồ, vì vậy chúng ta có thể viết một hàm định nghĩa thế nào là bằng nhau "đủ gần" (close enough) cho các góc của mình. Nhưng có một cách đơn giản để chúng ta lấy lại độ chính xác: chúng ta sắp xếp lại phương trình để không còn phải thực hiện phép chia xuống rồi nhân lên nữa. Chúng ta có thể thực hiện tất cả chỉ bằng phép chia.

Vì vậy thay vì

	numberOfSeconds * π / 30

chúng ta có thể viết

	π / (30 / numberOfSeconds)

cái mà tương đương.

Trong Go:

```go
func secondsInRadians(t time.Time) float64 {
	return (math.Pi / (30 / (float64(t.Second()))))
}
```

Và chúng ta có kết quả vượt qua.

```
PASS
ok      clockface     0.005s
```

Nó sẽ trông [giống như thế này](https://github.com/quii/learn-go-with-tests/tree/main/math/v3/clockface).

### Một lưu ý về phép chia cho số không

Máy tính thường không thích chia cho số không vì vô cực là một thứ gì đó hơi lạ lẫm.

Trong Go, nếu bạn cố gắng chia một cách rõ ràng cho số không, bạn sẽ nhận được lỗi biên dịch.

```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println(10.0 / 0.0) // không thể biên dịch
}
```

Rõ ràng là trình biên dịch không thể luôn dự đoán được bạn sẽ chia cho số không, chẳng hạn như `t.Second()` của chúng ta.

Hãy thử cái này:

```go
func main() {
	fmt.Println(10.0 / zero())
}

func zero() float64 {
	return 0.0
}
```

Nó sẽ in ra `+Inf` (vô cực dương). Việc chia cho +Inf dường như dẫn đến kết quả là không và chúng ta có thể thấy điều này qua ví dụ sau:

```go
package main

import (
	"fmt"
	"math"
)

func main() {
	fmt.Println(secondsinradians())
}

func zero() float64 {
	return 0.0
}

func secondsinradians() float64 {
	return (math.Pi / (30 / (float64(zero()))))
}
```

### Lặp lại cho các yêu cầu mới

Vậy chúng ta đã hoàn thành phần đầu tiên ở đây - chúng ta biết góc mà kim giây sẽ chỉ tới theo radian. Bây giờ chúng ta cần tính toán các tọa độ.

Lại một lần nữa, hãy làm cho mọi thứ đơn giản nhất có thể và chỉ làm việc với *vòng tròn đơn vị*; vòng tròn có bán kính là 1. Điều này có nghĩa là các kim đồng hồ của chúng ta đều có độ dài là 1 nhưng bù lại, toán học sẽ trở nên dễ dàng hơn cho chúng ta.

### Viết test trước tiên

```go
func TestSecondHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 0, 30), Point{0, -1}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondHandPoint(c.time)
			if got != c.point {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

### Thử chạy test

```
./clockface_test.go:40:11: undefined: secondHandPoint
```

### Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func secondHandPoint(t time.Time) Point {
	return Point{}
}
```

```
clockface_test.go:42: Wanted {0 -1} Point, but got {0 0}
```

### Viết đủ code để test chạy thành công

```go
func secondHandPoint(t time.Time) Point {
	return Point{0, -1}
}
```

```
PASS
ok  	clockface	0.007s
```

### Lặp lại cho các yêu cầu mới

```go
func TestSecondHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 0, 30), Point{0, -1}},
		{simpleTime(0, 0, 45), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondHandPoint(c.time)
			if got != c.point {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

### Thử chạy test

```
clockface_test.go:43: Wanted {-1 0} Point, but got {0 -1}
```

### Viết đủ code để test chạy thành công

Còn nhớ hình ảnh vòng tròn đơn vị của chúng ta chứ?

![picture of the unit circle with the x and y elements of a ray defined as cos(a) and sin(a) respectively, where a is the angle made by the ray with the x axis](math/images/unit_circle_params.png)

Cũng hãy nhớ rằng chúng ta muốn đo góc từ hướng 12 giờ (vốn là trục Y) thay vì đo từ trục X (hướng 3 giờ).

![unit circle ray defined from by angle from y axis](math/images/unit_circle_12_oclock.png)

Bây giờ chúng ta cần phương trình tạo ra X và Y. Hãy viết nó vào trong mã nguồn:

```go
func secondHandPoint(t time.Time) Point {
	angle := secondsInRadians(t)
	x := math.Sin(angle)
	y := math.Cos(angle)

	return Point{x, y}
}
```

Bây giờ chúng ta nhận được:

```
clockface_test.go:43: Wanted {0 -1} Point, but got {1.2246467991473515e-16 -1}

clockface_test.go:43: Wanted {-1 0} Point, but got {-1 -1.8369701987210272e-16}
```

Đợi đã, lại nữa sao? Có vẻ như chúng ta lại bị nguyền rủa bởi các số dấu phẩy động một lần nữa - cả hai con số không mong đợi đó đều cực kỳ nhỏ (infinitesimal) - tận vị trí thập phân thứ 16. Vì vậy, một lần nữa chúng ta có thể chọn tăng độ chính xác, hoặc chúng ta chấp nhận chúng xấp xỉ bằng nhau và tiếp tục cuộc sống.

Một tùy chọn để tăng độ chính xác của các góc này là sử dụng kiểu số hữu tỷ `Rat` từ package `math/big`. Nhưng với mục tiêu là vẽ một file SVG chứ không phải đáp phi thuyền xuống mặt trăng, tôi nghĩ chúng ta có thể chung sống với một chút sai số.

```go
func TestSecondHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 0, 30), Point{0, -1}},
		{simpleTime(0, 0, 45), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := secondHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}

func roughlyEqualFloat64(a, b float64) bool {
	const equalityThreshold = 1e-7
	return math.Abs(a-b) < equalityThreshold
}

func roughlyEqualPoint(a, b Point) bool {
	return roughlyEqualFloat64(a.X, b.X) &&
		roughlyEqualFloat64(a.Y, b.Y)
}
```

Chúng ta đã định nghĩa hai hàm để so sánh sự bằng nhau tương đối giữa hai `Points` - chúng sẽ hoạt động nếu các thành phần X và Y nằm trong khoảng sai số 0.0000001 với nhau. Độ chính xác đó vẫn là khá cao.

Và bây giờ test sẽ vượt qua:

```
PASS
ok  	clockface	0.007s
```

### Refactor

Tôi vẫn khá hài lòng với điều này.

Đầy là [hình dạng mã nguồn lúc này](https://github.com/quii/learn-go-with-tests/tree/main/math/v4/clockface).

### Lặp lại cho các yêu cầu mới

Chà, gọi là *mới* thì không hẳn chính xác - thực sự những gì chúng ta có thể làm bây giờ là làm cho bản acceptance test vượt qua! Hãy cùng nhắc lại nó trông như thế nào:

```go
func TestSecondHandAt30Seconds(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 30, 0, time.UTC)

	want := clockface.Point{X: 150, Y: 150 + 90}
	got := clockface.SecondHand(tm)

	if got != want {
		t.Errorf("Got %v, wanted %v", got, want)
	}
}
```

### Thử chạy test

```
clockface_acceptance_test.go:28: Got {150 60}, wanted {150 240}
```

### Viết đủ code để test chạy thành công

Chúng ta cần làm ba việc để chuyển đổi vector đơn vị của mình thành một điểm trên SVG:

1. Phóng to nó theo độ dài của kim
2. Lật ngược nó qua trục X để phù hợp với việc SVG có gốc tọa độ ở góc trên cùng bên trái
3. Dịch chuyển nó đến đúng vị trí (để nó bắt đầu từ tâm (150,150))

Thời gian vui vẻ đây rồi!

```go
// SecondHand là vector đơn vị của kim giây của đồng hồ kim tại thời điểm `t`
// được đại diện dưới dạng một Point.
func SecondHand(t time.Time) Point {
	p := secondHandPoint(t)
	p = Point{p.X * 90, p.Y * 90}   // scale
	p = Point{p.X, -p.Y}            // flip
	p = Point{p.X + 150, p.Y + 150} // translate
	return p
}
```

Phóng to, lật ngược và dịch chuyển đúng theo thứ tự đó. Hoan hô toán học!

```
PASS
ok  	clockface	0.007s
```

### Refactor

Có một vài con số "ma thuật" (magic numbers) ở đây nên được tách ra làm hằng số, vì vậy hãy làm điều đó:

```go
const secondHandLength = 90
const clockCentreX = 150
const clockCentreY = 150

// SecondHand là vector đơn vị của kim giây của đồng hồ kim tại thời điểm `t`
// được đại diện dưới dạng một Point.
func SecondHand(t time.Time) Point {
	p := secondHandPoint(t)
	p = Point{p.X * secondHandLength, p.Y * secondHandLength}
	p = Point{p.X, -p.Y}
	p = Point{p.X + clockCentreX, p.Y + clockCentreY} //translate
	return p
}
```

## Vẽ đồng hồ

Chà... ít nhất là kim giây đã xong...

Hãy cùng thực hiện điều này - vì không có gì tệ hơn việc không mang lại giá trị nào khi nó đang chờ đợi để ra mắt thế giới và làm mọi người kinh ngạc. Hãy vẽ một chiếc kim giây!

Chúng ta sẽ tạo một thư mục mới dưới package `clockface` chính, được gọi là (có chút gây bối rối) `clockface`. Ở đó chúng ta sẽ đặt package `main` để tạo ra file nhị phân xây dựng SVG:

```
|-- clockface
|       |-- main.go
|-- clockface.go
|-- clockface_acceptance_test.go
|-- clockface_test.go
```

Bên trong `main.go`, bạn sẽ bắt đầu với mã này nhưng thay đổi import cho package clockface để trỏ đến phiên bản của chính bạn:

```go
package main

import (
	"fmt"
	"io"
	"os"
	"time"

	"learn-go-with-tests/math/clockface" // THAY THẾ CHỖ NÀY!
)

func main() {
	t := time.Now()
	sh := clockface.SecondHand(t)
	io.WriteString(os.Stdout, svgStart)
	io.WriteString(os.Stdout, bezel)
	io.WriteString(os.Stdout, secondHandTag(sh))
	io.WriteString(os.Stdout, svgEnd)
}

func secondHandTag(p clockface.Point) string {
	return fmt.Sprintf(`<line x1="150" y1="150" x2="%f" y2="%f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
}

const svgStart = `<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg"
     width="100%"
     height="100%"
     viewBox="0 0 300 300"
     version="2.0">`

const bezel = `<circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/>`

const svgEnd = `</svg>`
```

Trời ạ, tôi không hề cố gắng giành bất kỳ giải thưởng nào cho mã nguồn đẹp đẽ với mớ hỗn độn *này* - nhưng nó hoàn thành công việc. Nó đang ghi một SVG ra `os.Stdout` - từng chuỗi một.

Nếu chúng ta build nó:

```
go build
```

và chạy nó, gửi output vào một file:

```
./clockface > clock.svg
```

Chúng ta sẽ thấy một cái gì đó như:

![a clock with only a second hand](math/v6/clockface/clockface/clock.svg)

Và đây là [mã nguồn trông như thế nào](https://github.com/quii/learn-go-with-tests/tree/main/math/v6/clockface).

### Refactor

Mã này bốc mùi. Chà, nó không hẳn là *quá tệ*, nhưng tôi không hài lòng về nó:

1. Toàn bộ hàm `SecondHand` này gắn chặt với việc tạo ra SVG... mà không hề đề cập đến SVG hay thực sự tạo ra một SVG nào...
2. ... đồng thời tôi cũng không kiểm thử bất kỳ mã SVG nào của mình.

Vâng, tôi đoán mình đã phá hỏng thứ gì đó. Cảm giác này thật sai trái. Hãy thử khắc phục bằng một bản kiểm thử tập trung vào SVG hơn.

Chúng ta có những lựa chọn nào? Chà, chúng ta có thể thử kiểm thử xem các ký tự xuất ra từ `SVGWriter` có chứa những thứ trông giống như thẻ SVG mà chúng ta mong đợi cho một thời điểm cụ thể hay không. Ví dụ:

```go
func TestSVGWriterAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)

	var b strings.Builder
	clockface.SVGWriter(&b, tm)
	got := b.String()

	want := `<line x1="150" y1="150" x2="150" y2="60"`

	if !strings.Contains(got, want) {
		t.Errorf("Expected to find the second hand %v, in the SVG output %v", want, got)
	}
}
```

Nhưng liệu đây có thực sự là một sự cải tiến?

Nó không chỉ vẫn vượt qua nếu tôi không tạo ra một SVG hợp lệ (vì nó chỉ kiểm thử xem một chuỗi có xuất hiện trong output hay không), mà nó còn thất bại nếu tôi thực hiện một thay đổi nhỏ nhất, không quan trọng đối với chuỗi đó - ví dụ: nếu tôi thêm một khoảng trắng dư thừa giữa các thuộc tính.

"Mùi" (smell) lớn nhất ở đây là tôi đang kiểm thử một cấu trúc dữ liệu - XML - bằng cách nhìn vào biểu diễn của nó dưới dạng một chuỗi ký tự. Đây *không bao giờ* là một ý tưởng hay vì nó tạo ra những vấn đề giống như tôi đã nêu ở trên: một bản kiểm thử vừa quá mong manh vừa không đủ nhạy bén. Một bản kiểm thử đang kiểm thử sai thứ!

Vì vậy, giải pháp duy nhất là kiểm thử output *dưới dạng XML*. Và để làm được điều đó, chúng ta cần phân tích (parse) nó.

## Phân tích XML

[`encoding/xml`][xml] là package của Go có thể xử lý tất cả những thứ liên quan đến phân tích XML đơn giản.

Hàm [`xml.Unmarshal`](https://pkg.go.dev/encoding/xml#Unmarshal) nhận vào một `[]byte` dữ liệu XML, và một con trỏ tới một struct để nó được giải mã (unmarshal) vào đó.

Vì vậy, chúng ta sẽ cần một struct để giải mã XML của mình vào. Chúng ta có thể dành thời gian để tìm ra tên chính xác cho tất cả các node và thuộc tính, và cách viết cấu trúc chính xác, nhưng may mắn thay, ai đó đã viết [`zek`](https://github.com/miku/zek) - một chương trình sẽ tự động hóa tất cả công việc nặng nhọc đó cho chúng ta. Thậm chí còn tuyệt hơn, có một phiên bản trực tuyến tại [https://xml-to-go.github.io/](https://xml-to-go.github.io/). Chỉ cần dán nội dung SVG từ đầu file vào một ô và - bùm - kết quả hiện ra:

```go
type Svg struct {
	XMLName xml.Name `xml:"svg"`
	Text    string   `xml:",chardata"`
	Xmlns   string   `xml:"xmlns,attr"`
	Width   string   `xml:"width,attr"`
	Height  string   `xml:"height,attr"`
	ViewBox string   `xml:"viewBox,attr"`
	Version string   `xml:"version,attr"`
	Circle  struct {
		Text  string `xml:",chardata"`
		Cx    string `xml:"cx,attr"`
		Cy    string `xml:"cy,attr"`
		R     string `xml:"r,attr"`
		Style string `xml:"style,attr"`
	} `xml:"circle"`
	Line []struct {
		Text  string `xml:",chardata"`
		X1    string `xml:"x1,attr"`
		Y1    string `xml:"y1,attr"`
		X2    string `xml:"x2,attr"`
		Y2    string `xml:"y2,attr"`
		Style string `xml:"style,attr"`
	} `xml:"line"`
}
```

Chúng ta có thể thực hiện các điều chỉnh đối với struct này nếu cần (như thay đổi tên struct thành `SVG`) nhưng nó chắc chắn đủ tốt để bắt đầu. Hãy dán struct này vào file `clockface_acceptance_test` và viết một bản kiểm thử với nó:

```go
func TestSVGWriterAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)

	b := bytes.Buffer{}
	clockface.SVGWriter(&b, tm)

	svg := Svg{}
	xml.Unmarshal(b.Bytes(), &svg)

	x2 := "150"
	y2 := "60"

	for _, line := range svg.Line {
		if line.X2 == x2 && line.Y2 == y2 {
			return
		}
	}

	t.Errorf("Expected to find the second hand with x2 of %+v and y2 of %+v, in the SVG output %v", x2, y2, b.String())
}
```

Chúng ta ghi output của `clockface.SVGWriter` vào một `bytes.Buffer` và sau đó `Unmarshal` nó vào một `Svg`. Tiếp theo, chúng ta xem xét từng `Line` trong `Svg` để xem liệu có bất kỳ dòng nào có giá trị `X2` và `Y2` như mong đợi hay không. Nếu tìm thấy sự trùng khớp, chúng ta thoát sớm (vượt qua bản kiểm thử); nếu không, chúng ta thất bại với một thông báo (hy vọng là) đầy đủ thông tin.

```sh
./clockface_acceptance_test.go:41:2: undefined: clockface.SVGWriter
```

Có vẻ như tốt hơn là chúng ta nên tạo `SVGWriter.go`...

```go
package clockface

import (
	"fmt"
	"io"
	"time"
)

const (
	secondHandLength = 90
	clockCentreX     = 150
	clockCentreY     = 150
)

// SVGWriter ghi lại biểu diễn SVG của một đồng hồ kim, hiển thị thời gian t, vào writer w
func SVGWriter(w io.Writer, t time.Time) {
	io.WriteString(w, svgStart)
	io.WriteString(w, bezel)
	secondHand(w, t)
	io.WriteString(w, svgEnd)
}

func secondHand(w io.Writer, t time.Time) {
	p := secondHandPoint(t)
	p = Point{p.X * secondHandLength, p.Y * secondHandLength} // scale
	p = Point{p.X, -p.Y}                                      // flip
	p = Point{p.X + clockCentreX, p.Y + clockCentreY}         // translate
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
}

const svgStart = `<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
<svg xmlns="http://www.w3.org/2000/svg"
     width="100%"
     height="100%"
     viewBox="0 0 300 300"
     version="2.0">`

const bezel = `<circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/>`

const svgEnd = `</svg>`
```

Một trình ghi SVG đẹp nhất ư? Không hẳn. Nhưng hy vọng nó sẽ hoàn thành công việc...

```
clockface_acceptance_test.go:56: Expected to find the second hand with x2 of 150 and y2 of 60, in the SVG output <?xml version="1.0" encoding="UTF-8" standalone="no"?>
    <!DOCTYPE svg PUBLIC "-//W3C//DTD SVG 1.1//EN" "http://www.w3.org/Graphics/SVG/1.1/DTD/svg11.dtd">
    <svg xmlns="http://www.w3.org/2000/svg"
         width="100%"
         height="100%"
         viewBox="0 0 300 300"
         version="2.0"><circle cx="150" cy="150" r="100" style="fill:#fff;stroke:#000;stroke-width:5px;"/><line x1="150" y1="150" x2="150.000000" y2="60.000000" style="fill:none;stroke:#f00;stroke-width:3px;"/></svg>
```

Ối! Chỉ thị định dạng `%f` đang in tọa độ của chúng ta với mức độ chính xác mặc định - sáu chữ số thập phân. Chúng ta nên nêu rõ mức độ chính xác mà mình mong đợi cho các tọa độ. Hãy giả định là ba chữ số thập phân.

```go
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
```

Và sau khi chúng ta cập nhật các mong đợi trong bản kiểm thử:

```go
	x2 := "150.000"
	y2 := "60.000"
```

Chúng ta nhận được:

```
PASS
ok  	clockface	0.006s
```

Bây giờ chúng ta có thể rút ngắn hàm `main` của mình:

```go
package main

import (
	"os"
	"time"

	"learn-go-with-tests/math/clockface"
)

func main() {
	t := time.Now()
	clockface.SVGWriter(os.Stdout, t)
}
```

Đây là [những gì mọi thứ trông như bây giờ](https://github.com/quii/learn-go-with-tests/tree/main/math/v7b/clockface).

Và chúng ta có thể viết một bản kiểm thử cho một thời điểm khác theo cùng một mẫu, nhưng chưa phải bây giờ...

### Refactor

Có ba điều cần chú ý:

1. Chúng ta chưa thực sự kiểm thử tất cả thông tin cần thiết để đảm bảo nó hiện diện - ví dụ, thế còn các giá trị `x1` thì sao?
2. Ngoài ra, các thuộc tính cho `x1`, v.v. đâu thực sự là `strings` đúng không? Chúng là những con số!
3. Tôi có thực sự quan tâm đến `style` của kim đồng hồ không? Hay, tương tự, cái node `Text` trống rỗng được tạo ra bởi `zek`?

Chúng ta có thể làm tốt hơn. Hãy thực hiện một vài điều chỉnh đối với struct `Svg`, và các bản kiểm thử, để làm cho mọi thứ sắc nét hơn.

```go
type SVG struct {
	XMLName xml.Name `xml:"svg"`
	Xmlns   string   `xml:"xmlns,attr"`
	Width   string   `xml:"width,attr"`
	Height  string   `xml:"height,attr"`
	ViewBox string   `xml:"viewBox,attr"`
	Version string   `xml:"version,attr"`
	Circle  Circle   `xml:"circle"`
	Line    []Line   `xml:"line"`
}

type Circle struct {
	Cx float64 `xml:"cx,attr"`
	Cy float64 `xml:"cy,attr"`
	R  float64 `xml:"r,attr"`
}

type Line struct {
	X1 float64 `xml:"x1,attr"`
	Y1 float64 `xml:"y1,attr"`
	X2 float64 `xml:"x2,attr"`
	Y2 float64 `xml:"y2,attr"`
}
```

Ở đây tôi đã:

- Làm cho các phần quan trọng của struct trở thành các kiểu dữ liệu có tên -- `Line` và `Circle`.
- Chuyển các thuộc tính dạng số thành `float64` thay vì `string`.
- Xóa các thuộc tính không dùng đến như `Style` và `Text`.
- Đổi tên `Svg` thành `SVG` bởi vì *đó là điều đúng đắn nên làm*.

Điều này cho phép chúng ta khẳng định chính xác hơn trên dòng kẻ mà mình đang tìm kiếm:

```go
func TestSVGWriterAtMidnight(t *testing.T) {
	tm := time.Date(1337, time.January, 1, 0, 0, 0, 0, time.UTC)
	b := bytes.Buffer{}

	clockface.SVGWriter(&b, tm)

	svg := SVG{}

	xml.Unmarshal(b.Bytes(), &svg)

	want := Line{150, 150, 150, 60}

	for _, line := range svg.Line {
		if line == want {
			return
		}
	}

	t.Errorf("Expected to find the second hand line %+v, in the SVG lines %+v", want, svg.Line)
}
```

Cuối cùng, chúng ta có thể học hỏi từ các bảng của unit test, và chúng ta có thể viết một hàm trợ giúp `containsLine(line Line, lines []Line) bool` để thực sự làm cho các bản kiểm thử này tỏa sáng:

```go
func TestSVGWriterSecondHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(0, 0, 0),
			Line{150, 150, 150, 60},
		},
		{
			simpleTime(0, 0, 30),
			Line{150, 150, 150, 240},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the second hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}

func containsLine(l Line, ls []Line) bool {
	for _, line := range ls {
		if line == l {
			return true
		}
	}
	return false
}
```

Đây là [những gì nó trông như thế nào](https://github.com/quii/learn-go-with-tests/tree/main/math/v7c/clockface).

Đó mới là cái tôi gọi là một acceptance test!

### Viết test trước tiên

Vậy là kim giây đã xong. Bây giờ hãy cùng bắt đầu với kim phút.

```go
func TestSVGWriterMinuteHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(0, 0, 0),
			Line{150, 150, 150, 70},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the minute hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}
```

### Thử chạy test

```
clockface_acceptance_test.go:87: Expected to find the minute hand line {X1:150 Y1:150 X2:150 Y2:70}, in the SVG lines [{X1:150 Y1:150 X2:150 Y2:60}]
```

Chúng ta nên bắt đầu xây dựng các kim đồng hồ khác, tương tự như cách chúng ta đã tạo ra các bản kiểm thử cho kim giây, chúng ta có thể lặp lại để tạo ra tập hợp các bản kiểm thử sau. Một lần nữa chúng ta sẽ tạm đóng acceptance test của mình trong khi làm cho nó hoạt động:

```go
func TestMinutesInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(0, 30, 0), math.Pi},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minutesInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### Thử chạy test

```
./clockface_test.go:59:11: undefined: minutesInRadians
```

### Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func minutesInRadians(t time.Time) float64 {
	return math.Pi
}
```

### Lặp lại cho các yêu cầu mới

Được rồi - bây giờ hãy bắt mình thực hiện một số công việc *thực sự*. Chúng ta có thể mô phỏng kim phút chỉ di chuyển sau mỗi phút đầy đủ - sao cho nó "nhảy" từ phút thứ 30 sang phút thứ 31 mà không di chuyển ở giữa. Nhưng điều đó trông sẽ hơi tệ. Những gì chúng ta muốn là nó di chuyển một *chút xíu* sau mỗi giây.

```go
func TestMinutesInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(0, 30, 0), math.Pi},
		{simpleTime(0, 0, 7), 7 * (math.Pi / (30 * 60))},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minutesInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

Cái "một chút xíu" đó là bao nhiêu? Chà...

- Sáu mươi giây trong một phút
- Ba mươi phút trong nửa vòng tròn (`math.Pi` radian)
- Vậy có `30 * 60` giây trong nửa vòng tròn.
- Do đó, nếu thời gian trôi qua là 7 giây sau mỗi giờ ...
- ... chúng ta mong đợi thấy kim phút ở vị trí `7 * (math.Pi / (30 * 60))` radian sau hướng 12 giờ.

### Thử chạy test

```
clockface_test.go:62: Wanted 0.012217304763960306 radians, but got 3.141592653589793
```

### Viết đủ code để test chạy thành công

Sử dụng lời bất hủ của Jennifer Aniston: [Đến phần khoa học đây](https://www.youtube.com/watch?v=29Im23SPNok)

```go
func minutesInRadians(t time.Time) float64 {
	return (secondsInRadians(t) / 60) +
		(math.Pi / (30 / float64(t.Minute())))
}
```

Thay vì tính toán xem kim phút được đẩy bao xa quanh mặt đồng hồ cho mỗi giây từ đầu, ở đây chúng ta có thể chỉ cần tận dụng hàm `secondsInRadians`. Đối với mỗi giây kim phút sẽ di chuyển bằng 1/60 góc mà kim giây di chuyển.

```go
secondsInRadians(t) / 60
```

Sau đó chúng ta chỉ cần cộng thêm chuyển động cho phút - tương tự như chuyển động của kim giây.

```go
math.Pi / (30 / float64(t.Minute()))
```

Và...

```
PASS
ok  	clockface	0.007s
```

Thật đẹp đẽ và dễ dàng. Đây là [những gì mọi thứ trông như bây giờ](https://github.com/quii/learn-go-with-tests/tree/main/math/v8/clockface/clockface_acceptance_test.go).

### Lặp lại cho các yêu cầu mới

Tôi có nên thêm nhiều kịch bản hơn vào cuộc kiểm thử `minutesInRadians` không? Hiện tại chỉ có hai. Tôi cần bao nhiêu kịch bản trước khi chuyển sang kiểm thử hàm `minuteHandPoint`?

Một trong những câu nói về TDD yêu thích của tôi, thường được gán cho Kent Beck,[^3] là:

> Hãy viết các bản kiểm thử cho đến khi nỗi sợ hãi chuyển thành sự nhàm chán.

Và, thành thật mà nói, tôi đang cảm thấy nhàm chán khi kiểm thử cái hàm đó. Tôi tự tin mình biết nó hoạt động thế nào. Vì vậy hãy chuyển sang cái tiếp theo.

### Viết test trước tiên

```go
func TestMinuteHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 30, 0), Point{0, -1}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minuteHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

### Thử chạy test

```
./clockface_test.go:79:11: undefined: minuteHandPoint
```

### Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func minuteHandPoint(t time.Time) Point {
	return Point{}
}
```

```
clockface_test.go:80: Wanted {0 -1} Point, but got {0 0}
```

### Viết đủ code để test chạy thành công

```go
func minuteHandPoint(t time.Time) Point {
	return Point{0, -1}
}
```

```
PASS
ok  	clockface	0.007s
```

### Lặp lại cho các yêu cầu mới

Và bây giờ cho một số công việc thực sự:

```go
func TestMinuteHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(0, 30, 0), Point{0, -1}},
		{simpleTime(0, 45, 0), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := minuteHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

```
clockface_test.go:81: Wanted {-1 0} Point, but got {0 -1}
```

### Viết đủ code để test chạy thành công

Một thao tác sao chép và dán nhanh hàm `secondHandPoint` với một vài thay đổi nhỏ chắc hẳn sẽ giải quyết được vấn đề...

```go
func minuteHandPoint(t time.Time) Point {
	angle := minutesInRadians(t)
	x := math.Sin(angle)
	y := math.Cos(angle)

	return Point{x, y}
}
```

```
PASS
ok  	clockface	0.009s
```

### Refactor

Chúng ta chắc chắn đã có một chút sự lặp lại trong `minuteHandPoint` và `secondHandPoint` - tôi biết điều đó vì chúng ta vừa sao chép và dán cái này để tạo ra cái kia. Hãy làm cho nó khô ráo (DRY) hơn bằng một hàm:

```go
func angleToPoint(angle float64) Point {
	x := math.Sin(angle)
	y := math.Cos(angle)

	return Point{x, y}
}
```

và chúng ta có thể viết lại `minuteHandPoint` và `secondHandPoint` thành các hàm một dòng:

```go
func minuteHandPoint(t time.Time) Point {
	return angleToPoint(minutesInRadians(t))
}

func secondHandPoint(t time.Time) Point {
	return angleToPoint(secondsInRadians(t))
}
```

```
PASS
ok  	clockface	0.007s
```

Bây giờ chúng ta có thể bỏ đóng acceptance test và bắt đầu vẽ kim phút.

### Viết đủ code để test chạy thành công

Hàm `minuteHand` là một bản sao chép-và-dán của `secondHand` với một vài điều chỉnh nhỏ, chẳng hạn như khai báo `minuteHandLength`:

```go
const minuteHandLength = 80

//...

func minuteHand(w io.Writer, t time.Time) {
	p := minuteHandPoint(t)
	p = Point{p.X * minuteHandLength, p.Y * minuteHandLength}
	p = Point{p.X, -p.Y}
	p = Point{p.X + clockCentreX, p.Y + clockCentreY}
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#000;stroke-width:3px;"/>`, p.X, p.Y)
}
```

Và một lời gọi nó trong hàm `SVGWriter` của chúng ta:

```go
func SVGWriter(w io.Writer, t time.Time) {
	io.WriteString(w, svgStart)
	io.WriteString(w, bezel)
	secondHand(w, t)
	minuteHand(w, t)
	io.WriteString(w, svgEnd)
}
```

Bây giờ chúng ta sẽ thấy `TestSVGWriterMinuteHand` vượt qua:

```
PASS
ok  	clockface	0.006s
```

Nhưng bằng chứng thép nằm ở kết quả thực tế - nếu bây giờ chúng ta biên dịch và chạy chương trình `clockface`, chúng ta sẽ thấy kết quả như sau:

![a clock with second and minute hands](math/v9/clockface/clockface/clock.svg)

### Refactor

Hãy loại bỏ sự lặp lại từ các hàm `secondHand` và `minuteHand`, đưa tất cả logic phóng to, lật ngược và dịch chuyển vào một nơi duy nhất.

```go
func secondHand(w io.Writer, t time.Time) {
	p := makeHand(secondHandPoint(t), secondHandLength)
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#f00;stroke-width:3px;"/>`, p.X, p.Y)
}

func minuteHand(w io.Writer, t time.Time) {
	p := makeHand(minuteHandPoint(t), minuteHandLength)
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#000;stroke-width:3px;"/>`, p.X, p.Y)
}

func makeHand(p Point, length float64) Point {
	p = Point{p.X * length, p.Y * length}
	p = Point{p.X, -p.Y}
	return Point{p.X + clockCentreX, p.Y + clockCentreY}
}
```

```
PASS
ok  	clockface	0.007s
```

Đây là [vị trí hiện tại của chúng ta](https://github.com/quii/learn-go-with-tests/tree/main/math/v9/clockface).

Xong... giờ chỉ còn kim giờ nữa thôi!

### Viết test trước tiên

```go
func TestSVGWriterHourHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(6, 0, 0),
			Line{150, 150, 150, 200},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the hour hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}
```

### Thử chạy test

```
clockface_acceptance_test.go:113: Expected to find the hour hand line {X1:150 Y1:150 X2:150 Y2:200}, in the SVG lines [{X1:150 Y1:150 X2:150 Y2:60} {X1:150 Y1:150 X2:150 Y2:70}]
```

Một lần nữa, hãy tạm đóng bản này cho đến khi chúng ta có một số độ bao phủ với các bản kiểm thử cấp độ thấp hơn:

### Viết test trước tiên

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### Thử chạy test

```
./clockface_test.go:97:11: undefined: hoursInRadians
```

### Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

```go
func hoursInRadians(t time.Time) float64 {
	return math.Pi
}
```

```
PASS
ok  	clockface	0.007s
```

### Lặp lại cho các yêu cầu mới

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### Thử chạy test

```
clockface_test.go:100: Wanted 0 radians, but got 3.141592653589793
```

### Viết đủ code để test chạy thành công

```go
func hoursInRadians(t time.Time) float64 {
	return (math.Pi / (6 / float64(t.Hour())))
}
```

### Lặp lại cho các yêu cầu mới

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(21, 0, 0), math.Pi * 1.5},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### Thử chạy test

```
clockface_test.go:101: Wanted 4.71238898038469 radians, but got 10.995574287564276
```

### Viết đủ code để test chạy thành công

```go
func hoursInRadians(t time.Time) float64 {
	return (math.Pi / (6 / (float64(t.Hour() % 12))))
}
```

Hãy nhớ rằng, đây không phải là đồng hồ 24 giờ; chúng ta phải sử dụng toán tử chia lấy dư để lấy phần dư của giờ hiện tại khi chia cho 12.

```
PASS
ok  	learn-go-with-tests/math/clockface	0.008s
```

### Viết test trước tiên

Bây giờ hãy thử di chuyển kim giờ quanh mặt đồng hồ dựa trên số phút và số giây đã trôi qua.

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(21, 0, 0), math.Pi * 1.5},
		{simpleTime(0, 1, 30), math.Pi / ((6 * 60 * 60) / 90)},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if got != c.angle {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

### Thử chạy test

```
clockface_test.go:102: Wanted 0.013089969389957472 radians, but got 0
```

### Viết đủ code để test chạy thành công

Một lần nữa, bây giờ cần một chút suy nghĩ. Chúng ta cần di chuyển kim giờ một chút xíu cho cả số phút và số giây. May mắn thay, chúng ta đã có sẵn góc cho phút và giây - chính là kết quả trả về bởi `minutesInRadians`. Chúng ta có thể tái sử dụng nó!

Vậy câu hỏi duy nhất là giảm góc trả về bởi `minutesInRadians` đi bao nhiêu lần. Một vòng quay đầy đủ là một giờ đối với kim phút, nhưng đối với kim giờ là mười hai giờ. Vì vậy, chúng ta chỉ cần chia góc trả về bởi `minutesInRadians` cho mười hai:

```go
func hoursInRadians(t time.Time) float64 {
	return (minutesInRadians(t) / 12) +
		(math.Pi / (6 / float64(t.Hour()%12)))
}
```

và bạn thấy đấy:

```
clockface_test.go:104: Wanted 0.013089969389957472 radians, but got 0.01308996938995747
```

Phép toán dấu phẩy động lại xuất hiện.

Hãy cập nhật bản kiểm thử để sử dụng `roughlyEqualFloat64` cho việc so sánh các góc.

```go
func TestHoursInRadians(t *testing.T) {
	cases := []struct {
		time  time.Time
		angle float64
	}{
		{simpleTime(6, 0, 0), math.Pi},
		{simpleTime(0, 0, 0), 0},
		{simpleTime(21, 0, 0), math.Pi * 1.5},
		{simpleTime(0, 1, 30), math.Pi / ((6 * 60 * 60) / 90)},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hoursInRadians(c.time)
			if !roughlyEqualFloat64(got, c.angle) {
				t.Fatalf("Wanted %v radians, but got %v", c.angle, got)
			}
		})
	}
}
```

```
PASS
ok  	clockface	0.007s
```

### Refactor

Nếu chúng ta định sử dụng `roughlyEqualFloat64` trong *một* bản kiểm thử radian, chúng ta có lẽ nên sử dụng nó cho *tất cả* chúng. Đó là một lần tái cấu trúc đơn giản và đẹp đẽ, nó sẽ để lại mọi thứ [trông như thế này](https://github.com/quii/learn-go-with-tests/tree/main/math/v10/clockface).

## Điểm đầu kim giờ

Đã đến lúc tính toán xem vị trí đầu kim giờ sẽ đi tới đâu bằng cách tính toán vector đơn vị.

### Viết test trước tiên

```go
func TestHourHandPoint(t *testing.T) {
	cases := []struct {
		time  time.Time
		point Point
	}{
		{simpleTime(6, 0, 0), Point{0, -1}},
		{simpleTime(21, 0, 0), Point{-1, 0}},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			got := hourHandPoint(c.time)
			if !roughlyEqualPoint(got, c.point) {
				t.Fatalf("Wanted %v Point, but got %v", c.point, got)
			}
		})
	}
}
```

Đợi đã, tôi chuẩn bị viết *hai* kịch bản kiểm thử *cùng lúc* ư? Chẳng phải điều này là *TDD tồi* sao?

### Về sự cố chấp trong TDD

Phát triển hướng kiểm thử (TDD) không phải là một tôn giáo. Một số người có thể hành động như vậy - thường là những người không thực hiện TDD nhưng thích phàn nàn trên Twitter hay Dev.to rằng nó chỉ dành cho những kẻ cuồng tín và họ đang "thực tế" (being pragmatic) khi không viết test. Nhưng nó không phải là một tôn giáo. Nó là một công cụ.

Tôi *biết* hai bản kiểm thử tiếp theo sẽ là gì - tôi đã kiểm thử hai kim đồng hồ kia theo đúng cách này - và tôi đã biết quá trình triển khai của mình sẽ như thế nào - tôi đã viết một hàm cho trường hợp tổng quát để chuyển đổi một góc thành một điểm trong lần lặp lại cho kim phút.

Tôi sẽ không thực hiện các nghi lễ TDD chỉ vì cho có lệ. TDD là một kỹ thuật giúp tôi hiểu mã nguồn mình đang viết - và mã nguồn mình sẽ viết - tốt hơn. TDD cho tôi phản hồi, kiến thức và sự sáng suốt. Nhưng nếu tôi đã có kiến thức đó rồi, thì tôi sẽ không lặp lại các nghi lễ đó một cách vô nghĩa. Cả sự kiểm thử hay TDD đều không phải là mục đích cuối cùng.

Sự tự tin của tôi đã tăng lên, vì vậy tôi cảm thấy mình có thể bước những bước dài hơn. Tôi sẽ "nhảy cóc" một vài bước, bởi vì tôi biết mình đang ở đâu, tôi biết mình đang đi đâu và tôi đã từng đi trên con đường này trước đây.

Nhưng cũng lưu ý: tôi không bỏ qua hoàn toàn việc viết các bản kiểm thử - tôi vẫn viết chúng trước tiên. Chỉ là chúng xuất hiện theo các mẩu ít chi tiết hơn.

### Thử chạy test

```
./clockface_test.go:119:11: undefined: hourHandPoint
```

### Viết đủ code để test chạy thành công

```go
func hourHandPoint(t time.Time) Point {
	return angleToPoint(hoursInRadians(t))
}
```

Như tôi đã nói, tôi biết mình đang ở đâu và biết mình đang đi đâu. Tại sao phải giả vờ khác đi? Các bản kiểm thử sẽ sớm cho tôi biết nếu tôi sai.

```
PASS
ok  	learn-go-with-tests/math/clockface	0.009s
```

## Vẽ kim giờ

Và cuối cùng chúng ta cũng đến phần vẽ kim giờ. Chúng ta có thể đưa bản acceptance test trở lại:

```go
func TestSVGWriterHourHand(t *testing.T) {
	cases := []struct {
		time time.Time
		line Line
	}{
		{
			simpleTime(6, 0, 0),
			Line{150, 150, 150, 200},
		},
	}

	for _, c := range cases {
		t.Run(testName(c.time), func(t *testing.T) {
			b := bytes.Buffer{}
			clockface.SVGWriter(&b, c.time)

			svg := SVG{}
			xml.Unmarshal(b.Bytes(), &svg)

			if !containsLine(c.line, svg.Line) {
				t.Errorf("Expected to find the hour hand line %+v, in the SVG lines %+v", c.line, svg.Line)
			}
		})
	}
}
```

### Thử chạy test

```
clockface_acceptance_test.go:113: Expected to find the hour hand line {X1:150 Y1:150 X2:150 Y2:200},
    in the SVG lines [{X1:150 Y1:150 X2:150 Y2:60} {X1:150 Y1:150 X2:150 Y2:70}]
```

### Viết đủ code để test chạy thành công

Và giờ chúng ta có thể thực hiện những điều chỉnh cuối cùng cho các hằng số và hàm ghi SVG:

```go
const (
	secondHandLength = 90
	minuteHandLength = 80
	hourHandLength   = 50
	clockCentreX     = 150
	clockCentreY     = 150
)

// SVGWriter ghi lại biểu diễn SVG của một đồng hồ kim hiển thị thời gian t, vào writer w
func SVGWriter(w io.Writer, t time.Time) {
	io.WriteString(w, svgStart)
	io.WriteString(w, bezel)
	secondHand(w, t)
	minuteHand(w, t)
	hourHand(w, t)
	io.WriteString(w, svgEnd)
}

// ...

func hourHand(w io.Writer, t time.Time) {
	p := makeHand(hourHandPoint(t), hourHandLength)
	fmt.Fprintf(w, `<line x1="150" y1="150" x2="%.3f" y2="%.3f" style="fill:none;stroke:#000;stroke-width:3px;"/>`, p.X, p.Y)
}
```

Và thế là...

```
ok  	clockface	0.007s
```

Hãy cùng kiểm tra lại bằng cách biên dịch và chạy chương trình `clockface`.

![a clock](math/v12/clockface/clockface/clock.svg)

### Refactor

Nhìn vào `clockface.go`, có một vài "số ma thuật" đang trôi nổi xung quanh. Tất cả chúng đều dựa trên số giờ/phút/giây có trong nửa vòng quanh mặt đồng hồ. Hãy tái cấu trúc để làm rõ ý nghĩa của chúng.

```go
const (
	secondsInHalfClock = 30
	secondsInClock     = 2 * secondsInHalfClock
	minutesInHalfClock = 30
	minutesInClock     = 2 * minutesInHalfClock
	hoursInHalfClock   = 6
	hoursInClock       = 2 * hoursInHalfClock
)
```

Tại sao phải làm như vậy? Chà, nó làm rõ ý nghĩa của từng con số trong phương trình. Khi chúng ta - *nếu* chúng ta - xem lại mã nguồn này, những cái tên này sẽ giúp chúng ta hiểu chuyện gì đang xảy ra.

Hơn nữa, nếu chúng ta muốn tạo ra một số chiếc đồng hồ thực sự KỲ QUẶC - ví dụ như những chiếc có kim giờ quay 4 tiếng một vòng, kim giây 20 giây một vòng - những hằng số này có thể dễ dàng trở thành các tham số. Chúng ta đang mở rộng cánh cửa đó (ngay cả khi chúng ta không bao giờ bước qua).

## Tổng kết

Chúng ta có cần làm gì khác không?

Đầu tiên, hãy tự khen ngợi bản thân - chúng ta đã viết một chương trình tạo ra mặt đồng hồ SVG. Nó hoạt động tốt và thật tuyệt vời. Nó sẽ chỉ tạo ra một loại mặt đồng hồ - nhưng thế cũng không sao! Có thể bạn chỉ *muốn* một loại mặt đồng hồ. Không có gì sai với một chương trình giải quyết một vấn đề cụ thể và không gì khác.

### Một chương trình... và một thư viện

Nhưng mã nguồn chúng ta viết *đã* giải quyết một tập hợp các vấn đề tổng quát hơn liên quan đến việc vẽ mặt đồng hồ. Bởi vì chúng ta đã sử dụng các bản kiểm thử để suy nghĩ về từng phần nhỏ của vấn đề một cách độc lập, và bởi vì chúng ta đã hệ thống hóa sự độc lập đó bằng các hàm, chúng ta đã xây dựng được một tập hợp API nhỏ khá hợp lý cho việc tính toán mặt đồng hồ.

Chúng ta có thể làm việc trên dự án này và biến nó thành một thứ gì đó tổng quát hơn - một thư viện để tính toán các góc và/hoặc vector mặt đồng hồ.

Thực tế, việc cung cấp thư viện cùng với chương trình là một *ý tưởng thực sự hay*. Nó chẳng tốn gì của chúng ta, trong khi lại tăng tính hữu dụng cho chương trình và giúp ghi lại tài liệu về cách nó hoạt động.

> Các thư viện (APIs) nên đi kèm với các chương trình, và ngược lại. Một API mà bạn phải viết mã C để sử dụng, cái mà không thể được gọi dễ dàng từ dòng lệnh, là khó học và khó sử dụng hơn. Và ngược lại, thật là một nỗi đau tột cùng khi có những interface mà hình thức công khai duy nhất được ghi chép là một chương trình, khiến bạn không thể gọi chúng dễ dàng từ một chương trình C.
>			-- Henry Spencer, trong sách _The Art of Unix Programming_

Trong [phiên bản cuối cùng của chương trình này](https://github.com/quii/learn-go-with-tests/tree/main/math/vFinal/clockface), tôi đã biến các hàm không xuất (unexported functions) bên trong `clockface` thành một API công khai cho thư viện, với các hàm để tính toán góc và vector đơn vị cho từng kim đồng hồ. Tôi cũng đã tách phần tạo SVG thành package của riêng nó, `svg`, cái mà sau đó được chương trình `clockface` sử dụng trực tiếp. Đương nhiên là tôi đã ghi lại tài liệu cho từng hàm và package.

Nói về SVG...

### Bản kiểm thử có giá trị nhất

Tôi chắc chắn bạn đã nhận ra rằng phần mã nguồn phức tạp nhất để xử lý SVG hoàn toàn không nằm trong mã nguồn ứng dụng; nó nằm trong mã nguồn kiểm thử. Điều này có khiến chúng ta thấy không thoải mái không? Chẳng lẽ chúng ta không nên làm điều gì đó như:

- sử dụng một template từ `text/template`?
- sử dụng một thư viện XML (giống như chúng ta đang làm trong test)?
- sử dụng một thư viện SVG?

Chúng ta có thể tái cấu trúc mã nguồn để thực hiện bất kỳ điều nào trong số này, và chúng ta có thể làm vậy bởi vì không quan trọng *cách thức* chúng ta tạo ra SVG, điều quan trọng là *cái gì* chúng ta tạo ra - *một file SVG*. Do đó, phần hệ thống của chúng ta cần biết nhiều nhất về SVG - cần phải nghiêm ngặt nhất về những gì cấu thành một SVG - chính là bản kiểm thử cho output SVG: nó cần có đủ ngữ cảnh và kiến thức về SVG để chúng ta tự tin rằng mình đang xuất ra một file SVG. Cái *cái gì* của một SVG sống trong các bản kiểm thử của chúng ta; cái *cách thức* nằm trong mã nguồn.

Chúng ta có thể cảm thấy kỳ quặc khi dành ra nhiều thời gian và công sức cho các bản kiểm thử SVG đó - import một thư viện XML, phân tích XML, tái cấu trúc các struct - nhưng mã nguồn kiểm thử đó là một phần giá trị của codebase - có lẽ còn giá trị hơn cả mã nguồn ứng dụng hiện tại. Nó sẽ giúp đảm bảo rằng output luôn là một SVG hợp lệ, bất kể chúng ta chọn sử dụng cái gì để tạo ra nó.

Kiểm thử không phải là những thứ hạng hai - chúng không phải là mã nguồn "vứt đi". Những bản kiểm thử tốt sẽ tồn tại lâu hơn nhiều so với phiên bản mã nguồn mà chúng đang kiểm thử. Bạn không bao giờ nên cảm thấy mình đang dành "quá nhiều thời gian" để viết các bản kiểm thử. Đó là một khoản đầu tư.

[^1]: Việc này dễ hơn nhiều so với việc viết tay tên dưới dạng một chuỗi rồi phải giữ cho nó đồng bộ với thời gian thực tế. Tin tôi đi, bạn sẽ không muốn làm điều đó đâu...

[^2]: Tóm lại là nó giúp việc thực hiện phép tính với các vòng tròn dễ dàng hơn vì số π cứ liên tục xuất hiện dưới dạng một góc nếu bạn sử dụng các độ thông thường, vì vậy nếu bạn tính các góc theo số π thì mọi phương trình sẽ đơn giản hơn.

[^3]: Bị gán sai vì giống như tất cả các tác giả vĩ đại khác, Kent Beck thường được trích dẫn nhiều hơn là được đọc. Chính Beck gán câu đó cho [Phlip][phlip].

[texttemplate]: https://golang.org/pkg/text/template/
[circle]: https://en.wikipedia.org/wiki/Sine#Unit_circle_definition
[mathcos]: https://golang.org/pkg/math/#Cos
[floatingpoint]: https://0.30000000000000004.com/
[phlip]: http://wiki.c2.com/?PhlIp
[xml]: https://pkg.go.dev/encoding/xml
