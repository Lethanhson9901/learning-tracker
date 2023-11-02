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

        + **Lưu trữ dữ liệu của ứng dụng hoặc cơ sở dữ liệu**: Khi bạn cần lưu trữ dữ liệu bền vững cho ứng dụng của bạn, chẳng hạn như dữ liệu cơ sở dữ liệu, tệp tin cấu hình, hình ảnh, video, hoặc bất kỳ dữ liệu nào mà ứng dụng của bạn sử dụng, bạn nên sử dụng Docker Volumes để đảm bảo tính bền vững.

        + **Chia sẻ dữ liệu giữa các container**: Khi bạn muốn chia sẻ dữ liệu giữa nhiều container, Docker Volumes là giải pháp lý tưởng. Ví dụ, bạn có thể sử dụng Docker Volumes để lưu trữ dữ liệu cơ sở dữ liệu chung và sau đó liên kết nó với nhiều container ứng dụng để chúng có thể truy cập và cập nhật cùng một dữ liệu.

        + **Backup và Restore dữ liệu dễ dàng**: Docker Volumes giúp bạn sao lưu và phục hồi dữ liệu dễ dàng. Bạn có thể sao lưu Docker Volume ra khỏi container và sau đó khôi phục nó khi cần thiết.

        + **Tích hợp với các dịch vụ lưu trữ dữ liệu bên ngoài**: Nếu bạn muốn tích hợp dữ liệu của container với các dịch vụ lưu trữ dữ liệu bên ngoài như Amazon S3, NFS, hoặc các dịch vụ lưu trữ dữ liệu đám mây khác, Docker Volumes cho phép bạn kết nối dễ dàng với những dịch vụ này.

        + **Quản lý dữ liệu của ứng dụng trong môi trường phát triển và sản xuất**: Docker Volumes giúp bạn quản lý dữ liệu ứng dụng trong cả môi trường phát triển và sản xuất. Bạn có thể sử dụng cùng một cấu hình Volume khi chạy ứng dụng trên máy tính cá nhân và trên máy chủ sản xuất.

        + **Dễ dàng thay thế và nâng cấp ứng dụng**: Khi bạn cần thay thế hoặc nâng cấp ứng dụng, bạn có thể chỉ cần cắt container cũ và triển khai container mới, giữ nguyên Docker Volume để đảm bảo dữ liệu không bị mất.




- ### <a name="bind-mounts">3.3 Trường hợp nào thì sử dụng bind mounts</a>
    Nhớ rằng, bind mounts thường dễ sử dụng và hiệu quả trong quá trình phát triển và thử nghiệm, nhưng chúng không cung cấp tính bền vững tương tự như Docker Volumes. Điều này có nghĩa rằng dữ liệu trong bind mounts có thể bị mất khi container bị xóa hoặc khi bạn di chuyển sang một máy chủ khác.

    - `bind mounts` có chức năng hạn chế so với `volumes`. Khi ta sử dụng `bind mounts` thì một file hoặc một thư mục trên Docker host sẽ được mount tới containers với đường dẫn đầy đủ.

    - Đây là các trường hợp phổ biến lựa chọn `bind mounts` đối với containers:

        + **Phát triển và Debugging ứng dụng**: Khi bạn đang phát triển và debug ứng dụng, việc sử dụng bind mounts cho phép bạn chia sẻ mã nguồn và tệp tin cấu hình từ máy tính phát triển của bạn vào container. Điều này giúp bạn thấy được các thay đổi ngay lập tức và tiện lợi khi thử nghiệm sửa lỗi và phát triển.

        + **Cập nhật cấu hình ứng dụng dễ dàng**: Khi bạn muốn cập nhật tệp tin cấu hình của ứng dụng mà không cần phải tạo lại container hoặc image. Bind mounts cho phép bạn cập nhật cấu hình từ bên ngoài và ứng dụng trong container sẽ thấy được các thay đổi ngay lập tức.

        + **Chia sẻ dữ liệu giữa container và host**: Bind mounts cho phép bạn chia sẻ dữ liệu giữa container và máy host. Điều này có thể hữu ích khi bạn muốn dữ liệu trong container có sẵn cho các ứng dụng hoặc dịch vụ bên ngoài container.

        + **Chia sẻ mã nguồn và tệp tin tĩnh**: Bind mounts thường được sử dụng để chia sẻ mã nguồn và tệp tin tĩnh, chẳng hạn như trang web tĩnh hoặc tệp tin hệ thống cục bộ mà bạn muốn sử dụng trong container.

        + **Chia sẻ dữ liệu trong quá trình thử nghiệm và thử nghiệm nhanh chóng**: Khi bạn cần triển khai và thử nghiệm một container tạm thời và bạn muốn chia sẻ dữ liệu hoặc tệp tin cụ thể với container mà không cần sử dụng Docker Volumes.

        + **Phối hợp với nhiều container**: Bind mounts cho phép bạn chia sẻ cùng một thư mục hoặc tệp với nhiều container đang chạy trên cùng máy host.

