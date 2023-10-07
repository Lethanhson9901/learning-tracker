# Fundamental of database engineering Udemy course

- **Section 2: ACID**
    
    [Khái niệm transaction trong database | TopDev](https://topdev.vn/blog/khai-niem-transaction-trong-database/)
    
    - **Transaction** (giao dịch) là một tập hợp các thao tác mà ở đó hoặc là các thao tác đó được thực hiện thành công, hoặc là không một thao tác nào được thực hiện thành công cả.
        - Ví dụ *Transaction chuyển tiền* từ tài khoản A sang tài khoản B. Đây là một tính năng mà quá trình xử lý sẽ gồm 2 bước:
            - Thứ nhất là trừ tiền trong tài khoản A
            - Và bước thứ hai là cộng tiền vào tài khoản B.
        - *Transaction chuyển tiền* thành công khi và chỉ khi cả 2 bước trên thành công. Các trường hợp còn lại tính là *transaction* thất bại
    - **ACID** là viết tắt của 4 thuộc tính quan trọng cần đảm bảo khi thực hiện bất cứ giao dịch nào với CSDL, bao gồm:
        - **Atomicity**: tính nguyên tử hay còn có thể hiểu là ***“tất cả hoặc không là gì cả”***
            - Toàn bộ các bước được thực hiện trong một transaction nếu thành công thì phải thành công tất cả, nếu thất bại thì tất cả cũng phải thất bại.
            - Nếu một transaction ***thành công thì tất cả*** những thay đổi phải được ***lưu vào database***.
            - Nếu ***thất bại thì tất cả*** những thay đổi trong transaction đó phải được ***rollback về trạng thái ban đầu***.
            - VD: khi đã trừ tiền trong tài khoản A thì bắt buộc việc cộng tiền vào tài khoản B phải xảy ra.
        - **Consistency**: ***tính nhất quán***, tức ***dữ liệu*** từ thời điểm start transaction với lúc kết thúc phải nhất quán.
            - VD: nếu đã trừ 10$ trong tài khoản A thì trong tài khoản B phải được cộng 10$.
        - **Isolation**: quy định rằng trong từng transaction sẽ đều phải thực hiện ***độc lập***. Nếu cần 2 transaction diễn ra cùng một thời điểm cần cơ chế bảo đảm transaction này tránh ảnh hưởng đến transaction khác.
            - Ví dụ trong trường hợp khách hàng cùng chuyển tiền vào tài khoản công ty đúng thời điểm kế toán rút tiền vậy hành động nào sẽ diễn ra?
            - Lúc này database sẽ thực hiện hai hành động là cập nhật số tiền trong tài khoản bằng cách trừ đi số tiền kế toán rút ra và ngay lập tức cộng thêm vào số dư hiện tại số tiền khách chuyển. Đặc tính độc lập này rất cần thiết giúp đảm bảo các giao dịch diễn ra thành công và không ảnh hưởng đến dữ liệu.
            - **Dirty reads, Nonrepeatable reads, phantom reads, Isolation levels**
                - **Dirty Reads** điều này xảy ra khi một transaction tiến hành đọc dữ liệu mà chưa được commited. Ví dụ: transaction A cập nhập 1 dữ liệu, transaction B đọc dữ liệu sau khi A cập nhật xong. Nhưng vì lý do nào đó A không commit thành công, dự liệu quay trở lại trạng thái ban đầu, khi đó dữ liệu của B trở thành Dirty.
                    
                    ![Untitled]static/Untitled.png)
                    
                - **Nonrepeatable reads** xảy ra khi một transaction đọc cùng 1 dữ liệu 2 lần nhưng lại nhận được giá trị khác nhau. Ví dụ: transaction A đọc 1 dữ liệu, transaction B cập nhật xóa dữ liệu đó. Nếu A đọc lại dữ liệu đó nó sẽ lấy các giá trị là khác nhau.
                - **Phantom reads** là rủi ro xảy ra với lệnh read có điều kiện. Ví dụ: giả sử transaction A đọc một tập hợp các dữ liệu đáp ứng một số điều kiện tìm kiếm, transaction B tạo ra một dữ liệu mới khớp với điều kiện được tìm kiếm cho transaction A. Nếu A thực hiện lại với điều kiện như vậy thì nó sẽ nhận dc một tập hợp các dữ liệu là không đồng nhất.
                    
                    ![Untitled]static/Untitled%201.png)
                    
                - Như vậy, để tránh được các trường hợp kể trên chúng ta cần phải khóa dữ liệu, không cho những tiến trình xử lý khác thực hiện các operations trên dữ liệu khi transaction hiện tại đang làm việc và việc khóa này sẽ được giải phóng ở cuối transaction. Có 3 loại khóa dữ liệu là: write locks, read locks, rang locks. ***Isolation Levels*** chỉ ra những mức độ khóa khác nhau.
                    
                    ![Untitled]static/Untitled%202.png)
                    
                    - **Read uncommitted** Khi transaction thực hiện ở mức này, các truy vấn vẫn có thể truy nhập vào các bản ghi đang được cập nhật bởi một transaction khác và nhận được dữ liệu tại thời điểm đó mặc dù dữ liệu đó chưa được commit. Nếu vì lý do nào đó transaction ban đầu rollback lại những cập nhật, dữ liệu sẽ trở lại giá trị cũ. Khi đó transaction thứ hai nhận được dữ liệu sai.
                    - **Read committed** Transaction sẽ không đọc được dữ liệu đang được cập nhật mà phải đợi đến khi việc cập nhật thực hiện xong. Vì thế nó tránh được dirty read như ở mức trên.
                    - **Repeatable read** Mức isolation này hoạt động nhứ mức read commit nhưng nâng thêm một nấc nữa bằng cách ngăn không cho transaction ghi vào dữ liệu đang được đọc bởi một transaction khác cho đến khi transaction khác đó hoàn tất. Tuy nhiên nó không bảo vệ được dữ liệu khỏi insert hoặc delete: *nếu bạn thay lệnh update ở cửa sổ thứ hai bằng lệnh insert, hai lệnh select ở cửa sổ đầu sẽ cho kết quả khác nhau.* Vì thế nó vẫn không tránh được hiện tượng phantom read.
                    - **Serializable** Mức isolation này tăng thêm một cấp nữa và khóa toàn bộ dải các bản ghi có thể bị ảnh hưởng bởi một transaction khác, dù là UPDATE/DELETE bản ghi đã có hay INSERT  bản ghi mới.
                    - Bonus **Snapshot:**  Mức độ này cũng đảm bảo độ cô lập tương đương với Serializable, nhưng nó hơi khác ở phương thức hoạt động. Khi transaction đang select các bản ghi, nó không khóa các bản ghi này lại mà *tạo một bản sao (snapshot) và select trên đó*. Vì vậy các transaction khác insert/update lên các bản ghi đó không gây ảnh hưởng đến transaction ban đầu. Tác dụng của nó là giảm blocking giữa các transaction mà vẫn đảm bảo tính toàn vẹn dữ liệu. Tuy nhiên cái giá kèm theo là cần thêm bộ nhớ để lưu bản sao của các bản ghi, và phần bộ nhớ này là cần cho mỗi transaction do đó có thể tăng lên rất lớn
                    
                    ![Untitled]static/Untitled%203.png)
                    
                    ![Untitled]static/Untitled%204.png)
                    
        - **Durability**: tính bền vững, điều này có nghĩa dữ liệu sau khi thực hiện transaction sẽ không thay đổi nếu chúng ta gặp vấn đề gì đó liên quan đến database.
            - Ví dụ với transaction chuyển tiền được diễn ra thành công. Tức là khi giao dịch đã hoàn tất thì tất cả những thay đổi sẽ ghi lại ở dạng bền như đĩa cứng và cả giao dịch đã hoàn thành cũng được ghi lại.
                - Sau khi chuyển tiền vào tài khoản B thành công thì dù database có gặp bất cứ vấn đề gì thì tài khoản B vẫn đảm bảo đã nhận đủ 10$.
            - Các kỹ thuật đảm bảo tính bền vững:
                - WAL - Write ahead log
                    
                    ![Untitled]static/Untitled%205.png)
                    
                - OS cache
                    
                    ![Untitled]static/Untitled%206.png)
                    
                - Asynchronous snapshot
                - AOF
        
        ![Untitled]static/Untitled%207.png)
        
    - ACID by practical examples:
        
        [co-so-du-lieu-phan-tan__cac-muc-isolation-level - [cuuduongthancong.com].pdf]static/co-so-du-lieu-phan-tan__cac-muc-isolation-level_-_cuuduongthancong.com.pdf)
        
        - Phantom reads:
            - Demo bằng 2 transaction như sau:
            
            ```sql
            --Transaction 1  
            BEGIN TRANSACTION;  
            SELECT ID FROM dbo.employee  
            WHERE ID &gt; 5 and ID &lt; 10;  
            --The INSERT statement from the second transaction occurs here.  
            SELECT ID FROM dbo.employee  
            WHERE ID &gt; 5 and ID &lt; 10;  
            COMMIT;
            ```
            
            ```sql
            --Transaction 2  
            BEGIN TRANSACTION;  
            INSERT INTO dbo.employee  
             (Id, Name) VALUES(6 ,'New');  
            COMMIT;
            ```
            
            - Để tránh hiện tượng Phantom reads sử dụng isolation levels (mức cô lập cao nhất)
            
            ```sql
            --Transaction 1  
            BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;  
            SELECT ID FROM dbo.employee  
            WHERE ID &gt; 5 and ID &lt; 10;  
            --The INSERT statement from the second transaction occurs here.  
            SELECT ID FROM dbo.employee  
            WHERE ID &gt; 5 and ID &lt; 10;  
            COMMIT;
            ```
            
    - **Eventual Consistency và Strong Consistency**
        
        [Eventual Consistency và Strong Consistency trong hệ thống Cơ sở dữ liệu phân tán](https://techmaster.vn/posts/34879/eventual-consistency-va-strong-consistency-trong-he-thong-co-so-du-lieu-phan-tan)
        
        - **Distributed Database** (hệ thống cơ sở dữ liệu phân tán): Là hệ thống Cơ sở dữ liệu (CSDL) mà có thể được phân tải, lưu trữ ở nhiều nơi. Ví dụ như ứng dụng sử dụng nhiều CSDL và các CSDL có thể nằm ở các máy chủ vật lý khác nhau.
        - **Strong Consistency** (tính nhất quán mạnh): Sau khi một cập nhật được diễn ra thì tất cả các lần đọc dữ liệu sau đó đều trả về giá trị mới được cập nhật.
        - **Eventual Consistency** (tính nhất quán cuối cùng, là một dạng của tính nhất quán yếu - Weak Consistency): Sau khi một cập nhật được diễn ra, các lần đọc sau đó không đảm bảo sẽ luôn trả về giá trị mới được cập nhật (có thể có lần đọc vẫn trả về dữ liệu cũ). Tuy nhiên sau một khoảng thời gian (đồng bộ giữa các CSDL) thì cuối cùng các lần đọc đều trả về giá trị mới nhất.
- **Section 3: Understanding Database Internals**
    - **How tables and indexes are store on disk** (Bảng và index được lưu trữ ntn trong ổ đĩa)
        - **Page**
            - Tùy thuộc vào kiểu lưu trữ dữ liệu của DB (row vs column store), hàng (hoặc cột) sẽ được lưu trữ và đọc dưới dạng *logical pages (trang logic)*
            - DB không đọc từng hàng một mà sẽ đọc theo trang (1 hoặc nhiều trang)
            - Mỗi trang sẽ có kích thước riêng tùy theo DB (VD 8KB trong postgree, 16KB trong MySQL)
            
            ![Untitled]static/Untitled%208.png)
            
        - **IO**
            - Các hoạt động I/O làm việc với ổ đĩa (có tốc độ chậm) nên ta sẽ cố gắng giảm thiểu tối đa chúng
            - IO sẽ lấy dữ liệu 1 hoặc nhiều trang phụ thuộc vào phân vùng ổ đĩa và các yếu tố khác.
            - IO cũng đọc theo trang và một vài hoạt động IO đi đến hệ thống cache thay vì ổ đĩa
            
            ![Untitled]static/Untitled%209.png)
            
        - **Heap**
            - *Heap* là cấu trúc dữ liệu nơi mà bảng được lưu cùng với tất cả các *page* theo thứ tự lần lượt.
            - Đây là nơi dữ liệu thực sử được lưu trữ
            - Việc duyệt toàn bộ *heap* rất tốn kém nên ta cần đánh *index* (cho chúng ta biết vị trí chính xác cần đọc trong heap)
            
            ![Untitled]static/Untitled%2010.png)
            
        - **Index**
            
            [sử dụng index trong sql query](https://viblo.asia/p/su-dung-index-trong-sql-query-1ZnbRlPQR2Xo)
            
            - Index trong SQL *tăng tốc độ của quá trình truy vấn dữ liệu* bằng cách cung cấp phương pháp truy xuất nhanh chóng tới các dòng trong các bảng, *tương tự như* cách mà *mục lục của một cuốn sách* giúp bạn nhanh chóng tìm đến một trang bất kỳ mà bạn muốn trong cuốn sách đó.
            - Có thể *index* 1 hoặc nhiều cột
            - Tuy nhiên chúng ta không nên tạo index trên các cột có kiểu dữ liệu quá lớn vì *để sử dụng index SQL server cần chi phí để quản lý* một vùng nhớ mình tạm gọi nó là *mục lục* ở đây. Độ lớn của mục lục sẽ tỉ lệ thuận với length index key bạn sử dụng.
            - Cấu trúc dữ liệu được sử dụng phổ biến cho index là *b-trees*
                - VD: Index trong SQL Server được tạo thành từ một tập hợp các page và chúng được tổ chức trong cấu trúc *b-tree*.
            
            ![Untitled]static/Untitled%2011.png)
            
            - Hình dưới minh họa việc sử dụng index:
                - Index theo cột EMP_ID: 10(1,0)
                - IO1 tại index lấy thông tin page/row theo EMP_ID: ⇒ employee id = 70 sẽ nằm ở dòng 7 page 2
                - IO2 tại heap sẽ tìm được chính xác pages từ thông tin do IO1 cung cấp tại index
                - Thử tưởng tượng trường hợp EMP_ID = 10000, sẽ tiết kiệm được rất nhiều thời gian query khi có index
                    
                    ```sql
                    select * from EMP where EMP_ID = 10000;
                    ```
                    
            
            ![Untitled]static/Untitled%2012.png)
            
    - **Row-based vs Column-based database**
        - Row-based DB
        
        ![Untitled]static/Untitled%2013.png)
        
        - Column Store
        
        ![Untitled]static/Untitled%2014.png)
        
        - So sánh:
        
        ![Untitled]static/Untitled%2015.png)
        
    - **Primary Key vs Secondary Key**
        - 
