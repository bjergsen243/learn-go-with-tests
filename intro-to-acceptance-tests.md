# Giới thiệu về Acceptance Tests

Tại nơi làm việc, chúng tôi cần tính năng graceful shutdown cho các dịch vụ. Graceful shutdown đảm bảo hệ thống hoàn thành công việc đang dở trước khi tắt — giống như kết thúc cuộc gọi lịch sự trước khi chuyển sang cuộc họp tiếp theo, thay vì dập máy giữa chừng.

Chương này sẽ giới thiệu về graceful shutdown trong bối cảnh của một máy chủ HTTP, và cách viết các acceptance test để giúp bạn tự tin vào hành vi của code.

Sau khi đọc xong chương này, bạn sẽ biết cách chia sẻ các package đi kèm với những bài test tốt, giảm công sức bảo trì và tăng độ tin cậy vào chất lượng công việc của bạn.

## Vừa đủ thông tin về Kubernetes

Chúng tôi chạy phần mềm trên [Kubernetes](https://kubernetes.io/) (K8s). K8s sẽ tắt các pod (tức phần mềm của chúng ta) vì nhiều lý do, phổ biến nhất là khi deploy code mới.

Chúng tôi đặt tiêu chuẩn cao về [các chỉ số DORA](https://cloud.google.com/blog/products/devops-sre/using-the-four-keys-to-measure-your-devops-performance), nên deploy lên production nhiều lần mỗi ngày.

Khi K8s muốn chấm dứt một pod, nó khởi tạo một [termination lifecycle](https://cloud.google.com/blog/products/containers-kubernetes/kubernetes-best-practices-terminating-with-grace). Một phần trong đó là gửi tín hiệu `SIGTERM` tới phần mềm của chúng ta. Đây là cách K8s thông báo với chương trình rằng:

> Bạn cần tự tắt đi. Hãy hoàn thành nốt mọi công việc đang làm dở, bởi vì sau một khoảng grace period, tôi sẽ gửi `SIGKILL`, và lúc đó bạn sẽ bị buộc dừng ngay lập tức.

Khi tín hiệu `SIGKILL` được gửi, mọi công việc đang được thực thi bởi chương trình sẽ bị dừng lại ngay lập tức.

## Nếu bạn không xử lý graceful shutdown

Tùy vào đặc điểm phần mềm của bạn, nếu bạn bỏ qua tín hiệu `SIGTERM`, bạn có thể gặp rắc rối.

Vấn đề cụ thể của chúng tôi nằm ở các in-flight HTTP requests. Khi một công cụ test tự động đang kiểm tra API của chúng tôi, nếu K8s quyết định dừng pod, máy chủ sẽ bị tắt đột ngột, bài test không thể nhận được phản hồi từ máy chủ, và từ đó test cũng thất bại.

Điều này sẽ kích hoạt thông báo cảnh báo trên kênh chat của chúng tôi, buộc các lập trình viên phải dừng công việc hiện tại để đi tìm nguyên nhân lỗi. Những thất bại ngắt quãng kiểu này là sự phân tâm rất khó chịu đối với cả team.

Vấn đề này không chỉ giới hạn ở test. Nếu một người dùng gửi request tới hệ thống và gặp tình trạng bị ngắt giữa chừng, rất có thể họ sẽ nhận được lỗi 5xx HTTP. Không hệ thống nào muốn gây ra trải nghiệm tồi tệ cho người dùng cả.

## Khi bạn xử lý graceful shutdown

Những gì chúng ta muốn làm là lắng nghe `SIGTERM`, và thay vì dừng máy chủ ngay lập tức, chúng ta muốn:

- Ngừng nhận thêm bất kỳ yêu cầu mới nào
- Cho phép các in-flight request được hoàn thành
- *Sau đó* mới chấm dứt tiến trình

## Làm thế nào?

Rất may, Go đã có sẵn cơ chế cho phép chúng ta tắt server một cách an toàn với [net/http/Server.Shutdown](https://pkg.go.dev/net/http#Server.Shutdown).

> Shutdown tắt máy chủ một cách mượt mà mà không làm gián đoạn các kết nối đang hoạt động. Đầu tiên, Shutdown đóng tất cả các listener đang mở, sau đó đóng các kết nối idle, rồi chờ vô thời hạn cho đến khi các kết nối đang hoạt động trở về trạng thái idle, và cuối cùng tắt hẳn. Nếu Context được cung cấp hết hạn trước khi Shutdown hoàn thành, Shutdown trả về lỗi từ Context. Ngược lại, Shutdown trả về bất kỳ lỗi nào phát sinh khi đóng các listener.

Để nhận tín hiệu `SIGTERM`, chúng ta sử dụng [os/signal.Notify](https://pkg.go.dev/os/signal#Notify). Hàm này cho phép chúng ta đăng ký một channel để nhận các tín hiệu từ hệ điều hành.

Kết hợp hai công cụ này từ standard library, bạn có thể lắng nghe tín hiệu `SIGTERM` và tắt server một cách an toàn.

## Package Graceful Shutdown

Vì lý do trên, tôi đã viết [https://pkg.go.dev/github.com/quii/go-graceful-shutdown](https://pkg.go.dev/github.com/quii/go-graceful-shutdown). Package này cung cấp một function bọc ngoài `*http.Server`, tự động gọi `Shutdown` khi nhận được tín hiệu `SIGTERM`.

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// this will typically happen if our responses aren't finished before ctx deadline, not much we can do
		log.Fatalf("uh oh, didn't shutdown gracefully, some responses may have been lost %v", err)
	}

	// hopefully, you'll always see this instead
	log.Println("shutdown gracefully! all responses were sent")
}
```

Chi tiết thiết kế bên trong đoạn code này không phải trọng tâm của chương này, nhưng tôi cũng nên dành chút thời gian xem qua phần code phía trên.

## Tests và vòng lặp phản hồi

Khi viết package `gracefulshutdown`, chúng tôi có các unit test để kiểm chứng thiết kế. Tuy nhiên, chúng chỉ hỗ trợ phần nào cho việc refactor. Thành thực mà nói, chúng tôi vẫn chưa đủ tự tin rằng mọi thứ sẽ **thực sự** hoạt động đúng.

Chúng tôi tạo thêm một thư mục `cmd` chứa chương trình thử nghiệm sử dụng code mà chúng tôi viết. Việc test phải thực hiện thủ công: bật chương trình lên, gửi các HTTP request tới endpoint, gửi tín hiệu `SIGTERM`, rồi kiểm tra kết quả.

**Là một kỹ sư, bạn nên cảm thấy không thoải mái với việc test thủ công.** Cách này không mở rộng được, thiếu chính xác, và lãng phí thời gian. Nếu bạn muốn chia sẻ code cho người khác sử dụng với yêu cầu bảo trì thấp nhất và dễ thay đổi nhất, thì test thủ công là không đủ.

## Acceptance tests

Nếu bạn đã đọc các chương trước trong cuốn sách này, chắc hẳn bạn đã quen với việc viết unit test. Unit test là công cụ mạnh mẽ giúp bạn refactor mà không sợ phá vỡ chức năng. Chúng giúp thiết kế module tốt hơn, ngăn ngừa lỗi phát sinh theo thời gian, và cho phản hồi nhanh.

Tuy nhiên, unit test chỉ kiểm tra được các phần nhỏ của phần mềm. Do đó, thông thường unit test *không đủ* để đánh giá toàn bộ hệ thống. Chúng ta cần đảm bảo hệ thống **always shippable**. Test thủ công thì quá mệt mỏi, vậy chúng ta cần thêm một loại test khác: **acceptance test**.

### Acceptance test là gì?

Acceptance test là một loại black-box test. Đôi khi chúng được gọi là functional test. Chúng kiểm tra hệ thống giống như cách một người dùng thực sự sẽ sử dụng hệ thống.

Thuật ngữ "black-box" có nghĩa là code test không biết cấu trúc bên trong hệ thống. Test chỉ tương tác với hệ thống thông qua public interface và đánh giá dựa trên hành vi quan sát được. Điều này cũng có nghĩa là bạn chỉ có thể test hệ thống như một tổng thể.

Đây là một ưu điểm, bởi vì khi test, chúng ta kiểm tra hệ thống giống hệt cách người dùng trải nghiệm. Test không "đi đường tắt" để pass mà không thực sự kiểm chứng đúng. Nếu bạn đã quen với nguyên tắc không test trực tiếp code nội bộ của package - tức là file test nên nằm trong `package mypkg_test` thay vì `package mypkg` - thì bạn sẽ hiểu ý tưởng này.

### Lợi ích của acceptance test

- Nếu tất cả test pass, bạn có thể khẳng định hệ thống đang hoạt động đúng như mong đợi.
- Cho phản hồi nhanh hơn rất nhiều so với test thủ công.
- Nếu được viết tốt, acceptance test còn đóng vai trò như tài liệu sống - chính xác và luôn được cập nhật. Điều này loại bỏ nguy cơ hệ thống hoạt động một kiểu, trong khi tài liệu lại mô tả một kiểu khác.
- Không có mock. Test chạy trên hệ thống thật.

### Nhược điểm của acceptance test so với unit test

- Viết phức tạp hơn.
- Chạy chậm hơn.
- Phụ thuộc nhiều vào kiến trúc và thiết kế hệ thống.
- Khi test fail, khó xác định root cause. Việc debug tốn nhiều công sức hơn.
- Acceptance test có thể pass trong khi internal quality của hệ thống vẫn kém, vì test không kiểm tra cấu trúc bên trong.
- Do bản chất black-box, khó tái tạo mọi tình huống để test.

Vì lý do này, sẽ là sai lầm nếu chỉ dựa vào acceptance test. Chúng không thay thế được unit test. Một hệ thống với quá nhiều acceptance test sẽ gặp vấn đề về chi phí bảo trì và lead time.

#### Lead time là gì?

Lead time là khoảng thời gian từ lúc merge code vào branch chính cho đến khi code được triển khai lên production. Con số này có thể dao động từ vài phút đến vài tuần, tùy thuộc vào tổ chức. Tại `$WORK`, chúng tôi luôn hướng tới các chỉ số DORA và cố gắng giữ lead time ở mức khoảng 10 phút.

Cần duy trì một chiến lược test cân bằng, đảm bảo chất lượng nhưng không ảnh hưởng đến lead time. Đây là lý do khái niệm [Test Pyramid](https://martinfowler.com/articles/practical-test-pyramid.html) ra đời.

## Cách viết một acceptance test cơ bản

Vậy điều này liên quan gì đến vấn đề ban đầu? Hãy nhớ rằng package chúng ta vừa đề cập không thể kiểm tra mọi khía cạnh chỉ bằng unit test.

Như đã nói ở trên, unit test chưa đủ để mang lại sự tự tin. Chúng tôi muốn khẳng định _chắc chắn_ rằng package hoạt động đúng khi được tích hợp vào một chương trình thực. Chúng ta hoàn toàn có thể tự động hóa các bước kiểm tra thủ công đã nói ở trên.

Hãy cùng xem đoạn code cần test:

```go
func main() {
	var (
		ctx        = context.Background()
		httpServer = &http.Server{Addr: ":8080", Handler: http.HandlerFunc(acceptancetests.SlowHandler)}
		server     = gracefulshutdown.NewServer(httpServer)
	)

	if err := server.ListenAndServe(ctx); err != nil {
		// this will typically happen if our responses aren't finished before ctx deadline, not much we can do
		log.Fatalf("uh oh, didn't shutdown gracefully, some responses may have been lost %v", err)
	}

	// hopefully, you'll always see this instead
	log.Println("shutdown gracefully! all responses were sent")
}
```

Bạn có thể thấy `SlowHandler` sử dụng `time.Sleep` để cố tình làm chậm phản hồi. Điều này giúp chúng ta có thời gian gửi `SIGTERM` trong khi request đang được xử lý. Phần còn lại chỉ là boilerplate:

- Tạo một `net/http/Server`
- Bọc nó bằng thư viện của chúng ta (theo [Decorator pattern](https://en.wikipedia.org/wiki/Decorator_pattern))
- Gọi `ListenAndServe` trên phiên bản đã bọc

### Các bước ở mức tổng quan cho acceptance test

- Build chương trình
- Chạy chương trình (và chờ cho đến khi nó lắng nghe ở cổng `8080`)
- Gửi một HTTP request tới máy chủ
- Gửi tín hiệu `SIGTERM` trong khi chờ phản hồi HTTP
- Kiểm tra xem có nhận được phản hồi hay không

### Build và chạy chương trình

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
		cleanup() // even though it errored, the incomplete process may have been started, so cleanup
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

Hàm `LaunchTestProgram` có nhiệm vụ:
- Build chương trình
- Chạy chương trình
- Chờ cho đến khi chương trình lắng nghe ở cổng chỉ định
- Trả về hàm `cleanup` để dọn dẹp sau khi test xong (kill process và xóa binary)
- Trả về hàm `sendInterrupt` để gửi tín hiệu `SIGTERM` tới chương trình đang chạy

Thú thật, đoạn code này không phải đẹp nhất, nhưng hãy tập trung vào hàm được export là `LaunchTestProgram`. Các hàm bên trong chỉ là boilerplate.

Như đã đề cập, acceptance test có xu hướng phức tạp hơn để thiết lập. Tuy nhiên, đoạn code thiết lập này giúp cho code test trở nên rõ ràng và dễ đọc hơn rất nhiều. Thông thường với acceptance test, một khi bạn đã viết xong phần thiết lập, bạn không cần phải quan tâm đến nó nữa.

### Acceptance test

Chúng tôi muốn có hai bài test cho hai chương trình: một chương trình có graceful shutdown và một chương trình không có. Qua đó, chúng ta có thể thấy rõ sự khác biệt trong hành vi. Nhờ `LaunchTestProgram` đã đóng gói phần thiết lập, việc viết hai bài test này trở nên dễ dàng và có thể tái sử dụng các helper function.

Dưới đây là bài test cho máy chủ _có_ graceful shutdown. Bạn có thể [xem bài test cho máy chủ không có graceful shutdown trên GitHub](https://github.com/quii/go-graceful-shutdown/blob/main/acceptancetests/withoutgracefulshutdown/main_test.go).

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

	// verify the server is running and can respond
	assert.CanGet(t, url)

	// send a request, and then send SIGTERM before the response is received
	time.AfterFunc(50*time.Millisecond, func() {
		assert.NoError(t, sendInterrupt())
	})
	// without graceful shutdown, this would fail
	assert.CanGet(t, url)

	// after the interrupt, the server should have shut down, no more requests should be served
	assert.CantGet(t, url)
}
```

Nhờ việc encapsulate phần thiết lập, bài test trở nên rõ ràng và dễ hiểu. Nó mô tả chính xác hành vi mà chúng ta muốn kiểm tra.

`assert.CanGet` và `assert.CantGet` là các helper function tôi viết để giữ cho code DRY (Don't Repeat Yourself), vì chúng được sử dụng lại trong nhiều bài test.

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

Hàm này gửi một HTTP GET request tới URL trong một goroutine, và kiểm tra xem request có thành công mà không có lỗi hay không. `CantGet` thì làm ngược lại. [Bạn có thể xem chi tiết trên GitHub](https://github.com/quii/go-graceful-shutdown/blob/main/assert/assert.go#L61).

Xin nhắc lại, Go cung cấp đầy đủ mọi thứ bạn cần để viết acceptance test ngay từ standard library. Bạn _không cần_ một framework đặc biệt nào để viết acceptance test.

### Đầu tư ít, thu lại nhiều

Với những bài test này, người đọc có thể chạy thử và tự tin rằng code _thực sự_ hoạt động. Điều này củng cố niềm tin vào package.

Là tác giả, điều này giúp tôi có **fast feedback** và **sự tự tin cao** rằng code hoạt động đúng trong thực tế.

```shell
go test -count=1 ./...
ok  	github.com/quii/go-graceful-shutdown	0.196s
?   	github.com/quii/go-graceful-shutdown/acceptancetests	[no test files]
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withgracefulshutdown	4.785s
ok  	github.com/quii/go-graceful-shutdown/acceptancetests/withoutgracefulshutdown	2.914s
?   	github.com/quii/go-graceful-shutdown/assert	[no test files]
```

## Tổng kết

Trong bài viết này, tôi đã giới thiệu về acceptance test. Acceptance test không thay thế unit test, mà bổ sung cho unit test để tạo nên một lớp bảo vệ toàn diện hơn.

*Cách* viết acceptance test phụ thuộc vào loại hệ thống bạn đang xây dựng, nhưng các nguyên tắc cốt lõi luôn giống nhau. Hãy coi hệ thống như một black box. Nếu bạn xây dựng một trang web, bài test nên mô phỏng hành vi của người dùng thực - click link, điền form - sử dụng headless browser như [Selenium](https://www.selenium.dev/). Nếu bạn xây dựng RESTful API, hãy gửi HTTP request thông qua một client.

### Mở rộng cho hệ thống phức tạp hơn

Ví dụ trong bài viết này khá đơn giản - chỉ là một máy chủ HTTP đơn lẻ. Trong thực tế, hệ thống của bạn có thể phụ thuộc vào nhiều thành phần khác (ví dụ: cơ sở dữ liệu). Trong trường hợp đó, bạn cần thiết lập môi trường tự động để chạy test. Các công cụ như [docker-compose](https://docs.docker.com/compose/) rất hữu ích để khởi tạo nhanh các container phục vụ cho việc test trên máy cục bộ.

### Hướng đi tiếp theo

Trong bài viết này, chúng ta viết acceptance test cho code đã có sẵn. Tuy nhiên, trong cuốn sách [Growing Object-Oriented Software](http://www.growing-object-oriented-software.com), tác giả khuyến khích sử dụng acceptance test như một "north star" - tức là viết acceptance test trước để định hướng cho quá trình phát triển theo phương pháp TDD.

Hệ thống càng phức tạp, thời gian và công sức để viết và duy trì acceptance test càng tăng. Nhiều đội kỹ sư đã gặp khó khăn vì chi phí bảo trì acceptance test quá cao.

Trong các chương tiếp theo, chúng ta sẽ tìm hiểu cách sử dụng acceptance test để định hướng thiết kế, cùng với các kỹ thuật giúp kiểm soát chi phí.

### Nâng cao chất lượng Open-Source

Nếu bạn định chia sẻ package cho cộng đồng, hãy cung cấp ví dụ minh họa cách sử dụng, kèm theo acceptance test để người dùng có thể tự kiểm chứng rằng code hoạt động đúng.

Tương tự như [Testable Examples](https://go.dev/blog/examples), việc đầu tư một chút vào developer experience sẽ giúp xây dựng uy tín và giảm bớt công việc bảo trì về sau.

## Tuyển dụng tại `$WORK`

Nếu bạn quan tâm đến việc làm việc cùng các kỹ sư đam mê code, tại London hoặc Porto, và thích nội dung của cuốn sách này - [hãy liên hệ với tôi trên Twitter](https://twitter.com/quii). Rất có thể chúng tôi sẽ hợp tác cùng nhau!