- ### <a name="tmpfs">3.4 Trường hợp nào thì sử dụng tmpfs mount</a>

    - `Tmpfs mount` là một loại mount point trong Linux mà bạn có thể sử dụng để tạo không gian lưu trữ trên bộ nhớ RAM. Khi sử dụng `Tmpfs mount` trong môi trường Docker, nó thường phù hợp trong các trường hợp sau:

        + **Cache tạm thời**: `Tmpfs mount` thường được sử dụng để tạo không gian lưu trữ tạm thời cho các ứng dụng hoặc dịch vụ mà cần sử dụng dữ liệu tạm thời trong quá trình hoạt động. Điều này có thể giúp cải thiện hiệu suất bằng cách đọc và ghi dữ liệu nhanh chóng từ bộ nhớ RAM.

        + **Giảm việc ghi lên ổ đĩa**: Khi bạn không muốn lưu trữ dữ liệu tạm thời trên ổ đĩa cứng vì nó có thể làm giảm tuổi thọ ổ đĩa hoặc tạo ra nhiều hoạt động đọc/ghi, `Tmpfs mount` trên RAM có thể giúp giảm tải cho ổ đĩa.

        + **Bảo mật hoặc dữ liệu nhạy cảm**: `Tmpfs mount` được tạo trên bộ nhớ RAM, điều này có nghĩa rằng dữ liệu tạm thời sẽ bị xóa khi máy chủ hoặc container bị khởi động lại. Điều này có thể hữu ích cho các ứng dụng hoặc dịch vụ yêu cầu tính bảo mật cao, vì dữ liệu không được lưu trữ trên đĩa.

        + **Tạo ổ đĩa ảo tạm thời**: Bạn có thể sử dụng `Tmpfs mount` để tạo một ổ đĩa ảo tạm thời cho container hoặc ứng dụng để lưu trữ tạm thời, tạo ra một không gian lưu trữ tạm thời như /tmp trên bộ nhớ RAM.

        + **Hiệu suất cao và đáng tin cậy**: `Tmpfs mount` trên RAM làm cho dữ liệu có thời gian truy cập thấp và tốc độ ghi rất nhanh. Điều này có thể hữu ích trong các tình huống yêu cầu hiệu suất cao và đáng tin cậy.

    Tuy nhiên, lưu ý rằng khi sử dụng `Tmpfs mount`, bạn cần quản lý việc sử dụng bộ nhớ RAM, vì dữ liệu được lưu trữ trong `Tmpfs mount` sẽ sử dụng bộ nhớ thực tế. Nếu bạn sử dụng quá nhiều `Tmpfs mount`s hoặc cấu hình `Tmpfs mount` quá lớn, có thể gây ảnh hưởng đến hiệu suất tổng thể của hệ thống hoặc container của bạn.

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