- **Section 4: Database Indexing**
    
    Chương này chúng ta sẽ thực hành index với Postgres DB
    
    - VD1: Lớp mầm non
        - Đầu tiên, cần các bước cài đặt môi trường. Tác giả sử dụng qua docker:
            
            ```bash
            #Pull postgres docker image về trong trường hợp chưa có
            docker pull postgres
            
            #Run docker images
            sudo docker run -e POSTGRES_PASSWORD=postgres --name pg1 postgres
            
            # Chạy command bên trong docker container
            docker exec -it pg1 psql -U postgres
            ```
            
            ![Untitled]static/Untitled%2016.png)
            
            - Thao tác vui với postgres: Tạo 1M dòng
        
        ```sql
        #Tạo bảng
        create table temp(t int);
        #tạo 1M dòng ngẫu nhiên
        insert into temp(t) select random()*100 from generate_series(0, 1000000);
        #check
        select count(*) from temp;
        ```
        
        ![Untitled]static/Untitled%2017.png)
        
    - VD2: Làm quen với Indexing
        
        B1: Import *employees.sql* file vào trong docker (hoặc là tạo 10M rows)
        
        [employees.sql]static/employees.sql)
        
        ```bash
        # run docker
        docker run --name pg -e POSTGRES_PASSWORD=postgres -d postgres
        
        #start docker
        docker start pg
        
        #chui vao postgres
        docker exec -it pg psql -U postgres
        
        #generate data
        create table employees( id serial primary key, name text);
        
        create or replace function random_string(length integer) returns text as 
        $$
        declare
          chars text[] := '{0,1,2,3,4,5,6,7,8,9,A,B,C,D,E,F,G,H,I,J,K,L,M,N,O,P,Q,R,S,T,U,V,W,X,Y,Z,a,b,c,d,e,f,g,h,i,j,k,l,m,n,o,p,q,r,s,t,u,v,w,x,y,z}';
          result text := '';
          i integer := 0;
          length2 integer := (select trunc(random() * length + 1));
        begin
          if length2 < 0 then
            raise exception 'Given length cannot be less than 0';
          end if;
          for i in 1..length2 loop
            result := result || chars[1+random()*(array_length(chars, 1)-1)];
          end loop;
          return result;
        end;
        $$ language plpgsql;
        
        insert into employees(name)(select random_string(10) from generate_series(0, 1000000));
        ```
        
        ```sql
        #Test thử select query sử dụng index
        explain analyze select id from employees where id = 2000;
        
        # select query không có index (lâu hơn rất nhiều)
        explain analyze select id from employees where name = 'Son';
        ```
        
        ![Untitled]static/Untitled%2018.png)
        
        ```sql
        #Tạo index cho cột name
        create index employees_name on employees(name);
        #Chạy lại câu query trên (lại nhanh như gió)
        explain analyze select id from employees where name = 'Son';
        ```
        
        ![Untitled]static/Untitled%2019.png)
        
        ```sql
        #Thay thế điều kiện '=' bằng 'like' (lâu hơn rất nhiều)
        explain analyze select id, name from employees where name like '%Son%';
        ```
        
        ![Untitled]static/Untitled%2020.png)
        
        ⇒ Lý do là vì toán tử ‘LIKE’ với các mẫu có ký tự đại diện không tận dụng được sức mạnh của index. Việc sử dụng pattern ‘%Son%’ *yêu cầu hệ thống cơ sở dữ liệu quét qua tất cả các giá trị chỉ mục để xác định các hàng phù hợp*. Quá trình quét này chậm hơn việc tra cứu trực tiếp, vì nó cần kiểm tra từng giá trị để xác định sự hiện diện của mẫu mong muốn. 
        
    - VD3: Bitmap Index Scan vs Index Scan vs Table scan
        
        [Có những cách nào để tối ưu SQL Query? - Viblo - Dat Bui](https://viblo.asia/p/001-co-nhung-cach-nao-de-toi-uu-sql-query-RnB5pVgrZPG)
        
        [Hiểu về Index để tăng performance với PostgreSQL P1 - Dat Bui](https://viblo.asia/p/002-hieu-ve-index-de-tang-performance-voi-postgresql-p1-3Q75wV3elWb)
        
        [Hiểu về Index để tăng performance với PostgreSQL P2 - Dat Bui](https://viblo.asia/p/003-hieu-ve-index-de-tang-performance-voi-postgresql-p2-m68Z049MZkG)
        
        - Bài viết rất hay, tất cả thứ bạn cần ở link trên.
        - **Scanning table** là task cơ bản nhất của **execution plan**, khởi nguồn của mọi thứ liên quan đến quá trình execute query. Hiểu đơn giản, nó là hành động tuyến tính, scan từng row từ đầu đến cuối, so sánh với các điều kiện (nếu có) và trả về kết quả. Do vậy, thời gian tìm kiếm sẽ phụ thuộc vào số lượng row table.
        - **indexing** là việc chuyển đổi một hoặc nhiều column sang table mới (DB System sẽ quản lý table này). **Index Scan** chính là scan trên table index
        - Bản chất của **bitmap index** vẫn sử dụng cấu trúc B-Tree, tuy nhiên nó lưu trữ thông tin khác với B-Tree index. **B-Tree index** mapping index với một hoặc nhiều rowId. **Bitmap index** mapping index với giá trị bit tương ứng của column. Ví dụ có 3 giá trị của gender: Male, Female, Unknown. Tạo ra 3 bit tương ứng 0, 1, 2 cho 3 giá trị đó và mapping với column trong table chính.
            - **bitmap index** có các tính chất:
                - Phù hợp với các column low cardinality.
                - Lưu bit cho mỗi giá trị nên giảm dung lượng lưu trữ cần dùng.
                - Chỉ hiệu quả với tìm kiếm full match value.
                - Kết hợp với nhiều index khác để tăng tốc độ với OR, AND.
    - VD4: Key vs Non-key indexes
        
        
    - VD5: Index Scan vs Index Only Scan
        - Đầu tiên, tạo một bảng 5M dòng với 3 trường: id, g, name
        - Thực hiện query *select name* mà chưa tạo index
            
            ![Untitled]static/Untitled%2021.png)
            
        - Tạo index
            
            ![Untitled]static/Untitled%2022.png)
            
        - Thực hiện query lại, khi này Index Scan sẽ được sử dụng
            
            ![Untitled]static/Untitled%2023.png)
            
        - Sửa query: Thay vì *select name ⇒ select id.* Index Only Scan được sử dụng
            
            ![Untitled]static/Untitled%2024.png)
            
            - So what happended ??? Lý do là vì mình đánh index cột *id*, query sau ko từ index ta đã có đầy đủ thông tin rồi nên ko cần phải truy cập vào bảng để lấy thông tin nữa
                1. **Index Scan:** An index scan, also known as a non-covering index scan, involves accessing the index structure to locate the relevant rows in the table. Once the index identifies the matching rows, the database engine retrieves the corresponding data from the table itself. In an index scan, the database engine needs to perform two I/O operations: one to read the index and another to retrieve the actual data from the table.
                2. **Index-Only Scan:** An index-only scan, on the other hand, is a more efficient method where the database engine can retrieve all the required data solely from the index, *without accessing the table*. In this case, the index contains all the necessary information, including the indexed columns and any additional columns required by the query. As a result, an index-only scan reduces the I/O overhead because it eliminates the need to read data from the table, making it faster and more efficient.
        - 
        
    - VD6: Bloom filter
        
        [Bloom Filters: Cấu trúc lưu trữ dữ liệu dựa trên xác suất. - Viblo](https://viblo.asia/p/bloom-filters-tai-sao-cac-mang-blockchain-lai-thuong-su-dung-no-GrLZD07eZk0)
        
        ![Untitled]static/Untitled%2025.png)
        
