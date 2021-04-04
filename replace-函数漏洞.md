---

title: replace()函数漏洞

date: 2021-03-31 23:16:50

tags: 练习

categories: test56

---
今天buu刷题，写了一道关于replace()函数漏洞的题目，感觉比较少见，现在记录一下。  

# [BJDCTF2020]ZJCTF，不过如此  

打开题目，看到源码。  

[![](https://img.imgdb.cn/item/6066b9e08322e6675cb9b9f2.png)](https://img.imgdb.cn/item/6066b9e08322e6675cb9b9f2.png)  

可以看到，get方式上传text和file参数，首先要满足上传的text是一个文件，并且这个文件中有"I have a dream"字符串。  

	file_get_contents($text,'r')==="I have a dream"
	这里的r参数表示只读形式打开。  

满足第一个条件后接下来有一个正则匹配，过滤了flag字符，而我们上传的file参数就进入了include()函数里。  

很明显，这道题第一步要用include()函数来读取文件，不能读取flag，那就读取它给我们的提示：next.php。  

而第一个text传参很明显我们可以利用data://伪协议来写入内容，第二个include函数我们可以使用php://filter协议来读取文件。  

于是构造payload。  

	?text=data://text/plain,I have a dream&file=php://filter/convert.base64-encode/resource=next.php  
	这里data://伪协议中的plain后面跟上的就是写入的内容。  

进入下一个界面，得到了next.php源码。  

[![](https://img.imgdb.cn/item/6066bc868322e6675cbe53b3.png)](https://img.imgdb.cn/item/6066bc868322e6675cbe53b3.png)  

解码后得到源码。  

	<?php
	$id = $_GET['id'];
	$_SESSION['id'] = $id;
	
	function complex($re, $str) {
    return preg_replace(
        '/(' . $re . ')/ei',
        'strtolower("\\1")',
        $str
    );
	}


	foreach($_GET as $re => $str) {
    echo complex($re, $str). "\n";
	}

	function getFlag(){
	@eval($_GET['cmd']);
	}

这个才是重头，一眼看到里面的eval()函数，这是我们读取flag文件的关键，我们通过上传cmd参数来执行命令，而上面的正则匹配会返回最后匹配后的值，但是我们注意到，这个正则匹配和之前见过的有点不太一样。  

	/e 修正符使 preg_replace() 将 replacement 参数（第二个参数，字符串）当作 PHP 代码执行。

也就是说我们传入的字符串在匹配后，它的第二个参数会被当作php代码执行，一般情况下，第二参数是用于替换的字符串或字符串数组，而这里却是'strtolower("\\\1")'这样的字符串，有点不寻常，查一下什么意思。  

	反向引用

    对一个正则表达式模式或部分模式 两边添加圆括号 将导致相关 匹配存储到一个临时缓冲区 中，所捕获的每个子匹配都按照在正则表达式模式中从左到右出现的顺序存储。缓冲区编号从 1 开始，最多可存储 99 个捕获的子表达式。每个缓冲区都可以使用 '\n' 访问，其中 n 为一个标识特定缓冲区的一位或两位十进制数。

	\\1也就表示\1，\1表示取出正则匹配后的第一个子匹配中的第一项，strtolower()的作用是把大写字母转换为小写字母。

所以说这里第二个参数就是我们匹配后的第一个子匹配中的第一项。  

而正则匹配里的$re和$str是通过foreach($_GET as $re => $str)历遍上传的，也就是说如果我们上传index.php?hello=world那么$re=hello，$str=world（关于foreach()函数可以看之前的一篇博客详解）。 

所以说上面的正则匹配也就相当于eval('strtolower("\\1");')，这里的\\\1也就是我们上传的$str,所以最后就是eval('strtolower("$str");')。所以只要我们构造好$str就可以执行命令。  

而我们前面已经注意到了代码中的getFlag()函数，这个函数是我们最终要利用的函数。

那么我们就可以这样构造payload。  

	?\S*=${getFlag()}&cmd=system('cat /flag');

我们来解释一下这个payload。  

首先是传入的参数\S* ，在官方的漏洞利用中，给出的是.* 作为参数上传，但是在php中当非法字符为首字母时，只有点号会被替换成下划线，所以这里改为\S* 。  

其次就是我们为什么传入的的是${getFlag()}而不是直接传入getFlag()，这里有关php的可变变量。  

	在php中，双引号里面如果包含有变量，php解释器会进行解析；单引号中的变量不会被处理。  

而我们看到strtolower("\\1")中的\\\1是被双引号括起来的，这也就是说我们如果直接上传getFlag()，那么结果就是strtolower("getFalg()")，输出的就是字符串，而不会执行getFlag()函数所以我们要上传${getFlag()}，而$$a = ${$a}，这样在经过双引号的解析后剩下的getFlag()就仍是函数，可以被eval()执行。  

最后我们上传cmd作为getFlag()的参数，执行命令，从而得到flag。（注意是在next.php页面上传。）  

[![](https://img.imgdb.cn/item/6066cbdd8322e6675cd02d67.png)](https://img.imgdb.cn/item/6066cbdd8322e6675cd02d67.png)

# 小结

这道题考察的知识点挺多的，首先是php伪协议的运用，然后又考察了常见的replace()的漏洞的运用，同时又考察了可变变量，这个漏洞挺少见的，还是要多做积累，以防以后遇到不认识。  

附上关于这个漏洞的详细讲解。  

[深入研究preg_replace与代码执行](https://xz.aliyun.com/t/2557)
	