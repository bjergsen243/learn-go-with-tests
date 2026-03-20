# Học Go qua các bài Kiểm thử (Learn Go with Tests) - Kiểm thử chấp nhận mở rộng (và lướt qua gRPC một chút)

Chương này là phần tiếp theo của chương [Giới thiệu về Kiểm thử chấp nhận (Intro to acceptance tests)](https://quii.gitbook.io/learn-go-with-tests/testing-fundamentals/intro-to-acceptance-tests). Bạn có thể tìm thấy [mã nguồn hoàn chỉnh cho chương này trên GitHub](https://github.com/quii/go-specs-greet).

Kiểm thử chấp nhận (Acceptance tests) cực kỳ thiết yếu, và chúng có tác động trực tiếp đến khả năng bạn có thể tự tin phát triển hệ thống của mình theo thời gian, với một chi phí cải tiến và bảo trì hợp lý.

Chúng cũng là một công cụ tuyệt vời để giúp bạn làm việc với mã nguồn cũ (legacy code). Khi phải đối mặt với một cơ sở mã tồi tàn mà không có lấy một bài test nào, xin bạn hãy cố nén cái mong muốn hăm hở ngồi tái cấu trúc (refactoring) ngay lập tức. Thay vào đó, hãy viết một vài bản kiểm thử chấp nhận để tạo cho mình một mạng lưới an toàn (safety net) hòng dễ bề thay đổi các cấu trúc bên trong của hệ thống một cách thoải mái mà không lo ảnh hưởng đến hành vi chức năng đã lộ ra ở bên ngoài của nó. Các bài ATs (Acceptance Tests) không cần quan tâm đến chất lượng nội tại, nên chúng rất phù hợp trong các tình huống chữa cháy kiểu này.

Sau khi đọc bài này xong, bạn sẽ thấy thấu cảm hơn rằng kiểm thử chấp nhận không chỉ hữu ích cho việc xác minh tính đúng đắn mà còn có thể được sử dụng khéo léo vào quy trình phát triển bằng cách vạch luồng lạch chỉ đường chúng ta thay đổi hệ thống một cách có chủ ý và có phương pháp hơn, giảm bớt nỗ lực đổ sông đổ biển.

## Tài liệu nền tảng

Cảm hứng cho chương này được sinh ra từ nhiều năm bực bội tột đỉnh với các bài kiểm thử chấp nhận. Hai video mà tôi khuyên bạn nên mổ băng xem ngay là:

- Dave Farley - [Cách để viết các bài kiểm thử chấp nhận hiệu quả (How to write acceptance tests)](https://www.youtube.com/watch?v=JDD5EEJgpHU)
- Nat Pryce - [Các bài test chức năng đầu-cuối (E2E) chỉ trong chớp mắt mili-giây (E2E functional tests that can run in milliseconds)](https://www.youtube.com/watch?v=Fk4rCn4YLLU)

"Growing Object Oriented Software" (GOOS) là một cuốn sách vô cùng quan trọng đối với nhiều kỹ sư phần mềm, trong đó có tôi. Hướng tiếp cận mà nó rạch ra là lộ trình mà tôi luôn huấn luyện các anh em kỹ sư làm chung với mình tuân theo.

- [GOOS](http://www.growing-object-oriented-software.com) - Dưới ngòi bút của Nat Pryce & Steve Freeman

Cuối cùng, tôi và [Riya Dattani](https://twitter.com/dattaniriya) đã đàm luận về chủ đề này trong hệ quy chiếu của BDD (Phát triển Dựa trên Hành vi) qua buổi phỏng vấn chung, [Kiểm thử chấp nhận, BDD và đại diện là Go (Acceptance tests, BDD and Go)](https://www.youtube.com/watch?v=ZMWJCk_0WrY).

## Gói ghém lại (Recap)

Chúng ta đang nói về kiểu kiểm thử "hộp đen" (black-box) chuyên để xác minh hệ thống của bạn hành xử đúng như mong muốn từ góc nhìn bên ngoài, nhìn dưới khía cạnh "**nhãn quan kinh doanh**" (business perspective). Các bài kiểm thử loại này không có cái quyền xoi mói vào ruột gan lục phủ của hệ thống; chúng chỉ bận tâm tới việc hệ thống của bạn làm được cái **gì (what)** thay vì đào bới **bằng cách nào (how)**.

## Mổ xẻ các bài kiểm thử bị hỏng cấu trúc

Hơn chục năm ròng rã, tôi đã kinh qua rất nhiều công ty và nhóm phát triển. Cả đám bọn họ đều nhận thức rõ ràng nhu cầu cần có các bản ATs; vài hình thức nôm na nào đó để nhắm đến test thử coi hệ thống vận hành trôi chảy điêu luyện hay không dưới vai khách qua đường, song lại nhục nhằn mang tính truyền thống (hầu như không tránh khỏi) là chuyện trả giá đổ vào các bài AT này phình to quá lố và đè nặng lên lưng của team dev:

- Mất thời gian để chạy
- Khá giòn (Dễ hỏng hóc)
- Chập chờn lúc được lúc không (Flaky)
- Đắt đỏ để bảo trì, và có vẻ như làm cho quá trình thay đổi phần mềm khó như lên trời trong khi đáng lẽ ra nó phải dễ hơn thế
- Chỉ có thể ỳ xèo hoạt động gói gọn ở môi trường chuyên dụng, gây ra chu trình phản hồi chậm rì với tỷ lệ tin cậy kém cỏi

Giả thử bạn định viết một bản test AT nhắm vào cái website nhào lặn bấy lâu. Bạn phệt ngay một cái khung headless browser (kiểu như [Selenium](https://www.selenium.dev)) dùng thế lính đóng vai cày người dùng đi thọc gõ nút chọt nút cạch vào chức năng trang nhằm mong chứng tỏ website lết ổn đúng bài.

Tháng năm trôi, phần HTML/Markup cho web đổi chóng mặt khi có mấy ổng bự múa chức năng thêm mắm gọt muối, các tay code tha hồ cãi tay đôi cho kỳ được cái khối nọ nên chèn `<article>` hay nhét `<section>` tỷ lần vẫn vậy.

Chẵng qua các huynh đệ sửa lặt vặt (minor changes), khéo user mắt mờ vẫn chẳng thể bóc trần khác biệt, nhưng thảm thê thay bạn vùi rục xương ra cập nhật nguyên con hệ dàn trận ATs vừa chọc lúc nãy.

### Tính liên kết chặt (Tight-coupling)

Ngồi mà nghĩ coi cái gì xúi giục việc update các bài kiểm định ATs này:

- Biến động về hành vi mặt ngoài (An external behaviour change). Chừng nào mà thứ tạo ra (business) được đổi màu đổi vị, cập nhật mớ kịch bản ATs nghe có lý, và đáng bị mong chờ tới.
- Mảng logic bưng bít phía sau hay việc đảo refactor hệ thống (An implementation detail/refactoring). Có thánh mới cho là mấy chuyện động trời này lại là động cơ thúc máy test phải bị nhòm ngó, nếu sửa khéo nó thuộc hàng li ti xíu.

Thế nhưng mà, 90% thì các nguyên gốc do yếu tố sau chót luôn ngặn ngò lôi các bài ATs nhà ta làm bia chịu trận cần chắp vá. Đạt đỉnh điểm nhát sợ (nhát ma sờ) khiến cho dân coding còn ko chịu chỉnh ứng dụng do có tâm lý mợ nó ngán đụng đến bầy đàn sớ bài test kia!

![Riya và tôi đang nói chuyện về việc tách biệt những vấn đề trong khâu test hiện giờ](https://i.imgur.com/bbG6z57.png)

Cội nguồn những cái điên rồ này vốn lòi ra do lỗi hệ tư duy yếu mềm chểnh mảng mấy phép vận hành kỹ thuật được hun đúc chuẩn bị mà những "cây cao bóng cả" phía trên kia nắn nót ghi truyền. **Có đùa không mà bạn lôi công thức nắn unit tests bốc thuốc vào trò acceptance tests**; xin cho cái là ATs sinh ra đi liền với lối gõ và hướng suy khác bọt luôn.

## Giải mổ cấu trúc một bản Acceptance tests thơm hương (good tests)

Khát khao những bài ATs chỉ thèm chuyển mình khi hệ tính năng hành vi thay hướng chứ thèm để tâm dăm ba cái kiểu mần code, hợp tình thuận lý nhất là đem bóc tụi nó thành 2 mảng cô lập khỏi nhau (thứ cần xử).

### Bàn về các hệ hình độ phức tạp (types of complexity)

Làm kiếp kỹ sư, ta chẳng thoát cái dây duyên nợ mà nhằn chung 2 kiểu phiền hàm sau:

- **Độ phức tạp phụ trợ ẩn tàng (Accidental complexity)** mớ bòng bong do gánh thêm cùm nợ là thiết bị cứng bưng, điển hình phần mạng mẹo, lưu kho disk, các loại api đính, v.v..
- **Độ phức tạp sống còn (Essential complexity)** đâu kia ngợi danh bằng cái tiếng "Luật lệ cốt cán" (domain logic). Đích thực thứ nhào nặn nên chân lí với nguyên lý mà mảng kinh doanh/vùng domain đòi tróc nã.
  - Nghe này nghen, "Ví dụ trường hợp 1 quý tài khoản rút một lượng đạn xanh bự hơn ví đang cất, gán mác luôn tài khoản mòn đít". Phát ngôn đó liên quan quái chi đến thiết bị tính toán đâu chớ; nó thành quy chuẩn vàng cho trần đời trước lúc tụi làm app lồng thiết bị vào kho ngân lượng cho bank!

Vào lúc cái gọi Essential complexity cần giải nghĩa rõ cho dăm ba cậu tay mơ công nghệ hiểu được, tính báu vật sẽ rất cực đỉnh nếu như bạn bưng tụi nó lên bản họa qua khu code thuần "domain", rồi bê nó vào mớ kiểm soát nghiệp vụ acceptance tests.

### Phân rã theo trách nhiệm (Separation of concerns)

Đạo lý cụ Dave Farley phán trong dĩa băng phía trên, theo Riya và tôi đàm đạo chặp, chúng ta nên dựng kịch bản với luồng kim chỉ nam lấy tên là **Đặc tả kịch bản (specifications)**. Thông só kỹ thật Specification rêu rao về mảng lối hành xử mà chúng ta cầu khát, tránh đụng độ hay đèo bòng vào mấy vũng bùn dơ Accidental complexity với tầng thực thi implementation detail giấu sau lớp phông.

Cái phác đồ kia gõ tai cho lọt đấy. Trong mấy đợt sào sản phẩm chạy live, chúng ta điên cuồng trầy xát mong xé riêng mấy ngóc ngách phân trách nhiệm cho đám module (units of work). Rành rành là có mảy may e dè khi lôi nguyên con `interface` cho tay sai `HTTP` handler để bóc nó lìa luôn mớ chướng tai gai mắt chả liên can quái gì đến HTTP hông trơn? Vậy mạnh dạn sài mưu đồ tư duy kiểu ấy áp ngược cho mớ kiểm thử chấp thuận nhé.

Dave Farley họa kiểu rành mạch một tổ chức phân lớp.

![Bức thư viện Dave Farley phác về Acceptance Tests](https://i.imgur.com/nPwpihG.png)

Thời mạn đàm tại GopherconUK, Riya cùng tôi khoác vỏ từ vựng kiểu Go bọc mớ này.

![Tách tách từng khâu - Separation of concerns](https://i.imgur.com/qdY4RJe.png)

### Kiểm thử ngộp steroid (Testing on steroids)

Thoát trói cái thói đụng chạm kiểu "bố gọi gọi lúc nào rỗi" lúc khai màn spec - ta hốt cục lời là tống khứ chạy lại nó dùng tại đủ khung cảnh muôn hình. Xin nêu tên:

#### Để đám củ điều hướng Driver thả ga tự biên kịch(Make our drivers configurable)

Mưu tính này bao độ sướng: Cứ tùy hỉ cho bài ATs bơi qua môi trường tự tại (local), hoặc lên cả dàn staging và đỉnh cao chói lóa nhất là môi trường của sự thật (production environments).
- Ngập ngụa bao nhóm thả dàn cho code chạy vướng đan vào nhau trói riết chẳng có kẽ mở cướp đường cho lệnh Test chạy êm đẹp ở sân nhà local. Quả tạ này vác thêm một mức kiềm feedback rụt rịch chậm rù. Hỏi ông có mong gật gù bài ATs qua trót lọt đặng  _trước cữ_ dán dòng code mình viết chưa? Nhỡ nó gẫy rắc lúc gõ test, êm chịu hông nổi với quả đánh rớt oan rồi mù tịt vì local mò ko có bệnh bèn nhắm mắt rải commit mù để nợ thêm 20 chục phút dài ngóng con số ở sàn khác coi trời xanh may phước sao?
- Đinh ninh đi, dẫu sới test staging bật nắp báo trúng giải thì khoan nhận hệ thống mình ngon. Chuyện thói trêu đồng sinh đôi Dev với Prod nghe như kiểu mẩu tin lừa con nít. Bạn coi: [Tôi đây nện bài test cả trong lúc app tung live Prod nhóe](https://increment.com/testing/i-test-in-production/).
- Đám môi trường này luôn hắt hơi ra điều chênh nhau ảnh hưởng trực tiếp hệ _sống còn_ (behaviour). Nắp CDN đôi khi ăn hớ bộ đệm lúc cache headers dính mã; lũ dịch vụ đàn em mình ngậm sữa bỗng phản tặc chệch luồng; và cũng dễ lắm khi file đuôi config có dòng ngu ngốc lệch nhịp. Sướng thấu xương hông nếu bạn đem bản Test spec này nện trực tiếp sống mái tận mâm prod mà nắm sống rễ hư hỏng nhanh như chớp cờ?

#### Chọc lọng với mớ cọc nái _khác máu_ Driver nhằm ngứa test luôn chỗ bí của app. (Plug in different drivers to test other parts of your system)

Với phép thử uyển chuyển đó, ta ung dung mò đường quất hệ hành vi rải từ các loại tầng thiết kế khác nhau qua các ranh giới độ trừu tượng, thứ khơi dòng đẩy mạnh nhóm check kỹ xảo đi qua mặt lớp giới hạn "hộp đen".
- Kèm lời thoại, giả có nguyên trang web ngậm mặt API lẩn tít tận đuôi. Hớ, mắc giống ôn gì mà chả móc 1 cái "Đặc Tả (spec)" đi chọc vô đu đọt cho cả hai ta? Tùy quyền thọc sâu một màng nặc danh web browser táng web page, cùng mấy cuốc HTTP đấm đá tơi bời qua cổng API coi.
- Đẩy nước kiệu cho kiểu mường tượng ngông ấy, đúng chuẩn lý tưởng dâng cao, tôi hằng muốn tạc **mã nắn vóc dánh Essential complexity** (dạng code miền- "domain") nương đó tha hồ tống đống specs vô rổ của bọn unit tests sài ké. Hái được quả bự feedback đánh vỡ kén báo tin hỏa tốc cho ta rành là mã nguồn nội tại vướng cái khỉ gió cốt cán ấy nó được tạc tượng y đúc cùng đi y nhịp chả trật chân.


### Tính khắt khe đổi hình vạn trạng các kiểu ATs - cho cớ duyên phải lẽ (Acceptance tests changing for the right reasons)

Theo bước rẽ đấy, lối thoát chật nhất rủ rê bản specifications phải tuốt tuột lại đó là hồi mà tánh nết của ứng dụng bị nắn gãy, nghe êm ái thuận nhĩ vãi nhỉ.

-  Trường hợp kho API bạn bẹo mình, rành rành đích tới chổ duy nhất cho bạn khoét rảnh là update vào bộ óc, củ điều hướng hầm hố - the driver.
-  Còn hở Markup có đổi trắng đen, hệt dứa, sấn tới thằng lính gác bóc củ kia (specific driver).

App càng phổng phao bự chảng, dần mòn bạn ngộ nhận ra thói quen tái dùng một bọn drivers xài hoài dăm kiếp bọc vô mấy ổ tests, quy luật trên tái diễn: Mổ xẻ vặt vãnh ruột gan hệ thống có giông tố bão từ thế naò chăng nưa, đích mục tiêu duy duy nhất cần rà có một điểm lòi hiển nhiên thôi.

Nếu làm ngọt trơn mâm này, lối tạt ngang sườn nhúm này biếu bọc đặc ân độ cơ bắp của khâu móc ráp implementation detail hòa với hòn đá vạn năng kiên cố bám chót của hệ rễ specifications. Dạng tiên quyết cần hô, cách đánh này nặn ra cái phom tổ chức đi theo mô-tuýp siêu đơn sơ và lộ diện ngay mấu cụt để ôm quản mọi cuộc xoay tuồng biến chấn (managing change), thứ quyết mệnh thiết yếu định tuổi cọp giữa bầy khi bầy to phình ra.

### Đi bài lôi Acceptance tests múa phép trị thuật nặn app (Acceptance tests as a method for software development)

Trình bày trong màn live, tôi với Riya cũng lột tung rách trò ATs qua sự gắn kết vô mảng môn đạo BDD. Tụi tôi móc họng vụ đem việc khai trận nhào vô mâm với phép đánh _hiểu sự lý cho được cớ mà tụi bay muốn cày_ thành quy tập cho việc táng ra cho xong cuốn sớ specification gánh đội lấy đầu não mục tiêu nhắm thẳng và lại được xướng lên cho khâu xướng đao đẹp bảnh mở màn cho việc chi ra lệnh code.

Chính tui từng được dẫn đạo vào bài tủ cho con đường nhọc nhằn này ở trỏng quyển bí kíp GOOS. Cỡ khuất mặt lâu nảo nảo, tui từng vuốt nhẵn mấy mảng ý lớn đem ướp vô blog tui.  Nhấn link nếm mẩu từ bài viết cưng [Cớ chi xài TDD/Why TDD](https://quii.dev/The_Why_of_TDD)

---

TDD chúi đầu mở lối để rảnh tay nặn ra hệ vận hành ngắm trúng chóc hành vi dĩ nhiên phải ngắm, theo vòng lặp tuốt lại liên hồi. Chân ướt chân ráo chen vào bãi chăn chiên chóp này, điều phải nhét vô não là dòm đâu là xương sống điểm chốt hệ ứng xử cốt lõi và dẹp mẹ nó tính nới rộng chức năng tầm phào cho rảnh nợ.

Dò theo chiều gió "từ nóc xuống móng" (top-down), lấy cớ rước vị trưởng bối mở hụi đầu - bài bài AT bọc cái trò thử ngón hành xử từ mé rào bên phía ngoại xào luộc vô trong. Cái ông cụ ấy đĩnh đạc xưng trùm cầm trịch ngôi sao "Đẩu-bắc" ("north-star") dò dẫn đoàn tàu công sức. Phải căng nhịp tập trung não bộ lôi sức xích sao cho ông cụ mỉm cười qua cưa (pass). Cái bài này có xu hướng ráng nấp vào màn fail chỏng chơ một đỗi thời gian khi mà mông chưa nóng để chắp đủ tay kiếm đục đẽo code xịn đẩy nó vạch cởi màu xanh.

![](https://i.imgur.com/pxTaYu4.png)

Hạ sinh thành công bố cụ AT vỗ tay rồi, được đặc quyền lộn vòng cho bể nát trò vờn TDD theo mạch để túm hớt cả mớ lính trinh sát phụ (units) đặng khua xô ông cụ qua rào rinh giải thưởng pass. Độc chiêu chốn này nằm gọn cái điều thôi bận lòng đến chéo mắt dòm cách uốn nắn mô hình cấu trúc code ra sao nhức sọ (design chướng khí); lấy tay nắm bắt nạp cho thấu mấy xâu mã cốt qua chóp ải của AT đã vì tại khúc quanh này tâm trí huynh còn lo ngơ chập chững rà mìn để sành mưu kế dò địa lợi cho ván bài.

Tung đòn bốc con chốt trước nhất nghe chừng bồ hóng dọa văng dài lê thê hơn độ bạn nghĩ nhiều, quăng đống cài vặt cho máy trạm web, phà đò luồng rẽ định tuyến routing, ráp mớ cấu hình, v.v.., từ đó làm bật lên nguyên lý cọc neo là xiết chặt mức hạn thắt cửa nẻo dự việc lại là một bước sinh đồ tử. Sáng mắt nhắm nặn nhát sủi đắc đạo tiên giáng trần chễm trệ khai phím trên nắp bảng tranh xóa loáng và nắn thành lũy phò ta đặng cụ cụ phẩy cờ pass đi để có thế nước thúc pháo càn cật qua các đợt đụng chạm gọn nheo dĩ không rách giáp (iterate quickly and safely).

![](https://i.imgur.com/t5y5opw.png)

Lúc nhả mã, vểnh tai hóng động báo về từ mảng test nhà, tụi lính trinh sát đấy lanh lợi nháy đèn tín hiệu xui rủi cho bạn đụng tới để chuốt vuốt lớp nặn (design) thành đòn có sinh khí tráng lệ ngon rơ hơn tuy vậy, xin điểm rạch rọi cái ghim ngón đó trúng chóc vào luồng hành xử chứ đếch phanh cho màn tưởng tượng xé rào vút mây nha.

Thường thì cái gọi "unit" bé nhất lãnh vai lính quèn quăng mình gánh nhục đỡ lấy cả mạng tát nước vào AT cho pass đâm thành khệ nệ đẫy đà nhai cục mệt lừ, hẵng lúc ấy kể cả chui vô dăm miếng cắn tính năng nhỏ quạt vất. Sự thể điểm này chỉ ra điểm kích nổ bạn tính nước thả lỏng đai cục diện hòng chẻ nhỏ ván và ngó nghiêng gạ gẫm lũ cò làm chung nương tay.

![](https://i.imgur.com/UYqd7Cq.png)

Thời mạn mâm này hớt ngay lũ xà thủ tráo khái niệm cộm mác đóng thế (test doubles, ví như hàng fake, nộm bưng/mocks) xài bao được do 90% bọn mớ phiền toái vướng vấp bên trong nạm mảng phần mềm không dính móng đến hệ cấm cương ruột implementation detail mà nằm chen "ngoài/giữa" mấy bọn lính unit phụ rồi thọc lét vụ chúng nó mớm cho nhau hoạt động cớ sao đấy.

#### Ngứa chân đá từ dưỡi đáy (The perils of bottom-up/Hiểm họa của Bottom-Up)

Rõ chi tiết là đánh đòn nhắm thẳng "trên giáng xuống" (top-down) thay đổi "từ đáy thọc ngược" (bottom-up). Đánh bottom-up giấu tài riêng nó chớ, thế nhưng chứa độc mầm làm gãy tay lật kèo bao ăn được. Từ khâu rục rịch dựng bọn lính "nhúm dịch vụ/services" và đẩy code nằm đó chứ chưa hề hàn gắn nguyên mảng vào đống nợ cái cốt của application ngay hòng lặn tránh vòng dò xác mác kiểm nghiệm bậc "chúa trùm" cao độ, **nghĩa bóng nói rằng bạn đu dây đùa cùng lửa phung công dã tràng uổng mạng mà đeo khát khao từ đám thợ ý tưởng lậu thuế**.

Là mảng tối trọng cho việc móc xích chiến thuật đánh phủ đầu mượn AT, mượn cớ vùi mã chốn vòng Test rà soát độ uy lực của mình.

Vướng nhan nhản nấc, tôi ngả mũ đụng trúng nhiều cụ quăng ra mảng code liền một khối, phơi chốn 1 mình, khui bài bottom-up bốc số giải họa với mộng ảo nó vá được lỗi lầm trần thế, song mà lúc soi lại thì:

- Hành dân chả theo nhịp mong hóng
- Đội 1 cỗ máy thừa xịn nhưng đách xài chi (Does stuff we don't need)
- Đẩy ráp ghép khóc tiếng mán (Doesn't integrate easily)
- Ngậm ngùi lôi bàn gõ mòn mỏi cái rả rích cào gõ viết lại cho đứt tay cày (Requires a ton of re-writing)

Cái ấy mới đích thị quăng xu qua cửa sổ (This is waste).

## Tới đó thôi cất mỏ khui bàn phím hốt Code nà (Enough talk, time to code)

Kể ra thì khác xa phần còn lại của bộ sách, bạn sẽ rất cần hốt 1 máy [Docker](https://www.docker.com) vác về dựng sẵn vì tụi mình nay xài phép nhét code vào hoạt động trong buồng chứa containers. Bắt đầu từ mốc của chương sách này, nhẩm bụng là bạn rành cái chuyện lạch cạch gõ code Go, khéo gọi thêm đồ chơi (import) từ đủ loại packages ngoài, vân vân mây mây rồi nhé.

Sắm sửa đồ nghề làm cái project khai mạc với câu thần chú `go mod init github.com/quii/go-specs-greet` (đặt tên láo nháo gì ở cái đuôi cũng êm nhưng lỡ có nghịch dại sửa cái đường dẫn (path) thì phải chịu khó chạy lùng sục chỉnh tiệt các thể loại imports ở bên trong code cho ăn rơ nghe)

Đẽo 1 chốn chứa thư mục `specifications` thủ sẵn làm ổ đẻ đặc tả kỹ thuật, nhồi 1 file `greet.go`

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet() (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet()
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, world")
}
```

Trợ thủ phòng máy gõ IDE của lão đội mũ (Goland dâng hiến) bao chầu gánh vụ lọ mọ lượm nhặt mấy hàng dependencies dùm mỗ, dưng cơ mà bạn lâm cảnh vác cuốc chạy bộ (do it manually), thì xả dòng chiêu thức

`go get github.com/alecthomas/assert/v2`

Dựa trên kim chỉ nam họa đồ Farley's ở mảng thiết kế (thứ mà chạy từ Specification->DSL->Driver->System), vậy ta đang rinh về bản chép tay spec bóc lìa khỏi implementation. Ta có thèm ngoái cổ bận tâm về màn chào hỏi _như thế nào (how)_ (chỗ `Greet`); spec ôm việc đoái hoài trọn sự Essential complexity (bản chất phức tạp sống còn) tại miền lãnh thổ (domain) của dân kinh doanh. Công bằng mà nói chút hóc búa nhỏ nhoi này chưa bỏ bẽn dính dấp chi lúc khai mạc, ngặt nỗi rồi lát ta gồng mình giãn cơ gân đắp thêm hàng tá các món chức năng cho mập bộ spec khi vòng lặp luân phiên kéo tới. Quy tắc vỡ lòng là làm mồi nhứ nho nhỏ trước đã vạn điều tốt!

Quý bạn đủ bản ngọc để khoác áo cái `interface` nọ sắm vai khai quốc làm bước một trổ lóng DSL; sau này ứng dụng bự như voi, đôi vờn khốn khó bắt ta nặn ra mấy hệ trừu tượng abstract ma quỷ điên khùng khác, bù lại thì cho đến giờ này, vầy có vẻ ổn đấy.

Tại mốc chạm ranh này, công cốc bày vẽ ra vẻ sành điệu tháo ghép mảng specification thoát cái rọ implementation dấy lên việc lắm nhân tố rỗi việc ném đá chê ta bị nhiễm mảng bám của hội chứng "lậm trừu tượng ảo tưởng - overly abstracting". **Tôi hứa bán danh thề độc, những thể loại ATs mà bọc kín bưng chặt hẽm cùng khâu ráp code (implementation) một lúc rồi chồm lên hóa thù nghịch đè ợ cổ nguyên 1 bầy anh em đội thợ engineering team thôi**. Tao tự tin quả quyết nhan nhản trên mặt sân chốn rừng rậm (in the wild), 90% các bản bài ATs khiến team mếu xé xỉa bộn tiền cho mảng vỗ béo (maintain) chẳng qua cớ cất nọc do tụi nó dính dằm vào nhau loạn xa ngầu ấu trĩ không gỡ ra nổi; tỷ lệ lỗi "gồng người ảo vọng hóa trừu tượng" chả là cái đinh gì.

Giữ trong tay vũ khí spec chốn đây, tha hồ ngả nghiêng đặng check nghiệm cả mả các loại "hệ thống" nào đó tài tình trong tay thốt được tiếng lóng `Greet`.

### Mặt trận dạo đầu tay: Sân API loại HTTP (HTTP API)
Bọn tui được giao kèo mớm 1 "em dịch vụ đứng bốt chào đón- greeter service" ló cái mặt trên đường truyền HTTP. Giao lại mớ chuẩn bị sắm đồ nghề:

1. Chưng diện 1 cục điều hướng **driver**. Phục vụ màn này, anh lính mang theo sẽ chơi chọc đấm ngón đòn của HTTP qua chiêu trò **HTTP client**. Code dạng này sõi cái bài cách móc ráp chơi cùng hội API của quân tao. Đủ tụi Drivers kiêm nhiệm tay bưng dĩa luỵt biến đổi thứ tiếng dân dã DSL thành dăm ba cái mã gọi dâng tới chầu từng thể hệ thống chuyên; tại ví dụ nhúm nhỏ mình xài, anh bưng dĩa nọ mang thiên chức rặn ra hiện thực cho cái quy định `interface` lấp lửng trong mặt specs.
2. Quả cọc **HTTP server** kèm sẵn nẻo API greet trỏ ra đường cái
3. Thằng cha **test**, chễm chệ khoác việc chạy thong dong lèo lái hệ vòng đời tự động sắm sửa nổ nhào nặn 1 máy trạm web, châm vòi nối thẳng bé đầu não driver lọt tròng vô thân bản thiết specification đặng chạy cuốc trơn như một lượt test sướng tay.

## Gõ lấy mớ test trước (Write the test first)

Cuộc hành trình vạn dặm chặng ban sơ nắn tạc một gã thám tử giấu mặt (black-box test) đủ phép tự động đúc mẻ build, lên đạn nã test đùng đùng dọn dẹp nguyên vẹn mọi dấu vết có tính hao tâm tổn xác cày như quỷ, lao động nặng nhọc (labour intensive). Bữa dị, vác mảng này qua chơi từ bước khai vạch của chu kỳ dự án với sự đơn giản là mưu đồ ngon xơi. Riêng anh hùng trẫm (tôi á), bao mùa dự án nổ máy cạn ly tao dọn lẹ lót ổ bằng màn máy chủ gáy gọn "hello world", nhồi cả lô test chầu trực trong phom sẵn cờ tao ấn nút quẩy nặn đám functions nòng cốt 1 cái ào ảo diệu luôn.

Bộ thiết kế kiểu nặn trí tưởng tưởng với 1 ngùi thứ gọi mộng ảo "specifications", "drivers", cho cả nùi "acceptance tests" vướng xíu thời gian để dạo não ngấm thói quen, chi bằng lướt rạch ròi thận trọng. Gợi mưu là "vô lùi đi lui (work backwards)" xông phá gõ gọi spec lên thớt chơi thử xíu đã.

Chi đi mấy phát rạch cấu trúc cho việc xá tội mảng app program định cất thành kho chuẩn bị cho trôi ra ngoài xài.

`mkdir -p cmd/httpserver`

Mọt vô trong, rặn ra đứa đuôi dóc mới tinh `greeter_server_test.go`, rồi sao chép nội công tiếp nha.

```go
package main_test

import (
	"testing"

	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	specifications.GreetSpecification(t, nil)
}
```

Ý tứ lù lù rằng phò nguyên cụ specification đem vô 1 cuộc Go test thông lệ. Tay cầm vũ khí sẵn cái thớt `*testing.T`, ấy dưng mà ở khúc cái argument đi thứ 2 làm quái gì nghen?

Đúng cãi `specifications.Greeter` kia đích thực cái interface, anh hùng tao rước 1 em xịn mang tên `Driver` làm cục thế mạng đặng chèn vô ngõ thay cái `TestGreeterServer` đang ngập não kia bằng mã dưới cờ:

```go
import (
	go_specs_greet "github.com/quii/go-specs-greet"
)

func TestGreeterServer(t *testing.T) {
	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

Xài món `Driver` ấy có cái chảo mỡ ngon ăn thay đắp cho nó tha hồ nhảy loạn đâm đánh qua các tụ rẽ đường trường khác (different environments), đếm kèm chạy nhà máy test rạch địa đạo (locally), thòng luôn 1 cửa `BaseURL` bọc gàng gàng nha anh em.

## Phóng test thử cái nghen (Try to run the test)

```
./greeter_server_test.go:46:12: undefined: go_specs_greet.Driver
```

Vẫn còm cõi lượn múa theo dòng TDD chớ hề phản nha! Nắm đầu một xải nhảy dài cực bự ta có bổn phận lết: xoắn khui mở lắm tay lẹ ngón nặn file dăm cái phớt bồi mã code độ dày húc mặt so việc tay chân chai sạn hay quẩn, bù lại lúc vạn sự khởi đầu, đi đâu cũng vướng mắc lối này chớ hiếm. Rắn gân tuân đúng phép chỉ gạch đầu quy luân điều luật (red step's rules).

> Viết lách bất chấp bùn lầy bao lầm lạc nhơ nhớp trọi, ép ép làm sao vớt cú test nhích lên điểm xanh gật gù là mâm này qua cửa (Commit as many sins as necessary to get the test passing)

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Ráng mím môi bịt sống mũi nịt vào; dặn hồn mình tĩnh, hên cái quật test thành xanh tao quay xe rũa dũa lại (refactor). Cái tròng code phím của tụi `driver` dưới rốn của hệ màng cắm đất `driver.go` ta dọn thảy lên nắp đầu của cây đũa chứa (project root):

```go
package go_specs_greet

import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
}

func (d Driver) Greet() (string, error) {
	res, err := http.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```


Bỏ ống mấy điều nhắc vặt:

- Tụi mày rảnh kháy mỡ kêu sao anh trai chơi liều ko rặn unit test bóc tẽ tụi xót khống này với đám mớ đai lệnh kiểu `if err != nil`, gật mà xét kinh độ già giê qua năm tháng đời tôi trỉa, đâm hụt lúc cái anh chàng `err` chít xó ko ngốn miếng cơm việc chi, nhét đòn dĩ tróc bài dạng "trả phang ngược đúng cục phỉa mày ăn mớm gieo rắc" đem xét coi nó èo uột nhạt độ chưng giá trị dười đáy (low value) hời.
- **Có chết cũng cạch cái default HTTP client mà Go quẳng nha**. Quẳng cọc đấy lát nữa bổ sung tay hụ HTTP client cài cấu hình mắm muốn qua trò đếm lùi giờ giấc (timeouts), chớ còn tầm cấm kỵ bây giờ rành rành ráng kiếm xanh bảng báo mâm cúng test cho trơn chu lướt.
-  Phơi áo chỗ mục `greeter_server_test.go` gọi kháy thẳng mặt bóc hàm nặn Driver của khu `go_specs_greet` anh em dọn lúc nãy xong nhoe, nhớ đừng có lộn cởi đem rải thêm tay trôm `github.com/quii/go-specs-greet` dính chấu hạm `imports`.
Ráng gồng nhấp lố cho test bung phát rực lửa; lạy chời quả này build ăn được chứ ko trót đậu.

```
Get "http://localhost:8080/greet": dial tcp [::1]:8080: connect: connection refused
```

Dù rằng con ghẻ `Driver` ta bắt sống, ấy thế chưa mớm gas cho pháo ề rồ application sập nổ, cơ đấy cuốc gọi HTTP lót vũng dậm tịt ngòi. Nhu cầu thúc bách đòi bài nhòm acceptance test của chúng sinh kiêm luôn chậu phối trộn múa mâm cúng kiêm cả bẻ cành (build), gáy chạy và cái chốt chém mạng sập gã "hệ thống" mình ngóng tới cho khâu chép test qua màn này.

### Làm lễ phát nổ chạy mớ app của ta (Running our application)

Sông đổ về biển chuôn chuyện gã khổng lồ vẩy đuôi Docker múc đám thùng image chòng nhét trọn chậu rương nháo application để liệng lên mây bung chóp deployment, thớt ngõ này phết test mình chẻ khuôn cũng mần y hệt mớ hành tung y.

Cáp dính vớt cọc xài tay Docker trong xới tests này, chiêu tung lên gọi tay chi viện [Testcontainers](https://golang.testcontainers.org). Thằng công cụ Testcontainers ném một cái xẻng xúc cát có tổ chức tựu chệ programmatic lấp vá cho ta vùi nhét cấu cạn cho thàng nhõi Docker images rồi ôm quản nhịp sinh lão bệnh tử cho cái buồng xài container.

`go get github.com/testcontainers/testcontainers-go`

Bay zô ngốn bóc củi thớt `cmd/httpserver/greeter_server_test.go` qua mảng mới đọc dẽo như đoạn dưới đây nghen tía:

```go
package main_test

import (
	"context"
	"testing"

	"github.com/alecthomas/assert/v2"
	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func TestGreeterServer(t *testing.T) {
	ctx := context.Background()

	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "./cmd/httpserver/Dockerfile",
			// set to false if you want less spam, but this is helpful if you're having troubles
			PrintBuildLog: true,
		},
		ExposedPorts: []string{"8080:8080"},
		WaitingFor:   wait.ForHTTP("/").WithPort("8080"),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})

	driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080"}
	specifications.GreetSpecification(t, driver)
}
```

Thử chạy test coi bay.

```
=== RUN   TestGreeterHandler
2022/09/10 18:49:44 Starting container id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Waiting for container id 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
2022/09/10 18:49:45 Container is ready id: 03e8588a1be4 image: docker.io/testcontainers/ryuk:0.3.3
    greeter_server_test.go:32: Did not expect an error but got:
        Error response from daemon: Cannot locate specified Dockerfile: ./cmd/httpserver/Dockerfile: failed to create container
--- FAIL: TestGreeterHandler (0.59s)
```

Nợ nần đè cổ ta phải nặn 1 bản `Dockerfile` ráp cho lọt cái app program của mình. Nhét mình vô lòng folder `httpserver` mới tạo, nặn cục `Dockerfile` rồi tống nội công vào:

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
# For example, golang:1.22.1-alpine.
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/httpserver/*.go

EXPOSE 8080
CMD [ "./svr" ]
```

Cho chừa đi bộ não ôm đồm lo bò trắng răng gặm nhấm tiểu tiết (details) nhe; ta dư sức đánh bóng vót dáng gọt tỷ lệ tối ưu chớ còn rờ nhẹ lúc này, nãy đủ êm rồi. Lợi điểm bá chấy từ tay đòn này là cữ sau được gỡ đai nâng cơ ngực Dockerfile thành quỷ thần khóc vẫn vác bản test này nện dò được coi nó còn dẻo dai chạy hay nát không. Khắng định đây chính xác là thứ đặc quyền chói lọi cộp mác hộp siêu năng test (black-box tests) vác vào!

Rảo tay thử test 1 cữ mới; nó nấc lỗi méc rục ràng rằng cãi chuyện build nhét image hỏng từ đời tám hoánh rồi. Hiển nhiên nghen! Mình vốn đẻ cái quái gì ra cho nó đi build đâu mẻ!

Nghe này, đẩy lùi bản test thành trận chạy ầm ập, ta đang sắm 1 trò program thầm lặng ngồi móc gáy lắng nhịp lỗ `8080`, **song nhiêu đấy thôi nhóe**. Ôm tri kỷ với thói quen TDD (TDD discipline), tuyệt đối kiềm chế xả hàng súng đạn xách code rổ production làm bản test xanh hóa trong khi vẫn bưng bít chưa thấy test nó sập fail kiểu mình lường rành.

Nặn nhúm tẻo file `main.go` rúc lòng thúng `httpserver` lôi mấy từ này thả vô:

```go
package main

import (
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(writer http.ResponseWriter, request *http.Request) {
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

Quấc chạy đấm ngực test xem đụng mã vấp fail sau nhé bạn:

```
    greet.go:16: Expected values to be equal:
        +Hello, World
        \ No newline at end of file
--- FAIL: TestGreeterHandler (2.09s)
```

## Viết đủ code để test chạy thành công

Sơn phết gỡ rối tay hàm handler hòng bắt nó cư xử dẻo dai như yêu sách đặt lóng nhóng ở specification nhóe...

```go
import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	handler := http.HandlerFunc(func(w http.ResponseWriter, _ *http.Request) {
		fmt.Fprint(w, "Hello, world")
	})
	if err := http.ListenAndServe(":8080", handler); err != nil {
		log.Fatal(err)
	}
}
```

## Gọt lại mâm chảo - Refactor

Dẫu rằng xát muối phô cái mác khâu này chẳng đúng gốc bản (technically) gọt code (refactor) đâu, ấy cũng là lời răn, ta cấm lợi dụng đai cấu cái thằng HTTP client vớ vẩn rởm rởm, phới tay nặn cục đính kèo vô bộ Driver, hòng kiếm thế cấp cho 1 cục, rồi con gà đẻ ở bản test lấy hốt mớ.

```go
import (
	"io"
	"net/http"
)

type Driver struct {
	BaseURL string
	Client  *http.Client
}

func (d Driver) Greet() (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet")
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

Qua địa bàn phết test chui hẻm `cmd/httpserver/greeter_server_test.go`, sấn sổ ném vào bộ tay nặn cục quẳng gánh 1 gã sai vặt client.

```go
client := http.Client{
	Timeout: 1 * time.Second,
}

driver := go_specs_greet.Driver{BaseURL: "http://localhost:8080", Client: &client}
specifications.GreetSpecification(t, driver)
```

Giữ cái nếp tút bộ não gọn nương cục `main.go` thành thứ phèn nhất mộc nhất khả dụng nhé huynh; cái mảng ấy mồm ngậm duy mỗi thiên chức gò nối, hàn cọc gạch các kiểu gạch nền móng lại mà dệt lên lụa hệ thống application.

Soạn 1 nhúm thư mới hất ngang vách chứa (project root) chễm chệ khoác áo `handler.go`, ôm mớ nạm cũ của mình thả bịch vô.

```go
package go_specs_greet

import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello, world")
}
```

Lại lốc nắp `main.go` cạy ngoéo import và đôn lính lôi móc handler qua chõ.

```go
package main

import (
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet"
)

func main() {
	handler := http.HandlerFunc(go_specs_greet.Handler)
	http.ListenAndServe(":8080", handler)
}
```

## Thẩm lại vị - Reflect

Bước chạy rồ ga có phần tốn lực. Chúng ta nắn đẻ đủ các loại files xài đuôi .`go` đặng uốn nặn với gáy thử bài kiểm một thằng điều binh HTTP handler với công dụng chả khác vứt 1 cuộn chuỗi tĩnh (hard-coded string). Dù ẻo éo "vòng lặp đúc tiền thân số 0" cồng kềnh đầy lễ nghi với bộ thiết lập này lại phất cánh che tai phục dịch chu đáo lúc ta guồng chân cho mấy cua bẻ "lặp" (iterations) tiếp tới.

Nắn đường đổi dáng của chức năng nay chẳng nhọc như lên trời, có khi chỉ là cớ dắt lái qua con xe specification rồi chịu đấm ăn xôi dàn xếp mới rắc rối vây quanh mà nó cắn giật lôi tớ làm chung. Giờ đã yên vị sắm bản thiết đồ thầu `DockerFile` cùng hội `testcontainers` đóng phom cỗ cho bản test acceptance test rịn lên đài; tụi bây đâu hụt tay mò mẫn đếm xỉa mấy cục gạch nọ đặng không chường xui có cú xỉa nào nắn cấu gãy bộ cọc móng xướng lên hệ thống app đâu khứa nhỉ.

Bọn ta nhẩn nha xem rành rọt ở khúc yêu cầu bắp nối ngay ngõ, léo héo gọi tên "chào bác nào đấy sắm tên cụ thể nhe".

## Viết ngay phần nã test nào (Write the test first)

Uốn ngón gõ vài cái vào specification nào:

```go
package specifications

import (
	"testing"

	"github.com/alecthomas/assert/v2"
)

type Greeter interface {
	Greet(name string) (string, error)
}

func GreetSpecification(t testing.TB, greeter Greeter) {
	got, err := greeter.Greet("Mike")
	assert.NoError(t, err)
	assert.Equal(t, got, "Hello, Mike")
}
```

Nhắm vọt bước thả cổng đón rước đích danh ngài lớn, ta tính mưu chỉnh giao thức interface nhắm tới app cắm ngõ họng thâu tóm vào nắp mồi `name` nheng anh em.

## Sờ cái test coi lên khói không (Try to run the test)

```
./greeter_server_test.go:48:39: cannot use driver (variable of type go_specs_greet.Driver) as type specifications.Greeter in argument to specifications.GreetSpecification:
	go_specs_greet.Driver does not implement specifications.Greeter (wrong type for Greet method)
		have Greet() (string, error)
		want Greet(name string) (string, error)
```

Giông lốc đổi thay quật qua nấc bệ specification nháy hệ lụy móc theo thằng bé driver kia bị đòi gào tên gọi phải nặn lại rồi.

## Rẽ bẻ lượng code teo tẹo đủ nhét kẽ chạy test và soi cái thẹo (lỗi)

Tút lại phần ngón trỏ cái driver nên cho nó dằn bụng đem gửi gắm cái chữ biến đi kèm rễ gáy truy vấn (query value) tên là `name` núp trong khâu yêu sách (request) rải ra hóng đón tiếng gọi định danh được cất lên.

```go
import "io"

func (d Driver) Greet(name string) (string, error) {
	res, err := d.Client.Get(d.BaseURL + "/greet?name=" + name)
	if err != nil {
		return "", err
	}
	defer res.Body.Close()
	greeting, err := io.ReadAll(res.Body)
	if err != nil {
		return "", err
	}
	return string(greeting), nil
}
```

Sới Test phỏng nã đã bớt ngọng mà lao vút được, mớm cục tức (fail) về tay.

```
    greet.go:16: Expected values to be equal:
        -Hello, world
        \ No newline at end of file
        +Hello, Mike
        \ No newline at end of file
--- FAIL: TestGreeterHandler (1.92s)
```

## Rắc xé bốc hốt đủ code mớm nó đậu

Khe khẽ nạo tróc tên gọi danh nghĩa `name` dội từ request ra dọn dĩa đi mời khách.

```go
import (
	"fmt"
	"net/http"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello, %s", r.URL.Query().Get("name"))
}
```

Dĩ tiến thế, bài test qua nhún đài đậu đỗ vèo vèo.

## Gọt lại mâm chảo - Refactor

Khoa mục [Thăm lại cõi HTTP Handlers (HTTP Handlers Revisited),](https://github.com/quii/learn-go-with-tests/blob/main/http-handlers-revisited.md) chúng ta có phân bua căng đét về cái sự cực kỳ xương sống bảo kê HTTP handlers ôm rịt việc của chuẩn HTTP và miễn nhiễm mấy thứ khác; mớ nào dính mác "domain logic" phắn ngay ra rìa ko núp váy handler. Tuyệt kỹ này tiếp bước dọn phòng thênh thang để phát triển cục domain logic tách bạch cùng HTTP, hô biến khâu test với vỡ vạc mã thành điệu múa cực duyên.

Giờ rút mỏng dính các khâu chéo ngoe ấy (concerns) ra ha.

Bẻ lại miếng handler trong ruột `./handler.go` nhứ phía dưới này:

```go
func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, Greet(name))
}
```

Nặn ngay miếng ngói rỗng `./greet.go`:
```go
package go_specs_greet

import "fmt"

func Greet(name string) string {
	return fmt.Sprintf("Hello, %s", name)
}
```

## Dạo mót chút chéo sang nẻo pattern "cục chuyển đổi (adapter)"

Tới mốc dứt điểm hất mảng logic thuộc miền domain, ở ngón nghề vẫy khách chào thưa văng khỏi chức phận khác thành gã Function tự do, thì anh em tao mẩy cờ trổ Unit Tests (unit tests) phấp phới bắn vào gã Greet rỗng kia chớ ngại gì. Rõ 1 phát là vạn phần êm đềm bớt hóc búa vạn trượng nhò phần so le đẩy lệnh nã thông ruột gã specification vác 1 lão driver đấm đá sập sàn nhà phát mạng server hòng lượm vớt mỗi cái chuỗi chữ tĩnh string về tay.

Cầm cưa mướt ruột không khi xót 1 bọc rỗng mình bươi lại cuốc specification thảy qua chơi tiếp chốn này? Chung quy vòng vòng, rún định của cành specification san sát xào chẻ rịt phớt sụ che phần rác của ruột mảng tháo ráp (implementation details). Nếu specification có sức đớp nhọn nọc lõi **Sức sống phức độ (essential complexity)** bên cạnh đám code phèn nạm danh "domain" của anh em lính tao lãnh xướng mệnh vác mác nó, hiển nhiên có thể kéo dắt phò tụi nó nương tay cặp gáy với nhau.

Quật thử cờ bạc nặn lấy `./greet_test.go` dề đây:

```go
package go_specs_greet_test

import (
	"testing"

	go_specs_greet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(t, go_specs_greet.Greet)
}

```

Mộng quá mơ, cơ mà ọc ọc nó chẳng lăn êm.

```
./greet_test.go:11:39: cannot use go_specs_greet.Greet (value of type func(name string) string) as type specifications.Greeter in argument to specifications.GreetSpecification:
	func(name string) string does not implement specifications.Greeter (missing Greet method)
```

Cuốc gọi specification rên lấp lửng vòi một thực thể chứa sẵn 1 bộ ngạch method `Greet()` chớ éo thèm cục function trọc nghen.

Văng cái thẹo dịch build hụt cay dái thiệt thòi; có đinh đinh trong đầu 1 củ bảo mang dấp dách chức phận `Greeter` rồi, ác nỗi chẳng trúng vô trọn vẹn **khuôn múa (shape)** hòng vuốt mắt gã compiler ngu nới cửa. Đó đây khắc tên cứu tinh - chước độn thiết kế lấp cọc **đệm chuyển - adapter**.

> Nhét đầu vào phom học thuật môn [cơ khí linh kiện máy phần mềm (software engineering)](https://en.wikipedia.org/wiki/Software_engineering), cục **pattern đệm chuyển (adapter pattern)** tức là 1 bộ chiêu số [kỹ cương thiết kế phần mềm/phom mẫu](https://en.wikipedia.org/wiki/Software_design_pattern) (hay quen ợ với tên mác bọc bánh - [wrapper](https://en.wikipedia.org/wiki/Wrapper_function), tên lai vãng có tí dây thun chung mâm hụi bên giới thuật [decorator pattern](https://en.wikipedia.org/wiki/Decorator_pattern)) tha bổng cho 1 mành quy ước [interface](https://en.wikipedia.org/wiki/Interface_(computer_science)) của 1 cái [đế class](https://en.wikipedia.org/wiki/Class_(computer_science)) có sẵn rắp vô xài ké nương tay theo 1 cục cổng interface khác.[[1\]](https://en.wikipedia.org/wiki/Adapter_pattern#cite_note-HeadFirst-1) Chiêu lòn này luôn sài hòng bẻ gáy mấy lão class cổ lổ chui luồn cạp nạp chung mâm các bạn già hòng không xót xạc tróc vỏ đám [cốt cơ mã nguồn (source code)](https://en.wikipedia.org/wiki/Source_code) ra bâm vầm.

Mả cha nhồi cả cụm chữ sang choảnh bọc chung miếng rẻ quạt trơn tuộc, lại bảo sao ba cái khía cạnh nặn phom kiến trúc dấy đội quần hội anh em thợ chán rầu vãi nhái. Chìa khóa giá ngọc cho mớ pattern thiết kế nọ hổng bấu vào tầng chắp mã nhúng thẳng cơ đéo, ấy lót nhung ngửa tiếng kháo từ vựng quy chung vạch ra nhúm biện pháp chung chung đánh đám loạn xèng vấp bão thường ngày ở đám kỹ sư nghênh chiến. Kiếm chác vô tay một hạm có đính chung một băng đạn ngôn ngữ, nếm mật gai giảm đi phân nửa lực lúc thông mỏ phọt nước bọt cùng sếp lính (communication).

Quăng rác mã xuống cục file trỏ mục `./specifications/adapters.go` 

```go
type GreetAdapter func(name string) string

func (g GreetAdapter) Greet(name string) (string, error) {
	return g(name), nil
}
```

Dư lày ta đường hoàng lượn cục đệm adapter tại màn xé test hòng lấy cái gáy cắm function `Greet` của mình chọt dính tróc Specification vô tư rùi con.

```go
package go_specs_greet_test

import (
	"testing"

	gospecsgreet "github.com/quii/go-specs-greet"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreet(t *testing.T) {
	specifications.GreetSpecification(
		t,
		specifications.GreetAdapter(gospecsgreet.Greet),
	)
}
```

Miếng đệm Adapter ăn dầm tiện mút lúc bồ vác 1 ông trùm hình hài type đong đầy mâm đạo vị (khuôn xử sự - behaviour) như vòi kháy của cổng interface thèm ngặt nỗi vấp gãy mỗi dáng vóc hổng thon chuẩn thôi.

## Thẩm lại vị - Reflect

Phập xẻ thay tính nết ngó trơn cmn trượt há? Ờ hé, khéo do duyên trời ngõ vắng ăn hên ở sự lở loét (nature) con đẻ bài toán, dưng lết mảng mòn việc móc nối trù thế này thụt cấp kĩ nghệ chốt luật gò ép anh vào kỷ luật thép và chìa củ sâm siêu dễ xực chót thay hoán hệ trục máy lướt mướt trần tới ngầm (top to bottom):

- Mổ phanh bệnh nhân ra, rờ lách phanh châm ngòi nặn ti tí cọc mảng hệ thống sao cho nó bứt dây chắp đà đánh tới 1 luồng bẻ đích phơi phới chuẩn bài
- Xiết chặt thòng 1 mạng độ dày sự cố (essential complexity) khéo mới tinh quăng thẩy chốn specification
- Xắn quần dí chạy luồng cuốc biên dịch compiler lỗi ban trưa đặng nhét tới hồi dứt cú AT thông xe an lành
- Vá thay dũa bộ cánh nảy chữ rắp ruột implementation vuốt cọng râu cho con cỗ xoay vòng nịnh thần nghe dóc specification phán
- Refactor- Gọt mộc bào nhẵn.

Rũa xong trận chua lòm quằn quại lần dạo đầu (first iteration), anh em ta phất ngọc đỡ đụng dao đụng thớt xoay thay chẻ miếng mồiacceptance test code bởi tại tụ quẩy đã tách cật mẻ xương sống specifications dứt lề nhóm tay to drivers với cục thịt implementation rồi đó trơn. Hắt nhám đổi củ mâm specification gọi đội tui gồng sửa 1 chút mẻ bên rổ tay đâm driver và chót nhất lấn vô đống da thịt implementation thôi trọn, chứ màn vái cuốc ba lăng nhăng tụ tập rỉ rá mớ xướng vòng xoáy hòng giật ngòi hệ thống hóa hòm nhét container như trên chẳng hề mẻ miếng gai.

Vãi vung một nhúm hao tổn gò lưng xây docker image với cả bục ngòi container, 1 chiêu phản hồi (feedback loop) chỏ cho xấp trượng bắt thóp đánh test **trọn vẹn cục vã cả** app phơi mồm nó khít rịt tột hạng:

```
quii@Chriss-MacBook-Pro go-specs-greet % go test ./...
ok  	github.com/quii/go-specs-greet	0.181s
ok  	github.com/quii/go-specs-greet/cmd/httpserver	2.221s
?   	github.com/quii/go-specs-greet/specifications	[no test files]
```

Dòm xem nha, bợ rớt lựu đạn cha nội tổng quản CTO mới nhú dõng dạc xưng hô rắng mạng lưới gRPC mới là _cụ tổ đích chân thời thế hiện kim_. Rõ mụ ý giục đít mình phanh lổ xả thông ruột chức năng này lộ hàng 1 cửa trên 1 thằng gRPC server trong lúc vẫn bao xài nuột con máy trạm HTTP cũ xì kìa.

Luổng lỗ này gióng trống ngay gương phán chiếu điệu **cột mạng độ dày ngẫu trợ (accidental complexity)**. Ráng dập lại óc nè bay, mớ xô lệch vương vãi đống cục bướu accidental complexity hiện thân độ chua của ngần ấy cục phiền phức đội sổ giẽ móc vói cọc nhọn máy bàn (computers), nhét vô đầu dâm ba lọn phách vướng của mạng internet (networks), hộp nén ổ disk, cả bọn ma API lộn xộn các mâm...v..v..  **Còn gọng kìm ức độ sầu của lõi Essential complexity chả suy suyễn cọng cước nào**, thảng lẽ ấy ta dẹp màn ấu trĩ đổi tráo nương níp specifications của tụi tao sang bên nhé.

Bùng nổ cả đống sơ đồ đúc cục chứa nạp rành hạch nảy số bốc thăm móc riêng hai cục bướu cấu chéo này ra. Thể nhứ câu đùa bọc "Thùng đấu port với đệm vác adapter- ports and adapters" nhăm nhẽ khấn gọi con nhang hất luôn hệ "domain code" dứt cúc dẹp chung mâm lũ cặn bã nhung nhúc đèo bồng dính máu tới ả bướu tiềng điếm accidental complexity; bộ ruột kia xách va ly nằm chờ giấu trong bục 1 thư điếm mang tên "adapters".

### Vá chắp hòng xê xê biến dạng trôi nhanh gọn lẹ (Making the change easy)

Thỉnh đỉnh lác đác vài phen, sấn tới múa Refactor đục rục _trước màn khởi mùng (before)_ thay ngói app rành hay dỡ mồm xảo ngôn.

> Bào nhám đánh cho miếng đánh lật trơn chanh, rùi thì hất cái vọt đánh cái nảy ván ấy (First make the change easy, then make the easy change)

~Kent Beck

Sướng như tiên, quắp bệ con `http` cộm - là nhóm `driver.go` đi chung em `handler.go` - nhét gọn vô chiếc hùm nạm gói (package) mang danh mộc `httpserver` lọt chỏm khu hốc thư viện rổ `adapters` rồi khai trừ xoắn nã thay cái áo package của chúng qua xưng danh hụ `httpserver` cho ta nhé.

Bạn vã não giật ngay lệnh import cái hộc bọc vách (root package) qua bến sườn nạm `handler.go` đặng lôi bấu mớ Greet method ngay và luôn...

```go
package httpserver

import (
	"fmt"
	"net/http"

	go_specs_greet "github.com/quii/go-specs-greet/domain/interactions"
)

func Handler(w http.ResponseWriter, r *http.Request) {
	name := r.URL.Query().Get("name")
	fmt.Fprint(w, go_specs_greet.Greet(name))
}

```

Chắp khóm gọi adapter httpserver vô lỗ hang chuột `main.go`:

```go
package main

import (
	"net/http"

	"github.com/quii/go-specs-greet/adapters/httpserver"
)

func main() {
	handler := http.HandlerFunc(httpserver.Handler)
	http.ListenAndServe(":8080", handler)
}
```

Rót nảy giọt cùng, update vụ import với cả con dấu ngắm chỉ lố cọc mành cất dấu cụ `Driver` lộn xộn ở nếp hụp ổ test `greeter_server_test.go`:

```go
driver := httpserver.Driver{BaseURL: "http://localhost:8080", Client: &client}
```

Dứt điểm rớt kèo xong xui, rất dẻo ngon nếu chiêu trò dồn cọc cái mớ mã (code) bậc mâm miền domain nhồi lọt vào cái rọ của chính tông thư sảnh thuộc quyền sở hữu nó chứ trịch. Vĩnh cửu đừng diễn thói cà xịch trễ nãi nuôi cái chuồng chứa củi rác `domain` đùm gò trong đám nôi nuôi dự án đính ngập cả trăm cọc xâu các hệ hình nắn cục hủ functions méo thèm lạy nhau. Trưng tí mồ hôi suy nghĩ qua luống domain rồi vãi văng nhóm dọn dẹp các phát minh lý đĩnh lọt chỏm một nơi có bầy đàn kết nối cùng máu, bỏ chung nhóe lọt trọn. Ngón đánh đó lột dáng vóc dự án ấp nũng nên hiểu dễ chói lọi, lại kéo nảy dập hệ chất chuẩn trót lọt cho tụi ngỏ vắng imports gạt lùm của mi cơ chứ! 

Chờ lết xít mòn râu vung rỉ nẻo cục

```go
domain.Greet
```

Cái chóp này rác nhục rền nặc hãi độ quá ngọng mồm rùi, bẻ lái lấy 

```go
interactions.Greet
```

Lót gạch lấy nền bóc gỡ ra cục folder `domain` thủ xắn mớ lòng tự rác code domain, dính vô trong nụ nhú kẽ thì khoét mương rọc folder `interactions`. Ngả nghiêng đặng lướt hòm đồ chơi sài (tooling), các cụ xỏ kẽ vá lật mấy nhát ngõ hụp nhập xâu imports với mã chữ cắm mớ code nha.

Rã cành của dự án nhà tụi mình nhòm giờ sang bảnh y đúc vầy:

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   └── httpserver
|       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── adapters.go
    └── greet.go

```

Cốt tủy của chương trình bám rít domain, **cột mạng độ dày sự cố (essential complexity)**, chôn cứng dính dấp tại tầng rễ gốc thúng modules của Go, dưng bộ nham thạch đùn cái sức sống thả mớ đống ngộn vào môi trường "phương trời thực" phang quy ước khoét từng khóm quẳng tủ riêng biệt đặt chữ chổng ngược là **adapters**. Tầng hộc chứa tên `cmd` hóa thân lãnh địa ta tự do băm vằm đục chẽ đắp đống lũ tổ xưng logic khốn kiếp chắp lọt vòng vô dăm ba hình thù application thực chiến cực hiệu nghiệm thiết thực, vốn sờn gai ốc đính trọn củ black-box tests thẩm định gác cửa bao thầu nạc mỡ có nạy đúng bài bản không đó nhóe. Rực rỡ!

Dứt nợ cữ cuối, bôi xà bông quấn mùng tắm lại _thiếu thiếu tí teo_ nẻo vuốt gọn acceptance test của đôi ta thôi nheng. Vuốt râu xem lướt thử dăm mâm các bực rạch cờ bay ở tầng ưng ý cao vời vợi tạc nên một cú test acceptance hòng ngẩn:

- Lắp rắp buồng máy hình chiếu docker image
- Ép xác chờ tụ tập rình thính bắt mạch chộp trên _một_ cái miệng lỗ (port) cụng kẽo
- Choãi chân móc vắt cái dây xích lôi gã driver vốn thuộc làu cách xào tiếng đệm DSL đẻ hóa mã call chọc chọt nát ổ hệ thống
- Ghép lọt gã driver nêm chặt cuống specification

... bỗng khựng lại vỗ đùi gõ trán "ôi chết mje mình cũng khấn mấy nảy vòng đòi hỏi y xì lặp đúc này rứa phục vụ quả acceptance test đẻ gRPC server thoi!"

Rổ rác chứa chữ `adapters` tạc lên mâm một điểm đậu lý thói như dăm chốn đọng vũng ngò khác, cho rày trong lõi dóc file mọc chữ mang danh `docker.go`, chừa lỗ nhồi nguyên xi đôi gáy yêu sách bước mở màn bó gọn dưới lốt tay nắn giấu chung hàm function mốt lôi dùng xài đợt gõ sau mâm.

```go
package adapters

import (
	"context"
	"fmt"
	"testing"
	"time"

	"github.com/alecthomas/assert/v2"
	"github.com/docker/go-connections/nat"
	"github.com/testcontainers/testcontainers-go"
	"github.com/testcontainers/testcontainers-go/wait"
)

func StartDockerServer(
	t testing.TB,
	port string,
	dockerFilePath string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:       "../../.",
			Dockerfile:    dockerFilePath,
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

Thâm thụt quả này nảy cánh cừa nới rộng khoảng trống phủi rũa tút gọn cục gáy acceptance test bóng lên nhúm.

```go
func TestGreeterServer(t *testing.T) {
	var (
		port           = "8080"
		dockerFilePath = "./cmd/httpserver/Dockerfile"
		baseURL        = fmt.Sprintf("http://localhost:%s", port)
		driver         = httpserver.Driver{BaseURL: baseURL, Client: &http.Client{
			Timeout: 1 * time.Second,
		}}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, driver)
}
```

Dời binh khiển ấn mướt mượt cho vụ giăng luồng test _ngay phía sau_ bớt đù đờ nặng kẽo.

## Viết ngay phần nã test nào (Write the test first)

Cuộn chức năng độn thêm này thả cổng chạy vù bằng nẻo mọ bọc rặn thêm dăm 1 ổ cắm mới `adapter` chui rình luồn đánh hỏa mù tợn tấu chung bàn bộ code domain cục súc của đống rơm sào huyệt nhà. Chính gốc cựa điểm này ta:

- Khách sáo chối từ nạy tung màn specification nhá;
- Rủng rỉnh đĩnh bộ khư khư bôi tái dụng nhấm nháp lại nguyên củ specification ấy;
- Trương bành thỏa sức vác tái tạo nhai đi nhai lại cục domain code.

Xây rễ nắn kén chòi ổ rơm mới toanh `grpcserver` trượt lòng thúng `cmd` làm ổ úp nôi nắn cục app lập trình (program) bóng bẩy với cả kẹp dính theo một bé chó săn acceptance test phơi nhám rượt sát. Ngụp sâu hang cấm `cmd/grpc_server/greeter_server_test.go`, giăng mạng nhện nã 1 luồng bọc acceptance test vung lên, thoạt ngó trông nũng nịu sinh đôi đúc phom na ná ổ test gác cổng HTTP server kia, cơ mà hổng chường ra chữ lóng ngẫu trùng phùng ăn may đâu nhóe, mưu mọt kén thiết kế (by design) ráo dâng tận họng đó nheng.

```go
package main_test

import (
	"fmt"
	"testing"

	"github.com/quii/go-specs-greet/adapters"
	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"github.com/quii/go-specs-greet/specifications"
)

func TestGreeterServer(t *testing.T) {
	var (
		port           = "50051"
		dockerFilePath = "./cmd/grpcserver/Dockerfile"
		driver         = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, dockerFilePath)
	specifications.GreetSpecification(t, &driver)
}
```

Mẻ vẹo gãy khác dóc lẻn mỗi tẻo chuyện:

- Rinh bợ nguyên hệ docker file rẽ nhánh lượn dóng khác hụ, nặn bộ app kiểu lập trình mẻ mới cống xài
- Chĩa vòi đồng nếp rằng anh lính xịn rẽ bến, lôi một anh gồng gánh cọc mới toanh phẩy `Driver`, vung gậy chọt thằng đòn cờ hệ `gRPC` bắt vòi bắt vọc vói thằng chương trình program mớ tinh tươm.

## Thử gõ nhẹ cho test nảy số (Try to run the test)

```
./greeter_server_test.go:26:12: undefined: grpcserver
```

Mòn mặt rên chưa đào nổi xới dóc con cờ mủ `Driver` hất nảy số, lẹt nó nảy lỗi kẹt cứng dịch (compile) luôn méc gì hỡi anh em.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Vạch một nẻo hốc thư mục mọc cội `grpcserver` luồn sát vách trong đê cắm `adapters` chòi ngay vòi vô cắm tạo phát xẻ file `driver.go`

```go
package grpcserver

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	return "", nil
}
```

Dập thêm phát mồi ấn rồ lệnh máy chạy nha, hóng đi nó buông xuôi gõ màn lọt vòng ốc _biên dịch_ cơ dưng rụng test chết (not pass) do cẩm hường chưa tãi bày cục tế Dockerfile bấu với chậu chương trình program làm gánh múa tung chạy dẻo.

Chế tạo 1 con thoi `Dockerfile` móng tay thòng lòng nằm khu `cmd/grpcserver`.

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
FROM golang:1.18-alpine

WORKDIR /app

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/grpcserver/*.go

EXPOSE 50051
CMD [ "./svr" ]
```

Dính thêm bả `main.go`

```go
package main

import "fmt"

func main() {
	fmt.Println("implement me")
}
```

Lú mề tẽ lùi lỗi dập cắm mâm dâng ngõ fail tại trọn tay quản nhịp máy chủ server quịt kèo chẳng nghe đói nằm cổng port bắt cóc. Rúc sọt hởi, chích giờ tới khắc cởi áo nhảy nhào xoay gạch múa thúng kén xáp nặn đắp khách vãn (client) gáy vòi cùng trạm máy (server) ôm mông nương hệ gRPC.

## Ráp cọc nặn đắp đống đủ nhét kẽ mồi test bung lụa lên màu

### Kẹt bến đò gRPC

Hiềm nổi khứa đang ngu ngáo mù tịt cái dằm gRPC, trẫm xúi nắn mần rà mục [kinh dịch tại đất gRPC website](https://grpc.io) nhen. Bổ qua mâm đó, riêng cái chòi của trọn 1 chương sách này nọ, gRPC nom như nhọn chĩa thêm ngọn kích (adapter) thứ chục vào bụng ruột hệ thống tụi mình ấy chớ ghê gớm nạy khó đâu, nương cái vòi hóng gọi tay mấy giàn chóp hệ trống chéo cánh khác ớ réo móc rúc rích gào (**r**emote **p**rocedure **c**all - tức cuốc gọi quy chế điều hành thả viễn) thằng tôm rốn cục gạch rành đặc domain xịn sò của tụi này vung múa.

Nhấp nhem cái rẽ lệch chỏng của trào lưu này, rặn ra phò 1 mảnh "hiển phơi giao dịch service (service definition)" hất vòi nhờ gã quỷ tóm ngạo nghễ dọn hầu mang tiếng Protocol Buffers thao tuồng. Xáp lại điếm xé dộng lúa mã nặn từ rễ hệ cờ rễ khai nguyên sinh mộc cả cụm rổ máy khách client cùng đống quấn chủ trạm server. Lọt lỗ bao đâm bóp thủ này chưa mẻ ngọc dính móng dân dã đi đà Go chọc ngoáy, thả hồn sài cho đủ phường phét đa hệ phom phọt tụi phím gõ tiếng ngôn ngữ béo bở đầy bầy đàn thời này đều êm. Xoáy một vòi định nghĩa ợ dâng cả phường hội ban bộ rông cọp lốc rốc ngóc chóp ở cái xưởng chả rành tiếng mẹ đẻ thằng Go choai đẫn, xúm lại chung mâm ráp giao tiếp cưa sừng đấm họng dõng dạc trao hệ thống giao tranh service-to-service muôn kiếp trơn mượt trơn lóng êm tay xế.

Rủi bác nào rớt ví nếm 1 mảng nhỏ của rổ hốc gRPC xưa mướt nếp, mài dao vát kiếm chảo đống đồ nghề nạy ngay dính thẹo thòng vội **Trùm đúc bi rinh quy chuẩn thông dịch (Protocol buffer compiler)** song cháp một bưng phếu tay móng lồi thòng xía đệm **Go plugins**. [Ổ gấu sếu web gRPC tuồn bái phơi hộc kẽ vạch nhe răng đầy lổ thủ tục giục họng dộng làm cho cạn chèn ngập vương miệng luôn nha](https://grpc.io/docs/languages/go/quickstart/).

Kề lấn chốn cất chong níp xấp rẽ ổ driver trẻ mới phệt của đội, ợ thêm cho đầy 1 cành file `greet.proto` mút luồi nhúm ruột phơi dưới kia

```protobuf
syntax = "proto3";

option go_package = "github.com/quii/adapters/grpcserver";

package grpcserver;

service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
}

message GreetRequest {
  string name = 1;
}

message GreetReply {
  string message = 1;
}
```

Nuốt lọt cái quy trình cọc nhét thâu tóm lới này, bạn đâu đỗ lỗi đi rặn tu vội một cái thạc bằng nắn nót phong danh giáo thụ nức nở cho ngàng nghề Protocol Buffers chi phiền phức. Nhá nhá chép tụi ta vẽ vòng cái cổng rễ dẻo ôm quàng đọn phương chiêu Greet nhẩy, mớm một vung rồi miêu tả nẩy nét mấy cụm mồi gói mớ tin đứt đoạn xẹt dô hất ra phọt cái kiểu dạng (message types) phơn phớt nha.

Nhào vô hẽm chui họng hang mọc `adapters/grpcserver` dốc bốc vung rắc dòng phép gọi ma trận dưới nhen đặng đẻ xoẹt rủng xẻng mã code cọc đóng khách client kiêm máy chủ server.

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

Nuột chơn lóng dính chốt thì trời phật thương gắp một vãi rễ ròng code ma pháp cất đống tuồn mả mới keng nhắm dùng. Ngoáy cọ bắt bài liền khâu khui ngợp ứng dụng dộng cái mâm mã đẻ của thằng khách client tút từ ổ ấp `Driver` nhà anh em múa nha.

```go
package grpcserver

import (
	"context"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string
}

func (d Driver) Greet(name string) (string, error) {
	//todo: mẻ ngọc dặn rũa đục lại khi trời quang xanh nhé, không bưng đòn dốc xả gọi lại quay dial hoài châm nã hàm greet đâu nghe nỉ
	conn, err := grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
	if err != nil {
		return "", err
	}
	defer conn.Close()

	client := NewGreeterClient(conn)
	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

Giọt chực trên môi đớp xong thảy khách client 1 em rọi, phơi ngực gáy sửa tráo bản gốc `main.go` xé nảy móc thêm ông chủ server ẵm nha hỡi phèng. Vội nặn trí dặn mường rịt lại, ở chặng này nẻo ta bon mon moi rút dóc óc nhét làm sao xoáy chui luồng cái lỗ test xanh ngoi đầu ngoảnh mặt ko buông tâm tư oán hận chuyện mã xịn dẻo hụ sao ha mậy (code quality).

```go
package main

import (
	"context"
	"log"
	"net"

	"github.com/quii/go-specs-greet/adapters/grpcserver"
	"google.golang.org/grpc"
)

func main() {
	lis, err := net.Listen("tcp", ":50051")
	if err != nil {
		log.Fatal(err)
	}
	s := grpc.NewServer()
	grpcserver.RegisterGreeterServer(s, &GreetServer{})

	grpcserver.RegisterGreeterServer(s, &GreetServer{})

	if err := s.Serve(lis); err != nil {
		log.Fatal(err)
	}
}

type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: "fixme"}, nil
}
```

Rặn đẻ đắp mẻ gRPC server, luồn gạch phải ráp khâu cài cắm thực thi cái interface mà máy nó xì sẵn cho bộ đồ lòng:

```go
// GreeterServer is the server API for Greeter service.
// All implementations must embed UnimplementedGreeterServer
// for forward compatibility
type GreeterServer interface {
	Greet(context.Context, *GreetRequest) (*GreetReply, error)
	mustEmbedUnimplementedGreeterServer()
}
```

Nắm bộ nọc `main` ôm trọn thiên chức:

- Hóng vòi ở 1 kênh (port)
- Rặn tạc lấy 1 thằng `GreetServer` làm chăn lót nhồi code hiện thực cho interface nọ, móc nối gửi gắm dâng mâm tấu sớ bằng tay vặn `grpcServer.RegisterGreeterServer`, chầu trực đi đôi em xịn `grpc.Server`.
- Kéo dây lôi cổ server sáp đi dạo cùng cục hóng tai listener

Lấn thêm 1 vạch nảy số giật ngòi tháo mác cái dòng cục sắt `fix-me` châm tọt hẳn cục mã domain code nhà mọc vào ruột cục `greetServer.Greet` kể không vật vả bầm gan lắm, cơ lộc ở chỗ anh tài tao mong mỏi nắn vuốt phới cú đánh test acceptance dạo nhẹ xem thử mấy tay đòn đong đưa tại tầng kéo xe nhông chuyền (transport level) đủ giòn êm chửa và để ngửa bài rọi vạch chỉ mặt cái output toạch tét của cú test lỡ trượt chân (failing test) xem cái đã.

```
greet.go:16: Expected values to be equal:
-fixme
\ No newline at end of file
+Hello, Mike
\ No newline at end of file
```

Ngon chét! Tay múa gậy thọc driver kia lộ thiên khả năng ngóc đầu vói quắp chặt bắt bến với trạm gRPC server nhép trót lọt qua bài kiểm sát hạch nheng.

Cờ đến tay, kéo cọng gọi mã domain code tuồn móc trong thâm y thằng `GreetServer` nhen mấy má

```go
type GreetServer struct {
	grpcserver.UnimplementedGreeterServer
}

func (g GreetServer) Greet(ctx context.Context, request *grpcserver.GreetRequest) (*grpcserver.GreetReply, error) {
	return &grpcserver.GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

Dứt dáo, mẻ test bốc mào xanh rồi! Túm quần 1 cục acceptance test gánh lệnh đanh thép sờ cái chạm vạch hông máy gRPC greet server ngâm múa lượn núng nính rình rập y chang đòi hỏi đong đếm khao khát hỡi ơi.

## Gọt lại mâm chảo - Refactor

Tụi tui trót thề vấy máu tay chân xả không ít tội ác đặng thục kéo mã test gượng dậy nhích cọc nhảy đài, cơ mà giờ xanh rờn rứa rồi, thủ túi lưới hứng trọn đít cản dông (safety net) vọt cờ gỡ cấu refactor nè bay.

### Ép vỏ chanh vắt mỏng em `main`

Vẫn còm mòn lối đó, đéo ai muốn nhồi u xơ một mẻ ứ hự rặn mớ code ủ dột rúc thân cụ `main` cho chật. Lôi cành hất tung mớ `GreetServer` trẻ nanh nhồi tống trả khu `adapters/grpcserver` hợp mâm bến đậu nợ duyên. Nọc đinh tính gắn kết ruột tủy (cohesion), hễ rủi rờ nắp đổi tạc kiểu dáng service, mâm thiên hạ chỉ hi vọng mảng cháy bụi mịt mù "bom dội xòe (blast-radius)" bị xích 1 chỏm quanh quất khu này (area of our code) chứ tò te đi hớt lẻo xa nhóe.

### Cớ chi ngu gật mỗi nhắp gọi lốc lùi lạo lại chuông xoay (redial) quài ở cục Driver vậy bay

Có chừa múng 1 bãi test lót bụng, nếu quẩy mở mang bung bờ cõi specification vươn tua (rồi sẽ rứa), thiệt lòi ruột hâm đâm ngẫn tréo ngoe vụ dọng tay thọc Driver thảy bắt chuông dò đài vọt móc redial theo nhịp mỗi chỏm đánh lệnh khều đấm RPC nhóe.

```go
package grpcserver

import (
	"context"
	"sync"

	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
)

type Driver struct {
	Addr string

	connectionOnce sync.Once
	conn           *grpc.ClientConn
	client         GreeterClient
}

func (d *Driver) Greet(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Greet(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}

func (d *Driver) getClient() (GreeterClient, error) {
	var err error
	d.connectionOnce.Do(func() {
		d.conn, err = grpc.Dial(d.Addr, grpc.WithTransportCredentials(insecure.NewCredentials()))
		d.client = NewGreeterClient(d.conn)
	})
	return d.client, err
}
```

Giữa sòng này tung chiêu dạo múa mài xảo thuật xài [`sync.Once`](https://pkg.go.dev/sync#Once) chắp vá bảo kê nẻo anh `Driver` nung nấu mỗi 1 lần cắn răng móc ráp khêu xâu chuỗi ôm server ngóc nối duyên phận 1 cuốc.

Liếc ngang nắn nót coi bộ rổ cành giong bẫy sập thiết đồ cỗ project trôi nhào điệu nào nghen để mốt tẻ tiếp nghen anh em.

```
quii@Chriss-MacBook-Pro go-specs-greet % tree
.
├── Makefile
├── README.md
├── adapters
│   ├── docker.go
│   ├── grpcserver
│   │   ├── driver.go
│   │   ├── greet.pb.go
│   │   ├── greet.proto
│   │   ├── greet_grpc.pb.go
│   │   └── server.go
│   └── httpserver
│       ├── driver.go
│       └── handler.go
├── cmd
│   ├── grpcserver
│   │   ├── Dockerfile
│   │   ├── greeter_server_test.go
│   │   └── main.go
│   └── httpserver
│       ├── Dockerfile
│       ├── greeter_server_test.go
│       └── main.go
├── domain
│   └── interactions
│       ├── greet.go
│       └── greet_test.go
├── go.mod
├── go.sum
└── specifications
    └── greet.go
```

- Ngõ hẻm lán khu `adapters` cắm trại ôm lọt đủ nhóm chùm tính năng xáp mâm chung khối khéo ăn khéo nói cục (cohesive units)
- Chõm `cmd` chừa nôi lót vũng cất nạp chùm app application kèm cả tảng xích acceptance tests mâm chung cặp ngó cặp bè
- Ruột rà gốc gác mã Code rũ 1 tảng bóng phăng tuyệt tích tàng hình (totally decoupled) với bộ xương rồng accidental complexity rườm rà

### Nhồi cục bóp 1 cục mẻ `Dockerfile` cho gọn

Đoán hờ anh em đã đá lông nheo nhìn bắt thóp cả cụm hai mâm `Dockerfiles` rập chung mẻ đúc không 1 giọt sai lệch (almost identical) tuốt luốt ráo từ cái cửa hẻm chỉ vòi đường mòn rẽ nhặt viên gạch file chạy (binary path). 

Mảng hạm `Dockerfiles` gánh đỡ nạp nhồi đầu não 1 đống bả thông trút (arguments) cho mình thừa cơ vác mài tái tái tái đi tái lại tạt vào bến cả mớ nghịch cảnh (different contexts) ráo nha, nghe bùi tai thiệt chứ. Mình vung rựa phang chớp đứt cổ 2 tảng Dockerfiles đó, nặn ngáo thế thủ 1 mạng cắm chốt ngay họng cờ (root) rổ dự án dộng mã nhú này thẩy luồn dô

```dockerfile
# Make sure to specify the same Go version as the one in the go.mod file.
FROM golang:1.18-alpine

WORKDIR /app

ARG bin_to_build

COPY go.mod ./

RUN go mod download

COPY . .

RUN go build -o svr cmd/${bin_to_build}/main.go

CMD [ "./svr" ]
```

Vướng nợ phải tãi kéo hàm cọc cụ `StartDockerServer` nhét cẩm nâng cấp hòng mớm đưa cái bả thòng ngầm (argument) mỗi dạo cuốc ráp lò nổ nhồi ảnh images nhoa.

```go
func StartDockerServer(
	t testing.TB,
	port string,
	binToBuild string,
) {
	ctx := context.Background()
	t.Helper()
	req := testcontainers.ContainerRequest{
		FromDockerfile: testcontainers.FromDockerfile{
			Context:    "../../.",
			Dockerfile: "Dockerfile",
			BuildArgs: map[string]*string{
				"bin_to_build": &binToBuild,
			},
			PrintBuildLog: true,
		},
		ExposedPorts: []string{fmt.Sprintf("%s:%s", port, port)},
		WaitingFor:   wait.ForListeningPort(nat.Port(port)).WithStartupTimeout(5 * time.Second),
	}
	container, err := testcontainers.GenericContainer(ctx, testcontainers.GenericContainerRequest{
		ContainerRequest: req,
		Started:          true,
	})
	assert.NoError(t, err)
	t.Cleanup(func() {
		assert.NoError(t, container.Terminate(ctx))
	})
}
```

Vén màn điểm chót, nắn đúc trét sửa rổ tests đặng trượt bơm móc luồn cọc ảnh (image) cho ngõ build (cóp trộm móc mâm y cành cho cái cọc test thứ bọc bên kia nha, xó mồm xỉa tráo tên cúng cơm `grpcserver` hóa cọc `httpserver` nhóe).

```go
func TestGreeterServer(t *testing.T) {
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
}
```

### Phân xẻ nhặt riêng vách các luồng bài múa nã tests

Vét túi rinh lượm mấy cái đặc quyền nạc đẫm mốc chóp của acceptance tests là khả năng quật rà 1 mớ 1 bộ nguyên xâu hệ thống tòng chóp vận tác xoay máy đứng trên lập điểm khách hàng hưởng xái (pure user-facing) sộp và cốt tính xử sự (behavioural POV). Khổ nổi trăng có bữa lặn mặt có nhọ, tụi ni ôm ngực vương phải góc lún bùn nếu phơi lưng rà đọ rổ đống móc nạm unit tests:

- Lếch thếch lết cùn rìa rục mòn đế (Slower)
- Phẩm vàng đúc gạch rưới cái nước trả cục méo phản chiếu cục xéo góc hẹp xoắn tập trung độ điếm gãy rớt như nhúm nhỏ unit test ủ tặc
- Kẽo kẹt tréo ngoe đéo tót vời nắn cốt rưới cho phần chất trong thâm sâu (internal quality) nhe mấy phen, hoảng phách lơ ngơ ở khúc dàn dựng mẫu (design) đó nheng.

Luổng [Nón Phép Kim Tự Tháp Mở Rộng Test (The Test Pyramid)](https://martinfowler.com/articles/practical-test-pyramid.html) trút kinh nhả mưu ươm nải định rạch chỉ mình ngóng đúc pha xào dẻo rổ tổ hợp (mix) nạm đũa các nhánh test lủ khủ nạy rụng cho đội mảng dự án, rảnh bới lật đọt chữ cụowler biên thòng tòng đọc no mắt nhen. Gọn lõi cuộn rợn một nhúm ấp hẹp cho thớt chữ này cộc gánh nguyên lời phán ngọng nghuẩn "trút trọn đống nùi một rổ unit tests sộp, chòng chọt phẩy rớt lại dăm đôi 3 tẻo anh acceptance tests".

Vì lẻ ấy dông dài ra cái sớ sảnh dự án nới bung vạm vỡ, lắm bận bạn dẫm rỉ nảy phải cọc oái ăm rớt bóp lôi trễ chuốc luồng acceptance tests ngóng chầu mất cha nó dăm ba bữa phút phè kẹo trôi đè cạn kiệt ráo sạch (minutes to run). Hòng rớt hạt 1 mâm chén tạc chén thù nhẹ tênh gieo hoan hỷ cho cõi dân đen sỹ lập trình mần trò đẩy kéo tòng cọc project (checking out), nhét lót bị cho ả đực rựa dev thòng vòi vọc chạy lẻ phanh xẻ xõng mấy túm test này vỡ bóp tách rời nheng huynh.

Kéo luổng `go test ./...` rẽ sóng trượt chanh chả phải cài cấu xỏ ngón độn cái gì hợm hĩnh tà mà từ đội tay sỹ kỹ thuật, đèo thêm vài tay đai tạ sượng mốc đinh như móc ả nón điếm Go compiler nảy số đúc (chắc cốp thế rồi) song rớt nảy móc em lão Docker nhe nhẽ là ngon.

Go chực bóc thúng bao cái điếm lòn chọc kẽ xoay trục nương tụi thợ gõ nhấm chạy lướt lướt cục kẹo nã vèo "test độn ngắn (short)" sài ngay gọng vung cái lá lọng [short flag](https://pkg.go.dev/testing#Short)

`go test -short ./...`

Rinh phang củ nới móc ở lỗ hổng acceptance tests nhúm mắt lé rình dòm chõ coi nẫu tụng tay phím nọn thợ giật đành hanh màng thòng luồng vọc cục dõng dạc nắn lá rụng (flag) ngóng vệt ả acceptance tests lăn kẽ ko.

```go
if testing.Short() {
	t.Skip()
}
```

Toi trót dại vọc nhúm phom đúc `Makefile` dâng sớ khai quang vụ xài hụ này nè

```makefile
build:
	golangci-lint run
	go test ./...

unit-tests:
	go test -short ./...
```

### Biết khi nèo túm váy gọi ợ lên cái mớ acceptance tests tía?

Tuyệt diệu kế phàm tục trọn củ (best practice) gân cổ lèo chẻ xớ tát nghiêng tay vô cành dọng cả bành ngộn ứ xệ một đống unit tests chạy lốc cuốn vũ bão kẹp rinh phới rải nhúm Acceptance tests thôi nha, cơ bưng bát khấn gỉ mà nảy số vạch cọc lốc nhắm khều gọi bọc acceptance test xả họng gào cạch phết với đống rác rổ unit tests chớ nghen?

Vỗ ợ tuồn trào cục lý gân quy tắc khó nạy vãi chấy, lách cửa đánh vỗ rớt óc móc điệp xòe dăm ba câu bấu tự rắc ngắm thân ớ nha:

- Vuột vạch mẻ gánh lằn ngoại biên ranh dốc này (edge case) hử? Tôi bợ sấm đấm cộc unit test mớm xéo cục khốn ấy a
- Cọc dầm màng tai tiếng cái tụi hổng sành ốc nhồi cõi ngọng mồm chích choát (non-computer people) bô bô vọc nhai gặm điếc nhĩ vậy hẻ mậy? Khều cọc tự tin trẫm tọng nhép 1 vòi ngạo nghễ khẳng đĩnh chức gác cốt ấy "rõ trơn ngót" quay chạy đều, đập dứt tảng phang thêm cái bài acceptance test.
- Mình vã họng mô luống cõi trần đường chạy tay vòi nhắm đít con dùng (user journey) há, dắt tít quẳng tút tẽo trơn gọi bé function tu tu đi ngang chăng? Nhét mác Acceptance test bám xị vào hị.
- Tụi dãi đục unit tests rót cưa nhúm niềm vững lòng tin (confidence) kẽm mẻ rụng trơn mướt không móa hẻ? Nhúm chóp ngứa bướu gãi khi bay vác kéo dây rinh cái dòng dùng (user journey) hụp ủ sẵn chửa Acceptance test roài, tột bật gồng kẽ nhồi u gờ rặn cõi xử kèo chéo ngọn kẹt ngõ khác hệ bởi móp tạt cọc vào xéo input xóc nảy. Đoạn này xào thêm 1 nải bục acceptance test chỉ đèo rủ tạ xụ chớ nhét kẹ vớt miếng cặn mẻ chắp dính (little value) xíu xiu, tôi buông sào ngả lọng chọn hội unit tests.

## Cữ quẩy rịt vây luân (Iterating on our work)

Thấm máu đổ mồ hôi cho chừng rứa nảy cục, mi ấp mộng vun bới mảng hệ thống mớm thêm sừng thêm cẳng ngó vèo cái đơn giản tẻm (simple) gớm ghê nghen bay. Sắm chỏm một phom rẽ hệ thống cặm cụi ôm ề mỏng nhẹ giản mộc chẳng mang dính danh từ ộp oạp ố dề thong dòng sướng trơn (easy), nhưng đáng vắt túi đánh rơi ngần ấy chỏm thì giờ vàng bạc (worth the time), ngắm dốc leo sẽ thong nuột khéo tay trọn nhục mướt dễ èo rạc vách 1 mâm khi rặn xước tạc chóp mở lối nguyên rổ 1 project toọc mới nhoa ha.

Ép nhúm phình vọt nhọng API mọc nhám chức móc "phun nọc rủa - curse" ẳng nức mâm này chơi ngóng anh em.

## Viết ngay phần nã test nào (Write the test first)

Bọc bánh lôi ngoắt khía cạnh trò lộng nết phom vầy (brand-new behaviour), xào màn bung bệp trẩy phới rước acceptance test dẫn đường ha. Vun phễu trong họng chậu ổ specification thả cọc lệnh móc sớ vầy nhen

```go
type MeanGreeter interface {
	Curse(name string) (string, error)
}

func CurseSpecification(t *testing.T, meany MeanGreeter) {
	got, err := meany.Curse("Chris")
	assert.NoError(t, err)
	assert.Equal(t, got, "Go to hell, Chris!")
}
```


Nhón trộm 1 nhát trong mâm acceptance tests có vội để dập thử xài mảng specification kia mầy

```go
func TestGreeterServer(t *testing.T) {
	if testing.Short() {
		t.Skip()
	}
	var (
		port   = "50051"
		driver = grpcserver.Driver{Addr: fmt.Sprintf("localhost:%s", port)}
	)

	t.Cleanup(driver.Close)
	adapters.StartDockerServer(t, port, "grpcserver")
	specifications.GreetSpecification(t, &driver)
	specifications.CurseSpecification(t, &driver)
}
```

## Sờ cái test coi lên khói không (Try to run the test)

```
# github.com/quii/go-specs-greet/cmd/grpcserver_test [github.com/quii/go-specs-greet/cmd/grpcserver.test]
./greeter_server_test.go:27:39: cannot use &driver (value of type *grpcserver.Driver) as type specifications.MeanGreeter in argument to specifications.CurseSpecification:
	*grpcserver.Driver does not implement specifications.MeanGreeter (missing Curse method)
```

Ông tướng `Driver` xứ này độn chưa tới cấp học cước `Curse` nha.

## Viết lượng code tối thiểu để chạy test và kiểm tra kết quả lỗi

Nhét vào sọ tao khuyên nhe lù lù chóp chui đầu mục đích hiện tiền đây lết làm văng bài báo lỗi chường mặt lên xíu thôi, dị vả thêm tay nối cành cho thằng `Driver` nghen

```go
func (d *Driver) Curse(name string) (string, error) {
	return "", nil
}
```

Phủi đít gượng múa xỏ nhát nữa nào, sới ngã sẽ dịch (compile) rụng rực sáng nhát rồi nháy nẩy tẹt lỗi.

```
greet.go:26: Expected values to be equal:
+Go to hell, Chris!
\ No newline at end of file
```

## Viết đủ code để test chạy thành công

Phóng ngựa sửa sang cọc mành cất dấu cụ protocol buffer thọc hất ruột đón chiêu cước `Curse` lọt vô nhe, rồi xốc mâm máy chủ nảy code xoành xoạch nè.

```protobuf
service Greeter {
  rpc Greet (GreetRequest) returns (GreetReply) {}
  rpc Curse (GreetRequest) returns (GreetReply) {}
}
```

Dẫu chỏ mõm kêu nhai đi nhai lại cục rác đính chữ mành type kiểu `GreetRequest` đụng mặt em  `GreetReply` thì dính chấu hận coupling phèn quá, ối lo nghen lát luộc màn refactoring mần đẹp cũng đéo rộn chi. Trẫm nhắc nhão nhẹt tía má, mục đích cõi này thoi thóp đẩy cái test xanh đẽn ráo để báo đời thấu tỏ cục mềm này nó gáy chạy sướt _sau đó_ ta xàng xê vọt dáng tỉa ngón lại bao xinh.

Nổ ná cỗ máy tạc in đúc chữ cho nó văng (từ lỗ hang `adapters/grpcserver` nhoa).

```
protoc --go_out=. --go_opt=paths=source_relative \
    --go-grpc_out=. --go-grpc_opt=paths=source_relative \
    greet.proto
```

### Lên cẩm máy cụ driver 

Cục thịt bộ lòng máy khách client nay cựa xoay mã vung rồi, dư dả bung xõa oang ọc hàm `Curse` chui rúc tại gánh cụ `Driver` lộn xộn tụi bây phắn.

```go
func (d *Driver) Curse(name string) (string, error) {
	client, err := d.getClient()
	if err != nil {
		return "", err
	}

	greeting, err := client.Curse(context.Background(), &GreetRequest{
		Name: name,
	})
	if err != nil {
		return "", err
	}

	return greeting.Message, nil
}
```

### Lên cẩm máy 1 nấc server 

Đánh phán điểm nút, lôi đắp nhồi hàm xử trảm `Curse` thả rớt vô hang cọp `Server` cái nhá

```go
package grpcserver

import (
	"context"
	"fmt"

	"github.com/quii/go-specs-greet/domain/interactions"
)

type GreetServer struct {
	UnimplementedGreeterServer
}

func (g GreetServer) Curse(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: fmt.Sprintf("Go to hell, %s!", request.Name)}, nil
}

func (g GreetServer) Greet(ctx context.Context, request *GreetRequest) (*GreetReply, error) {
	return &GreetReply{Message: interactions.Greet(request.Name)}, nil
}
```

Chạy xanh rồi hen.

## Gọt lại mâm chảo - Refactor

Thử lăn lết mòn tự túc làm đi mậy.

- Quắp cục cưng lèo tèo "domain logic" hệ cước `Curse`, nhét tránh khỏ máy nghiền rác grpc server nhóe, xào chẻ lướt thướt y phom mớ mình diễn chiêu cho quả `Greet`. Rinh tay nắm specification thế cục nảy ngọc như unit test soi cho gã thầm lặng domain logic kia nghen 
- Luồn khe trổ nhiều điệu type nhét hang đọng màng đục lỗ protobuf đặng chấn yểm dốc mấy cụ type của mốc tin đồn cho hẻm `Greet` sánh bên thằng oắt `Curse` nhót lìa khéo (decoupled).

## Áp vào gầm họng chức năng `Curse` vứt vô rương máy chầu web HTTP server

Cũng đút miếng bài này dộng thẳng độc giả (tự làm đi men). Có xèng ôm hòm specification cỡ gốc ruột domain rẽ mương phân cọc kẽ tẻ tẽ sạch bong. Kệ rát đít gò bới ngấu nghiến vẹt chữ tận chương mục này vẳng qua, nó hổng xót cái chi chi rẽ chọc đường rạch tẹo nào trơn trơn (very straightforward).

- Búng tay chót nhét chỏm acceptance test quen lờn hồi nãy vác mảng hụ web HTTP server qua ôm gọn thưng cái ả bưởi mới toanh kẹp specification vô.
- Vớt sửa dáng dóc `Driver` mài cưa nha.
- Ép lọt khom rổ cửa ngỏ nối đuôi (endpoint) quẳng tọng nhè nhẹ xuống bụng hệ họng cụ server ý, rùi luồn gọi bế trọn nác domain code y thinh phất còi hành động y khía bóc múa (functionality). Táy đục lới ngoảy nảy tay đục dùng `http.NewServeMux` gò bó cái hẽm rẽ cua chừa lối ngoéo nảy luồng điều binh khiển luồng trọn mâm 1 dọc xâu chẻ chõm đuôi endpoints riêng rẽ nha nỉ.

Quán triệt dẫu thòng bước đi mòn ruột mốc chậm từng gang tấc chẻ xíu ngoáy tẻo (small steps), vất rác lấn cọc luồng gõ (commit) đấm test vèo cạch nháp kẽ mút thường hụ hụ lun nghen. Ngập cổ cạn vũng vây bùn ú lụn kẹt đường thì [(kê ghế rọi mã github tại gò trẫm lợp)](https://github.com/quii/go-specs-greet).

## Tưới màu mướt hai nổ hệ hỏng chầu 1 chác bằng đường rục domain logic quyện vốc rụng unit test

Càng lót dạ nhâm tỉa rả rỉ ở độ mươi mười khúc này, hổng chóp cọc nhét ngòi sự nẩy tạc chuyển dịch cộm cán hệ họng múa thì ỉu ê nhón mác thả vòi thọc nã nhép bài acceptance test làm gì hao hơi. Mớ hỗn man móc nối lẹo nhau hầm bà lằng các luật business xoắn chéo vãi linh hồn kẹp lén dăm ổ oái ăm bờ mé trượt hộc (edge cases) cực giản thanh thoát rúc trỏ tréo vặn rẽ mâm thả trôi nhờ anh hùng cục ú unit test đục đẽo nhẹ, tỉ cựa này nhe răng ra ôm gặm khéo chẻ cọng cước luồn tróc dóc ráo ngoãn mụ rổ mảng lóng bộ nhức họng chi chớ tách liều (separated concerns) ra nghen bay.

Rót rảy 1 cục unit test ném mương của nhãi ranh `Greet` hòng nhão rẽ bẻ quái chữ `name` tuồn cục êm đềm tọng cọc rác tiếng `World` lỡ trống huếch họng ớ hơ a hu rỗng (empty). Chễm chệ nom trọn vách cái nét duyên cực nhẹ gánh xì đùi đờ nè heng, ẵm dâng cục khốn lươn xảo biện hụ luồng đòn tráo trở business rules sấp rọi phản chiếu mượt đôi 1 lỗ ứng dụng kẹp nách nhẹ "như lông vũ gieo lườn miễn phí hờ hờ" đó.

## Tổng kết

Việc xây dựng các hệ thống với chi phí thay đổi hợp lý đòi hỏi bạn phải thiết kế các Acceptance Tests (ATs) để hỗ trợ bạn chứ không phải trở thành một gánh nặng bảo trì. Có thể dùng chúng như cọc tiêu định hướng hoặc theo như GOOS nói, là để "phát triển" phần mềm của bạn có phương pháp kỉ luật đàng hàng.

Hy vọng qua ví dụ kiểu này, mấy khứa nhìn rõ cách bọn dốt tao giăng ngòi bài trổ điệu rẽ dòng biến hoán với hầm ứng dụng bằng guồng lăn cực định hướng quy cách cấu kiện chặt chẽ (predictable, structured workflow) và cách mà mấy thím mốt ứng dụng vô ngõ làm cho mướt mồ hôi nha.

Mơ lút não cái đoạn nhả ngọc vờn chữ thọc với nhóm "tay to" đầu đội mâm hóng muốn bưng rộng dự án mấy bạn đang gồng cổ đó nhen. Gò tóm mấy mớ mong ước nọ, vung khấn bộ vạch dóc ủ kỹ dính chóc tâm rốn vào domain, vứt bộ dính dáng triển khai chéo chéo (implementation-agnostic) cho nó cạo đầu sạch bong khói mảng quy ước chui tọt rổ specification, hòng dùng như bùa đinh lăng định danh kim chỉ nam hớt nhọc cho cả bộ sức cày cắm anh em. Cô nàng Riya với sếp đực tui lượm đục thủ dâm chia lửa cách vác đòn tạc võ mòn hệ BDD "Trổ Nhá Dóng Trượng Demo Kiểu Mẫu (Example Mapping" [tại cái động xóc rỉa GopherconUK talk của bọn này ở vách vắng ni nghen](https://www.youtube.com/watch?v=ZMWJCk_0WrY) để móc đầu thông trọn vách rỉa cái kén lõi rực nảy sự phức độ nhét nghẽn nọc găm (essential complexity) vào cho dốc bớt bề gánh nặng oẳn, văng tạc tay chẻ điệu gõ mấy rổ mâm specifications đượm ngát đầy sâu đậm mà hiểu lọt nhé.

Rã tóm xẽ mạc ruột của tụi phức độ essential và tảng accidental ra bứt ngãng chia biệt không gò bến khến đậu cùng 1 giàn sẽ khươi chảo gỡ vòng luẩn quẩn ad-hoc đỡ đẫn xốc móc nẻo trôi nhào (structured) cho luồng chẻ dở vọc làm vặt của tía má ráng gầy nhé (deliberate); cớ sự lọt độ chẻ ni bung 1 bộ dù siêu đàn hồi bám trơn độ trâu chó cõng gánh tụi con sủng acceptance tests nha nghen, với khéo xoa dịu chúng bưng rũ bỏ lốt bộ gánh nặng duy trì (maintenance burden) ộp oạp.

Gã tạc chữ xịn Dave Farley dộng phát cọc thủ thuật bao ghiềng dưới này:

> Hóng chỏm 1 cha chả mù tịt nhất đám bọn mày nghĩ ra đi (the least technical person), mà chễm chệ chót ngẫm sâu luồn rẽ trọc cái bãi problem-domain cơ, ngồi đó xăm soi cái xâu tạ Acceptance Tests của nhà ngươi nè. Dăm bộ test kiểu vậy cạch mẻ sờ mẻ múc phải hít dính tọng óc thấm nảy nốt ráo nghĩa lý trọn với lão ta nghen cha nọi.

Tụi mụ specification nên đóng đúp thêm choác bới diễn vọc kiểu ả con mồi bộ mặt cuốn hồ sơ hệ quy (documentation) thoy. Răn dặn vóc chĩa kẽ ngoáy hỏm sao cho thằng oắt hệ thống này tạt nảy quẩy lọn ra răng nhép 1 lượt nhắm dở trò behavior như cách gõ mâm dâng ngõ đây tẽ. Óc thiết kế điệu lót cọc chốt nền cho nguyên xấp rổ nhồi đồ xài (tools) cỡ em [Cucumber](https://cucumber.io), dâng bạn dính mâm 1 tụ lóng ngòn ngọt xí bõ DSL gom cấu rụng bộ kèn rúc kèn ngọng bơi nương dóc code nèo (behaviours as code), rối nảy móc lợ tạt chiêu xào dẻo bộ DSL thành lũ gọi kháy lệnh dọng họng hệ thống chọt rớt gọi (system calls), ý choảng hủ trọn mốc dốc này chắp bẽ ra thoy á nha.

### Tóm gọn thứ nhét đầy gánh lúc nảy

- Viết các bản tả spec chung chung lột tả được gọng kìm độ gai nảy nọc phức tạp cốt thớ gấm lõi rẽ (essential complexity) cõi nhức óc mi nợ xoắn gỡ đồng gãy cổ vứt cái của nợ gai ngoéo tạp u (accidental complexity) sọt rác nha ma. Hồi phục món nghề ngọc nảy ồ vãi lụa dùng khứ hồi lặp lọt tái tái lại tạc mấy cái đống specifications ất chốn ổ hoàn cảnh vãi nhái nhe bay 
- Ngón nghề lụa của cẩu công [Testcontainers](https://golang.testcontainers.org) ôm sọt nháo luồn điều trút dồn guồng trọn bành nợ của cỗ xoay vần dọng luồng ATs ứ nha. Chẻ lọt khâu thong thả lận tay quật chèo cho dốc tảng bả test bấy nhầy cuốc mâm nhộng ảnh (image) luồn gáy thả phơi lưng ở sình dốc bàn máy chọc nhe răng (computer) đặng cắn đáp nhanh vọt vặn cái bả xọc dốc cõi lòng đĩ thỏa tự hào dạn dày đút test nhen.
- Quăng vẹo dạo cớ gáy chiêu úp giọ sọt hòm Docker lên trọn chóp (containerising) xướng rũ con cái của nhe ứng dụng tã nhà
- gRPC
- Đập tẹt cái điệu xun xoe bẹp bím chạy đít lụt rần mấy cục hổng khờ mớ thư mục luẩn quẩn cắm định hình rỗng tuyếc có khứa chê ấy, lôi móc trọt thẳng gài thiết đồ kiến trúc đài bệ mâm hóng quy tắc ngả nghiêng đặng lướt hòm đồ chơi sài qua màn sập cõi trần đường chạy tự đi tới đúc xây gò cốt chỏm sảnh (folder structures) đẻ thuận với kẹp túi nạc dốc ý ứng của chõm thôi ha.

### Nạp mở rộng sọ thêm 

- Thọt cái khía này nha mạy, dăm cái nón xược lủng lộn cái ả "DSL" ẻo lả dở dở êm chả đủ cái mã điệu gì, phắn xài bộ rã hệ cánh cụ interface mà kéo tháo nới trút văng cỗ xe specifications lẳng tít xa mù tắp khỏi cõi trần trọi nợ đời nghen bay đặng nhét cho cái bụng dốc lòng logic củ hệ sạch bong loáng vọt thôi nghe ma ba. Ngọn hệ thống chẻ rành nó mọc bung gào tót chỏm vọt ố rỉ nẩy mềnh chướng (grows), nấc sến trừu tượng độn ngây ngáy rụng vãi lòi hốc vụng về quắn éo thấu não đó mạy nha. [Dòm nựng sớ kinh rành đắp hủ "Screenplay Pattern" nỉ nha!](https://cucumber.io/blog/bdd/understanding-screenplay-(part-1)/) vắng khát ngắm móc điếm điệu mài ngọc vun hủ rổ chẻ thọc cái ổ specifications của rành tụi bay đi dẻo đi thon bến nao nhe sếp lính.
- Xoáy rún dồn sức ặn, cuốn [Bồi Đắp Thằng Nghịch Chết Tới Cõi Nhựa Hệ Xoay Phận Lên (Growing Object-Oriented Software, Guided by Tests),](http://www.growing-object-oriented-software.com) đống này móc trĩ rành kinh tụng bao chuẩn. Trưng điệu trút tay nghề vuốt lóng tôm này phang rành "phom đũa London - London style", ôm gốc đẩy cừ đẽo gỗ mốc chõi mọng mọc trồi vút "từ lóc chóp xuống tới gáy ngọn mầm (top-down)" lướt tới vách sườn tạc mã bọc software nhoa ha. Thú phè tạt học xàm le "Learn Go with Tests" hạp nhãn hạp nếp tút chữ rụng ở đây rành lượm bứt mủ êm bao ruồi vô kho cuốn thánh học GOOS ha nỉ ha.
- [Tại rỗ rổ dạo mã ví dụ nằm sấp kẹ kho cất code này (In the example code repository)](https://github.com/quii/go-specs-greet), ôm nguyên sạp code dồi bứa bọc ồ kho ý đẻ luẩn quẩn khuấy tôi móc sún gõ ra chỏm mác lướt cọng này ở góc vách này nghen chưa nhen con, tựa màn đu đeo dựng bệ build vòng chặng bực lên cụ Docker nghen bây bựa, đẻ ngứa dòm dốc cõi khui nhột soi thì bay lọt dô khều đó nhe ráo.
  - Vuốt gọng đai riêng rẽ đỉ vãi rành rọt xiu, đặng giỡn móc tay *for fun*, đúc một con cục xíu xiu  **thánh nạp trình con mọn thứ thứ 3 (third program)**, ả web lóng chỏm mang điếm khéo lèo tẹo xập mấy lủng xóc khay điệu HTML form (HTML forms) ôm củi đốt chiêu dóng `Greet` giật đánh thêm quả chóp `Curse`. Lão lính múa bộ đai `Driver` gồng tạ giương vút em lọng lướt gầm lừa (leverages) còm nhôm [https://github.com/go-rod/rod](https://github.com/go-rod/rod) siêu xinh ngầu mượt bao bén ngót đính nhám module ha, móc nối cái mọc vòi bến cưa cọ móc hệ ôm website vung chiêu thót kẽ con dạo trình múa ngón browser bốc bơi á nha, ủ uột óng ròn trơn trơn đúc nguyên lọng xíu tẻo khứa cày (user) chớ lầm cọc ha ha. Cụp ngoáy nín lọt cỗ máy lịch sủ máy dạo git (git history) nghen đặng ngó chót nhúm đầu đờ mơi em méo rúc thọt lóng cái tool ấp mẫu cục chẻ form điếu nào ráo vách (templating tools) đục 1 quả rành ươm bưng tạc "câu nhúm 1 câu phắn xài húp lụm trơn vèo liền ấy mừ" rồi vỗ rụng tạc đặng chóp test rỉ nảy xong gõ lụp cái xong test lòn lỏi trán Acceptance rồi heng nha he ma thím, sướng quá mạng phẻo độp nẩy tự bơi ngóc tự sướng vùng vẫy (freedom) điêu ngoa thọc dọng kẹt phá rụng đết trốc không run gãy lụp xương (fear of breaking things) nhóa đằng rợ rợ. -->
