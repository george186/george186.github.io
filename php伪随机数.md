---

title: php伪随机数

date: 2021-05-05 22:38:32

tags: 练习

categories: test79

---
今天做了一道php随机数的题目。一个新的知识点记录一下。  

打开题目，发现要我们抽奖，要填对20个字符才能得到flag，但是他给了我们一部分flag。  

靠猜和一个个试显然是不可能的，那么我们看一下有无源码，f12发现有一个/check.php网页。  

[![](https://pic.imgdb.cn/item/609b3e28d1a9ae528ff07ad2.png)](https://pic.imgdb.cn/item/609b3e28d1a9ae528ff07ad2.png)  

打开这个网页得到源码  

	qmC46RE62n
	<?php
	#这不是抽奖程序的源代码！不许看！
	header("Content-Type: text/html;charset=utf-8");
	session_start();
	if(!isset($_SESSION['seed'])){
	$_SESSION['seed']=rand(0,999999999);
	}
	
	mt_srand($_SESSION['seed']);
	$str_long1 = "abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";
	$str='';
	$len1=20;
	for ( $i = 0; $i < $len1; $i++ ){
    $str.=substr($str_long1, mt_rand(0, strlen($str_long1) - 1), 1);       
	}
	$str_show = substr($str, 0, 10);
	echo "<p id='p1'>".$str_show."</p>";
	
	
	if(isset($_POST['num'])){
    if($_POST['num']===$str){x
        echo "<p id=flag>抽奖，就是那么枯燥且无味，给你flag{xxxxxxxxx}</p>";
    }
    else{
        echo "<p id=flag>没抽中哦，再试试吧</p>";
    }
	}
	show_source("check.php");  

可以看到显示随机生成了一个种子seed，然后用mt_srand()函数和这个种子随机生成字符串，现在给我们展现的是前生成的十个字符。  

总所周知，伪随机并非真的随机，而在mt_srand()函数中，当种子不变时，生成的随机数是一样的，那么我们就有可能通过他生成的随机数来反推出种子，有了种子我们就可以得到完整的字符串。  

这里使用php _mt _seed这个工具来反推出种子  

	php_mt_rand 工具只能用于爆破mt_rand()函数产生的随机数的种子值， 无论是否显式调用mt_srand()函数播种，但不能用于mt_rand(1,1000)这种指定范围的和rand函数的爆破  

首先我们先用脚本将前十个我们已经知道的字符串转换成对应的随机数。（大佬的脚本，tql）  

	str1='abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ'
	str2='qmC46RE62n'
	str3 = str1[::-1]
	length = len(str2)
	res=''
	for i in range(len(str2)):  
    for j in range(len(str1)):
        if str2[i] == str1[j]:
            res+=str(j)+' '+str(j)+' '+'0'+' '+str(len(str1)-1)+' '
            break
	print(res)  

[![](https://pic.imgdb.cn/item/609b3fd7d1a9ae528ffd5525.png)](https://pic.imgdb.cn/item/609b3fd7d1a9ae528ffd5525.png)

得到随机数  

	16 16 0 61 12 12 0 61 38 38 0 61 30 30 0 61 32 32 0 61 53 53 0 61 40 40 0 61 32 32 0 61 28 28 0 61 13 13 0 61  

然后将随机数放入php _mt _seed来反推种子。  

[![](https://pic.imgdb.cn/item/609b40e5d1a9ae528f06a3aa.png)](https://pic.imgdb.cn/item/609b40e5d1a9ae528f06a3aa.png)  

得到种子为655093557，php版本为7.1.0+(这个版本很重要，不同的php版本的mt_srand()函数使用相同的种子得到的随机数不同)  

接下来我们只需要在源码上稍作修改，让他显示全部字符串并将我们得到的种子输入mt_srand()函数就能得到完整字符串。  

	
	<?php
	mt_srand(655093557);
	$str_long1 = "abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";
	$str='';
	$len1=20;
	for ( $i = 0; $i < $len1; $i++ ){
    $str.=substr($str_long1, mt_rand(0, strlen($str_long1) - 1), 1);       
	}
	echo "<p id='p1'>".$str."</p>";


[![](https://pic.imgdb.cn/item/609b41d9d1a9ae528f0e569d.png)](https://pic.imgdb.cn/item/609b41d9d1a9ae528f0e569d.png)  

得到完整字符串qmC46RE62nCuh0IzcrY2，输入就得到flag。  

# 小结  

这道题目考的就是php的伪随机数的生成函数mt_srand()以及它的一些规律，当种子不变时，生成的随机数不变。同时知道了一个专门针对这个函数的脚本工具来反推种子。题目不难，重在对新工具和知识的掌握。  

附上链接：  

[php伪随机数](https://blog.csdn.net/zss192/article/details/104327432)