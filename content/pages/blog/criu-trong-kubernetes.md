---
type: PostLayout
title: >-
  [Devops] CRIU trong Kubernetes: Hướng dẫn từ A-Z để "Save Game" cho Pods của
  bạn
date: '2025-11-09'
author: content/data/team/Le-Huan.json
excerpt: >-
  Đây là một bài blog "deep dive" hơn, hướng dẫn anh em các bước kỹ thuật để
  kích hoạt và sử dụng CRIU với Pods.
featuredImage:
  type: ImageBlock
  url: /images/ciu-k8s.png
  altText: Post thumbnail image
  caption: Caption of the image
  elementId: ''
media:
  type: ImageBlock
  url: /images/ciu-k8s.png
  altText: Post image
  caption: Caption of the image
  elementId: ''
bottomSections:
  - type: LabelsSection
    title: Keywords
    subtitle: ''
    items:
      - type: Label
        label: CRIU
        url: ''
      - type: Label
        label: Kubernetes
        url: ''
      - type: Label
        label: DevOps
        url: ''
      - type: Label
        label: Linux
        url: ''
      - type: Label
        label: Container Runtime
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
addTitleSuffix: true
metaTags: []
colors: colors-a
backgroundImage:
  type: BackgroundImage
  url: /images/bg2.jpg
  backgroundSize: cover
  backgroundPosition: center
  backgroundRepeat: no-repeat
  opacity: 100
---
**Tiêu đề: CRIU trong Kubernetes: Hướng dẫn kỹ thuật để "Save" và "Load" Pods của bạn**

Chào anh em,

Ở bài trước, chúng ta đã tìm hiểu CRIU là gì và khả năng "đóng băng" ứng dụng thần thánh của nó. Giờ là lúc "vào việc" với câu hỏi quan trọng hơn: Làm thế nào để mang "phép thuật" này vào Kubernetes, để chúng ta có thể "live migrate" (di trú trực tiếp) một con Pod?

Bài viết này sẽ tập trung vào các yêu cầu kỹ thuật, các "công tắc" anh em cần bật, và các bước thực hiện.

**Lưu ý:** Đây là một tính năng nâng cao, không phải "plug-and-play" (cắm là chạy). Anh em cần chuẩn bị tinh thần "vọc vạch" một chút.

***

### 1. Điều kiện cần: Các "Công tắc" phải được Bật

Để Kubernetes "hiểu" và "cho phép" CRIU hoạt động, chúng ta cần sự đồng ý từ hai "ông lớn": **Kubelet** và **Container Runtime**.

#### A. Kubelet Feature Support

`Kubelet` là "tay chân" của Kubernetes trên mỗi node. Mặc định, nó không cho phép các tính năng thử nghiệm (experimental). Anh em cần "bật" một **Feature Gate** (cổng tính năng) có tên là `ContainerCheckpoint`.

*   **Cách làm:** Anh em cần chỉnh sửa file cấu hình của `kubelet` (thường nằm ở `/var/lib/kubelet/config.yaml`) trên **tất cả các node** mà anh em muốn dùng tính năng này.
*   **Thêm vào:**
    ```yaml
    featureGates:
      ContainerCheckpoint: true
    ```
*   Sau đó, khởi động lại `kubelet`: `systemctl restart kubelet`.

#### B. Container Runtime Support

Kubelet chỉ là "sếp" ra lệnh. Người "làm việc" (đóng băng container) chính là **Container Runtime**. Cả CRI-O và Containerd đều hỗ trợ, nhưng anh em cần đảm bảo phiên bản của mình "đủ tuổi":

*   **Với CRI-O:** CRI-O có hỗ trợ checkpoint khá tốt. Anh em nên sử dụng các phiên bản gần đây (ví dụ: v1.18 trở lên) để đảm bảo tính ổn định.
*   **Với Containerd:** Hỗ trợ checkpoint cũng đã có (từ v1.5+), nhưng anh em có thể cần phải kích hoạt một số cấu hình bổ sung trong file `config.toml` của Containerd.

**LƯU Ý CỰC KỲ QUAN TRỌNG:**

> **Checkpoint và Restore phải được thực hiện trên cùng một hệ điều hành VÀ cùng một loại (và phiên bản) container runtime.**
>
> Anh em không thể "save game" trên CRI-O rồi "load game" trên Containerd. Chúng nó "nói chuyện" khác ngôn ngữ.

***

### 2. Các bước thực hiện "Save Game" (Checkpoint)

Quy trình chuẩn để "đóng băng" một Pod đang chạy như sau:

1.  Anh em có một Pod đang chạy (ví dụ: `my-app-pod`).
2.  Để "đóng băng" nó, anh em (hoặc một công cụ tự động hóa) phải **gọi trực tiếp vào Checkpoint API của `kubelet`** trên cái node đang chạy con Pod đó.
    *   Đây là một lệnh API, kiểu như `curl` (với chứng chỉ) hoặc gọi gRPC tới endpoint của `kubelet`.
3.  Khi nhận được lệnh, `kubelet` sẽ "gõ đầu" Container Runtime.
4.  Container Runtime sẽ dùng CRIU để "đóng băng" container.

