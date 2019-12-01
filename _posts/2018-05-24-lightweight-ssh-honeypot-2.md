---
layout: post
title: "Lightweight SSH Honeypot (part 2)"
date: 2018-05-24 00:00:00.000000000 +07:00
type: post
published: true
status: publish
categories:
- Tutorials
tags:
- lsh
- honeypot
excerpt: "Xây dựng chương trình đánh dấu vị trí của các máy tính đang bruteforce dịch vụ SSH trên thế giới. Phần này tôi sẽ viết chi tiết cách cụ thể tôi thực hiện dự án này."
---
## Giới thiệu
Trong bài viết trước [Lightweight SSH Honeypot](https://develbranch.com/tutorials/lightweight-ssh-honeypot.html), tôi đã mô tả về một dự án cho phép tôi xác định các máy tính đang bruteforce dịch vụ SSH trên thế giới. Tuy nhiên, có một số bạn đã inbox cho fanpage của tôi và mong muốn tôi viết rõ hơn, chi tiết hơn về các thứ mà tôi đã làm. Bài viết này tôi sẽ mô tả cụ thể phương pháp tôi đã làm cũng như đưa các đoạn chương trình tôi thực hiện để có thể hoạt động được.

## Phát hiện những IP đang thực hiện bruteforce
### Sử dụng phương pháp kiểm tra các kết nối đang mở bằng netstat
_netstat_ là một tiện ích của linux cho phép chúng ta liệt kê tất cả các thông tin liên quan tới các kết nối tới máy tính. Chúng ta có thể kiểm tra các kết nối đang bruteforce tới máy tính bằng lệnh:

```
netstat -nat |  grep -v LISTEN | grep ":22"
```
Lệnh này sẽ liệt kê tất cả các kết nối TCP, lọc ra các kết nối tới cổng 22 (cổng mặc định của dịch vụ SSH). Do các máy bruteforce thường chỉ kết nối tới cổng mặc định nên các thao tác này đã khá đủ.

### Sử dụng phương pháp kiểm tra log auth.log
_"auth.log"_ là log chứa các thông tin khi có người dùng đăng nhập vào hệ thống. Đăng nhập thông qua SSH cũng sẽ được lưu vào đây. Chúng ta có thể kiểm tra các thông tin này bằng một lệnh đơn giản sau:

```
cat /var/log/auth.log | grep "authentication failures" | grep "rhost" | more
```
Lệnh này sẽ liệt kê toàn bộ các lượt đăng nhập sai (hoặc sai tên hoặc sai mật khẩu) thông qua SSH, và có kèm cả ngày giờ đăng nhập.

## Xây dựng chương trình giả lập SSH server
Như đã trình bày trong bài viết trước, tôi có rất nhiều lựa chọn từ các dự án mã nguồn mở: [Kippo](https://github.com/desaster/kippo) hoặc [Kojoney](http://kojoney.sourceforge.net/). Trong dự án này, tôi đã tự xây dựng hệ thống cho riêng mình dựa trên Twisted với thư viện Conch SSH. Các bước tôi làm như sau:
 * Chỉnh sửa thư viện Conch của Twisted để tôi có thể ghi nhận các thông tin đăng nhập. Tôi sẽ đi chi tiết mục này do đây là phần khó nhất. 2 phần còn lại rất đơn giản, tôi chỉ sẽ không đi vào chi tiết. Khi có dữ liệu, việc hiển thị chỉ là thứ yếu.
 * Lưu lại các thông tin đăng nhập vào database để có thể tra cứu về sau.
 * Xây dựng một web đơn giản cho phép hiển thị các thông tin đã ghi nhận được và đánh dấu trên bản đồ.
 
### Chỉnh sửa Conch
Thư viện Conch khá hoàn thiện, cho phép chúng ta xây dựng SSH server với đầy đủ các mô tả của giao thức SSH2. Tuy nhiên, chúng ta chỉ cần ghi nhận địa chỉ của máy thực hiện bruteforce, username và password là đủ. Chúng ta chưa cần phải thực hiện giả lập các thao tác sau khi hacker đăng nhập được vào hệ thống. Do đó, chúng ta sẽ **luôn luôn** trả về kết quả _đăng nhập không thành công_. Để làm điều này, tôi sẽ patch thư hiện Conch của Twisted bằng một lớp mới (`AuthServerWithPeer`) của tôi.

```python
class AuthServerWithPeer(userauth.SSHUserAuthServer):
    def auth_password(self, packet):
        password = getNS(packet[1:])[0] # parse nhận được trong quá trình authentication

        # Tạo một Object chứa các thông tin đăng nhập, object này phải phù hợp với interface
        # của hàm auth_password(). Hoàn toàn không hề có tài liệu mô tả chính xác cho interface này.
        # credentials.UsernamePassword là một implementation của interface nên tôi kế thừa class này
        # Để biết được interface này, tôi đã đọc mã nguồn của Conch.
        # Đây là ưu điểm của các ứng dụng mã nguồn mở do chúng ta luôn có source code của toàn bộ chương trình
        c = UsernamePasswordPeer(self.user, password, self.transport.getPeer()) 
        return self.portal.login(c, None, interfaces.IConchUser).addErrback(
            self._ebPassword)

# khởi tạo một instance của SSHFactory			
factory = SSHFactory()
# patch module SSH Authentication mặc định thành AuthServerWithPeer
factory.services[b'ssh-userauth'] = AuthServerWithPeer
```

`UsernamePasswordPeer` sẽ thực hiện ghi log khi có bất kì yêu cầu đăng nhập nào gửi tới server. Thông qua đọc mã nguồn, tôi biết được phương thức `requestAvatarId` sẽ nhận thông tin đăng nhập. Tôi đặt code ghi log vào đó.

Toàn bộ code của quá trình như sau:
```python
@implementer(ICredentialsChecker)
class UsernamePasswordLogger(object):
    """
    A simple credentials logger.
    """

    credentialInterfaces = (credentials.IUsernamePassword,
                            credentials.IUsernameHashedPassword)

    def requestAvatarId(self, user_cred):
        self.write_log(user_cred)
        return defer.fail(error.LoginFailed())

    @staticmethod
    def write_log(cred):
        # thực hiện ghi log ở đây
        pass


class UsernamePasswordPeer(credentials.UsernamePassword):
    def __init__(self, username, password, peer):
        credentials.UsernamePassword.__init__(self, username, password)
        self.peer = peer


class AuthServerWithPeer(userauth.SSHUserAuthServer):
    def auth_password(self, packet):
        password = getNS(packet[1:])[0]
        c = UsernamePasswordPeer(self.user, password, self.transport.getPeer())
        return self.portal.login(c, None, interfaces.IConchUser).addErrback(
            self._ebPassword)


class SimpleRealm(object):
    def requestAvatar(self, avatarId, mind, *i):
        user = ConchUser()
        user.channelLookup['session'] = SSHChannel

        def nothing():
            pass

        return interfaces.IConchUser, user, nothing


factory = SSHFactory()
# patch module SSH Authentication mặc định thành AuthServerWithPeer
factory.services[b'ssh-userauth'] = AuthServerWithPeer
# sửa thông tin version của SSH cho giống với 1 server thật
factory.protocol.ourVersionString = 'SSH-2.0-OpenSSH_7.2p2 Ubuntu-4ubuntu2.4'
factory.privateKeys = {'ssh-rsa': privateKey}
factory.publicKeys = {'ssh-rsa': publicKey}
# thiết lập moduli file để khởi tạo cho server, nếu không thiết lập thì sẽ dùng các giá trị mặc định
if MODULI_FILE:
    factory.primes = primes.parseModuliFile(MODULI_FILE)
factory.portal = Portal(SimpleRealm())
factory.portal.registerChecker(UsernamePasswordLogger())
reactor.listenTCP(port=2022, factory=factory)
reactor.run()
```

Để tránh chương trình phải chạy với quyền root (do 22 là một cổng privileged port), tôi sẽ lắng nghe ở cổng 2022 sau đó dùng luật iptables để redirect kết nối từ cổng 22 về 2022.

```
 iptables -A PREROUTING -t nat  -p tcp -m tcp --dport 22 -j REDIRECT --to-ports 2022
```

Một cách phức tạp khác là chúng ta thêm CAP_NET_BIND_SERVICE cho tiến trình lắng nghe. 


### Lưu các thông tin đăng nhập
Các thông tin đăng nhập có thể kể đến là :
```
login_info = { 
                'host': cred.peer.address.host,
                'username': cred.username,
                'password': cred.password,
                'last_update': int(time.time())
            }
```
Trong đó: 
 * _host_ chứa địa chỉ của máy tính thực hiện bruteforce.
 * _username_ username đăng nhập.
 * _password_ password đăng nhập.
 * _last_update_ Thời gian đăng nhập.
 
Khi có các thông tin này, các bạn có thể sử dụng bất cứ một cơ sở dữ liệu nào để lưu, tôi sử dụng mongodb để lưu lại.

### Xây dựng giao hiện hiển thị
Quá trình xây dựng giao hiện hiển thị khá đơn giản, gồm 2 phần là front-end và back-end
 * **front-end**: Chúng ta cần hiển thị lên một bản đồ. Tôi dùng Google Maps API. Code sử dụng rất đơn giản và có sẵn trên mạng. Ngoài ra chúng ta cần một số bảng thống kê một số thông tin như: máy chủ quét nhiều nhất, cặp username - password được sử dụng nhiều nhất....
 * **back-end**: Chúng ta cần một webservice để lấy các thông tin và hiển thị lên giao diện. Do tôi sử dụng mongodb nên mọi thứ khá đơn giản khi lấy về danh sách các host đã bruteforce. Tuy nhiên cần lưu ý rằng để thống kê với mongodb thì chúng ta phải tối ưu. Các bạn có thể thấy tuy số lượng bản ghi rất nhiều nhưng quá trình thống kê kết quả của tôi là realtime và tốc độ khá nhanh. Tôi thực hiện điều này bằng cách sử dụng tính năng `aggregate` của mongodb. Aggregate của mongodb cho phép chúng ta xây dựng các pipeline để tính các giá trị thống kê. Một số bạn chưa biết tính năng này sẽ query dữ liệu về dưới dạng 1 list hoặc dùng cursor để walk và đếm. Cách đó hoàn toàn không đúng và sẽ làm treo toàn bộ mọi thứ. 

## Kết quả

Tôi bắt đầu vào ngày 26 tháng 3 và tới nay đã có hơn 495000 lượt bruteforce, từ hơn 1867 ip khác nhau và sử dụng trên 90000 cặp username password. Tôi có đánh dấu tất cả các ip đó trên bản đồ. Để xem chi tiết hơn, các bạn có thể vào đường link bên dưới:

https://develbranch-lsh.herokuapp.com/lsh


<iframe width="100%" height="600px" src="https://develbranch-lsh.herokuapp.com/lsh" frameborder="0" allowfullscreen></iframe>
