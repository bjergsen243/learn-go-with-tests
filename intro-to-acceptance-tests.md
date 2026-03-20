# Giới thiệu về kiểm thử chấp nhận (Acceptance testing)

Tại `$WORK`, chúng tôi đã và đang gặp phải nhu cầu cần có tính năng "tắt an toàn" (graceful shutdown) cho các dịch vụ của mình. Tính năng tắt an toàn đảm bảo hệ thống của bạn hoàn thành công việc một cách thích hợp trước khi nó bị chấm dứt. Một ví dụ trong thế giới thực sẽ là ai đó cố gắng kết thúc một cuộc gọi điện thoại một cách đàng hoàng trước khi chuyển sang cuộc họp tiếp theo, thay vì chỉ dập máy giữa chừng.

Chương này sẽ giới thiệu qua về việc tắt an toàn trong bối cảnh của một máy chủ HTTP, và cách viết các "bản kiểm thử chấp nhận" (acceptance tests) để giúp bạn tự tin vào hành vi mã nguồn của mình.

Sau khi đọc xong chương này, bạn sẽ biết cách chia sẻ các gói (packages) đi kèm với những bản kiểm thử xuất sắc, giảm bớt nỗ lực bảo trì và tăng độ tin cậy vào chất lượng công việc của bạn.

## Vừa đủ thông tin về Kubernetes