**Kết quả:**
Checkpoint sẽ được lưu dưới dạng các file ngay trên node, tại đường dẫn mặc định (hoặc do anh em cấu hình), ví dụ: `/var/lib/kubelet/checkpoints/`

**Vấn đề:** Giờ anh em có một đống file checkpoint nằm "chỏng chơ" trên một node. Làm sao để mang nó sang node khác mà "load game"?

Đây là lúc chúng ta sang bước 3.

***

### 3. Đóng gói Checkpoint thành Image

Đây là bước then chốt. Anh em phải "đóng gói" đống file checkpoint kia thành một **container image** thì mới "ship" đi đâu thì ship được.

Anh em sẽ cần một công cụ như `buildah` (hoặc `podman`) và thực hiện các bước:

1.  Tạo một image mới (thường là "from scratch").
2.  Copy toàn bộ file checkpoint (từ `/var/lib/kubelet/checkpoints/...`) vào trong image này.
3.  **Quan trọng nhất:** "Đánh dấu" image này bằng các **annotation** (chú thích) đặc biệt. Đây chính là thứ sẽ "bảo" container runtime "đây là file save game, đừng chạy như image bình thường".

**A. Nếu anh em dùng CRI-O:**
(Dùng `$IMAGE_NAME` là tên image checkpoint mới, và `$BASE_IMAGE` là tên image *gốc* mà pod đã chạy)

```
buildah config --annotation=io.kubernetes.cri-o.annotations.checkpoint.name="$IMAGE_NAME" "$newcontainer"
buildah config --annotation=io.kubernetes.cri-o.annotations.checkpoint.rootfsImageName="$BASE_IMAGE" "$newcontainer"
```

**B. Nếu anh em dùng Containerd:**

```
buildah config --annotation=org.criu.checkpoint.container.name="$IMAGE_NAME" "$newcontainer"
buildah config --annotation=org.criu.checkpoint.rootfsImageName="$BASE_IMAGE" "$newcontainer"
```

Sau khi "đánh dấu", anh em buildah push cái image checkpoint này lên registry.

### 4. Các bước thực hiện "Load Game" (Restore)

Đây là phần "vi diệu" nhất của cả quy trình.

Không có lệnh "restore" đặc biệt nào cả.

Anh em chỉ cần làm một việc duy nhất: Deploy một Pod mới, và trỏ trường image: vào cái image checkpoint mà anh em vừa đẩy lên registry.

Ví dụ:

```
apiVersion: v1
kind: Pod
metadata:
  name: my-restored-app
spec:
  containers:
  - name: my-app
    image: my-registry/my-app-checkpoint:v1 # Trỏ vào image checkpoint!
```

**Chuyện gì xảy ra?**

*   Kubernetes "tưởng" đây là deploy Pod bình thường, nó ra lệnh cho kubelet ở node mới.
*   Kubelet ra lệnh cho Container Runtime (CRI-O/Containerd) kéo cái image my-app-checkpoint:v1 về.
*   Container Runtime kéo image về. Nó đọc các annotation mà anh em đã thêm ở Bước 3.
*   Nó thấy "À, đây là image checkpoint!"
*   Nó sẽ tự động gọi CRIU để "restore" (phục hồi) từ các file checkpoint trong image, thay vì "start" (chạy mới) cái container.

Kết quả: Pod "tỉnh dậy" y hệt như lúc anh em "save game", với đầy đủ bộ nhớ và trạng thái.

### 5. Những lưu ý "Xương Máu"

1.  Cùng OS, Cùng Runtime: Vẫn là yêu cầu bắt buộc. Phải checkpoint và restore trên cùng một hệ điều hành, cùng một loại (và phiên bản) container runtime.

2.  CRIU chỉ lo Memory/CPU: Nó không di chuyển Persistent Volumes (PV). Nếu Pod của anh em có dùng PV, anh em phải tự lo vụ di chuyển/đồng bộ dữ liệu trên ổ cứng. Đây là một câu chuyện cực kỳ phức tạp khác.

3.  Network (Mạng): Các kết nối mạng sẽ được cố gắng khôi phục, nhưng nếu Pod bị "dịch chuyển" sang node/cluster khác có IP khác, các kết nối cũ có thể sẽ "đứt" và ứng dụng cần có khả năng tự kết nối lại.

**Kết luận**

Như vậy, "live migration" trong Kubernetes (bằng CRIU) là một quy trình 2 bước rõ ràng:

1.  Checkpoint (Save): Gọi API của kubelet -> Lấy file checkpoint.

2.  Restore (Load): Đóng gói file thành image có annotation -> Deploy Pod mới trỏ đến image đó.

Đây là một kỹ thuật cực kỳ mạnh mẽ, dù còn nhiều rào cản. Nó đòi hỏi anh em phải setup kỹ càng từ Kubelet, Container Runtime, cho đến cách anh em build image. Nhưng khi đã chạy được, "trái ngọt" là anh em có thể bảo trì, nâng cấp cluster mà không sợ làm phiền người dùng.

Hy vọng bài viết này giúp ích được cho anh em!
