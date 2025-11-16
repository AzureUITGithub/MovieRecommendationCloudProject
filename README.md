# Đồ án: Hệ thống Đề nghị Phim trên Nền tảng Điện toán Đám mây AWS

Đây là đồ án xây dựng một hệ thống đề nghị phim (Movie Recommendation System) có khả năng mở rộng, sử dụng kiến trúc "tính toán trước" (pre-computation) trên nền tảng AWS.

Mục tiêu là phục vụ các đề nghị "item-to-item" (phim tương tự) với độ trễ cực thấp (low latency) bằng cách tách biệt hoàn toàn giữa việc xử lý dữ liệu nặng (Batch Processing) và việc phục vụ API (Real-time Serving).

## 1. Kiến trúc Hệ thống

Dự án này sử dụng kiến trúc tách biệt giữa xử lý (batch) và phục vụ (real-time):



### Luồng 1: Xử lý (ETL & Tính toán trước) - (Batch Processing)

1.  [cite_start]Dữ liệu thô (MovieLens 25M) được tải lên **Amazon S3**. [cite: 7]
2.  Một **Amazon SageMaker Notebook** (trên instance RAM lớn, vd: `ml.m5.4xlarge`) đọc dữ liệu từ S3.
3.  SageMaker huấn luyện mô hình **SVD (Matrix Factorization)** trên toàn bộ 25M ratings để lấy "đặc trưng" (item factors) của 59,047 bộ phim.
4.  SageMaker thực hiện tính toán "Just-In-Time" (JIT) để tìm 10 phim tương tự nhất cho mỗi phim (bằng `cosine_similarity`) và lưu kết quả (`movieId` -> `[top_10_recs]`) vào **Amazon DynamoDB**.
5.  SageMaker cũng lưu file tra cứu tên phim (`movies_df.pkl`) trở lại S3.

### Luồng 2: Phục vụ (Serving) - (Real-time)

1.  Người dùng truy cập **Giao diện Web Streamlit** (chạy local).
2.  Streamlit gọi đến một API endpoint (ví dụ: `/recommend?title=Matrix`).
3.  Endpoint này được host trên **Amazon EKS** (Kubernetes), chạy một API (Flask/Gunicorn) từ image lưu trên **Amazon ECR**.
4.  API **không** chạy model. Nó chỉ thực hiện một truy vấn Key-Value cực nhanh vào **DynamoDB** để lấy 10 ID phim đã được tính toán sẵn.
5.  [cite_start]API đọc file `movies_df.pkl` từ S3 để tra cứu tên phim, ID, thể loại [cite: 41] và trả về kết quả JSON cho Streamlit.

## 2. Công nghệ sử dụng

| Lĩnh vực | Công nghệ | Mục đích |
| :--- | :--- | :--- |
| **Cloud (AWS)** | **Amazon S3** | Lưu trữ dữ liệu thô (Dataset 25M) và file tra cứu (`.pkl`). |
| | **Amazon SageMaker**| Môi trường (Jupyter) để chạy ETL và huấn luyện SVD (Plan A). |
| | **Amazon DynamoDB** | Cơ sở dữ liệu NoSQL, lưu kết quả đã tính toán trước (Key-Value). |
| | **Amazon ECR** | Lưu trữ Docker image cho API. |
| | **Amazon EKS** | Dịch vụ Kubernetes, điều phối và phục vụ API (Mục 4.2). |
| **Backend** | **Python** | Ngôn ngữ lập trình chính. |
| | **Flask / Gunicorn** | [cite_start]Xây dựng và phục vụ API đơn giản. [cite: 3] |
| | **Docker** | Đóng gói API thành image. |
| **Frontend** | **Streamlit** | Xây dựng giao diện web demo (Mục 4.4). |
| **Data Science**| **Pandas** | Xử lý, chuẩn bị dữ liệu. |
| | **Surprise** | Thư viện Recommender System (huấn luyện mô hình SVD). |
| | **Scikit-learn** | Tính toán `cosine_similarity` (tối ưu RAM). |
| | **Boto3** | AWS SDK cho Python (giao tiếp với S3, DynamoDB). |

## 3. Hướng dẫn cài đặt và triển khai

Đây là 7 giai đoạn để dựng lại toàn bộ hệ thống từ đầu.

### Giai đoạn 0: Chuẩn bị

1.  [cite_start]Tải bộ **MovieLens 25M Dataset**. [cite: 7]
2.  Clone repository này:
    ```bash
    git clone [URL-REPO-CUA-BAN]
    cd [TEN-REPO]
    ```

### Giai đoạn 1: Amazon S3

1.  Tạo một S3 Bucket (ví dụ: `movielens-25m-project-user-12345`).
2.  Tải 2 file `ratings.csv` và `movies.csv` lên bucket.

### Giai đoạn 2: Amazon DynamoDB

1.  Tạo một bảng DynamoDB tên là **MovieRecommendations**.
2.  **Partition key (Khóa phân vùng):** `movieId` (Kiểu dữ liệu: **String**).

### Giai đoạn 3: Amazon SageMaker (Huấn luyện & ETL)

1.  Tạo một SageMaker Notebook Instance (ví dụ: `ml.m5.4xlarge` hoặc instance 32GB RAM+).
2.  **IAM Role:** Tạo một IAM Role mới cho instance, và **Attach Policy** cho phép:
    * Quyền đọc/ghi S3 (hoặc chỉ bucket của bạn).
    * Quyền đọc/ghi DynamoDB (`AmazonDynamoDBFullAccess`).