Chúng tôi chạy phần mềm của mình trên [Kubernetes](https://kubernetes.io/) (K8s). K8s sẽ chấm dứt các "pods" (trong thực tế là phần mềm của chúng ta) vì nhiều lý do khác nhau, và một lý do phổ biến là khi chúng ta đẩy (push) mã mới mà chúng ta muốn triển khai (deploy).

Chúng tôi đặt ra cho bản thân những tiêu chuẩn cao về [các chỉ số DORA](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance), do đó chúng tôi làm việc theo cách mà chúng tôi triển khai những cải tiến và tính năng nhỏ, tăng dần lên môi trường sản xuất (production) nhiều lần mỗi ngày.

Khi k8s muốn chấm dứt một pod, nó khởi tạo một ["chu kỳ sống chấm dứt" (termination lifecycle)](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace), và một phần trong đó là gửi đi một tín hiệu SIGTERM tới phần mềm của chúng ta. Đây là cách K8s báo với đoạn mã của mình rằng:

> Ngươi cần phải tự tắt máy đi, hoàn thành nốt bất cứ việc gì đang làm lỡ dở bởi vì sau một "thời gian ân hạn" (grace period) nhất định, ta sẽ gửi `SIGKILL`, và lúc đó thì ngươi tắt điện hằn.

Vào lúc tín hiệu `SIGKILL` được gọi ra, mọi công việc hiện thời được thực thi bởi chương trình của bạn sẽ bị dừng lại ngay tắp lự.

## Nếu bạn không cúp máy an toàn kiểu (graceful shutdown)

Tùy vào đặc điểm phần mềm của bạn, nếu bạn bỏ ngoài tai thông điệp `SIGTERM`, bạn có thể gặp rắc rối.

Vấn đề cụ thể của chúng tôi nằm ở các yêu cầu HTTP đang được xử lý (in-flight HTTP requests). Khi một công cụ kiểm thử tự động đang kiểm tra API của chúng tôi, nếu K8s quyết định dừng máy con (pod), theo lẽ thường máy chủ sẽ "đi đời", bài test không thể nhận phản hồi gì sất từ máy chủ nữa, và từ đó kiểm thử cũng chẳng thành công.

Điều này sẽ kích hoạt thông báo đỏ trên kênh chat của chúng tôi về vấn đề sự cố, một thông báo ép các dev phải ngưng tất tần tật công việc hiện tại rồi quay lại xúm vào tìm lỗi phát sinh. Những thất bại ngắt quãng kiểu này là sự phân tâm cực kỳ khó chịu đối với cả team.

Các nhức nhối trên thì đâu có loại trừ gì đâu, với bài test hay với cái khác cũng thế. Nếu như một người khách gửi truy vấn tới hệ thống rồi gặp tình cảnh cúp điện giữa chừng (mid-flight), khả năng xui xẻo cao anh bạn đó sẽ ăn quả thông báo điếc 5xx HTTP. Chẳng hệ thống nào lại muốn nhai phải trãi nghiêm tồi tệ của khách hàng đâu nhỉ.

## Khi bạn xử lý được kiểu "graceful"

Những gì chúng ta muốn làm là lắng nghe `SIGTERM`, và thay vì giết chết máy chủ ngay lập tức, chúng ta muốn:

- Ngừng lắng nghe thêm bất kỳ yêu cầu nào nữa
- Cho phép bất kỳ yêu cầu nào đang giải quyết dở (in-flight) được hoàn thành nốt
- *Sau đó* mới chấm dứt tiến trình

## Làm thế nào để có "grace"?

Rất đáng mừng thay, bản thân Go vốn dĩ đã có sẵn một cơ chế cho phép chúng ta chủ động dừng lại (shutdown) server cách an toàn với đoạn code [net/http/Server.Shutdown](https://pkg.go.dev/net/http#Server.Shutdown).

> Shutdown sẽ tắt máy chủ một cách mượt mà mà không làm gián đoạn mọi kết nối đang diễn ra (dù là một). Shutdown xử lý qua thuật toán dừng lại hết toàn bộ danh sách "open listeners" hiện tại (nghe kết nối mở), kể cả các kết nối trạng thái nghỉ, từ từ nhâm nhi ly trà đợi ngóng nốt kết quả các active response trên kia phản hồi ra sao và đẩy vào "idle", và đoạn cuối là sập nguồn hẳn. Nếu mà gói Context đi kèm hết hạn chót thời điểm trước lúc lệnh Shutdown hoàn thành công việc của nó, thì Shutdown lại báo ngược lại lỗi do gói Context, không thì báo lỗi là kết quả do gói listeners ẩn của Server thu về lúc ép buộc kết nối ngừng lại.

Để thu nhận `SIGTERM` ta tận dụng module [os/signal.Notify](https://pkg.go.dev/os/signal#Notify), đây là chỗ báo các kênh Channel chúng ta cung cấp bất kỳ các signals đến nào.

Tối đa hóa sử dụng 2 món kỹ năng thượng thừa này do thư viện chuẩn (standard library) nhà làm, nay bạn đã có thể lắng tai nghe tín hiệu của thiên sứ báo ngày phán xét `SIGTERM` rồi dừng server lại lúc nào mình muốn một cách khá nhẹ nhàng êm ái.

## Gói (Package) Graceful-Shutdown

Vì lẽ đó, tôi đã viết ra [https://pkg.go.dev/github.com/quii/go-graceful-shutdown](https://pkg.go.dev/github.com/quii/go-graceful-shutdown). Nó cung cấp cho một function như một chức năng mở rộng cho `*http.Server` để gọi hàm `Shutdown` ngay thời khắc nó nhìn thấy có tiếng hò hét của thằng tín hiệu `SIGTERM`.

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// Nhảy thông báo này về lý do phản hồi của bạn chưa in ra được thời điểm trước Context deadline, cũng bất lực chả thể làm gì sất.
		log.Fatalf("uh oh, didn't shutdown gracefully, some responses may have been lost %v", err)
	}

	// thay vì vậy hy vọng luôn gặp dòng dưới này nhé
	log.Println("shutdown gracefully! all responses were sent")
}
```

Nội tiết tố (The specifics), thiết kế nội bên đoạn mã kia vốn không đao búa lắm cho chương này, ấy nhưng tôi cũng nên bỏ đi ít thời gian nghía ngang qua chót vài phần source code phía bên trên một xíu mới đành đi.

## Các bản kiểm thử (Tests) và vòng lặp phản hồi (Feedback loops)

Khi chúng tôi viết ứng dụng gói `gracefulshutdown`, chúng tôi có các phần mềm kiểm tra mức bộ phận (unit tests) để chứng thực thiết kế cho dự án vận hành mượt. Dẫu gì thì, đó được xem là trợ giúp nhỏ bé tạo thêm dũng cảm cho việc cắt xét, xây dựng (refactor) lại cấu trúc hệ thống. Cơ mà thành thực thì, chúng tôi vẫn cảm thấy thiếu "tự tin" rằng mọi thứ sẽ vận hành và nó **thực sự** không bung bét cả ra.

Chúng tôi tạo thêm một tập code package mang tên `cmd` (lệnh) kết hợp cùng xây chương trình thử thực nghiệm qua mớ code chúng tôi dựng. Việc thử có những đoạn là phải thực hiện trên máy thủ công (manual) qua trò "bật chương trình - kích hoạt các http calls vào endpoint đó, chốt màn gửi cước lệnh tặc tử `SIGTERM` - đếm kết quả của màn thi coi như nào".

**The engineer in you should be feeling uncomfortable with manual testing (người làm kỹ thuật nếu vướng vụ kiểm tra thủ công sẽ sinh tâm lý chướng vô cùm).**
Rõ chán, cái đó sao dùng quy mô to được, thiếu tính sát thật (inaccurate) và hoàn toàn lãng phí của giời. Cứ giả thiết nếu bạn muốn viết nguyên 1 mảng lớn rồi tung hê nó để sài chung (share), nhưng mà kèm thêm tiêu chí ít rối não nhất rồi dể quất (change) nhất thì xin báo lun, làm manual có bằng cắt bỏ cho nó rảnh nợ.

## Bản kiểm thử chấp nhận (Acceptance tests)

Nãy giờ đọc được mấy chương của sách rồi, kiểu gì mớ bòng bong code mà các bạn phải viết ở dạng "tập tính khối lẻ" của unit tests thì chắc kha khá rồi đó nhỉ. Ở khía cạnh đó unit tests thực chất rèn vũ khí (tools) mang lại ưu quyền đập đi xây lại không phải ngán 1 bố con ai (fearless refactoring). Việc dẫn lái chúng giúp có định hướng cho sự phân vùng module cho tốt (drive good module), giảm thiểu vấp lỗi phát sinh theo thời gian (prevent regressions), từ đó tối cần các câu trả lời ngắn (fast feedbacks)

Chỉ có điều, chúng chỉ kiểm tra được các mảnh rất bé tẹo của phần mềm theo đúng chức năng nhiệm vụ duyên cớ sinh ra chúng. Do đó thông thường phần unit tests không có cửa ( *không đủ* ) cho tư cách đánh giá toàn cục hệ thống được. Giữ cho vững, ta cần cho đứa con hệ thống nhà mình **luôn phải có thể xách kho mà đem giao** (always be shippable). Cái mớ manual tests mệt não quá, thế làm thế nào với rào cản mang tên "thủ công" ta cần gánh vào đây loại chiêu thức rà soát mới. Một tên là: Kiểm tra đánh giá thông qua nghiệm thu được gọi với cái mác **acceptance tests**.

### Chúng là cái quái gì?

Kiểm thử chấp nhận là một loại "kiểm thử hộp đen" (black-box test). Đôi khi chúng được gọi là "kiểm thử chức năng" (functional tests). Chúng nên dùng vào việc vận hành hệ thống giống như cách một người dùng hệ thống sẽ làm.

Thuật ngữ hộp đen (black-box) hướng đến quan điểm chỉ ra là mã thực hiện testing chẳng thể soi dòm cấu kiện nội bộ bên trong hệ thống ra sao, những thứ dùng cho đánh giá là qua việc điều khiển các cổng chóp của nó (mặt tiền) rồi rút kết ra sự nhận định thông qua cái nết ứng xử thực thi bị đem ra khảo sát. Điều này cũng có nghĩa là người ta chỉ có tùy chọn khả dĩ là test cái hệ thống y một hệ thống mà thôi (system as a whole).

Điều này là một ưu điểm, mang lợi ích cho quy trình bởi vì lúc test, ta kiểm duyệt nó cũng in hệt cái thao tác như người khách có trải nghiệm vậy, test chả thèm chơi chiêu "hack, vượt rào" - (kẻ phá rào, đi thủng đường tắt) nhằm cố mà gỡ cho bài báo cáo rớt đài lúc pass mà không soi xét thứ mà bạn cần nó pass. Bạn còn giữ phương châm không test trực quan code của package là thay vì là file test bạn không nên nằm ở `package mypkg` mà nên ném nó ra package chửi sau mới ngoan đúng luật `package mypkg_test` thì sẽ ngấm ra.

### Lợi ích của các Acceptance tests

- Nếu đặng mọi bài test đạt tiêu chuẩn (pass), bạn khẳng định hệ thống đang hành xử rạp y phăng rắc thứ bạn ra lệnh.
- Rõ ràng nó cho phản hồi độ chuẩn cũng không tồi, báo kết quả lại lướt mây siêu cấp không tốn nhọc nhằn tý nào nếu đi đem so kè so đao với hàng làm bằng sức người chay test manual.
- Còn nếu được gõ khéo code lại còn đẹp trai sáng láng thay thế một mớ tài liệu chuẩn cơm mẹ nấu với chứng chỉ chứng thực do nó đảm nhận luôn (accurate, verified documenation). Điều này dập tắt nguy cơ hệ thống hành động "một đằng", trong khi bản khai báo văn bản trôi qua thời gian lại thành mớ bòng bong ghi "một nẻo" chả giống cái mô tê chi dáo.
- 100% người thực việc thực, giấu luôn kỹ năng diễn tuồng (Không Mock!). 

### Mặt trái nếu đặt test rà nghiệm thu so đo cùng mảng unit tests

- Vắt mồ hôi viết mã nên chát phết.
- Chạy thì ỳ ạch chậm chứ bộ.
- Sự phụ thuộc vào mặt định kỳ hay tính thiết kế mảng hệ thống là nhiều.
- Rớt nhịp (fail) kiểm thử là dính chưởng luôn tại hông bao h cho mình ngó root cause là gì ở tận gốc lụi lỗi lầm. Rút cục công cán để debug kiếm cớ nó chua cực á.
- Ngụy trang, chênh chếch với mã dỏm mà các acceptance này pass ầm ầm là chiêu quen do ẻm không thèm báo mãng của bác ntn về mặt bản chất đâu (internal quality system).
- Bản chất hòm đen gây ngộp thở nên đâm ra việc lôi ứng dụng giả ra đục cho thành thật khó lắm nha, đâu phải ngữ cảnh gì mình cũng mò được mà test.

Vì lý do này, sẽ là ngu ngốc nếu chỉ dựa vào mỗi kiểm thử chấp nhận. Chúng không có nhiều phẩm chất như bài unit tests gánh lấy, và một hệ thống với số lượng bài test chấp nhận khổng lồ thường sẽ chịu đựng theo nghĩa đen liên đới vì mấy khoản phí bảo hành cũng tốn hao kinh khủng nhọc mệt cả "lead time".

#### "Lead time"? Ái dà có thế chứ? (Lead Time là thời điểm cái khỉ khô gì)

Lead time (thời điểm tung lệnh gọi gộp, tính từ phút giây bắt tay đưa thay đổi mã xuống kho `merge commit -> branch chính` qua dóng hàng để xuất đi cho ra biển biển khai mạc - production deployment). Khoảng mốc báo cáo "hồn bay phách lạc" nay mai này nhỉnh cả đến mấy mùa, cũng chừng đó tuần với lắm lúc vác team tới gục cho được qua vài phút đồng hồ cho vài nhà phát triển nào trót xài. Cho nói rõ nốt cái, cái đám bọn `$WORK` thì lúc nào cũng chăm soi theo quan hệ với con số mà nhà DORA nó đề xướng làm gương mẫu nên ép bản thân cứ hỡ tầm chục phút 10 là chọt cái lead time vào rảnh cho đỡ xót.


Phải giữ độ chuẩn nhịp kiểm tra bằng một chiến thuật cho một môi giới tốt đáng đồng tiền bát gạo nhưng ăn đứt vụ `lead time` này ngon nghẻ mới xong, do đó thông thường nó được gói thành hình tựa khái niệm như con [Hình tháp kiểm thử (Test Pyramid)](https://martinfowler.com/articles/practical-test-pyramid.html).

## Cách để viết một bản kiểm thử chấp nhận cơ bản

Thế thì nó có gì liên hệ qua trục trặc nguyên mẫu hông nhỉ? Để ý chút, bạn có thấy gói (package) mình nhào lặn vừa rồi ấy chẳng phải mọi khía cạnh là vọc unit test đều xử tuốt được hay sao.

Như tui nói ở trên, cái đoạn code test Unit thì dường như nó thiếu mắm thiếu muối sao đó mới lột tả được niềm tin để ta dựa nương vào đó. Chúng tôi có mưu cầu khẳng định thật _chac chắn_ (phải là thật sự chắc) là gói tin của tôi còn xài vào hệ chương trình chần mứa sống ngắc ngứ kia thì nó vẫn trân tru vận hành. Ta hoàn toàn có thể nhồi nhét code tự giải những màn check nảy não như thế ở phía người dùng thật khi chuyển cái này sang tự động được.

Nào bóc tách đoạn test nhúng vô nhé

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// Nhảy thông báo này về lý do phản hồi của bạn chưa in ra được thời điểm trước Context deadline, cũng bất lực chả thể làm gì sất.
		log.Fatalf("uh oh, didn't shutdown gracefully, some responses may have been lost %v", err)
	}

	// thay vì vậy hy vọng luôn gặp dòng dưới này nhé
	log.Println("shutdown gracefully! all responses were sent")
}
```

Hẳn là bạn cũng soi ra cái rắp tâm vụ `SlowHandler` phải lù lù kèm con `time.Sleep` thì cũng mường tượng một xíu là chủ ý làm trễ quá trình đáp ứng để tôi cho nó tý đất diễn vụ tống ném biến thần thánh `SIGTERM` đặng xem phản ứng ntn. Cớ dư vậy chốc lát còn lại thì chỉ xem boilerplate là ra:

- Vòi cái mảng máy web `net/http/Server`;
- Bọc nó vào trong thư viện (xem: [Mô hình Decorator](https://en.wikipedia.org/wiki/Decorator_pattern));
- Triệu hồi phiên bản đã gói đấy vào sài tại `ListenAndServe`.

### Liệt kê theo chuỗi thao tác (high-level) tại bộ kiểm duyệt cấp acceptances tests

- Cài build vào trình phần mềm
- Ra lệnh cho chạy lên (mà chịu rình cho đên lúc nó nhạy ở tận `8080`)
- Xin gửi 1 phong HTTP lên bác cửa máy cái
- Nhá 1 phát cho sếp bự `SIGTERM` xuống cước nhưng phải giựt chớp mắt trước lúc có đường đáp trả lời response qua HTTP của máy chủ.
- Xem bộ dạng em có chịu trả mình gì hông

### Xây dựng (Building) và Chạy chương trình

```go
package acceptancetests

import (
	"fmt"
	"math/rand"
	"net"
	"os"
	"os/exec"
	"path/filepath"
	"syscall"
	"time"
)

const (
	baseBinName = "temp-testbinary"
)

func LaunchTestProgram(port string) (cleanup func(), sendInterrupt func() error, err error) {
	binName, err := buildBinary()
	if err != nil {
		return nil, nil, err
	}

	sendInterrupt, kill, err := runServer(binName, port)

	cleanup = func() {
		if kill != nil {
			kill()
		}
		os.Remove(binName)
	}

	if err != nil {
		cleanup() // dủ không thấy tín hiệu nó hoạt động, khả năng ngầm ứng dụng đang bay lạng quạng cũng không phải nhỏ
		return nil, nil, err
	}

	return cleanup, sendInterrupt, nil
}

func buildBinary() (string, error) {
	binName := randomString(10) + "-" + baseBinName

	build := exec.Command("go", "build", "-o", binName)

	if err := build.Run(); err != nil {
		return "", fmt.Errorf("cannot build tool %s: %s", binName, err)
	}
	return binName, nil
}

func runServer(binName string, port string) (sendInterrupt func() error, kill func(), err error) {
	dir, err := os.Getwd()
	if err != nil {
		return nil, nil, err
	}

	cmdPath := filepath.Join(dir, binName)

	cmd := exec.Command(cmdPath)

	if err := cmd.Start(); err != nil {
		return nil, nil, fmt.Errorf("cannot run temp converter: %s", err)
	}

	kill = func() {
		_ = cmd.Process.Kill()
	}

	sendInterrupt = func() error {
		return cmd.Process.Signal(syscall.SIGTERM)
	}

	err = waitForServerListening(port)

	return
}

func waitForServerListening(port string) error {
	for i := 0; i < 30; i++ {
		conn, _ := net.Dial("tcp", net.JoinHostPort("localhost", port))
		if conn != nil {
			conn.Close()
			return nil
		}
		time.Sleep(100 * time.Millisecond)
	}
	return fmt.Errorf("nothing seems to be listening on localhost:%s", port)
}

func randomString(n int) string {
	var letters = []rune("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789")

	s := make([]rune, n)
	for i := range s {
		s[i] = letters[rand.Intn(len(letters))]
	}
	return string(s)
}
```

Hàm thiết lập chương trình `LaunchTestProgram` có nghĩa vụ phải:
- dựng (build) phần mềm
- chạy và đẩy cỗ máy ấy hoạt động
- ngây mặt mà chờ với rốn cho nó đáp tại cảng mạng `8080`
- cấp phát một thủ tục để gọi dọn rửa sạch sẽ qua trình dọn gọi là `cleanup` rình lúc kill process là tiện tay đem luôn code giùm thùng rác đặng sau khi các khâu kiểm thử nghỉ ngơi tay ngực tui được thảnh thơi nhấm trà sảng khoái với cái môi trường được dọn xạch gọn y bong (rồi có sạch mà).
- sắm luôn ra cái trò chơi lệnh ngắt `interrupt` mà trỏ cược qua em yêu chốt lời báo sập `SIGTERM` tiện bề thao túng cái hành trình đang xem.
Thú nhận một điều thì, đoạn mã này không phải là đoạn code đẹp nhất trên đời, nhưng hãy chỉ tập trung vào cái hàm được đưa ra cộng đồng mạng dùng chung là (exported function) `LaunchTestProgram`, mấy cái hàm mờ nhạt gọi bên trong kia toàn mấy mớ rập khuôn vô vị (boilerplate).

Như đã bàn, kiểm thử chấp nhận (acceptance testing) có xu hướng rắc rối hơn để thiết lập. Đoạn code này thực sự làm cho mã code _kiểm thử_ giản tiện một cách đáng kể tới lúc đọc, và thường với các bài lập trình kiểm tra sự chấp nhận một lúc bạn mà nhọc công code xong được đống tổ bu nghi thức cúng kiến lằng nhằng này rồi, nó là cái vĩnh cửu, bạn chả cần phải xỉa não quan tâm nữa.

### Bản kiểm thử chấp nhận (acceptance tests)

Chúng tôi muốn có hai bản kiểm tra nhắm đích cho hai chương trình, một cái dùng cách thoát an nhàn (`graceful shutdown`) và cái còn lại kệ đời (without), qua đó, bọn mình và đương nhiên cả mấy ông bà đọc bài này mới rõ thấu được sự biến hóa trong cư xử của tụi nó. Cùng `LaunchTestProgram` làm đại ca sai múa may gõ búa đẽo code chạy lên app chương trình, chuyện đặt bút viết cho 2 mảng test này hóa ra trơn tuột như bôi mỡ, đã dứa tụi mình còn được nhờ ở phần dùng lại mớ `function helpers`.

Phác họa kiểm thử viết ra sài qua một máy chủ _có mặt_ chiêu rút lui trong danh dự, cho tiện bạn có thể [dạo quanh bản test vắng mặt tính năng ấy ngay trên repo kho GitHub này](https://github.com/quii/go-graceful-shutdown/blob/main/acceptancetests/withoutgracefulshutdown/main_test.go)

```go
package main

import (
	"testing"
	"time"

	"github.com/quii/go-graceful-shutdown/acceptancetests"
	"github.com/quii/go-graceful-shutdown/assert"
)

const (
	port = "8080"
	url  = "<http://localhost:" + port
)

func TestGracefulShutdown(t *testing.T) {
	cleanup, sendInterrupt, err := acceptancetests.LaunchTestProgram(port)
	if err != nil {
		t.Fatal(err)
	}
	t.Cleanup(cleanup)

	// rẽ qua ngó máy chủ chạy vẫn phà phà xem chửa dính bùa lịm nào nhé
	assert.CanGet(t, url)

	// bắn bốc hơi một cái request, với lanh lẹ chợp thời cơ cho phi tiêu găm luôn SIGTERM khi ứng dụng ấp úng định nhả lời.
	time.AfterFunc(50*time.Millisecond, func() {
		assert.NoError(t, sendInterrupt())
	})
	// Không có thuốc tiên giải huyệt mang danh tắt có dự tính (graceful shutdown), khúc này sẽ rụng ngay.
	assert.CanGet(t, url)

	// sau quả choáng váng do ngắt đột ngột cúp cầu giao mạng, server sẽ tắt dần đều, đừng hòng mống request nào ăn thua nữa nha.
	assert.CantGet(t, url)
}
```

Bởi khâu nhúng chìm việc thiết lập vô trong vỏ (encapsulated) như vậy, những bài tính kiểm ngặt trở nên toàn diện hơn, lột tả được sắc nét cái tánh hành xử của đối tượng mà còn siêu đỗi dễ hình dung tuân theo.

`assert.CanGet/CantGet` đóng vai hai đứa sai vặt mà tôi chế ra hòng giữ châm ngôn tái chế code (chạm chuẩn DRY) dành cho cái luật lệ hay đem dùng đi sài lại với nguyên bộ bài (suite) test rả rích này.

```go
func CanGet(t testing.TB, url string) {
	errChan := make(chan error)

	go func() {
		res, err := http.Get(url)
		if err != nil {
			errChan <- err
			return
		}
		res.Body.Close()
		errChan <- nil
	}()

	select {
	case err := <-errChan:
		NoError(t, err)
	case <-time.After(3 * time.Second):
		t.Errorf("timed out waiting for request to %q", url)
	}
}
```

Một chú sẽ khai phát cuốc gọi điện kiểu `GET` gọi vào thẳng `URL`, bọc trong cái áo `goroutine`, với phước đức để lại mà nó réo chuông báo hoàn trót lọt với bằng chứng là ko một vết nhơ lỗi thì lúc ấy chốt sổ (not fail). `CantGet` nhường lối cất giấu qua khe do tốn hàng chứ chẳng có mưu mẹo gì, [chỉn chu hơn thì xem ngay tại cái nôi là trang chủ GitHub chốn này](https://github.com/quii/go-graceful-shutdown/blob/main/assert/assert.go#L61).

Hãy cho đĩa than lên một lần nhắc sâu sắc lại, dòng Go cấp trọn thảy cho túi đồ mưu cầu việc nắn test rà nghiệm thu chả thiếu thứ chi, nguyên bản sẵn phom (out of the box). Mình _đâu rãnh_ kiếm chi trọn cái khung công trình (framework) cầu kì cho cất được loại kiểm thử đánh giá tính chấp nhận (acceptance tests) của sản phẩm.

### Ngân lượng còm, sắm vàng ròng (bỏ chút sức thu hời cực đậm)

Cảm giác đem được những bài test tung ra, hội độc giả có thể nhâm nhi nghiên cứ các phiên bản mồi mà mạnh tay ôm lấy cái tự tin bảo chứng: thí dụ này _chắc cú_ chạy được; thành thử củng cố lòng nương cậy vô bộ `packages`.

Khẳng khái vỗ ngực là tay "cha chú" chắp rặn nó ra, điều này giúp thu nhặt **phản hồi tức tốc (fast feedback)** song song châm trọn vẹn **đỉnh cao tự tin** báo hiệu nguyên kiện mã xài cực mướt bên luồng hiện thực.

```shell
go test -count=1 ./...
ok  	github.com/quii/go-graceful-shutdown	0.196s
?   	github.com/quii/go-graceful-shutdown/acceptancetests	[no test files]
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withgracefulshutdown	4.785s
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withoutgracefulshutdown	2.914s
?   	github.com/quii/go-graceful-shutdown/assert	[no test files]
```

## Tổng kết

Trong bài blog này, tôi vừa "đầu độc" bạn vào mảng bài kiểm thử tính chấp thuận sắm trong túi áo chọc nghẹo mấy gã dev. Không có chúng sao trị được thiên hạ rộng mở, tụi nó sẽ được ươm vào chéo mảng với loại Unit test để dệt nên một hàng lá chắn.

Phương cách *ra sao* (how) mới viết nổi ba kiểm chứng vặt này thì lại dắt qua dựa trên quy chuẩn cho dạng cái "kiểu hệ thống" mấy cha đang cày cục dựng; dẫu rằng kim chỉ nam cốt yếu (principles) luôn chôn một chỗ như đinh rỉ. Hình dung bộ máy mình nắm dưới dạng khối mưu sầu quỷ dạ "Hộp đen" (black box). Bạn nặn ra cái mặt tiền của 1 cái Web thì, dàn test bạn biên nên hóa dạng một user bằng xương thịt, cờ nhét (click) qua links thì hãy mài rũa tool hỗ trợ trình duyệt kiểu headless như của [Selenium](https://www.selenium.dev/), hòng đọ phím tút léo tót chọc nhét, múa võ điền trang thông tin này (fill...forms),v.v.. Đẩy lên thành bản API của mảng RESTful, đâm lệnh đạn yêu cầu lọt cổng HTTP thì bạn quẩy tung lồng qua một `client` thế là đặng.

### Nâng cấp mở khóa vùng độ sâu với Hệ Thống chằng chịt rích rắc

Cân đong vào cho cái thứ ko phải trò của dân chít chăn như loại dùng máy làm "đèn xanh đỏ một hướng" chúng tôi lỡ trót múa miệng bên trên á. Mà ở thực tại phũ phàng hơn, bạn đè nặng vác thêm một đàn các nền tảng khác phụ theo (ví dụ DB). Giải những con số cho vế đố ấy, đụng trúng bạn phải điều tự động cỗ môi trường nội bộ hòng làm đất cho code đi test. Có vài cái rổ kéo (tools) đồ nghề bọc mang tiếng tuổi oanh vàng như [docker-compose](https://docs.docker.com/compose/) sài rất xịn sò dùng sút xoáy sinh sản nhanh chớ mấy lớp chứa môi trường nội bộ hòng phục vụ quá trình ứng dụng của tay bạn hoạt động trơn khi ngụp lặn tại rốn con PC.

### Nẻo tới tại chương tiếp chắp bút

Ngụp lặn tại trang post này ta sào nấu mã acceptance tests nhưng thiên về mảng hướng quy trình bồi đắp nhặt dần lại thứ đã đẽo. Xong xui mà nói, vùi trong quyền sách [Growing Object-Oriented Software](http://www.growing-object-oriented-software.com), cái tác giả chỉ thẳng mặt cướp điểm là bộ khung kiểm định của tests dọn đường dồn tâm làm mục tiêu chốt mỏm "chó-sói-ngắm-trăng" ("north-star") soi lấy cái rắp tâm thành thực của chún mình theo đúng tôn chỉ Test-Driven-Approach.

Cái tính "vùng sâu, hiểm hóc" nhú càng nhiều, mâm cúng thời lượng lẫn đạn tiền cho khâu chế cháo và sửa soạn mã acceptance tests này đội lên phi mã khó cản vô thiên. Cả thúng các màn kịch đội kỹ sư chán rầu tới phát điên trói bộ mặt do nguyên cây tiền đổ ập mà mua dầm rả cái bộ cùm mã đắt sắt này.

Ngay tại đoạn đường tương lai nãy mình trình lên cái khía sài nó cho hướng điều tâm vào thiết kế mang kèm tính tôn chỉ lẫn các xảo thực kìm dây nịt cho tiền phung phí chi ba mảng này thôi.

### Nới Cửa sổ đánh bóng lớp chất của Open-Source 

Tính đường đi dạo vứt mớ packages mà tụi bay muốn thiên hạ cũng sờ tay mó dạo, trẫm khuyên bay rành đôi chút làm ví dụ cho tụi nó hiểu cái cốt tinh chương trình chạy là cái qué dóc gì đi cùng văng não chút lãng mạn vô chiêu tạo ra đám test nhắm lính trơn nhất cho họ soi để mình lẫn những ông con ghẻ muốn mớm có chỗ thấy niềm an ủi yên tâm.

Mang nghĩa như dạng bài tủ [Bài ví dụ đem Thể Kiểm Được (Testable Examples)](https://go.dev/blog/examples), dẫu chắp mồ hôi liếc cho vài con chữ trải đường (dev exp) sài cực dễ bắt tay thì lại ăn dăm ba con xảo xây cả môt kho uy tín độ nét công sức, với cả nhẩm cho dễ là bớt việc cho bác gánh sau này.

## Gạch dòng "Chào Hàng" lôi bạn tới công ty làm tại `$WORK`

Một khi bạn tăm tia cái chốn nán lại làm chung mâm các sư huynh (engineers) vừa ăn chè vừa phá cỗ mấy thứ lạ lạ để code, vị trí bạn định vị chui góc London hay phi sang tận Porto cơ đấy, lẫn máu lửa chực cày nội thất cái book với mớ dở của chương này - [tìm tui trên thánh địa chim xanh Twitter đê cha nội (nay là X nha)](https://twitter.com/quii), cũng khả quan tới vụ mình dắt tay vô đội đó mấy khứa ơi!
