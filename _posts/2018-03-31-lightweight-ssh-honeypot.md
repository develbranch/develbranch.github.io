---
layout: post
title: "Lightweight SSH Honeypot"
date: 2018-03-31 00:00:00.000000000 +07:00
type: post
published: true
status: publish
categories:
- Tutorials
tags:
- lsh
- honeypot
excerpt: "Xây dựng chương trình đánh dấu vị trí của các máy tính đang bruteforce dịch vụ SSH trên thế giới."
---
## Giới thiệu
Gần đây, tôi có dựng một VPS để tổ chức CTF. Tôi nhận ra rằng trong lúc setup iptable, tôi nhận được rất nhiều log kết nối tới cổng dịch vụ SSH (22) từ IP trên toàn thế giới. Chẳng có lý do gì để một máy chủ vừa dựng lên, đã có người biết đến nhanh như thế, ngoại trừ những người đó đang dùng các máy tính chuyên để quét cổng dịch vụ SSH (SSH bruteforce). 

Chúng ta có nhiều cách để phát hiện những kẻ khó chịu này:
 * kiểm tra các kết nối đang mở bằng `netstat -ntl`
 * kiểm tra log ssh (auth.log)
 * cài đặt suricata 
 * ...
 
Tôi quyết định xem thật sự đang có những tài khoản nào đăng nhập vào dịch vụ SSH của tôi. Tôi dựng một honeypot đơn giản: LSH - Lightweight SSH Honeypot. Có nhiều dự án mã nguồn mở khác tương tự thế này: [Kippo](https://github.com/desaster/kippo) hoặc [Kojoney](http://kojoney.sourceforge.net/). Chúng khá phức tạp và nhiều tính năng hay ho. Tôi cần một thứ đơn giản hơn. Tôi dùng Twisted với thư viện Conch SSH. Tôi sử dụng SSHFactory của Conch, chỉnh sửa một chút để tôi có thể ghi lại các thông tin đăng nhập (username và password). Tôi bắt đầu bật service ở cổng 2022, redirect kết nối từ cổng 22 về 2022 do cổng 22 là privileged port. Mọi tiến trình muốn sử dụng privileged port đều phải có quyền root, hoặc chúng ta cần thêm CAP_NET_BIND_SERVICE cho tiến trình lắng nghe. Khi có các thông tin này, tôi sử dụng mongodb để lưu lại. Thật tuyệt vời là có nhiều hãng cung cấp dịch vụ database online như MongoDB Atlas hay mLab. Việc tiếp theo là tôi cần một dịch vụ để có thể deploy toàn bộ code lên đó. Tôi không muốn dựng thêm một dịch vụ gì trên VPS giá $3/tháng của tôi nữa. Tôi chọn heroku app (platform as a service (PaaS)) do tính tiện dụng của nó. Quan trọng hơn, heroku cho phép sử dụng free nếu chúng ta chỉ có một app duy nhất. 

## Kết quả

Tôi bắt đầu vào ngày 26 và trong vòng 1 tuần thì có hơn 15000 lượt bruteforce, từ hơn 60 ip khác nhau và sử dụng trên 10000 cặp username password. Tôi có đánh dấu tất cả các ip đó trên bản đồ. Để xem chi tiết hơn, các bạn có thể vào đường link bên dưới:

https://develbranch-lsh.herokuapp.com/lsh


<iframe width="100%" height="600px" src="https://develbranch-lsh.herokuapp.com/lsh" frameborder="0" allowfullscreen></iframe>
