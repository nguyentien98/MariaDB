[Nguồn](https://mariadb.com/kb/en/library/compound-composite-indexes/ "Permalink to Compound (Composite) Indexes - MariaDB Knowledge Base")

# Chỉ mục ghép (hỗn hợp) - kiến thức MariaDB cơ sở 

## Một bài học nhỏ trong "chỉ mục ghép" ("chỉ mục hỗn hợp")

Tài liệu này mở đầu bằng những thứ có vẻ tầm thường và có thể nhàm chán, nhưng càng về sau tài liệu được xây dựng bởi những thông tin thú vị hơn. Những điều đó có thể là những điều bạn chưa biết về cách hoạt động của chỉ mục trong MariaDB và MySql.

Nó cũng giải thích cho [EXPLAIN][1] (trong vài khía cạnh nào đó).

(Hầu hết những thứ được này áp dụng cho những cơ sở dữ liệu không phải là MySQL)

## Câu truy vấn để bàn luận

Câu hỏi là "Khi nào Andrew Johnson là tổng thống của nước Mỹ?".

Bảng `Presidents` có sẵn như sau:
    
    
    +-----+------------+----------------+-----------+
    | seq | last_name  | first_name     | term      |
    +-----+------------+----------------+-----------+
    |   1 | Washington | George         | 1789-1797 |
    |   2 | Adams      | John           | 1797-1801 |
    ...
    |   7 | Jackson    | Andrew         | 1829-1837 |
    ...
    |  17 | Johnson    | Andrew         | 1865-1869 |
    ...
    |  36 | Johnson    | Lyndon B.      | 1963-1969 |
    ...
    

("Andrew Johnson" đã được chọn cho bài học này vì có sự lặp lại)

Chỉ mục nào sẽ là tốt nhất cho câu hỏi này? Cụ thể hơn là, những gì sẽ là tốt nhất cho câu truy vấn này:
    
    
        SELECT  term
            FROM  Presidents
            WHERE  last_name = 'Johnson'
              AND  first_name = 'Andrew';
    

Một vài chỉ mục để thử

* Không chỉ mục
* Chỉ mục(first_name), Chỉ mục(last_name) (hai chỉ mục tách rời) 
* "Index Merge Intersect" 
* Chỉ mục(last_name, first_name) (một chỉ mục "ghép") 
* Chỉ mục(last_name, first_name, term) (một chỉ mục "bao trùm") 
* Các biến thể

## Không chỉ mục

Tốt rồi, bây giờ tôi đang phán đoán một chút ở đây. Tôi có một khóa chính là `seq`, nhưng nó ko có ích trong câu truy vấn chúng ta đang tìm hiểu.
    
    
    mysql>  SHOW CREATE TABLE Presidents G
    CREATE TABLE `presidents` (
      `seq` tinyint(3) unsigned NOT NULL AUTO_INCREMENT,
      `last_name` varchar(30) NOT NULL,
      `first_name` varchar(30) NOT NULL,
      `term` varchar(9) NOT NULL,
      PRIMARY KEY (`seq`)
    ) ENGINE=InnoDB AUTO_INCREMENT=45 DEFAULT CHARSET=utf8
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew';
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    | id | select_type | table      | type | possible_keys | key  | key_len | ref  | rows | Extra       |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    |  1 | SIMPLE      | Presidents | ALL  | NULL          | NULL | NULL    | NULL |   44 | Using where |
    +----+-------------+------------+------+---------------+------+---------+------+------+-------------+
    
    # Hoặc, bằng cách sử dụng hình thức hiển thị khác:  EXPLAIN ... G
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ALL        <-- Ngụ ý là scan bảng
    possible_keys: NULL
              key: NULL       <-- Ngụ ý là không có index nào hữu ích, do đó scan bảng
          key_len: NULL
              ref: NULL
             rows: 44         <-- Điều này là có bao nhiêu hàng trong bảng, vậy scan bảng
            Extra: Using where
    

## Chi tiết triển khai

Đầu tiên, hãy mô tả cách mà InnoDB lưu trữ và sử dụng các chỉ mục.

* Dữ liệu và khóa chính được nhóm lại cùng nhau trong BTree. 
* Tra cứu BTree khá nhanh và hiệu quả. Đối với một bảng triệu hàng có thể có 3 mức BTree và hai mức cao nhất có thể được lưu trong bộ nhớ cache. 
* Mỗi chỉ mục phụ trong một BTree khác nhau, với khóa chính PRIMARY KEY ở lá. 
* Tìm nạp các mục 'liên tiếp' (theo chỉ mục) từ BTree rất hiệu quả vì chúng được lưu trữ liên tục. 
* Vì mục đích đơn giản, chúng ta có thể đếm từng tra cứu BTree dưới dạng 1 đơn vị công việc, và bỏ qua các lần quét cho các mục liên tiếp. Điều này xấp xỉ số lần truy cập đĩa cho một bảng lớn trong một busy system.
Với MyISAM, khóa chính không được lưu cùng dữ liệu, vì vậy hãy nghĩ về nó như một chìa khóa phụ (quá đơn giản).

## Chỉ mục(first_name), Chỉ mục(last_name)

Người mới làm quen, một khi anh ấy biết về lập chỉ mục, quyết định lập chỉ mục nhiều cột, mỗi cái một lần. Nhưng...

MySQL hiếm khi sử dụng nhiều hơn một chỉ mục tại một thời điểm trong một truy vấn. Vì vậy, nó sẽ phân tích các chỉ mục có thể.

* first_name -- có hai hàng khả thi (một lần tra cứu BTree, sau đó quét liên tục) 
* last_name -- có hai hàng khả thi. Giả sử nó chọn last_name. Đây là một bước thực hiện SELECT: 1\. Sử dụng Chỉ mục(last_name), tìm 2 mục nhập với chỉ mục last_name = 'Johnson'. 2\. Lấy khóa chính (ngầm được thêm vào mỗi chỉ số phụ trong InnoDB); get (17, 36). 3\. Tiếp cận dứ liệu bằng cách sử dụng seq = (17, 36) để lấy hàng cho Andrew Johnson and Lyndon B. Johnson. 4\. Sử dụng phần còn lại của mệnh đề WHERE lọc tất cả trừ hàng mong muốn. 5\. Đưa ra kết quả (1865-1869). 
    
    
    mysql>  EXPLAIN  SELECT  term
                FROM  Presidents
                WHERE  last_name = 'Johnson'
                  AND  first_name = 'Andrew'  G
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: last_name, first_name
              key: last_name
          key_len: 92                 <-- VARCHAR(30) utf8 may need 2+3*30 bytes
              ref: const
             rows: 2                  <-- Two 'Johnson's
            Extra: Using where
    

## "Index Merge Intersect" 

Ok, vậy bạn thực sự thông minh và cho rằng MySQL đủ thông minh để sử dụng cả hai chỉ mục tên để tìm được câu trả lời. Nó được gọi là  "điểm giao". 1\. Sử dụng Chỉ mục(last_name), tìm 2 chỉ mục nhập vào với last_name = 'Johnson'; get (7, 17) 2\. Sử dụng Chỉ mục(first_name), tìm hai chỉ mục nhập vào với first_name = 'Andrew'; get (17, 36) 3\. "And" hai danh sách với nhau (7,17) & (17,36) = (17) 4\. Tiếp cận dữ liệu sử dụng seq = (17) để lấy hàng cho Andrew Johnson. 5\. Đưa kết quả (1865-1869).
    
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: index_merge
    possible_keys: first_name,last_name
              key: first_name,last_name
          key_len: 92,92
              ref: NULL
             rows: 1
            Extra: Using intersect(first_name,last_name); Using where
    

EXPLAIN không cung cấp thông tin chi tiết về số lượng hàng được thu thập từ mỗi chỉ mục, vân vân.

## Chỉ mục(last_name, first_name)

Cái này được gọi là một chỉ mục "ghép" hoặc "hỗn hợp" khi nó có nhiều hơn một cột. 1\. Tìm kiếm  BTree để lấy được chỉ mục có được chính xác hàng chỉ mục cho Johnson+Andrew; get seq = (17). 2\. Tiếp cận dữ liệu sử dụng seq = (17) để lấy hàng cho Andrew Johnson. 3\. Đưa ra kết quả (1865-1869). Cái này tốt hơn nhiều. Sự thất nó luôn "tốt nhất".
    
    
        ALTER TABLE Presidents
            (drop old indexes and...)
            ADD INDEX compound(last_name, first_name);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: compound
              key: compound
          key_len: 184             <-- The length of both fields
              ref: const,const     <-- The WHERE clause gave constants for both
             rows: 1               <-- Goodie!  It homed in on the one row.
            Extra: Using where
    

## "Bao trùm": Chỉ mục(last_name, first_name, term)

Ngạc nhiên chưa! Chúng ta thực sự có thể làm tốt hơn. Một chỉ mục "bao trùm" tất cả các trường của SELECT được tìm thấy trong chỉ mục. Nó có thêm điểm cộng là không phải tiếp cận vào "dữ liệu" để hoàn thành nhiệm vụ. 1\. Tìm kiếm BTree để lấy được chỉ mục có được chính xác hàng chỉ mục cho Johnson+Andrew; get seq = (17). 2\. Đưa ra kết quả (1865-1869). BTree "dữ liệu" không được chạm vào; đây là một cải tiến so với "ghép".
    
    
        ... ADD INDEX covering(last_name, first_name, term);
    
               id: 1
      select_type: SIMPLE
            table: Presidents
             type: ref
    possible_keys: covering
              key: covering
          key_len: 184
              ref: const,const
             rows: 1
            Extra: Using where; Using index   <-- Note
    

Mọi thứ giống như là sử dụng "ghép", ngoại trừ việc bổ sung "Sử dụng chỉ mục".

## Các biến thể 

* Điều gì sẽ xảy ra khi bạn xáo trộn các trường trong câu lệnh WHERE? Trả lời: Thứ tự mọi thứ trong AND không quan trọng. 
* Điều gì sẽ xảy ra khi bạn xáo trộn các trường trong Chỉ mục? Trả lời: Nó có thể tạo ra sự khác biệt lớn. More in a minute. 
* Điều gì sẽ xảy ra nếu có các trường bổ sung ở cuối? Trả lời: một chút tác hại; có thể rất nhiều cái tốt(ví dụ, 'bao trùm'). 
* Thừa thãi? Đúng vậy, Điều gì sẽ xảy ra khi có cả hai: Chỉ mục(a), Chỉ mục(a,b)? Trả lời : Thừa chi phí gì đó trong INSERTs; nó hiếm khi sử dụng cho SELECTs. 
* Tiền tố? Đúng vậy, Chỉ mục(last_name(5). first_name(5)) Trả lời: đừng bận tâm, nó hiếm khi giúp ích, và thường làm hại. (Chi tiết trong một chủ đề khác.) 

## Ví dụ thêm:
    
    
        INDEX(last, first)
        ... WHERE last = '...' -- good (even though `first` is unused)
        ... WHERE first = '...' -- index is useless
    
        INDEX(first, last), INDEX(last, first)
        ... WHERE first = '...' -- 1st index is used
        ... WHERE last = '...' -- 2nd index is used
        ... WHERE first = '...' AND last = '...' -- either could be used equally well
    
        INDEX(last, first)
        Both of these are handled by that one INDEX:
        ... WHERE last = '...'
        ... WHERE last = '...' AND first = '...'
    
        INDEX(last), INDEX(last, first)
        In light of the above example, don't bother including INDEX(last).
    

## Postlog

Refreshed -- Oct, 2012; more links -- Nov 2016