- **Section 5: B-Tree and B+Tree in Production DB system**
    - **Full Table Scan**
        
        ![Untitled]static/Untitled%2026.png)
        
    - **Original B-Tree**
        
        **B-Tree** lưu trữ dữ liệu theo kiểu mỗi node lưu trữ key theo thứ tự tăng dần, mỗi node này lại chứa 2 liên kết đến những node trước và sau nó. Node bên trái có key <= key node hiện tại, còn node bên phải thì có key >= key của node hiện tại. Nếu một node mà có n keys thì nó sẽ có tối đa là n + 1 node con.
        
        ![Untitled]static/Untitled%2027.png)
        
        ![Untitled]static/Untitled%2028.png)
        
        ![Untitled]static/Untitled%2029.png)
        
    - **How the Original B-Tree Helps Performance**
        
        [Giới thiệu về B-tree index trong database](https://viblo.asia/p/gioi-thieu-ve-b-tree-index-trong-database-ByEZkQn25Q0)
        
        **B-Tree** index tăng tốc độ truy vấn dữ liệu vì storage engine không phải duyệt cả bảng để tìm kiếm dữ liệu mà nó sẽ đi từ node root. các vị trí root node sẽ chưa con trỏ tới những node con, nó tìm đúng con trỏ bằng cách nhìn vào các giá trị trong node con và bằng việc xác định các giới hạn trên và dưới của một node, giúp storage engine dễ dàng tìm kiếm sự tồn tại hay không của một giá trị.
        
        VD về B-Tree index:
        
        ```sql
        CREATE TABLE People (
        last_name varchar(50) not null,
        first_name varchar(50) not null,
        dob date not null,
        gender enum('m', 'f')not null,
        key(last_name, first_name, dob)
        );
        ```
        
        - Index ở đây chính là last_name, first_name, dob trong bảng People.
        
        ![Untitled]static/Untitled%2030.png)
        
        - **Những kiểu truy vấn có thể dùng với B-Tree**
            - **Tìm kiếm đầy đủ giá trị (Full value)**
            Là việc tìm kiếm tất cả đầy đủ giá trị trong những column đã được đánh index ví dụ chúng ta có thể tìm kiếm những người có tên Cuba Allen và sinh năm 1960-01-01
            - **Tìm kiếm tiền tố bên trái**
            Chúng ta có thể tìm kiếm tất cả những người có tên Allen, tuy nhiên điều này chỉ ứng dụng với column đầu tiên trong index.
            - **Tìm kiếm tiền tố column**
            Chúng ta có thể tìm kiếm những người có last_name bắt đầu bằng chữ cái J, và điều này cũng chỉ hữu ích cho column đầu tiên của index.
            - **Tìm kiếm trong một giải các giá trị**
            Điều này giúp chúng ta tìm kiếm những người có tên nằm trong khoảng từ Allen đến Barymore. Tuy nhiên nó cũng chỉ hữu ích với column đầu tiên của Index.
            - **Tìm kiếm một phần và một dải trong phần tiếp theo**
            Giups chúng ta tìm kiếm những người có last_name là Allen và có first_name bắt đầu bằng ký tự K. Nó chỉ có tác dụng với last_name và một dải của first_name.
                
                Bởi vì các node trên tree đã được sắp xếp chính vì thế chúng có thể thực hiện cả 2 việc là tìm kiếm giá trị và tìm kiếm trên một tập đã sắp xếp.
                
        - **Một số giới hạn của B-Tree**
            - Giới hạn lớn nhất của B-Tree chính là việc nó c**hỉ tìm kiếm tốt tại cột ngoài cùng bên trái của index**.
            - Bạn **không thể bỏ qua columns trong index** ví dụ không thể chỉ tìm kiếm từ last_name và dob mà bỏ qua first_name storage engine không thể tối ưu truy vấn với bấy kỳ column nào nằm bên phải một dải giá trị, ví dụ nếu như bạn query WHERE last_name="Smith" AND first_name LIKE 'J%' AND dob='1976-12-23' Thì sẽ chỉ có tác dụng trên 2 cột đầu tiên bên trai trong index. Bởi vì LIKE ở đây được hiểu như một dải các giá trị.
            
            ![Untitled]static/Untitled%2031.png)
            
        - Qua đây chúng ta có thể thấy rằng *việc sắp xếp các column trong index là vô cùng quan trọng , Để có hiệu suất tối ưu việc tạo chỉ mục cho cùng một column với các thứ tự khác nhau là cần thiết*.
        
    - **B+Tree**
        
        ![Untitled]static/Untitled%2032.png)
        
        ![Untitled]static/Untitled%2033.png)
        
    - **B+Tree DBMS Consideration**
        
        ![Untitled]static/Untitled%2034.png)
        
        - 
    - **B+Tree Storage cost in MySQL vs Postgres**
        
        ![Untitled]static/Untitled%2035.png)
        
