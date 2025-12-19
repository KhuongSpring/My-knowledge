# SYSTEM DESIGN NOTE

![Status](https://img.shields.io/badge/Status-Learning-blue) ![Topic](https://img.shields.io/badge/Topic-System%20Design-orange)

System Design cho người mới bắt đầu.
Những kiến thức cơ bản giúp bạn phỏng vấn hiệu quả.

> **Lưu ý:** Đây là tài liệu ghi chép, tổng hợp và tóm tắt lại kiến thức cá nhân của tôi trong quá trình đọc Ebook về System Design.
>
> **© Bản quyền gốc thuộc về:** Tác giả **Nguyễn Thế Huy** (https://huynt.dev). Mọi trích dẫn và nội dung cốt lõi đều nhằm mục đích học tập và lưu trữ kiến thức.

---

## Mục lục

### [Lời mở đầu](#lời-mở-đầu-1)

### [Giới thiệu](#giới-thiệu-1)

- [System Design Interview là gì?](#system-design-interview-là-gì)
- [Tại sao vòng phỏng vấn này quan trọng?](#tại-sao-vòng-phỏng-vấn-này-quan-trọng)
- [Những khó khăn mà người mới thường gặp](#những-khó-khăn-mà-người-mới-thường-gặp)
- [Các kỹ năng mà nhà tuyển dùng mong muốn](#các-kỹ-năng-mà-nhà-tuyển-dùng-mong-muốn)

### [Các khái niệm nền tảng trong Thiết kế hệ thống](#các-khái-niệm-nền-tảng-trong-thiết-kế-hệ-thống-1)

- [Latency (Độ trễ)](#latency-độ-trễ)
- [Throughput (Thông lượng)](#throughput-thông-lượng)
- [Scalability (Khả năng mở rộng)](#scalability-khả-năng-mở-rộng)
- [Availability (Tính khả dụng)](#availability-tính-khả-dụng)
- [Reliability (Độ tin cậy)](#reliability-độ-tin-cậy)
- [High-Level Architecture (Kiến trúc cấp cao)](#high-level-architecture-kiến-trúc-cấp-cao)
- [Các thành phần phổ biến trong hệ thống](#các-thành-phần-phổ-biến-trong-hệ-thống)
    - [Web Server (Máy chủ Web)](#web-server-máy-chủ-web)
    - [Application Server (Máy chủ ứng dụng)](#application-server-máy-chủ-ứng-dụng)
    - [Database (Cơ sở dữ liệu)](#database-cơ-sở-dữ-liệu)
    - [Load Balancer (Bộ cân bằng tải)](#load-balancer-bộ-cân-bằng-tải)
    - [Cache (Bộ nhớ đệm)](#cache-bộ-nhớ-đệm)
    - [CDN (Mạng phân phối nội dung)](#cdn-mạng-phân-phối-nội-dung)
    - [Message Queue (Hàng đợi thông điệp)](#message-queue-hàng-đợi-thông-điệp)

### [Tính nhất quán và khả dụng trong System Design](#tính-nhất-quán-và-khả-dụng-trong-system-design-1)

- [Giới thiệu](#giới-thiệu-2)
- [Tính nhất quán và tính khả dụng](#tính-nhất-quán-và-tính-khả-dụng)
- [Các mô hình nhất quán (Consistency Models)](#các-mô-hình-nhất-quán-consistency-models)
    - [Nhất quán mạnh (Strong Consistency)](#nhất-quán-mạnh-strong-consistency)
    - [Nhất quán cuối cùng (Eventual Consistency)](#nhất-quán-cuối-cùng-eventual-consistency)
    - [Nhất quán yếu (Weak Consistency)](#nhất-quán-yếu-weak-consistency)
- [Cơ chế đảm bảo tính nhất quán và khả dụng](#cơ-chế-đảm-bảo-tính-nhất-quán-và-khả-dụng)
    - [Nhân bản dữ liệu (Replication)](#nhân-bản-dữ-liệu-replication)
    - [Chuyển đổi dự phòng (Failover)](#chuyển-đổi-dự-phòng-failover)
    - [Quorum (Đa số phiếu)](#quorum-đa-số-phiếu)
- [Định lý CAP và các đánh đổi kinh điển](#định-lý-cap-và-các-đánh-đổi-kinh-điển)
- [Cách trình bày về nhất quán và khả dụng khi phỏng vấn](#cách-trình-bày-về-nhất-quán-và-khả-dụng-khi-phỏng-vấn)

### [Lựa chọn Cơ sở dữ liệu, Sharding và Tối ưu hóa lưu trữ](#lựa-chọn-cơ-sở-dữ-liệu-sharding-và-tối-ưu-hóa-lưu-trữ-1)

- [SQL vs NoSQL: Lựa chọn cơ sở dữ liệu phù hợp](#sql-vs-nosql-lựa-chọn-cơ-sở-dữ-liệu-phù-hợp)
- [Phân mảnh dữ liệu (Sharding/Partitioning)](#phân-mảnh-dữ-liệu-shardingpartitioning)
- [Chỉ mục (Index): Tối ưu hóa truy vấn](#chỉ-mục-index-tối-ưu-hóa-truy-vấn)
- [Cân nhắc khi chọn kiểu lưu trữ cho hệ thống lớn](#cân-nhắc-khi-chọn-kiểu-lưu-trữ-cho-hệ-thống-lớn)
- [Dữ liệu phân tán và tính nhất quán (Consistency)](#dữ-liệu-phân-tán-và-tính-nhất-quán-consistency)
- [Gợi ý trình bày phần thiết kế Database khi phỏng vấn](#gợi-ý-trình-bày-phần-thiết-kế-database-khi-phỏng-vấn)

### [Mở rộng hệ thống và tối ưu hiệu suất](#mở-rộng-hệ-thống-và-tối-ưu-hiệu-suất-1)

- [Cân bằng tải (Load Balancer)](#cân-bằng-tải-load-balancer)
    - [Load Balancer là gì và vai trò của nó?](#load-balancer-là-gì-và-vai-trò-của-nó)
    - [Cách hoạt động và các thuật toán phân phối](#cách-hoạt-động-và-các-thuật-toán-phân-phối)
    - [Load Balancer tầng 4 vs tầng 7 (Layer 4 vs Layer 7)](#load-balancer-tầng-4-vs-tầng-7-layer-4-vs-layer-7)
    - [Khi nào cần sử dụng Load Balancer?](#khi-nào-cần-sử-dụng-load-balancer)
- [Bộ nhớ đệm (Caching)](#bộ-nhớ-đệm-caching)
    - [Cache là gì và có lợi ích gì?](#cache-là-gì-và-có-lợi-ích-gì)
    - [Các vị trí có thể đặt cache](#các-vị-trí-có-thể-đặt-cache)
    - [Các chiến lược cập nhật cache (Cache update strategies)](#các-chiến-lược-cập-nhật-cache-cache-update-strategies)
    - [Gợi ý khi phỏng vấn về Caching](#gợi-ý-khi-phỏng-vấn-về-caching)
- [Hàng đợi thông điệp (Message Queue)](#hàng-đợi-thông-điệp-message-queue)
    - [Khái niệm hàng đợi thông điệp & xử lý bất đồng bộ](#khái-niệm-hàng-đợi-thông-điệp--xử-lý-bất-đồng-bộ)
    - [Khi nào nên dùng hàng đợi](#khi-nào-nên-dùng-hàng-đợi)
    - [Một số hệ thống Message Queue phổ biến](#một-số-hệ-thống-message-queue-phổ-biến)
    - [Mô hình Producer - Queue - Consumer](#mô-hình-producer---queue---consumer)
    - [Cơ chế backpressure (phản áp lực) trong hàng đợi](#cơ-chế-backpressure-phản-áp-lực-trong-hàng-đợi)
    - [Ví dụ minh họa sử dụng Message Queue](#ví-dụ-minh-họa-sử-dụng-message-queue)
    - [Gợi ý khi phỏng vấn về Message Queue](#gợi-ý-khi-phỏng-vấn-về-message-queue)

### [Idempotency và cơ chế khóa trong hệ thống phân tán](#idempotency-và-cơ-chế-khóa-trong-hệ-thống-phân-tán-1)

- [Pessimistic Locking (Khóa bi quan)](#pessimistic-locking-khóa-bi-quan)
- [Optimistic Locking (Khóa lạc quan)](#optimistic-locking-khóa-lạc-quan)
- [Khi nào dùng & lựa chọn](#khi-nào-dùng--lựa-chọn)

### [Các phương thức giao tiếp và thiết kế API](#các-phương-thức-giao-tiếp-và-thiết-kế-api-1)

- [HTTP: Giao thức request/response phổ biến nhất](#http-giao-thức-requestresponse-phổ-biến-nhất)
- [TCP: Giao thức kết nối tin cậy](#tcp-giao-thức-kết-nối-tin-cậy)
- [UDP: Giao thức "gửi là quên"](#udp-giao-thức-gửi-là-quên)
- [RPC: Gọi thủ tục từ xa như gọi hàm nội bộ](#rpc-gọi-thủ-tục-từ-xa-như-gọi-hàm-nội-bộ)
- [REST: Phong cách thiết kế API rõ ràng, thống nhất](#rest-phong-cách-thiết-kế-api-rõ-ràng-thống-nhất)

### [Đảm bảo khả năng chịu lỗi và phục hồi hệ thống](#đảm-bảo-khả-năng-chịu-lỗi-và-phục-hồi-hệ-thống-1)

- [Khả năng chịu lỗi là gì và tại sao quan trọng?](#khả-năng-chịu-lỗi-là-gì-và-tại-sao-quan-trọng)
- [Các sự cố phổ biến trong hệ thống phân tán](#các-sự-cố-phổ-biến-trong-hệ-thống-phân-tán)
- [Chiến lược tăng độ bền vững (resilience) cho hệ thống](#chiến-lược-tăng-độ-bền-vững-resilience-cho-hệ-thống)
- [Phân biệt High Availability và Fault Tolerance](#phân-biệt-high-availability-và-fault-tolerance)
- [Đánh giá độ tin cậy: Uptime, SLA, MTTR, MTBF](#đánh-giá-độ-tin-cậy-uptime-sla-mttr-mtbf)
- [Gợi ý trình bày chiến lược phục hồi sự cố trong phỏng vấn](#gợi-ý-trình-bày-chiến-lược-phục-hồi-sự-cố-trong-phỏng-vấn)

### [Bảo mật và phân quyền](#bảo-mật-và-phân-quyền-1)

- [Vai trò của bảo mật trong hệ thống phân tán hiện đại](#vai-trò-của-bảo-mật-trong-hệ-thống-phân-tán-hiện-đại)
    - [Các khái niệm cơ bản](#các-khái-niệm-cơ-bản)
        - [Xác thực (Authentication)](#xác-thực-authentication)
        - [Phân quyền (Authorization)](#phân-quyền-authorization)
        - [Bảo mật API](#bảo-mật-api)
        - [Bảo vệ dữ liệu](#bảo-vệ-dữ-liệu)
        - [HTTPS, TLS và bảo mật cookie](#https-tls-và-bảo-mật-cookie)
    - [Ví dụ minh họa](#ví-dụ-minh-họa)
    - [Lời khuyên khi phỏng vấn về bảo mật trong System Design](#lời-khuyên-khi-phỏng-vấn-về-bảo-mật-trong-system-design)
- [Monitoring và Observability](#monitoring-và-observability)
    - [Monitoring là gì?](#monitoring-là-gì)
    - [Observability là gì?](#observability-là-gì)
    - [Trình bày về Monitoring và Observability trong buổi phỏng vấn](#trình-bày-về-monitoring-và-observability-trong-buổi-phỏng-vấn)

### [Lời kết](#lời-kết-1)

---

## <a id="lời-mở-đầu-1"></a>Lời mở đầu

> "Chào bạn, và cám ơn bạn đã đọc ebook này của mình.
>
> Mình là Huy, hiện tại mình là một Technical Leader với trên 10 năm kinh nghiệm.
> Ebook này của mình nhằm cung cấp cho bạn những khái niệm cốt lõi, cũng như kinh nghiệm tham gia phỏng vấn System Design (thiết kế hệ thống).
>
> Dù bạn là người mới bắt đầu hay đã có kinh nghiệm, mình tin những kiến thức trong ebook này sẽ giúp bạn rất nhiều trong việc review lại kiến thức về thiết kế, cũng như chuẩn bị cho các cuộc phỏng vấn về System Design.
>
> Chúc bạn có những giây phút thú vị, bổ ích khi đọc ebook."

## <a id="giới-thiệu-1"></a> Giới thiệu

### System Design Interview là gì?

[System Design](https://roadmap.sh/system-design) Interview hay phỏng vấn thiết kế hệ thống là một vòng phỏng vấn kỹ thuật trong đó bạn được yêu cầu thiết kế kiến trúc cho một hệ thống phần mềm giả định nào đó, thay vì coding trực tiếp. Thông thường, người phỏng vấn sẽ đưa ra một bài toán mở.  

Nhiệm vụ của bạn là thảo luận và đề xuất một giải pháp kiến trúc tổng thể: bao gồm các thành phần chính của hệ thống, cách các thành phần này tương tác với nhau, và làm thế nào hệ thống đáp ứng được các yêu cầu đề ra.

Trong vòng phỏng vấn này, bạn sẽ không viết code chi tiết mà tập trung vào việc phác thảo và giải thích ý tưởng thiết kế. Người phỏng vấn thường quan sát cách bạn tiếp cận một bài toán lớn: từ việc làm rõ yêu cầu, đề xuất các thành phần kiến trúc, cho đến việc phân tích những điểm mạnh, điểm yếu của từng lựa chọn thiết kế. Mục đích là để đánh giá khả năng của bạn trong việc xây dựng một hệ thống phần mềm có tính mở rộng, ổn định và hiệu quả dựa trên yêu cầu đưa ra. 

Nói cách khác, cuộc phỏng vấn đề system design nhằm kiểm tra kỹ năng giải quyết vấn đề kỹ thuật phức tạp ở tầm kiến trúc của bạn: hiểu bài toán, đề ra hướng giải quyết, thảo luận các phương án và đưa ra quyết định phù hợp. 

### Tại sao vòng phỏng vấn này quan trọng?

Vòng phỏng vấn system design ngày càng trở thành yếu tố then chốt trong quy trình tuyển dụng kỹ sư phần mềm, đặc biệt tại các công ty công nghệ lớn. Lý do là bởi việc xây dựng và vận hành các hệ thống phần mềm quy mô lớn tiêu tốn rất nhiều nguồn lực, và các công ty muốn đảm bảo rằng những kỹ sư họ tuyển vào có thể nhanh chóng bắt tay vào thiết kế những hệ thống như vậy. Nói cách khác, họ dùng vòng phỏng vấn này như phương pháp chính để đánh giá liệu ứng viên có kiến thức và tư duy thiết kế hệ thống đủ mạnh hay không. 

Vòng phỏng vấn này đôi khi còn được đánh giá cao hơn vòng coding, đặc biệt trong thời kỳ AI thống trị, việc coding càng ngày càng được chuyển giao nhiều hơn cho trí tuệ nhân tạo. Coding interview thường chỉ kiểm tra khả năng giải thuật và coding ở quy mô nhỏ, trong khi thiết kế hệ thống kiểm tra tư duy chiến lược hơn: bạn hiểu bức tranh toàn cảnh ra sao, bạn sẽ thiết kế hệ thống phục vụ hàng triệu người dùng như thế nào, và bạn ra quyết định kỹ thuật ra sao khi đối mặt với các ràng buộc thực tế (hiệu năng, chi phí, độ phức tạp, v.v.). Đó là lý do các công ty lớn rất coi trọng vòng này, họ muốn tuyển người không chỉ code giỏi, mà còn thiết kế hệ thống tinh gọn, linh hoạt và có thể mở rộng cho tương lai.

### Những khó khăn mà người mới thường gặp

- **Câu hỏi mở, không có đáp án cố định:** Các câu hỏi thiết kế hệ thống thường rất mở và không có một đáp án duy nhất đúng hẳn. Không có tiêu chí "đúng" hoặc "sai" rõ ràng như trong bài lập trình, nên bạn dễ cảm thấy lúng túng, không biết nên bắt đầu từ đâu và định hướng giải quyết vấn đề ra sao. Mỗi người có thể đưa ra một cách tiếp cận khác nhau cho cùng một bài toán thiết kế, miễn là đáp ứng được các yêu cầu cơ bản - do đó sẽ không có phương án nào được coi là hoàn toàn đúng hoặc hoàn toàn sai.

- **Thiếu kinh nghiệm thực tế về hệ thống lớn:** Nhiều lập trình viên trẻ chưa từng tham gia thiết kế hoặc vận hành những hệ thống lớn trong công việc thực tế. Điều này dẫn đến việc bạn thiếu kiến thức về các thành phần kiến trúc quan trọng (ví dụ: máy chủ ứng dụng, cơ sở dữ liệu phân tán, caching, message queue, load balancing, v.v.) và không biết cách áp dụng chúng trong thiết kế. Khi nhận một bài toán thiết kế, bạn có thể bối rối không rõ nên dùng công nghệ nào, module nào cho phù hợp, đơn giản vì bạn chưa từng làm việc với hệ thống tương tự quy mô đó. 

- **Áp lực thời gian và tâm lý:** Mỗi buổi phỏng vấn thiết kế hệ thống thường chỉ kéo dài khoảng 45 - 60 phút, đây không phải là nhiều thời gian để bạn trình bày một kiến trúc phức tạp. Bạn phải vừa tư duy giải pháp, vừa diễn đạt nó một cách mạch lạc dưới áp lực thời gian, nên rất dễ căng thẳng. Nhiều người mới có thể bị "khớp" khi đối diện với bảng trắng và một vấn đề lớn, hoặc phân bổ thời gian không hợp lý (ví dụ: mải mê vẽ chi tiết mà không kịp nói hết ý tưởng chính). Kết quả là bài thiết kế trình bày dang dở hoặc thiếu nhiều phần quan trọng. Rõ ràng, **kỹ năng quản lý thời gian** và **bình tĩnh xử lý tình huống áp lực** là một thách thức lớn ở vòng này. 

- **Chưa có phương pháp tiếp cận bài bản:** Thiết kế một hệ thống quy mô lớn là vấn đề phức tạp, đòi hỏi một phương pháp luận rõ ràng. Thực tế, đa số ứng viên thất bại ở vòng System Design là do thiếu một quy trình tư duy lặp lại được cho các câu hỏi mở. Nếu bạn không có sẵn cho mình một chiến lược (chẳng hạn: bước 1 hỏi rõ yêu cầu, bước 2 đề xuất thiết kế tổng quan, bước 3 phân tích chuyên sâu các thành phần...), bạn sẽ dễ bị lúng túng trước lượng thông tin và lựa chọn khổng lồ. Việc thiếu phương pháp cũng khiến bạn có thể bỏ sót những khía cạnh quan trọng của hệ thống. 

- **Ám ảnh về “giải pháp hoàn hảo”:** Không ít bạn lo lắng rằng mình phải đưa ra một thiết kế hoàn mỹ không có bất kỳ sai sót nào thì mới được đánh giá cao. Thực ra, người phỏng vấn không kỳ vọng một giải pháp 100% hoàn hảo từ bạn. Không hề có “đáp án đúng tuyệt đối” trong phỏng vấn thiết kế hệ thống, vì vậy điều quan trọng hơn là cách bạn tiếp cận vấn đề. Họ muốn thấy cách bạn đưa ra quyết định khi thông tin chưa đầy đủ, cách bạn xử lý các tình huống không chắc chắn, và liệu bạn có đủ linh hoạt để thích ứng khi yêu cầu thay đổi hay không. Hiểu được điều này sẽ giúp bạn tự tin hơn và tập trung vào quy trình giải quyết vấn đề thay vì cố tìm một đáp án hoàn hảo duy nhất. Hãy nhớ rằng phỏng vấn thiết kế hệ thống chủ yếu đánh giá tư duy và kỹ năng giao tiếp kỹ thuật của bạn, chứ không phải tìm kiếm một thiết kế hoàn chỉnh để đem triển khai ngay. 

### Các kỹ năng mà nhà tuyển dùng mong muốn

- **Kiến thức nền tảng vững chắc:** Bạn cần nắm vững các kiến thức cơ bản về thiết kế và kiến trúc hệ thống. Điều này bao gồm hiểu biết về các thành phần như máy chủ, cơ sở dữ liệu, hệ thống storage, networking, caching, load balancing, v.v. Bên cạnh đó, bạn cũng nên nắm các khái niệm như tính mở rộng (scalability), tính sẵn sàng (availability), độ trễ (latency), tính nhất quán (consistency)... Kiến thức nền tảng vững giúp bạn tự tin đề xuất các giải pháp phù hợp và giải thích được tại sao chọn thiết kế đó. 

- **Tư duy phân tích và giải quyết vấn đề:** Kỹ năng phân tích một bài toán lớn thành các phần nhỏ hơn là rất quan trọng. Nhà tuyển dụng muốn thấy bạn biết cách làm rõ yêu cầu bài toán (functional & non-functional requirements), biết đặt câu hỏi để bổ sung thông tin, và từ đó định hình được vấn đề cần giải quyết. Bạn cũng cần có tư duy hệ thống để xâu chuỗi các phần lại thành một giải pháp hoàn chỉnh. Khả năng xác định trọng tâm của vấn đề và giải quyết lần lượt từng khía cạnh (dữ liệu, giao tiếp giữa các dịch vụ, giao diện API, v.v.) cho thấy bạn có phương pháp làm việc rõ ràng và hiệu quả. 

- **Kỹ năng thiết kế hệ thống quy mô lớn:** Đây là khả năng hình dung và thiết kế một hệ thống có thể hoạt động tốt ở quy mô hàng triệu người dùng hoặc hơn. Bạn cần thể hiện được việc cân nhắc đến yếu tố tải cao. Đồng thời, kỹ năng này bao gồm việc thiết kế hệ thống có tính chịu lỗi (fault-tolerance): hệ thống vẫn tiếp tục hoạt động ngay cả khi một phần nào đó gặp sự cố. Nhà tuyển dụng muốn thấy rằng bạn biết nghĩ đến tương lai của hệ thống: cách hệ thống mở rộng khi lưu lượng tăng, cách sao lưu dữ liệu, cách xử lý khi một dịch vụ con bị hỏng, v.v. Đây là những yếu tố phân biệt một thiết kế tạm thời với một thiết kế tốt cho sản phẩm thực tế. 

- **Khả năng ra quyết định và cân bằng đánh đổi:** Trong thiết kế hệ thống, không có giải pháp nào là hoàn hảo toàn diện: mỗi lựa chọn đều đi kèm đánh đổi (trade-off). Người phỏng vấn sẽ đánh giá cao khả năng bạn đưa ra quyết định hợp lý và giải thích được lý do cho các lựa chọn thiết kế của mình. Họ muốn thấy bạn cân nhắc và cân bằng giữa các yếu tố đánh đổi: như hiệu năng vs. chi phí, tốc độ phát triển vs. khả năng mở rộng, độ phức tạp vs. tính linh hoạt, v.v.. Kỹ năng này thể hiện tư duy của một technical leader: biết lựa chọn hướng đi tối ưu dựa trên bối cảnh cụ thể của bài toán.

- **Kỹ năng giao tiếp và cộng tác:** Khả năng trình bày rõ ràng ý tưởng kỹ thuật của bạn là vô cùng quan trọng. Trong phỏng vấn thiết kế hệ thống, bạn cần giao tiếp mạch lạc để người phỏng vấn hiểu được kiến trúc bạn đề xuất. Điều này bao gồm việc dùng ngôn ngữ dễ hiểu, tổ chức ý tưởng có thứ tự, và minh họa bằng sơ đồ/trạng thái khi cần thiết. Bên cạnh đó, thái độ cởi mở và hợp tác cũng được đánh giá cao: người phỏng vấn có thể sẽ gợi ý hoặc đặt câu hỏi thử thách vào thiết kế của bạn, và họ muốn thấy bạn biết lắng nghe, trao đổi lại một cách tích cực, giống như khi làm việc nhóm thực sự. Việc truyền đạt ý tưởng rõ ràng và tương tác nhịp nhàng sẽ gây ấn tượng tốt rằng bạn có 
thể làm việc hiệu quả trong team. 

Tóm lại, để vượt qua vòng phỏng vấn thiết kế hệ thống, bạn cần kết hợp cả kiến thức kỹ thuật nền tảng lẫn kỹ năng mềm về phân tích, ra quyết định và giao tiếp. Nhà tuyển dụng tìm kiếm một người kỹ sư có thể nhìn toàn cảnh, thiết kế giải pháp phù hợp và thuyết phục được người khác về giải pháp đó.

## <a id="các-khái-niệm-nền-tảng-trong-thiết-kế-hệ-thống-1"></a>Các khái niệm nền tảng trong Thiết kế hệ thống

Chúng ta sẽ cùng khám phá những khái niệm nền tảng nhất trong thiết kế hệ thống. Đây là những viên gạch đầu tiên giúp bạn xây dựng tư duy System Design vững chắc.

### [Latency](https://cs.fyi/guide/latency-vs-throughput) (Độ trễ)

Độ trễ là thời gian từ lúc bạn bắt đầu gửi yêu cầu đến lúc bạn nhận được phản hồi. Ví dụ trong web: khi bạn nhấp vào một liên kết, độ trễ là thời gian từ lúc bạn nhấn chuột đến khi trang web hiển thị nội dung. Độ trễ thường được đo bằng mili giây (ms) - con số càng nhỏ thì phản hồi càng nhanh. 

Hãy tưởng tượng một tình huống đời thường: bạn gửi một bưu phẩm từ Hà Nội vào TP.Hồ Chí Minh. Thời gian từ lúc bạn gửi đến khi người nhận nhận được chính là “độ trễ” của việc vận chuyển. Tương tự, trong hệ thống máy tính, nếu máy chủ ở xa hoặc đường truyền chậm, độ trễ mạng sẽ cao - nghĩa là bạn phải chờ lâu để nhận kết quả. Ngược lại, máy chủ gần và đường truyền tốt sẽ giúp độ trễ thấp, bạn nhận được phản hồi nhanh hơn rất nhiều. 

Latency ảnh hưởng trực tiếp đến trải nghiệm người dùng. Ví dụ: một trang web có độ trễ cao (phản hồi chậm) sẽ làm người dùng khó chịu và có thể rời đi, trong khi trang web có độ trễ thấp (phản hồi gần như tức thì) sẽ tạo cảm giác mượt mà. Trong thiết kế hệ thống, chúng ta luôn cố gắng giảm độ trễ - bằng cách tối ưu đường truyền, đặt máy chủ gần người dùng, hoặc cải thiện tốc độ xử lý, để đem lại trải nghiệm tốt nhất. 

### [Throughput](https://cs.fyi/guide/latency-vs-throughput) (Thông lượng)

Thông lượng đo lường xem có bao nhiêu dữ liệu hoặc bao nhiêu yêu cầu được xử lý thành công mỗi giây (hoặc mỗi phút, mỗi giờ). Nôm na, đây là “sức tải” của hệ thống.

Hãy lấy ví dụ đời thường để hình dung: tưởng tượng bạn đang quản lý một trạm thu phí trên cao tốc. Độ trễ tương tự như thời gian một chiếc xe cần để qua trạm thu phí (mỗi xe qua nhanh hay chậm). Còn thông lượng là số lượng xe có thể qua trạm trong một phút. Một trạm thu phí có thể cho 60 xe qua mỗi phút thì có thông lượng cao hơn trạm chỉ cho 30 xe/phút. Trong hệ thống cũng vậy, một server xử lý được 1000 yêu cầu/giây có thông lượng cao hơn server chỉ xử lý 100 yêu cầu/giây. 

Throughput và latency có mối quan hệ nhưng không phải lúc nào cũng tỷ lệ thuận với nhau. Ví dụ: một hệ thống có thể có độ trễ thấp (phản hồi rất nhanh cho mỗi yêu cầu) nhưng nếu chỉ phục vụ được ít người dùng cùng lúc, thì thông lượng vẫn thấp. Ngược lại, hệ thống có thể phục vụ rất nhiều yêu cầu mỗi giây (thông lượng cao) nhưng mỗi yêu cầu lại mất nhiều thời gian (độ trễ cao cao) nếu hệ thống bị quá tải. Do đó, khi thiết kế hệ thống, chúng ta cần cân bằng cả hai yếu tố: vừa đảm bảo latency thấp để người dùng có phản hồi nhanh, vừa có throughput cao để phục vụ được nhiều người dùng đồng thời. 

### [Scalability](https://cs.fyi/guide/scalability-for-dummies) (Khả năng mở rộng)

Khả năng mở rộng là khả năng của hệ thống thích nghi và xử lý hiệu quả khi khối lượng công việc tăng lên. Nói cách khác, một hệ thống có khả năng mở rộng tốt sẽ “lớn lên” cùng với số lượng người dùng hoặc dữ liệu ngày càng tăng mà không bị suy giảm hiệu năng. Điều này đặc biệt quan trọng đối với các ứng dụng có tham vọng phục vụ hàng triệu 
người dùng. 

Hãy tưởng tượng bạn mở một quán phở nhỏ. Ban đầu quán chỉ có 5 bàn, phục vụ 20 khách mỗi sáng là tối đa. Nhưng nếu phở của bạn quá ngon và khách kéo đến ngày càng đông, bạn sẽ làm gì? Bạn có hai lựa chọn: 

**1. Mở rộng chiều dọc ([Vertical Scaling](https://www.geeksforgeeks.org/system-design/system-design-horizontal-and-vertical-scaling/)):** 

- Tăng công suất của quán hiện tại - thuê thêm đầu bếp giỏi hơn, nâng cấp nồi nấu to hơn, nguyên liệu nhiều hơn. Trong hệ thống, điều này tương tự với việc nâng cấp máy chủ hiện có (thêm CPU, thêm RAM, ổ cứng nhanh hơn). Quán phở phục vụ được nhiều khách hơn trên cùng một địa điểm, nhưng sẽ có giới hạn (một nồi phở to cỡ nào cũng có hạn chế, máy chủ cũng vậy). 

- **Ưu điểm:**
    - **Tăng dung lượng (Increased capacity):** Hiệu suất và khả năng quản lý các yêu cầu đến của máy chủ đều có thể được nâng cao bằng cách nâng cấp phần cứng.
    - **Quản lý dễ dàng hơn (Easier management):** Nâng cấp một máy chủ duy nhất thường là trọng tâm của việc mở rộng theo chiều dọc, điều này đơn giản hơn so với việc duy trì nhiều máy chủ.

- **Nhược điểm:**
    - **Khả năng mở rộng hạn chế (Limited scalability):** Mở rộng theo chiều dọc bị hạn chế bởi các giới hạn vật lý của phần cứng.
    - Một máy chủ sẽ nhận tất cả các request, do đó làm tăng khả năng xảy ra downtime trong trường hợp máy chủ gặp sự cố.
    - Scaling up thường yêu cầu khởi động lại hoặc thay thế máy, gây ra downtime.

**2. Mở rộng chiều ngang ([Horizontal Scaling](https://www.geeksforgeeks.org/system-design/system-design-horizontal-and-vertical-scaling/)):** 

- Mở thêm nhiều quán phở ở các địa điểm khác hoặc thêm nhiều nồi phở và đầu bếp hoạt động song song. Trong hệ thống, đây là việc thêm nhiều máy chủ vào hệ thống. Mỗi máy chủ mới sẽ chia sẻ gánh nặng xử lý, giống như có thêm chi nhánh phục vụ khách. Cách này thường linh hoạt và hiệu quả hơn: nếu một quán (hay một máy chủ) bận hoặc hỏng, quán khác vẫn phục vụ được.

- **Ưu điểm:**
    - **Tăng dung lượng (Increased capacity):** Nhiều máy chủ có thể xử lý số lượng request lớn hơn.
    - **Cải thiện hiệu suất (Improved performance):** Bằng cách phân phối tải trên nhiều máy chủ, khả năng một máy chủ bất kỳ nào đó bị quá tải sẽ giảm đi.
    - **Tăng khả năng chịu lỗi (Increased fault tolerance):** Các request có thể được gửi đến một máy chủ khác trong trường hợp máy chủ này bị lỗi, giảm khả năng xảy ra downtime.

- **Nhược điểm:**
    - Yêu cầu kiến ​​trúc phức tạp (bộ cân bằng tải, cơ sở dữ liệu phân tán, v.v.).
    - Khó duy trì tính nhất quán mạnh mẽ trên các máy chủ phân tán. Yêu cầu đồng bộ hóa, thông điệp hoặc nhân bản dữ liệu giữa các máy chủ.
    - Nhiều máy hơn -> cần nhiều mạng, điện năng (power) và bảo trì hơn.
    - Cần các công cụ điều phối (orchestration) (ví dụ: [Kubernetes](https://kubernetes.io/), [Ansible](https://docs.ansible.com/projects/ansible/latest/index.html)) để quản lý nhiều máy chủ.
    - Các sự cố có thể lan rộng trên nhiều máy chủ, khiến việc phân tích nguyên nhân gốc rễ trở nên khó khăn.
    - Giao tiếp giữa các nút làm tăng độ trễ và độ phức tạp.

Trong thiết kế hệ thống thực tế, mở rộng chiều ngang (thêm server) thường được ưu tiên vì nó tăng khả năng chịu tải và dự phòng lỗi tốt hơn. Tuy nhiên, triển khai mở rộng chiều ngang cần có cơ chế cân bằng tải (sẽ nói ở phần sau) để phân phối công việc giữa các máy chủ. Tóm lại, khả năng mở rộng đảm bảo rằng hệ thống của bạn có thể phình to khi cần thiết (và thậm chí thu nhỏ lại khi nhu cầu giảm) một cách trơn tru mà không làm gián đoạn dịch vụ hoặc giảm hiệu năng. 

### [Availability](https://www.geeksforgeeks.org/system-design/availability-in-system-design/) (Tính khả dụng)

Tính khả dụng thể hiện mức độ mà hệ thống luôn sẵn sàng để phục vụ: hay nói nôm na,hệ thống chạy “ổn định” được bao nhiêu phần trăm thời gian. Một hệ thống có tính khả dụng cao nghĩa là hầu như bất kỳ lúc nào người dùng cần, hệ thống đều hoạt động và đáp ứng. Ngược lại, nếu hệ thống hay downtime (ngừng hoạt động) thường xuyên, tính khả dụng sẽ thấp. 

Chúng ta thường nghe những con số “chín” huyền thoại như 99.9%, 99.99%... Ví dụ 99.99% thời gian hoạt động nghĩa là trong 10000 phút thì hệ thống chỉ được phép ngưng tối đa 1 phút. Những dịch vụ lớn đặt mục tiêu “năm số 9” (99.999%) tức downtime chỉ vài giây mỗi năm, tức là gần như không bao giờ “chết”. Tất nhiên, đạt được điều này không dễ, nhưng nó cho thấy tầm quan trọng của tính khả dụng. 

Trong thiết kế hệ thống, để tăng tính khả dụng, người ta thường sử dụng các giải pháp như dự phòng: có nhiều máy chủ chạy song song, nếu một máy hỏng sẽ có máy khác thay thế ngay. Load balancing (cân bằng tải) cũng giúp tăng khả dụng (vì có thể rút một máy ra sửa mà người dùng không bị ảnh hưởng). Ngoài ra còn có các kỹ thuật như triển khai ở nhiều trung tâm dữ liệu khác nhau (multi-zone, multi-region) để tránh sự cố ở một nơi làm sập toàn bộ hệ thống. Mục tiêu cuối cùng là hệ thống luôn “sẵn sàng” phục vụ bất kể hoàn cảnh.

### [Reliability](https://www.geeksforgeeks.org/system-design/reliability-in-system-design/) (Độ tin cậy)

Độ tin cậy thể hiện mức độ hệ thống vận hành ổn định và nhất quán trong thời gian dài, đảm bảo tính toàn vẹn của dữ liệu và dịch vụ. Một hệ thống tin cậy là hệ thống mà bạn có thể tin tưởng rằng nó sẽ không bị lỗi nghiêm trọng, không mất mát dữ liệu, và nếu có sự cố nhỏ thì hệ thống cũng tự phục hồi hoặc có cơ chế khôi phục mà không ảnh hưởng đến người dùng.

Ví dụ, trong một hệ thống thương mại điện tử hoặc ngân hàng, độ tin cậy là vô cùng quan trọng - mỗi giao dịch của khách hàng phải được lưu lại chắc chắn, không thể “mất giữa chừng” chỉ vì một server nào đó sập. Nếu một máy chủ xử lý giao dịch bị lỗi, một máy chủ khác phải tiếp quản ngay với đầy đủ dữ liệu (nhờ sao lưu thời gian thực), đảm bảo giao dịch không bị mất hoặc sai sót. 

Bạn có thể nghe người ta so sánh tính khả dụng và độ tin cậy. Hai khái niệm này liên quan nhưng không giống hệt nhau: một hệ thống có độ tin cậy cao thì gần như chắc chắn cũng khả dụng cao, vì nó hiếm khi gặp sự cố. Nhưng một hệ thống khả dụng cao (ít downtime) chưa chắc đã tin cậy về dữ liệu. Nói cách khác, reliability bao hàm cả high availability (khả dụng cao) nhưng còn thêm yếu tố toàn vẹn và đúng đắn của hoạt động. 

Trong thực tế, đạt được độ tin cậy tuyệt đối là rất khó, nên các kiến trúc sư thường tập trung vào tính khả dụng cao và giảm thiểu rủi ro mất dữ liệu đến mức thấp nhất có thể. Các kỹ thuật nâng cao độ tin cậy thường gồm: sao lưu và phục hồi dữ liệu, replication (nhân bản dữ liệu sang nhiều máy), thiết kế loại bỏ điểm lỗi đơn (single point of failure), v.v. 

Tóm lại, độ tin cậy giúp người dùng yên tâm rằng hệ thống của bạn “sống dai sống khỏe” và giữ gìn dữ liệu của họ an toàn. 

### [High-Level Architecture](https://faculty.sites.iastate.edu/tesfatsi/archive/tesfatsi/HLAIntro.RMcFarlane.pdf) (Kiến trúc cấp cao)

> Bạn có thể tìm hiểu thêm về [High-Level Design](https://www.geeksforgeeks.org/system-design/what-is-high-level-design-learn-system-design/).

Khi thiết kế hệ thống, trước khi đi vào chi tiết cụ thể, chúng ta thường vẽ ra kiến trúc cấp cao: một bức tranh tổng quát về hệ thống. 

Kiến trúc cấp cao là cái nhìn toàn cảnh về các thành phần chính của hệ thống và cách chúng tương tác với nhau, không đi sâu vào chi tiết. Nó giống như bản thiết kế tổng thể của một ngôi nhà trước khi bạn lo chi tiết từng phòng ốc, hay như bản đồ thành phố trước khi zoom vào từng con hẻm. 

Ví dụ, khi thiết kế một hệ thống mạng xã hội đơn giản, kiến trúc cấp cao có thể bao gồm: người dùng trên ứng dụng di động/website gửi yêu cầu lên máy chủ ứng dụng, máy chủ này đọc/ghi dữ liệu từ cơ sở dữ liệu, có thể gọi thêm một số dịch vụ phụ trợ khác (ví dụ dịch vụ lưu trữ hình ảnh, dịch vụ tìm kiếm), và tất cả được đặt sau một load balancer (cân bằng tải). 

Kiến trúc cấp cao có thể được vẽ dưới dạng sơ đồ các khối (boxes) và đường nối giữa chúng, mỗi khối là một thành phần (như web server, database, cache, v.v.). Mục đích của kiến trúc cấp cao là giúp bạn và đội ngũ của bạn cùng hiểu rõ cấu trúc tổng thể của hệ thống trước. Nhờ đó, ta xác định được các thành phần chính, mối quan hệ giữa chúng, điểm mạnh, điểm yếu và các điểm có thể là nút thắt cổ chai. 

Kiến trúc cấp cao tốt sẽ đơn giản, rõ ràng, tránh ôm đồm quá nhiều chi tiết, nhưng cũng phải đủ thành phần để đáp ứng yêu cầu. Trong các buổi phỏng vấn thiết kế hệ thống, người ta rất hay yêu cầu bạn phác thảo kiến trúc cấp cao trước - đây là kỹ năng quan trọng, giúp bạn trình bày ý tưởng một cách mạch lạc và có tổ chức. 

### Các thành phần phổ biến trong hệ thống

Bây giờ, chúng ta cùng điểm qua những thành phần phổ biến thường xuất hiện trong kiến trúc của một hệ thống lớn. Hiểu rõ vai trò của từng thành phần này sẽ giúp bạn rất nhiều khi phân tích hoặc thiết kế hệ thống: 

1. <a id="web-server-máy-chủ-web"></a>**[Web Server](https://www.geeksforgeeks.org/node-js/web-server-and-its-type/) (Máy chủ Web)**

Web server là thành phần chịu trách nhiệm tiếp nhận các yêu cầu HTTP từ client (trình duyệt hoặc ứng dụng) và phản hồi lại kết quả (thường là nội dung trang web hoặc dữ liệu API). Web server giống như lễ tân ở đầu hệ thống - mỗi khi có “khách” (request) đến gõ cửa, lễ tân này sẽ xem khách yêu cầu gì, sau đó quyết định sẽ đưa khách đó tới phòng ban nào (ví dụ đưa yêu cầu cho Application server xử lý) rồi nhận kết quả trả về cho khách. 

Các web server phổ biến hiện nay như [Apache](https://www.geeksforgeeks.org/linux-unix/what-is-apache-server/), [Nginx](https://www.geeksforgeeks.org/operating-systems/what-is-nginx-web-server-and-how-to-install-it/), hoặc IIS… không chỉ đơn thuần chuyển tiếp yêu cầu mà còn có thể phục vụ các nội dung tĩnh (static content) như hình ảnh, file CSS, JS trực tiếp cho client, hoặc thực hiện cân bằng tải đơn giản. Trong một số kiến trúc, web server và application server có thể tích hợp làm một (ví dụ nhiều framework tự nhúng server). Nhưng về khái niệm, web server là cửa ngõ giao tiếp giữa bên ngoài và hệ thống của bạn, quản lý phiên làm việc (session), bảo mật đầu vào ở mức cơ bản (như chặn một số request xấu) trước khi chuyển cho tầng xử lý bên trong. 

Ví dụ: Khi bạn gõ URL https://facebook.com vào trình duyệt, yêu cầu đó sẽ được gửi tới web server của Facebook trước. Web server sẽ kiểm tra bạn có đăng nhập chưa, bạn yêu cầu trang nào, sau đó mới chuyển tiếp yêu cầu đến các dịch vụ bên trong để lấy dữ liệu về trang Facebook cá nhân của bạn rồi gửi lại trình duyệt.

2. <a id="application-server-máy-chủ-ứng-dụng"></a>**[Application Server](https://www.geeksforgeeks.org/computer-networks/difference-between-web-server-and-application-server/) (Máy chủ ứng dụng)**

Nếu web server là lễ tân, thì application server chính là “phòng bếp” hoặc “đầu bếp” nơi thực sự xử lý yêu cầu và thực thi nghiệp vụ. Đây là nơi chứa logic chính của ứng dụng. Application server nhận các yêu cầu đã được web server “chuyển vào”, sau đó thực hiện các công việc như: kiểm tra thông tin người dùng, xử lý các tính toán, truy vấn hoặc cập nhật dữ liệu trong database, gọi tới các dịch vụ khác nếu cần, rồi tạo ra kết quả (ví dụ nội dung trang HTML, hoặc dữ liệu JSON) để trả về cho web server gửi lại người dùng. 

Application server có thể được xây dựng bằng nhiều ngôn ngữ và framework khác nhau - ví dụ Java với [Spring Boot](https://spring.io/projects/spring-boot), Python với [Django](https://www.djangoproject.com/)/[Flask](https://flask.palletsprojects.com/en/stable/), JavaScript/Node.js với [Express](https://expressjs.com/), C# với [.NET](https://dotnet.microsoft.com/en-us/download), v.v. Dù công nghệ gì thì vai trò chung của nó vẫn là thực thi logic nghiệp vụ (business logic). Trong hệ thống lớn, ta thường có nhiều application server chạy song song (đặt sau load balancer) để đảm bảo phục vụ lượng lớn người dùng. Mỗi application server thường giống nhau (cùng một ứng dụng được deploy), nhờ đó bất kỳ cái nào cũng có thể xử lý yêu cầu như nhau - việc này giúp dễ mở rộng và dự phòng. 

3. <a id="database-cơ-sở-dữ-liệu"></a>**[Database](https://www.geeksforgeeks.org/dbms/what-is-database/) (Cơ sở dữ liệu)**

Database (cơ sở dữ liệu) là thành phần đóng vai trò lưu trữ dữ liệu lâu dài cho hệ thống. Nếu coi hệ thống như một ứng dụng thì database chính là “bộ nhớ” của ứng dụng đó. Mọi thông tin cần được ghi nhớ và truy xuất sau này: từ tài khoản người dùng, bài viết, đơn hàng, tin nhắn… đều sẽ lưu trong database. 

Có nhiều loại database, nhưng phổ biến nhất trong hệ thống web là: 

- Database quan hệ (SQL): như [MySQL](https://www.mysql.com/), [PostgreSQL](https://www.postgresql.org/), [Oracle](https://www.oracle.com/), [Microsoft SQL Server](https://www.microsoft.com/en-us/sql-server/#tabs-pill-bar-oc2ce4_tab0),... Lưu dữ liệu thành bảng, có cấu trúc rõ ràng, và dùng ngôn ngữ truy vấn SQL. Thích hợp cho dữ liệu có cấu trúc chặt chẽ và cần các truy vấn phức tạp (như join nhiều bảng). 

- Database phi quan hệ (NoSQL): như [MongoDB](https://www.mongodb.com/?msockid=0af41d1827fe67273b750e8e23fe616c), [Cassandra](https://cassandra.apache.org/_/index.html), [Redis](https://redis.io/), [DynamoDB](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html),... Lưu dữ liệu dạng linh hoạt hơn (document, key-value, graph,...). Thích hợp cho dữ liệu không có cấu trúc cố định, hoặc cần hiệu năng cao, mở rộng dễ dàng. 

Trong một hệ thống lớn, database thường chạy trên một (hoặc nhiều) máy chủ cơ sở dữ liệu riêng, tách biệt với application server. Điều này cho phép ta mở rộng từng thành phần độc lập và đảm bảo hiệu năng. Đôi khi, để tăng khả năng đáp ứng, người ta còn triển khai replication (nhân bản cơ sở dữ liệu sang nhiều máy: một máy chính để ghi, nhiều máy phụ để đọc), giúp chia tải và tăng độ tin cậy.

4. <a id="load-balancer-bộ-cân-bằng-tải"></a>**[Load Balancer](https://www.geeksforgeeks.org/system-design/what-is-load-balancer-system-design/) (Bộ cân bằng tải)**

Load balancer là thành phần đặc biệt có nhiệm vụ phân phối đều tải công việc đến nhiều máy chủ phía sau nó. Bạn có thể hình dung load balancer như một anh điều phối giao thông hoặc người gác cổng thông minh: mọi yêu cầu từ client thay vì đi thẳng đến một máy chủ cố định, sẽ đi qua load balancer trước. Load balancer sau đó quyết định gửi yêu cầu đó đến máy chủ nào đang rảnh rỗi hoặc phù hợp, đảm bảo không máy nào bị quá tải trong khi máy khác nhàn rỗi. 

Vai trò chính của load balancer gồm: 

- Phân phối tải: Ví dụ có 5 server ứng dụng, load balancer sẽ đảm bảo 5 server này mỗi cái nhận ~20% lượng request, không ai phải “gánh” 90% trong khi người khác chỉ 10%. Thuật toán phân phối có thể là round-robin (xoay vòng lần lượt), least connections (ưu tiên server nào đang ít kết nối), hoặc thông minh hơn tùy hệ thống. 
    > Có thể tìm hiểu thêm về [Load Balancing Algorithms](https://www.geeksforgeeks.org/system-design/load-balancing-algorithms/) tại đây.

- Tính dự phòng và khả dụng cao: Nếu một server phía sau bị sập, load balancer có thể loại server đó ra khỏi danh sách tạm thời và chuyển hướng request sang các server còn lại. Nhờ đó người dùng hầu như không nhận ra có sự cố. Khi server kia hoạt động lại, load balancer lại thêm vào. Điều này tăng tính khả dụng cho hệ thống (hệ thống vẫn phục vụ bình thường dù một máy hỏng). 

- Tính linh hoạt: Bạn có thể thêm server mới hoặc gỡ server cũ xuống để bảo trì dễ dàng mà không phải dừng toàn bộ dịch vụ: chỉ cần báo cho load balancer biết để điều chỉnh phân phối. 

Có các loại load balancer phần cứng (thiết bị chuyên dụng) hoặc phần mềm (ví dụ [HAProxy](https://www.haproxy.org/), [Nginx](https://www.geeksforgeeks.org/operating-systems/what-is-nginx-web-server-and-how-to-install-it/), hoặc các dịch vụ cloud load balancing). Đối với bạn phỏng vấn system design ở mức cơ bản, hiểu khái niệm là đủ: load balancer giúp mở rộng hệ thống theo chiều ngang và tránh điểm quá tải, điểm thất bại duy nhất. 

5. <a id="cache-bộ-nhớ-đệm"></a>**Cache (Bộ nhớ đệm)**



6. <a id="cdn-mạng-phân-phối-nội-dung"></a>**CDN (Mạng phân phối nội dung)**

7. <a id="message-queue-hàng-đợi-thông-điệp"></a>**Message Queue (Hàng đợi thông điệp)**

## <a id="tính-nhất-quán-và-khả-dụng-trong-system-design-1"></a>Tính nhất quán và khả dụng trong System Design

### <a id="giới-thiệu-2"></a>Giới thiệu

### Tính nhất quán và tính khả dụng

### Các mô hình nhất quán (Consistency Models)

1. <a id="nhất-quán-mạnh-strong-consistency"></a>**Nhất quán mạnh (Strong Consistency)**

2. <a id="nhất-quán-cuối-cùng-eventual-consistency"></a>**Nhất quán cuối cùng (Eventual Consistency)**

3. <a id="nhất-quán-yếu-weak-consistency"></a>**Nhất quán yếu (Weak Consistency)**

### Cơ chế đảm bảo tính nhất quán và khả dụng

1. <a id="nhân-bản-dữ-liệu-replication"></a>**Nhân bản dữ liệu (Replication)**

2. <a id="chuyển-đổi-dự-phòng-failover"></a>**Chuyển đổi dự phòng (Failover)**

3. <a id="quorum-đa-số-phiếu"></a>**Quorum (Đa số phiếu)**

### Định lý CAP và các đánh đổi kinh điển

### Cách trình bày về nhất quán và khả dụng khi phỏng vấn

## <a id="lựa-chọn-cơ-sở-dữ-liệu-sharding-và-tối-ưu-hóa-lưu-trữ-1"></a>Lựa chọn Cơ sở dữ liệu, Sharding và Tối ưu hóa lưu trữ

### SQL vs NoSQL: Lựa chọn cơ sở dữ liệu phù hợp

### Phân mảnh dữ liệu (Sharding/Partitioning)

### Chỉ mục (Index): Tối ưu hóa truy vấn

### Cân nhắc khi chọn kiểu lưu trữ cho hệ thống lớn

### Dữ liệu phân tán và tính nhất quán (Consistency)

### Gợi ý trình bày phần thiết kế Database khi phỏng vấn

## <a id="mở-rộng-hệ-thống-và-tối-ưu-hiệu-suất-1"></a>Mở rộng hệ thống và tối ưu hiệu suất

### Cân bằng tải (Load Balancer)

1. <a id="load-balancer-là-gì-và-vai-trò-của-nó"></a>**Load Balancer là gì và vai trò của nó?**

2. <a id="cách-hoạt-động-và-các-thuật-toán-phân-phối"></a>**Cách hoạt động và các thuật toán phân phối**

3. <a id="load-balancer-tầng-4-vs-tầng-7-layer-4-vs-layer-7"></a>**Load Balancer tầng 4 vs tầng 7 (Layer 4 vs Layer 7)**

4. <a id="khi-nào-cần-sử-dụng-load-balancer"></a>**Khi nào cần sử dụng Load Balancer?**

### Bộ nhớ đệm (Caching)

1. <a id="cache-là-gì-và-có-lợi-ích-gì"></a>**Cache là gì và có lợi ích gì?**

2. <a id="các-vị-trí-có-thể-đặt-cache"></a>**Các vị trí có thể đặt cache**

3. <a id="các-chiến-lược-cập-nhật-cache-cache-update-strategies"></a>**Các chiến lược cập nhật cache (Cache update strategies)**

4. <a id="gợi-ý-khi-phỏng-vấn-về-caching"></a>**Gợi ý khi phỏng vấn về Caching**

### Hàng đợi thông điệp (Message Queue)

1. <a id="khái-niệm-hàng-đợi-thông-điệp--xử-lý-bất-đồng-bộ"></a>**Khái niệm hàng đợi thông điệp & xử lý bất đồng bộ**

2. <a id="khi-nào-nên-dùng-hàng-đợi"></a>**Khi nào nên dùng hàng đợi**

3. <a id="một-số-hệ-thống-message-queue-phổ-biến"></a>**Một số hệ thống Message Queue phổ biến**

4. <a id="mô-hình-producer---queue---consumer"></a>**Mô hình Producer - Queue - Consumer**

5. <a id="cơ-chế-backpressure-phản-áp-lực-trong-hàng-đợi"></a>**Cơ chế backpressure (phản áp lực) trong hàng đợi**

6. <a id="ví-dụ-minh-họa-sử-dụng-message-queue"></a>**Ví dụ minh họa sử dụng Message Queue**

7. <a id="gợi-ý-khi-phỏng-vấn-về-message-queue"></a>**Gợi ý khi phỏng vấn về Message Queue**

## <a id="idempotency-và-cơ-chế-khóa-trong-hệ-thống-phân-tán-1"></a>Idempotency và cơ chế khóa trong hệ thống phân tán

### Pessimistic Locking (Khóa bi quan)

### Optimistic Locking (Khóa lạc quan)

### Khi nào dùng & lựa chọn

## <a id="các-phương-thức-giao-tiếp-và-thiết-kế-api-1"></a>Các phương thức giao tiếp và thiết kế API

### HTTP: Giao thức request/response phổ biến nhất

### TCP: Giao thức kết nối tin cậy

### UDP: Giao thức "gửi là quên"

### RPC: Gọi thủ tục từ xa như gọi hàm nội bộ

### REST: Phong cách thiết kế API rõ ràng, thống nhất

## <a id="đảm-bảo-khả-năng-chịu-lỗi-và-phục-hồi-hệ-thống-1"></a>Đảm bảo khả năng chịu lỗi và phục hồi hệ thống

### Khả năng chịu lỗi là gì và tại sao quan trọng?

### Các sự cố phổ biến trong hệ thống phân tán

### Chiến lược tăng độ bền vững (resilience) cho hệ thống

### Phân biệt High Availability và Fault Tolerance

### Đánh giá độ tin cậy: Uptime, SLA, MTTR, MTBF

### Gợi ý trình bày chiến lược phục hồi sự cố trong phỏng vấn

## <a id="bảo-mật-và-phân-quyền-1"></a>Bảo mật và phân quyền

### Vai trò của bảo mật trong hệ thống phân tán hiện đại

1. <a id="các-khái-niệm-cơ-bản"></a>**Các khái niệm cơ bản**
    - <a id="xác-thực-authentication"></a>**Xác thực (Authentication)**
    - <a id="phân-quyền-authorization"></a>**Phân quyền (Authorization)**
    - <a id="bảo-mật-api"></a>**Bảo mật API**
    - <a id="bảo-vệ-dữ-liệu"></a>**Bảo vệ dữ liệu**
    - <a id="https-tls-và-bảo-mật-cookie"></a>**HTTPS, TLS và bảo mật cookie**

2. <a id="ví-dụ-minh-họa"></a>**Ví dụ minh họa**

3. <a id="lời-khuyên-khi-phỏng-vấn-về-bảo-mật-trong-system-design"></a>**Lời khuyên khi phỏng vấn về bảo mật trong System Design**

### Monitoring và Observability

1. <a id="monitoring-là-gì"></a>**Monitoring là gì?**

2. <a id="observability-là-gì"></a>**Observability là gì?**

3. <a id="trình-bày-về-monitoring-và-observability-trong-buổi-phỏng-vấn"></a>**Trình bày về Monitoring và Observability trong buổi phỏng vấn**

## <a id="lời-kết-1"></a>Lời kết