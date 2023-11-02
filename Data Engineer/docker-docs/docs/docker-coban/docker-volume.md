# 3. Quản lý dữ liệu trong Docker

____

# Mục lục


- [3.1 Giới thiệu cách thức quản lý](#about)
- [3.2 Trường hợp nào thì sử dụng volumes](#volumes)
- [3.3 Trường hợp nào thì sử dụng bind mounts](#bind-mounts)
- [3.4 Trường hợp nào thì sử dụng tmpfs mount](#tmpfs)
- [3.5 Cách sử dụng volumes](#use-volumes)
- [3.6 Cách sử dụng bind mounts](#use-bind)
- [3.7 Cách sử dụng tmpfs mounts](#use-tmpfs)
- [Các nội dung khác](#content-others)

____

# <a name="content">Nội dung</a>

- ### <a name="about">3.1 Giới thiệu cách thức quản lý</a>

    Trong Docker, việc quản lý dữ liệu bên trong các container có thể gặp một số khó khăn. Cụ thể, khi container ngừng hoạt động, dữ liệu bên trong nó có thể mất đi và không dễ dàng trích xuất ra ngoài cho mục đích khác. Dữ liệu của container thường liên quan chặt chẽ với máy chủ Docker, khó di chuyển đến các máy chủ khác. Việc ghi dữ liệu vào container cũng ảnh hưởng đến hiệu suất.

    Docker giải quyết vấn đề này bằng cách cung cấp ba cách để quản lý dữ liệu:

    - Volumes: Đây là cách phổ biến nhất và được Docker quản lý. Volumes cho phép bạn lưu trữ dữ liệu một cách an toàn và có thể sử dụng cho nhiều mục đích.

    - Bind Mounts: Bind mounts cho phép bạn chia sẻ dữ liệu từ máy chủ Docker với container. Điều này thường phù hợp khi bạn muốn chia sẻ các tệp cấu hình hoặc có cấu trúc thư mục cố định trên máy chủ Docker phù hợp với container.

    - Tmpfs Mounts: Tmpfs mounts cho phép bạn lưu trữ tạm thời dữ liệu trong bộ nhớ RAM của máy chủ Docker, phù hợp cho các trường hợp bạn không muốn dữ liệu tồn tại trên máy chủ Docker hoặc trong container vì lý do bảo mật hoặc hiệu suất.

    `volumes` thường luôn là cách được lựa chọn sử dụng nhiều nhất đối với mọi trường hợp

    - Không có vấn đề nào xảy ra khi ta lựa chọn cách chia sẻ dữ liệu để sử dụng, vì các dữ liệu đều giống nhau trong container. Chúng được quy định giống như một thư mục hoặc một file trong filesystem của containers. Sự khác biệt giữa `volumes`, `bind mounts` và `tmpfs mounts` chỉ đơn giản là khác nhau về vị trí lưu trữ dữ liệu trên Docker host.

        > ![types-of-mounts.png](../../images/types-of-mounts.png)

    - Hình ảnh trên mô tả vị trí lưu trữ dữ liệu của container trên Docker host. Theo đó, ta có thể thấy được:

        + `volumes` được lưu trữ như một phần của filesystem trên Docker host và được quản lý bởi Docker (xuất hiện trong /var/lib/docker/volumes trên Linux).  Đây được xem là cách tốt nhất để duy trì dữ liệu trong Docker

        + `bind mounts` cho phép lưu trữ bất cứ đâu trong host system. 

        + `tmpfs mounts` cho phép lưu trữ tạm thời dữ liệu vào bộ nhớ của Docker host, không bao giờ ghi vào filesystem của Docker host.

- ### <a name="volumes">3.2 Trường hợp nào thì sử dụng volumes</a>

    - `volumes` được tạo và quản lý bởi Docker. Ta có thể tạo `volumes` với câu lệnh `docker volume create` hoặc tạo `volumes` trong khi tạo containers, ...

    - Khi tạo ra `volumes`, nó sẽ được lưu trữ trong một thư mục trên Docker host. Khi ta thực hiện mount `volumes` vào container thì thư mục này sẽ được mount vào container. Điều này tương tự như cách `bind mounts` hoạt động ngoại trừ việc được Docker quản lý.

    - `volumes` có thể được mount vào nhiểu containers cùng một lúc. Khi không có containers nào sử dụng `volumes` thì `volumes` vẫn ở trạng thái cho phép mount vào containers và không bị xóa một cách tự động.

    - `volumes` hỗ trợ volume drivers, do đó ta có thể sử dụng để lưu trữ dữ liệu từ remote hosts hoặc cloud providers.

    - Đây là cách phổ biến được lựa chọn để duy trì dữ liệu trong services và containers. Một số trường hợp sử dụng `volumes` có thể bao gồm:

        + Lưu trữ dữ liệu của ứng dụng hoặc cơ sở dữ liệu: Khi bạn cần lưu trữ dữ liệu bền vững cho ứng dụng của bạn, chẳng hạn như dữ liệu cơ sở dữ liệu, tệp tin cấu hình, hình ảnh, video, hoặc bất kỳ dữ liệu nào mà ứng dụng của bạn sử dụng, bạn nên sử dụng Docker `volumes` để đảm bảo tính bền vững.

        + Chia sẻ dữ liệu giữa các container: Khi bạn muốn chia sẻ dữ liệu giữa nhiều container, Docker Volumes là giải pháp lý tưởng. Ví dụ, bạn có thể sử dụng Docker Volumes để lưu trữ dữ liệu cơ sở dữ liệu chung và sau đó liên kết nó với nhiều container ứng dụng để chúng có thể truy cập và cập nhật cùng một dữ liệu.

        + Backup và Restore dữ liệu dễ dàng: Docker Volumes giúp bạn sao lưu và phục hồi dữ liệu dễ dàng. Bạn có thể sao lưu Docker Volume ra khỏi container và sau đó khôi phục nó khi cần thiết.

        + Tích hợp với các dịch vụ lưu trữ dữ liệu bên ngoài: Nếu bạn muốn tích hợp dữ liệu của container với các dịch vụ lưu trữ dữ liệu bên ngoài như Amazon S3, NFS, hoặc các dịch vụ lưu trữ dữ liệu đám mây khác, Docker Volumes cho phép bạn kết nối dễ dàng với những dịch vụ này.

        + Quản lý dữ liệu của ứng dụng trong môi trường phát triển và sản xuất: Docker Volumes giúp bạn quản lý dữ liệu ứng dụng trong cả môi trường phát triển và sản xuất. Bạn có thể sử dụng cùng một cấu hình Volume khi chạy ứng dụng trên máy tính cá nhân và trên máy chủ sản xuất.

        + Dễ dàng thay thế và nâng cấp ứng dụng: Khi bạn cần thay thế hoặc nâng cấp ứng dụng, bạn có thể chỉ cần cắt container cũ và triển khai container mới, giữ nguyên Docker Volume để đảm bảo dữ liệu không bị mất.




- ### <a name="bind-mounts">3.3 Trường hợp nào thì sử dụng bind mounts</a>

    - `bind mounts` có chức năng hạn chế so với `volumes`. Khi ta sử dụng `bind mounts` thì một file hoặc một thư mục trên Docker host sẽ được mount tới containers với đường dẫn đầy đủ.

    - Đây là các trường hợp phổ biến lựa chọn `bind mounts` đối với containers:

        + Chia sẻ các file cấu hình từ Docker host tới containers. 

        + Chia sẻ khi các file hoặc cấu trúc thư mục trên Docker host có cấu trúc cố định phù hợp với yêu cầu của containers.

        + Kiểm soát được các thay đổi của containers đối với filesystem trên Docker host. Do khi sử dụng `bind mounts`, containers có thể trực tiếp thay đổi filesystem trên Docker host.

- ### <a name="tmpfs">3.4 Trường hợp nào thì sử dụng tmpfs mount</a>

    - `tmpfs mounts` được sử dụng trong các trường hợp ta không muốn dữ liệu tồn tại trên Docker host hay containers vì lý do bảo mật hoặc đảm bảo hiệu suất của containers khi ghi một lượng lớn dữ liệu một cách không liên tục.

____

- `bind mounts` và `volumes` đều có thể được mount vào container khi sử dụng flag `-v` hoặc `--volume` nhưng cú pháp sử dụng có một chút khác nhau. Đối với `tmpfs mounts` có thể sử dụng flag `--tmpfs`. Tuy nhiên, từ bản Docker 17.06 trở đi, chúng ta được khuyến cáo dùng flag `--mount` cho cả 3 cách, để cú pháp câu lệnh minh bạch hơn.

- Sự khác nhau giữa `--volume, -v` và `--mount` đơn giản chỉ là về cách khai báo các giá trị:

    + `--volume, -v` các giá trị cách nhau bới `:`. theo dạng source:target. Ví dụ: `-v myvol2:/app`
    + `--mount` khai báo giá trị theo dạng `key=values`. Ví dụ: `--mount source=myvol2,target=/app`. Trong đó `source` có thể thay thế bằng `src`, `target` có thể thay thế bằng `destination` hoặc `dst`.

- Khi sử dụng volumes cho services thì chỉ `--mount` mới có thể sử dụng.

- ### <a name="use-volumes">3.5 Cách sử dụng volumes</a>

    - Volumes là cơ chế ưa thích cho việc duy trì dữ liệu được tạo ra bởi Docker containers và được quản lý bởi Docker. Trong khi `bind mounts` phụ thuộc vào cấu trúc thư mục của Docker host. Do đó, volumes có một số tính năng khác biệt so với `bind mounts` như sau:

        + Volumes dễ dàng backups, migrate hơn so với `bind mounts`.
        + Có thể quản lý volumes sử dụng Docker CLI hoặc Docker API.
        + Volumes làm việc trên cả Linux containers và Winodws containers.
        + Volumes an toàn hơn khi sử dụng để chi sẻ giữa nhiều containers.

    - Để tạo một volume, ta sử dụng câu lệnh:

            docker volume create my-vol

    - Khi khởi chạy containers với volume chưa có (tồn tại) thì Docker sẽ tự động tạo ra volume với tên được khai báo hoặc với một tên ngẫu nhiên và đảm bảo tên ngẫu nhiên này là duy nhất. Ví dụ:

            docker run -d \
              -it \
              --name devtest \
              --mount type=volume,source=myvol2,target=/app \
              nginx:latest

        câu lệnh trên sẽ thực hiện mount volume `myvol2` tới thư mục `/app` trong container `devtest`.

    - Nếu muốn mount volume với chế độ `readonly`, ta có thể thêm vào trong `--mount`. Với ví dụ trên có thể là làm như sau:

            --mount source=myvol2,target=/app,readonly

- ### <a name="use-bind">3.6 Cách sử dụng bind mounts</a>

    - Sử dụng tương tự như volume, ta chỉ cần thay đổi giá trị của `type` trong `--mount`. Theo đó ta có câu lệnh ví dụ như sau:

            docker run -d \
              -it \
              --name devtest \
              --mount type=bind,source=myvol2,target=/app \
              nginx:latest

- ### <a name="use-tmpfs">3.7 Cách sử dụng tmpfs mounts</a>

    - Khi sử dụng `tmpfs mounts` thì ta không thể chia sẻ dữ liệu giữa containers.
    - `tmpfs mounts` chỉ làm việc đối với Linux containers.

    - Tương tự như khi sử dụng volumes, ta chỉ cần thay đổi giá trị `type` trong `--mount`:

            docker run -d \
              -it \
              --name devtest \
              --mount type=tmpfs,source=myvol2,target=/app \
              nginx:latest
              
____

# <a name="content-others">Các nội dung khác</a>