- **Bonus: Joining Table**
    
    [Hiểu về Join để tăng performance với PostgreSQL - Viblo](https://viblo.asia/p/005-hieu-ve-join-de-tang-performance-voi-postgresql-924lJjpXlPM)
    
    - Nested Loop Join
    - Hash Join
    - Merge Join
- **Section 6: Database Partitioning**
    
    Chương này giới thiệu một hướng tiếp cận khác có thể tăng query performance là áp dụng **Database Partitioning.**
    
    [Partitioning data với PostgreSQL P1 - Viblo - Dat Bui](https://viblo.asia/p/006-partitioning-data-voi-postgresql-p1-1VgZvr87ZAw)
    
    - **What is Partitioning ?**
        - Là việc chia bảng thành các phần nhỏ hơn
    - ****Horizontal & Vertical partitioning****
        - **Horizontal partitioning**: Chia table lớn thành nhiều table nhỏ hơn, các table nhỏ hơn gọi là **partition table**, kế thừa toàn bộ cấu trúc của parent table, từ column cho đến kiểu dữ liệu
            
            ![Untitled]static/Untitled%2036.png)
            
            - **Horizontal partitioning** có những ưu điểm sau:
                - Giới hạn vùng dữ liệu phải scan trên table trong một vài trường hợp. Nếu ta cần tìm một học sinh tên John Doe không phân biệt giới tính thì việc partition như ví dụ trên không đem lại hiểu quả.
                - Partition table cũng giống như một table thường nên ta có thể thực hiện index cho nó. Dẫn đến việc tốn ít cost hơn để maintain index table, do số lượng record ít hơn.
                - Ngoài ra, việc xóa các dữ liệu trên partition table sẽ nhanh hơn và không ảnh hưởng đến các partition khác.
            - Với những ưu điểm trên, ta cần lưu ý khi thực hiện **horizontal partitioning** để đạt hiệu quả tối đa:
                - Áp dụng với các table rất lớn. Thường là quá size của memory.
                - Việc partition trên điều kiện nào phải dựa vào tính chất và tần suất của các query.
        - **Vertical partitioning** cũng là việc chia một table ra thành nhiều **partition table** nhưng theo chiều.. dọc. Ví dụ một table 100 columns được partition thành 4 table mỗi table 25 columns. Về cơ bản sẽ không có một tiêu chuẩn hay công thức cụ thể nào cho việc **vertical partitioning**. Ta chỉ cần chú ý đến việc nhóm các columns có tần suất query cùng nhau thành một partition.
            
            Ngoài ra, với **vertical partitioning**, best practice là *sử dụng chung một PK cho toàn bộ các partition table.*
            
            ![Untitled]static/Untitled%2037.png)
            
            - Vậy lợi ích của **vertical partitioning** là gì?
                - Về cơ bản, các records được lưu thành một khối dữ liệu có độ lớn gần tương tự như nhau được gọi là block. *Do đó, nếu một table chứa số lượng column ít đồng nghĩa với việc tăng đương số lượng records lưu trữ trên một block*. Như vậy nếu query các column trong cùng block, các xử lý tính toán I/O sẽ giảm đi phần nào dẫn tới việc tăng performance
    - **Phân biệt Horizontal Partitioning và Sharding**
        
        ![Untitled]static/Untitled%2038.png)
        
    - **Demo**
        
        ```sql
        create table grades_org (id serial not null, g int  not null);
        insert into grades_org(g) select floor(random()*100) from generate_series(0, 10000000);
        #create index
        create index grades_org_index on grades_org(g);
        ```
        
        ```sql
        #test select
        explain analyze select count(*) from grades_org where g = 30; 
        explain analyze select count(*) from grades_org where g between 30 and 35;
        ```
        
        ![Untitled]static/Untitled%2039.png)
        
        ```sql
        #Tao Partition
        create table grades_parts (id serial not null, g int not null) partition by range(g);
        create table g0035 (like grades_parts including indexes);
        create table g3560 (like grades_parts including indexes);
        create table g6080 (like grades_parts including indexes);
        create table g80100 (like grades_parts including indexes);
        
        #attach partitioned tables
        alter table grades_parts attach partition g0035 for values from (0) to (35);
        alter table grades_parts attach partition g3560 for values from (35) to (60);
        alter table grades_parts attach partition g6080 for values from (60) to (80);
        alter table grades_parts attach partition g80100 for values from (80) to (100);
        
        #insert data into grades_parts
        insert into grades_parts select * from grades_org;
        
        #create index on grades_parts
        create index grades_parts_idx on grades_parts(g);
        
        #see the magic
        explain analyze select count(*) from g0035 where g between 30 and 35;
        explain analyze select count(*) from grades_parts where g between 30 and 35;
        ```
        
        ![Untitled]static/Untitled%2040.png)
        
    - **Automate Partitioning in Postgres**
        
        [create_partitions.mjs]static/create_partitions.mjs)
        
        [package.json]static/package.json)
        
        [package-lock.json]static/package-lock.json)
        
        [populate_customers.mjs]static/populate_customers.mjs)
        
        - 
