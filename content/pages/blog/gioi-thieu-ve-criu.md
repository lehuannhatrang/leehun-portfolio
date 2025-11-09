---
type: PostLayout
title: >-
  [Devops] CRIU là gì? Khám phá công nghệ "Save Game" cho Server, tạm biệt nỗi
  lo Downtime!
date: '2025-11-09'
excerpt: ''
featuredImage:
  type: ImageBlock
  url: /images/criu.svg
  altText: Post thumbnail image
  caption: Caption of the image
  elementId: ''
media:
  type: ImageBlock
  url: /images/criu.svg
  altText: Post image
  caption: Caption of the image
  elementId: ''
addTitleSuffix: true
colors: colors-a
backgroundImage:
  type: BackgroundImage
  url: /images/bg2.jpg
  backgroundSize: cover
  backgroundPosition: center
  backgroundRepeat: no-repeat
  opacity: 100
author: content/data/team/Le-Huan.json
bottomSections:
  - type: LabelsSection
    title: Keywords
    subtitle: ''
    items:
      - type: Label
        label: CRIU
        url: ''
      - type: Label
        label: Devops
        url: ''
      - type: Label
        label: Linux
        url: ''
    colors: colors-f
    elementId: ''
    styles:
      self:
        height: auto
        width: wide
        padding:
          - pt-36
          - pb-36
          - pl-4
          - pr-4
        textAlign: left
---
Chào anh em, hôm nay chúng ta sẽ cùng "mổ xẻ" một công nghệ cực kỳ thú vị đang thay đổi cuộc chơi của giới hạ tầng: CRIU.

Anh em có bao giờ ở trong tình huống phải loay hoay với một con server đang chạy dịch vụ quan trọng không? Kiểu như máy chủ cần bảo trì, cần nâng cấp phần cứng, hay đơn giản là muốn dời nó sang một cụm (cluster) khác, nhưng cứ hễ nghĩ đến việc phải "tắt" ứng dụng và đối mặt với downtime là thấy đau đầu?

Những lúc như thế, chắc hẳn anh em đều ước có một cái nút "Save Game" thần thánh, bấm một cái để lưu lại toàn bộ trạng thái, rồi bình tĩnh mang sang máy khác "Load Game" là xong.

Tin vui cho anh em: Công nghệ đó có thật! Và nó chính là CRIU.

**Vậy CRIU là gì?**

Nói một cách dễ hiểu nhất cho anh em, CRIU (viết tắt của Checkpoint/Restore In Userspace) chính là "nút Save Game" cho các ứng dụng Linux.

Nó là một công cụ cho phép anh em "đóng băng" một ứng dụng (hoặc cả một nhóm tiến trình) đang chạy. Nó sẽ lưu lại *tất cả* trạng thái của ứng dụng—từ bộ nhớ RAM, thanh ghi CPU, các file đang mở, cho đến các kết nối mạng...—thành một bộ sưu tập các file trên đĩa.

Phần hay nhất là đây: Anh em có thể dùng chính các file này để "phục hồi" (restore) ứng dụng, và nó sẽ tiếp tục chạy *giống hệt* như tại thời điểm bị "đóng băng", mà không hề biết mình vừa có một "giấc ngủ sâu". Anh em thậm chí có thể phục hồi nó trên một máy chủ hoàn toàn khác.

**Nó hoạt động ra sao?**

Quá trình này có hai phần chính:

Checkpoint (Lưu trạng thái): Khi anh em ra lệnh, CRIU sẽ "tiếp cận" ứng dụng và thu thập toàn bộ thông tin trạng thái của nó (memory, CPU, I/O...) rồi ghi ra file. Sau khi hoàn tất, ứng dụng gốc sẽ được dừng lại.

Restore (Phục hồi): Anh em mang các file trạng thái này sang một máy chủ mới (hoặc dùng tại chỗ). Khi lệnh restore được thực thi, CRIU sẽ đọc các file này, tái tạo lại các tiến trình, nạp lại bộ nhớ, và kết nối lại các luồng mạng. Ứng dụng sẽ "tỉnh dậy" và tiếp tục công việc của mình như chưa hề có cuộc chia ly.

**Tại sao nó "lợi hại"? (Các ứng dụng thực tế)**

Đây không chỉ là một công nghệ "hay ho" về mặt lý thuyết. CRIU mở ra rất nhiều khả năng giá trị cho anh em SRE/DevOps:

*   Live Migration (Di trú trực tiếp): Đây là ứng dụng giá trị nhất. Anh em có thể di chuyển một ứng dụng đang chạy (kể cả các workload có trạng thái như database) từ server A sang B để bảo trì, cân bằng tải, hoặc nâng cấp mà không làm gián đoạn dịch vụ.

*   High Availability (Tăng tính sẵn sàng): Anh em có thể định kỳ "checkpoint" các dịch vụ quan,". Nếu dịch vụ gặp sự cố, thay vì mất thời gian khởi động lại từ đầu, anh em chỉ cần "restore" từ lần checkpoint gần nhất.

*   Fast Startup (Khởi động nhanh): Nhiều ứng dụng phức tạp (như Java) mất rất nhiều thời gian để "làm nóng" (warm-up). Với CRIU, anh em có thể chạy ứng dụng cho đến khi nó "nóng", checkpoint nó lại, và các lần khởi động sau chỉ cần restore, nhanh hơn đáng kể.

*   Debugging (Gỡ lỗi): Khi gặp một lỗi khó tái hiện, anh em có thể dùng CRIU để "chụp nhanh" (snapshot) toàn bộ trạng thái của ứng dụng ngay lúc xảy ra lỗi. Đội dev có thể "restore" trạng thái này bao nhiêu lần cũng được để phân tích và tìm ra nguyên nhân.

**CRIU, Container và Kubernetes**

Các công nghệ container runtime (như CRI-O) và "bộ não điều phối" Kubernetes (thông qua `kubelet`) đang dần tích hợp sâu hơn với CRIU.

Điều này mở đường cho một thứ mà trước đây cực kỳ nan giải: di trú trực tiếp các workload có trạng thái (stateful workloads) giữa các cluster. Hãy tưởng tượng anh em có thể "bê" một con database đang chạy trong pod từ cụm on-premise của mình lên mây (hoặc ngược lại) mà không mất một giao dịch nào. Đây cũng là một phần trong dự án mà tôi đang nghiên cứu (chia sẻ một chút).

**Chốt lại**

Tóm lại, CRIU là một công cụ đắc lực, cung cấp sự linh hoạt và độ tin cậy mà các hệ thống phân tán hiện đại yêu cầu. Nó biến việc di dời các ứng dụng phức tạp từ một quy trình rủi ro, gây gián đoạn thành một tác vụ có kiểm soát và an toàn.

Nếu anh em đang làm về hạ tầng, DevOps, hoặc các hệ thống cần độ sẵn sàng cao, CRIU chắc chắn là một công nghệ rất đáng để tìm hiểu.