3.  Mở JupyterLab, tải file Notebook `.ipynb` (Plan A - SVD JIT) từ repo này lên.
4.  **Quan trọng:** Mở **Cell 1**, cập nhật 2 biến `S3_BUCKET_NAME` và `AWS_REGION` cho đúng với của bạn.
5.  Chạy tuần tự các Cell (1 đến 5) để:
    * **Cell 1:** Cài đặt thư viện (`surprise`, `scikit-learn`...).
    * **Cell 2:** Tải data từ S3.
    * **Cell 3:** So sánh model (SVD vs KNN) trên mẫu 1M (Giải quyết Rubric 4.3).
    * **Cell 4:** Huấn luyện SVD trên **toàn bộ 25M data** và lưu `movies_df.pkl` (Giải quyết Rubric 4.1).
    * **Cell 5:** Tính toán JIT và lưu kết quả vào DynamoDB (có progress bar).
6.  **DỪNG (STOP)** instance SageMaker ngay sau khi Cell 5 chạy xong để tiết kiệm chi phí.
7.  **Kiểm tra:** Vào S3 (thư mục `model/`) và DynamoDB (Bảng `MovieRecommendations`) để xác nhận dữ liệu đã được lưu.

### Giai đoạn 4: Docker & ECR (Build API)

*(Thực hiện trên máy cá nhân)*

1.  Cài đặt Docker và AWS CLI (đã cấu hình `aws configure`).
2.  Trong file `app.py`, kiểm tra lại 3 biến `S3_BUCKET_NAME`, `DYNAMO_TABLE_NAME`, và `AWS_REGION` cho chính xác.
3.  Tạo một ECR repository (ví dụ: `movie-rec-api`).
4.  Vào ECR, nhấn vào repo và chọn **"View push commands"**.
5.  Chạy 4 lệnh này trong terminal (tại thư mục dự án) để build, tag, và push Docker image.

### Giai đoạn 5: Amazon EKS (Deploy API)

1.  Tạo một EKS Cluster (ví dụ: `movie-cluster`) và một Node Group (ví dụ: `t3.medium`, 2 nodes).
2.  **IAM Role:** Tìm IAM Role của Node Group (ví dụ: `EKS-Node-Role`), và **Attach Policy** cho phép:
    * `AmazonS3ReadOnlyAccess` (để API đọc file `.pkl`).
    * `AmazonDynamoDBReadOnlyAccess` (để API đọc bảng `MovieRecommendations`).
3.  Cấu hình `kubectl` trên máy cá nhân:
    ```bash
    aws eks update-kubeconfig --region [YOUR_REGION] --name movie-cluster
    ```
4.  Mở file `deployment.yaml`, tìm dòng `image:` và thay thế bằng **Image URI** từ ECR của bạn (Giai đoạn 4).
5.  Triển khai API:
    ```bash
    kubectl apply -f deployment.yaml
    kubectl apply -f service.yaml
    ```
6.  Chờ vài phút và lấy địa chỉ API (Load Balancer):
    ```bash
    kubectl get service movie-api-service
    ```
    (Sao chép giá trị ở cột `EXTERNAL-IP`).

### Giai đoạn 6: Streamlit (Chạy Giao diện)

1.  Cài đặt thư viện:
    ```bash
    pip install streamlit requests
    ```
2.  Mở file `webapp.py`, tìm biến `API_URL` và dán địa chỉ `EXTERNAL-IP` (kèm `http://`) của bạn vào.
3.  Chạy ứng dụng:
    ```bash
    streamlit run webapp.py
    ```
4.  Mở trình duyệt (thường là `http://localhost:8501`) và sử dụng!

### Giai đoạn 7: Dọn dẹp (RẤT QUAN TRỌNG)

Để tránh phát sinh chi phí, hãy xóa tài nguyên theo thứ tự:

1.  **EKS:**
    * `kubectl delete -f service.yaml`
    * `kubectl delete -f deployment.yaml`
    * Vào AWS Console, xóa **Node Group** trước, sau đó xóa **Cluster**.
2.  **ECR:** Xóa repository `movie-rec-api`.
3.  **SageMaker:** Xóa Notebook Instance (nếu chưa xóa).
4.  **DynamoDB:** Xóa bảng `MovieRecommendations`.
5.  **S3:** Làm rỗng (Empty) bucket, sau đó xóa (Delete) bucket.
6.  **IAM:** Xóa các IAM Roles đã tạo (cho SageMaker, EKS Cluster, EKS Node).

## 4. Ánh xạ Rubric (Theo yêu cầu Đồ án)

Dự án này được thiết kế để đáp ứng các mục tiêu trong Rubric:

* **Mục 1 (Giới thiệu):** Sử dụng S3 (Lưu trữ), SageMaker (Xử lý), EKS (Trực quan/API), Dataset 25M (GB), và các dịch vụ PaaS (S3, EKS, DynamoDB).
* **Mục 2 (Lý thuyết):** Lưu trữ (S3, DynamoDB), Thuật toán (SVD, Cosine Similarity), Dịch vụ (SageMaker, EKS).
* **Mục 3 (Mô hình dữ liệu):** Tối ưu tốc độ đọc (DynamoDB) và ghi (batch_writer). Luồng ETL (S3->SageMaker->DynamoDB). Tối ưu độ trễ (cùng Region). Tối ưu kiến trúc (Pre-computation + SVD JIT).
* **Mục 4 (Hiện thực):** Huấn luyện (SageMaker Cell 4), API (EKS Giai đoạn 5), Trực quan tham số (SageMaker Cell 3), Web (Streamlit Giai đoạn 6).
* **Mục 5 (Báo cáo):** Quản lý dự án (Repo GitHub này).