- **Section 7: Database Sharding**
    - Khái niệm Database Sharding ?
        - Sharding database là một mẫu kiến trúc cơ sở dữ liệu liên quan đến phân vùng ngang. Theo đó, một bảng dữ liệu lớn sẽ được *chia thành nhiều phân vùng khác nhau, được lưu trữ trên các server khác nhau*. Mỗi phân vùng này sẽ chứa một phần của dữ liệu và được xử lý độc lập với các phân vùng khác.
    - Consistent Hashing:
        
        [System Design Cơ Bản - Consistent Hashing | TopDev](https://topdev.vn/blog/system-design-co-ban-consistent-hashing/)
        
- **Section 8: Concurrency Control**
    - **Exclusive lock vs shared lock**
    
    [Exclusive lock và Shared lock - Viblo - Dat Bui](https://viblo.asia/p/010-exclusive-lock-va-shared-lock-924lJjn0lPM)
    
    - **Dead lock**
    - **Two phase Locking**
    - **Solving the Double Booking Problem**
    - ******************************************************************************************SQL Pagination with Offset is very slow******************************************************************************************
        
        ```sql
        #query using offset:
        explain analyze select title from news order by id desc offset 0 limit 10;
        #result: using index scan backward => 0.291s
        
        #without offset:
        explain analyze select title from news order by id desc limit 10;
        #result: down to => 0.197s
        ```
        
        ![Untitled]static/Untitled%2041.png)
        
        ![Screenshot from 2023-07-05 14-21-36.png]static/Screenshot_from_2023-07-05_14-21-36.png)
        
    
    [booking-system.zip]static/booking-system.zip)
    
