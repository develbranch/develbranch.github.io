---
layout: post
title: "Làm thế nào để tìm các URL trong các biến thể của Win32-Sality (How to extract URLs of Win32-Sality variants)"
date: 2017-10-30 00:00:00.000000000 +07:00
type: post
published: true
status: publish
categories:
- Tutorials
tags:
- Sality
- keystone
- unicorn
- capstone
- pefile
excerpt: "Tôi thử tìm hiểu và thực hiện nhiều phương pháp khác nhau để có thể lấy được các URL chứa trong Win32-Sality. Các URLs này thường tồn tại trong thời gian rất ngắn, nhưng điều quan trọng là nó chứa các binary mới của Sality. Tôi cần một phương pháp tự động, hiệu quả và có thể sử dụng tài nguyên hạn chế để thực hiện. Các phương pháp sử dụng môi trường ảo hay sandbox rất hay và khả thi nhưng trong bài viết này, tôi muốn cung cấp một góc nhìn mới, một phương pháp tiếp cận khác để có thể tìm và phát hiện các URL. Tôi sử dụng emulator. Sau đây, tôi sẽ mô tả cách làm của tôi để có thể tìm các URL điều khiển trong các biến thể khác nhau của virus lây file Win32-Sality"
---


Tôi là một lập trình viên. Trong thời gian rảnh, tôi có nghiên cứu một vài vấn đề liên quan tới dịch ngược và virus máy tính. Tôi phân tích biến thể đầu tiên của Win32-Sality vào năm 2011. Từ đó đến nay, dù Win32-Sality vẫn tiếp tục tồn tại và lây lan nhưng những thay đổi của dòng virus này không nhiều. Trước đây, tôi đã thử tìm hiểu và thực hiện nhiều phương pháp khác nhau để có thể lấy được các URL chứa trong Win32-Sality. Các URLs này thường tồn tại trong thời gian rất ngắn. Tôi cần một phương pháp tự động, hiệu quả và có thể sử dụng tài nguyên hạn chế để thực hiện. Các phương pháp sử dụng môi trường ảo hay sandbox rất hay và khả thi. Trong bài viết này, tôi muốn cung cấp một góc nhìn mới, một phương pháp tiếp cận khác để có thể tìm và phát hiện các URL. Tôi sử dụng Emulator.