- **Section 9: Database Replication**
    
    [Database Replication là gì?](https://devera.vn/blog/our-blog-1/post/database-replication-la-gi-75)
    
    *Database Replication* là một kỹ thuật/nguyên lý thiết kế sử dụng trong việc mở rộng (scale) cơ sở dữ liệu bằng cách duy trì các bản sao của dữ liệu trên nhiều server cơ sở dữ liệu khác nhau.
    
    - **Master/Stanby Replication**
        
        ![Untitled]static/Untitled%2042.png)
        
        ![Untitled]static/Untitled%2043.png)
        
    - **Multi-Master Replication**
        
        Mỗi node đều đóng vai trò là master node nơi mà Máy khách có thể thực hiện cả việc đọc và ghi dữ liệu. Giữa các node có thể được đồng bộ bằng cả 2 cách nêu trên (Synchronous và Asynchronous)
        
        Nhưng khi sử dụng Asynchronous vấn đề về xung đột dữ liệu rất dễ xảy ra khi các Máy khách cập nhật cùng một đơn vị dữ liệu nhưng trên các master-node khác nhau.
        
        ![Untitled]static/Untitled%2044.png)
        
    - Synchronous vs Asynchronous Replication
        
        ![Untitled]static/Untitled%2045.png)
        
- **Section 10: Database System Design**
    
    **Twitter Database System Design**
    

- **[Learning Database with Tran Quoc Huy](https://www.youtube.com/@tranquochuywecommit)**
    - **Bài 1: Select 1 cột và Select * cái nào nhanh hơn? - Phân tích chi tiết**
        - **Case 1: So sánh khi bảng không có Index**
            
            ![Untitled]static/Untitled%2046.png)
            
            - NX: Query 1 cột hay tất cả các cột là như nhau.
        - **Case 2: So sánh khi bảng đánh Single Index**
            - Thực hiện đánh index trên cột amount ⇒ Index Only Scan cho kết quả nhanh nhất
            
            ![Untitled]static/Untitled%2047.png)
            
        - **Case 3: So sánh khi bảng đánh Index trên nhiều cột**
            - Thực hiện đánh index trên 2 cột payment_date và amount ⇒ index only scan vẫn cho thời gian chạy tốt nhất.
            
            ![Untitled]static/Untitled%2048.png)
            
        - Tips ứng dụng thực tế:
            - Nhanh hay chậm phụ thuộc phần lớn vào chiến lược thực thi.
    - **Bài 2: Bản chất hoạt động của SQL trong cơ sở dữ liệu & ứng dụng trong các dự án tối ưu**
        
        [Tối ưu cơ sở dữ liệu cải thiện 97% thời gian thực hiện chỉ bằng một “chấm nhẹ” thế nào? - WE COMMIT](https://wecommit.com.vn/database-performance-tuning-speed-up-97/)
        
        *6 bước thực thi 1 câu lệnh – phải biết khi tối ưu cơ sở dữ liệu*
        
        ![Untitled]static/Untitled%2049.png)
        
        INDEX RANGE SCAN
        
        [Phân tích nguyên lý sử dụng RESULT CACHE giúp câu lệnh tăng tốc về 0s ngay lập tức - WE COMMIT](https://wecommit.com.vn/courses/chuong-trinh-dao-tao-toi-uu-co-so-du-lieu-cao-cap/lesson/phan-tich-nguyen-ly-su-dung-result-cache-giup-cau-lenh-tang-toc-ve-0s-ngay-lap-tuc/)
        
        build career path system
        
    - ****Bài 3: Tối ưu SQL và tối ưu Database: Kinh nghiệm và cách thức thực hiện****
        - Tối ưu SQL tách thành các luồng riêng biệt.
        - Tối ưu theo weight
        - Dùng Toast for oracle DB để theo dõi các thông số
            
            ![Untitled]static/Untitled%2050.png)
            
            ![Untitled]static/Untitled%2051.png)
            
            ![Untitled]static/Untitled%2052.png)
            
            ![Untitled]static/Untitled%2053.png)
            
            ![Untitled]static/Untitled%2054.png)
            
        - Partition DB chủ yếu dựa theo dung lượng
        - Tải thấp liu riu ⇒ tối ưu câu lệnh k ảnh hưởng nhiều.
    - **Bài 4:** ****Chia sẻ những trải nghiệm dự án tối ưu****
        
        ![Untitled]static/Untitled%2055.png)
        
        - SQL1 và SQL2 có chiến lược thực thi như nhau.
        - SQL3 index hiệu quả
        
        Làm dự án, mắc sai lầm, note lại, check list cho dự án mới
        
        Tư duy phép trừ, cái nào nhiều người làm đc, ko nên đi theo hướng đấy. (not to do list)
        
    - **Bài 5: Đơn giản hóa PostgreSQL**
        - Tiếp cận vấn đề kiến trúc theo hướng tại sao
    - ****Bài 6: Hiểu toàn bộ PostgreSQL trong 1h30p - 2023****
        
        [Mindmap kiến thức PostgreSQL trong 1h30ph.xmind]static/Mindmap_kin_thc_PostgreSQL_trong_1h30ph.xmind)
        
        - Sao lưu và khôi phục
            - Export ra file csv, sql
    - ****Bài 7: Có nên định kỳ Rebuild Index?****
        
        [Có nên định kỳ Rebuild Index không? - WE COMMIT](https://wecommit.com.vn/co-nen-dinh-ky-rebuild-index-khong/)
        
        -