Có thể nói, tôi là một fan của bộ ba công cụ: Reversing Trilogy: [Capstone](http://capstone-engine.org), [Unicorn](http://unicorn-engine.org) & [Keystone](http://keystone-engine.org). Đây là nhưng công cụ rất cơ bản để có thể thực hiện nhiều ý tưởng khác nhau của tôi trong quá trình phân tích hay dịch ngược. Nói vòng vo một chút, tôi biết ngôn ngữ Ruby của người Nhật. Những tài liệu, những thư viện đầu tiên đều là của người Nhật viết để hỗ trợ cho ngôn ngữ này. Tôi muốn tinh thần đó cũng có trong các sản phẩm của người Việt. Có thể đâu đó trong blog của tôi, tôi đã nhắc tới bộ ba Reversing Trilogy. Qua bài viết này, tôi muốn được một lần nữa giới thiệu Reversing Trilogy, một công cụ của người Việt nhưng chưa được thật sự nhiều người Việt biết tới.

## Đôi nét về virus lây file

Trước tiên, tôi muốn các bạn có một hình dung cụ thể hơn về một virus lây file (file infector). Chúng ta tạm gọi một file thực thi chưa bị tấn công là **host file** (lưu ý: đây không phải là file `/etc/hosts` hay `C:\Windows\System32\drivers\etc\hosts`). **host file** (hay gọi tắt là *host*) là một file chương trình bình thường: Các file của hệ điều hành mà thực thi được (notepad.exe, calc.exe), các chương trình được cài đặt trong máy tính, các file cài đặt có phần mở rộng là exe hay các file trò chơi... Các file này có thể có phần mở rộng là .exe, .dll, .sys. Máy tính không thể hoạt động nếu không có một file thực thi nào. Một virus lây file khi vào máy tính và được thực thi, sẽ tìm kiếm các host và sửa file này: Chúng thêm vào host các đoạn mã của virus. Các đoạn mã này sẽ chạy virus lên khi host được kích hoạt, và sau khi virus đã chạy thành công, quyền thực thi sẽ trả về cho host để host tiếp tục thực hiện nhiệm vụ. Người dùng sẽ hoàn toàn không phát hiện ra đã có virus hoạt động. Host đã trở thành file bị lây (**infected file**). Khi người dùng copy những file bị lây này qua một máy tính khác (qua USB, qua email...), Virus sẽ tiếp tục lây lan. Đó là phương thức lây lan chủ yếu của virus lây file. Nếu trong một máy có nhiều file bị lây cùng thực thi, virus sẽ sử dụng các phương pháp như tạo mutex, event, ... để virus chỉ thực thi một lần duy nhất.

Các chương trình Antivirus có nhiệm vụ bảo vệ máy tính có thể phát hiện ra các virus lây file này, gỡ bỏ đoạn mã độc hại và khôi phục lại file như trước khi bị lây. Tuy nhiên, vì không phải virus lây file nào cũng giống nhau, nên phương pháp gỡ bỏ sẽ rất đặc trưng cho từng virus. Người phân tích sẽ phải đọc và hiểu cách thức lây của virus để tìm phương pháp khôi phục file thích hợp. Không hề có một công thức chung cho việc này. Do vậy, Antivirus có thể dùng nhiều phương pháp **phát hiện** khác nhau dành cho virus lây file ( heuristics, emulator, phân tích theo hành vi - behavior analysis, ...) nhưng tất cả chỉ dừng ở mức độ **"phát hiện"**. Muốn gỡ bỏ hoàn toàn, cần có sự phân tích cụ thể của người phân tích để đảm bảo chính xác. Phương pháp phát hiện và gỡ bỏ hiệu quả nhất cho dòng virus này vẫn là sử dụng chữ kí (signature): tức là một đoạn mã virus đặc trưng. Tuy nhiên, theo thời gian, các phương pháp sử dụng chữ kí sẽ khiến cho Antivirus ngày càng cồng kềnh và kém hiệu quả. 

Để cải thiện khả năng "ẩn thân", các nhà phát triển mã độc cũng nghĩ ra công cụ cho riêng mình: virus đa hình ([polymorphic virus](https://en.wikipedia.org/wiki/Polymorphic_code#Malicious_code)). Loại virus này có đặc điểm là sẽ mã hóa hầu hết các đoạn mã của virus, chỉ giải mã các đoạn mã đó trên bộ nhớ của chương trình khi thực thi. Do toàn bộ virus đã mã hóa, nên các Antivirus sẽ lựa chọn đoạn code giải mã (những phần code có thể coi là cố định) để làm signature. Các đoạn code giải mã này thường nhỏ. Tất nhiên, các nhà phát triển mã độc không dừng lại ở đây, họ tiếp tục cải tiến và phát triển virus siêu đa hình ([Metamorphic code](https://en.wikipedia.org/wiki/Metamorphic_code)). Dòng này có đặc điểm là nó tạo ra các đoạn code giải mã khác nhau ở mỗi infected file. Cùng một biến thể, nhưng nếu lây 2 host, sẽ cho ra 2 đoạn code giải mã khác nhau. Có nhiều phương pháp khác nhau để làm việc này, nhưng một trong các phương pháp phổ biến là đưa các đoạn code "rác" vào code giải mã, để làm rối Antivirus. Hiện nay có khá nhiều project cho để biến đổi giống như thế này, bạn có thể thử xem qua: [metame](https://github.com/a0rtega/metame) hay [pymetamorph](https://github.com/JuanJMarques/pymetamorph).

## Phân tích Win32-Sality và phương pháp tìm các URL độc hại
Phân tích kĩ Win32-Sality, có thể thấy rằng đây là Metamorphic virus. Điều đó có nghĩa là, với cùng một mẫu virus và cùng một file bị lây, nhưng mỗi lần lây sẽ tạo ra các đoạn mã hoàn toàn khác nhau. Cần phải giải mã được toàn bộ virus để tìm được các URL được lưu trong virus. Để vượt qua các đoạn mã đa hình, tôi sử dụng emulator. Tôi dùng Unicorn-engine để chạy giả lập lại toàn bộ quá trình chạy của virus. Vì thế tôi không cần đọc hiểu thuật toán giải mã của chương trình, nhưng vẫn có thể giải mã chính xác. Win32-Sality có "đem" theo một DLL, đó là engine lây file. Tiếc rằng DLL này lại được pack bằng UPX. Đến bước này tôi sẽ phải dùng emulator một lần nữa để unpack và tìm các URL bên trong đó. Trong script, tôi có sử dụng [pefile](https://github.com/erocarrera/pefile) để parse nội dung của một file thực thi.

Các bước thực hiện như sau:
 * sử dụng pefile để parse file PE.
 * sử dụng unicorn-engine làm emulator để thực thi các đoạn mã giải mã
   * dùng `mem_map` của unicorn để copy toàn bộ file PE vào memory của emulator
   * tạo một vùng nhớ (khoảng 0x4000 - con số này tôi tự chọn) làm stack
   * thiết lập thanh ghi stack, do chương trình cần sử dụng stack trong khi thực thi. Các thanh ghi khác chỉ cần để mặc định
   * cài đặt một hàm callback để trace các lệnh đã được thực thi
   * bắt đầu chạy emulator từ entrypoint của chương trình 
   
Trong quá trình phân tích Win32-Sality, tôi nhận thấy: sau khi thực hiện lệnh `retn` trong quá trình giải mã, EIP sẽ trỏ về đoạn code thật. Tuy nhiên, lệnh `retn` có thể xuất hiện nhiều chỗ khác nhau trong chương trình. Để tránh phát hiện nhầm, tôi có lấy 1 đoạn làm chữ kí cho Sality (hàm `check_sality`) để biết được đã giải mã xong hay chưa.

Sau khi giải mã thành công toàn bộ đoạn code gốc của sality, chúng ta lưu ý: DLL chứa engine lây file của sality lại được pack bằng UPX:
 * Sử dụng pefile để parse file 
 * sử dụng unicorn-engine làm emulator để thực thi các đoạn mã giải mã của UPX (UPX stub code): bước này tôi làm tương tự như đã trình bày ở trên
 * Fix Import Address Table (IAT) cho PE
 
Ở đây, chúng ta nhận thấy có IAT: UPX stub code cần gọi các hàm Windows API trong quá trình chạy của mình để: Build IAT cho file (`LoadLibraryA`, `GetProcAddress` của thư hiện kernel32.dll) hay gọi `VirtualProtect` để thiết lập thuộc tính cho các section. Rõ ràng, trong emulator hoàn toàn không có hệ điều hành nên tôi sẽ không có `kernel32.dll`. Tôi cũng không muốn emulate cả thư viện `kernel32.dll`. Tôi tạo một bảng fake IAT và dùng các ngắt (interrupt) để "bắt chước" lời gọi API. Các bước như sau:
   * sử dụng keystone-engine tạo một đoạn code mô phỏng một API, gồm 2 lệnh: `mov eax, number` và `int 0xff`
   * `number` là một số bất kì, đại diện cho API
   * ngắt `0xff` là một ngắt không ai dùng.
   * sử dụng tính năng `UC_HOOK_INTR` để hook interrupt
   
Khi một API được thực thi, EAX sẽ chứa số hiệu tương ứng với hàm cần gọi. Emulator sẽ thực thi ngắt, gọi tới `hook_intr`. Bên trong hàm hook, thanh ghi EAX sẽ chứa kết quả trả về của hàm API. Bằng phương pháp này, chúng ta có thể mô phỏng bất cứ API nào.

Ngoài ra, UPX stub code không phải code đa hình, chúng ta có thể biết rõ: bắt đầu của stub là lệnh `pushad` và kết thúc là `popad`. Tôi sử dụng capstone-engine để disassemble đoạn code, tìm lệnh `popad` và lấy đó làm địa chỉ kết thúc quá trình emulate.

## Kết quả
Sau khi thực hiện tất cả các bước trên, chúng ta có được toàn bộ code của sality, không mã hóa. Chúng ta hoàn toàn toàn có thể dump nó ra file để phân tích sâu hơn. Đơn giản hơn, tôi dùng một đoạn biểu thức chính quy (regular expression) để tìm các URL. 

Kết quả thực hiện với một mẫu tôi kiếm được:

![Sality Extractor]( {{site.url}}/assets/2017/10/sality_extractor.PNG) 

## Script
Đây là mã nguồn của toàn bộ script tôi đã sử dụng:

{% gist 41deada8a936a1877a6c6c757ce73800 %}
