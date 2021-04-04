---

title: 有关flack框架的ssrf

date: 2021-04-02 16:02:55

tags: 练习

categories: test57

---
今天做到了一道和以往都不同的ssrf题目，现在记录下来。扩展一下自己的见识。  

# [De1CTF 2019]SSRF Me   
题目说的很明白，就是ssrf题目，那么打开题目，发先代码好像有点不对劲。  

[![](https://img.imgdb.cn/item/606946698322e6675c200e1c.png)](https://img.imgdb.cn/item/606946698322e6675c200e1c.png)  

代码有点乱，需要整理一下，整理后如下。  

	#! /usr/bin/env python
	#encoding=utf-8
	from flask import Flask
	from flask import request
	import socket
	import hashlib
	import urllib
	import sys
	import os
	import json
	
	reload(sys)
	sys.setdefaultencoding('latin1')
	
	app = Flask(__name__)
	
	secert_key = os.urandom(16)
	
	class Task:
    def __init__(self, action, param, sign, ip):
        self.action = action
        self.param = param
        self.sign = sign
        self.sandbox = md5(ip)
        if(not os.path.exists(self.sandbox)):          #SandBox For Remote_Addr
            os.mkdir(self.sandbox)

    def Exec(self):
        result = {}
        result['code'] = 500
        if (self.checkSign()):
            if "scan" in self.action:
                tmpfile = open("./%s/result.txt" % self.sandbox, 'w')
                resp = scan(self.param)
                if (resp == "Connection Timeout"):
                    result['data'] = resp
                else:
                    print(resp)
                    tmpfile.write(resp)
                    tmpfile.close()
                result['code'] = 200
            if "read" in self.action:
                f = open("./%s/result.txt" % self.sandbox, 'r')
                result['code'] = 200
                result['data'] = f.read()
            if result['code'] == 500:
                result['data'] = "Action Error"
        else:
            result['code'] = 500
            result['msg'] = "Sign Error"
        return result

    def checkSign(self):
        if (getSign(self.action, self.param) == self.sign):
            return True
        else:
            return False

	#generate Sign For Action Scan.
	@app.route("/geneSign", methods=['GET', 'POST'])
	def geneSign():
    param = urllib.unquote(request.args.get("param", ""))
    action = "scan"
    return getSign(action, param)

	@app.route('/De1ta',methods=['GET','POST'])
	def challenge():
    action = urllib.unquote(request.cookies.get("action"))
    param = urllib.unquote(request.args.get("param", ""))
    sign = urllib.unquote(request.cookies.get("sign"))
    ip = request.remote_addr
    if(waf(param)):
        return "No Hacker!!!!"
    task = Task(action, param, sign, ip)
    return json.dumps(task.Exec())
	@app.route('/')
	def index():
    return open("code.txt","r").read()

	def scan(param):
    socket.setdefaulttimeout(1)
    try:
        return urllib.urlopen(param).read()[:50]
    except:
        return "Connection Timeout"

	def getSign(action, param):
    return hashlib.md5(secert_key + param + action).hexdigest()

	def md5(content):
    return hashlib.md5(content).hexdigest()

	def waf(param):
    check=param.strip().lower()
    if check.startswith("gopher") or check.startswith("file"):
        return True
    else:
        return False

	if __name__ == '__main__':
    app.debug = False
    app.run(host='0.0.0.0',port=80)  

好家伙，第一次看到这么长的代码，而且很明显这个不是php代码，在开头也说明了from flask import Flask，说明这个是py的flask框架，这个框架我们之前遇到过，但是考察的一般都是ssti注入，而这次它来考察我们ssrf，也就是说我们第一步就是要进行代码审计。  

其实flask框架的代码与之前我们认识的代码原理上没有太大的差距（就好比学会了c语言，别的语言也能触类旁通一样），只要学过别的语言，对于这一大段代码还是能看懂七七八八的。而这里最大的不同就是多了一个路由的概念，就像下面的这些：  

	
	@app.route("/geneSign", methods=['GET', 'POST'])
	def geneSign():
    param = urllib.unquote(request.args.get("param", ""))
    action = "scan"
    return getSign(action, param)

	@app.route('/De1ta',methods=['GET','POST'])
	def challenge():
    action = urllib.unquote(request.cookies.get("action"))
    param = urllib.unquote(request.args.get("param", ""))
    sign = urllib.unquote(request.cookies.get("sign"))
    ip = request.remote_addr
    if(waf(param)):
        return "No Hacker!!!!"
    task = Task(action, param, sign, ip)
    return json.dumps(task.Exec())
	@app.route('/')
	def index():
    return open("code.txt","r").read()  

可以看到有@app.route开头的字符，这个就是路由的标志，关于路由有以下作用：   

	在这个小的应用里，值得注意的是路由的使用，即@app.route()。这个路由便是之前说的装饰器，也就是说flask通过装饰器来识别用户需要访问的网址路径，并在对应的网址路径里做对应的应用。  

用我的理解，这个就相当于编程里的一个自定义的一个函数，用户通过get或者post方式访问不同的网址，也就是**/geneSign**或者**/De1ta**然后就会触发相应的函数的操作，所以这道题目也就是让我们通过这些给出的网址路径来进行ssrf攻击，那么我们首先应该分析的就是这三个路由以及里面的操作了。  

对于这道题我们只需分析前两个路由即可，第三个就是正常的路由。

首先看到第一个路由/geneSign

	@app.route("/geneSign", methods=['GET', 'POST'])
	def geneSign():
    param = urllib.unquote(request.args.get("param", ""))
    action = "scan"
    return getSign(action, param)  

看到网址路径是/geneSign，方式是get，上传的变量是param，而action变量的值被固定了，是“scan”，最后返回经过函数getsign(action, param)操作后的值，那么我们跟进一下函数getsign()。  

	def getSign(action, param):
	return hashlib.md5(secert_key + param + action).hexdigest()  

可以看到函数getsign()会接受action和param的值，然后进行一个md5加密，最后返回加密后的值，其中md5加密方式是**md5(secert_key + param + action)**其中secert _key的值我们是不知道的。  

接下来看第二个路由/De1ta。  

	@app.route('/De1ta',methods=['GET','POST'])
	def challenge():
	action = urllib.unquote(request.cookies.get("action"))
	param = urllib.unquote(request.args.get("param", ""))
	sign = urllib.unquote(request.cookies.get("sign"))
	ip = request.remote_addr
	if(waf(param)):
    return "No Hacker!!!!"
	task = Task(action, param, sign, ip)
	return json.dumps(task.Exec())  

和上个相同，网址路径是/De1ta上传方式是get，上传的变量是action，param，sign，**这里注意action和sign是通过cookies来上传的，和param不同**。接下来会对param的参数进行一个waf()的检测，如果通过了检测就可以将这四个值传进Task类，然后将其赋值给变量task，而task会进行Exec()函数操作，最后返回Exec()函数操作后的值。  

那么我们先看一下waf()函数过滤了什么。  
	
	def waf(param):
	check=param.strip().lower()
	if check.startswith("gopher") or check.startswith("file"):
    return True
	else:
    return False  

很明显，过滤了协议的开头，使我们无法通过伪协议来读取文件。  

接下来我们看一下Task类是什么。  

	class Task:
	def __init__(self, action, param, sign, ip):
    self.action = action
    self.param = param
    self.sign = sign
    self.sandbox = md5(ip)
    if(not os.path.exists(self.sandbox)):          #SandBox For Remote_Addr
        os.mkdir(self.sandbox)  

就是正常的赋值，ip经过了一个md5的加密，但是对于我们这道题ip没有什么影响。  

然后我们来看最重点的Exec()函数。  

	def Exec(self):
    result = {}
    result['code'] = 500
    if (self.checkSign()):
        if "scan" in self.action:
            tmpfile = open("./%s/result.txt" % self.sandbox, 'w')
            resp = scan(self.param)
            if (resp == "Connection Timeout"):
                result['data'] = resp
            else:
                print(resp)
                tmpfile.write(resp)
                tmpfile.close()
            result['code'] = 200
        if "read" in self.action:
            f = open("./%s/result.txt" % self.sandbox, 'r')
            result['code'] = 200
            result['data'] = f.read()
        if result['code'] == 500:
            result['data'] = "Action Error"
    else:
        result['code'] = 500
        result['msg'] = "Sign Error"
    return result

可以看到，这个函数会先进行一个checkSign()函数的检验，如果通过才能进行下面的if语句，我们跟进一下这个函数。  

	def checkSign(self):
    if (getSign(self.action, self.param) == self.sign):
        return True
    else:
        return False  

可以看到这个函数会将action和param的值传入getsign()函数，然后用getsgin()函数返回的值和sign的值进行对比，如果相同就返回真，反之就返回假。对于getsign()函数我们之前已经分析过了，他就是一个md5加密：md5(secert_key + param + action)但是我们并不知道sign的值，我们先跳过这一点，接着往下看：  

可以看到如果我们通过检测，接下来会有两个if语句判断：  

	if "scan" in self.action:
        tmpfile = open("./%s/result.txt" % self.sandbox, 'w')
        resp = scan(self.param)
        if (resp == "Connection Timeout"):
            result['data'] = resp
        else:
            print(resp)
            tmpfile.write(resp)
            tmpfile.close()
        result['code'] = 200
    if "read" in self.action:
        f = open("./%s/result.txt" % self.sandbox, 'r')
        result['code'] = 200
        result['data'] = f.read()  

如果action参数里有scan字符，那么他会执行下面的操作：  

	tmpfile = open("./%s/result.txt" % self.sandbox, 'w')
        resp = scan(self.param)  
	.....
	print(resp)
        tmpfile.write(resp)
        tmpfile.close()

不难看出，这个就是将param的参数内容写入文件，而下面的if语句大体相同：  

	if "read" in self.action:
        f = open("./%s/result.txt" % self.sandbox, 'r')
        result['code'] = 200
        result['data'] = f.read()  

如果action参数里有read字符，那么他就会读取文件的内容。  

到这里就很明白了，题目想让我们用的就是这个write和read，由于题目刚开始就给出了提示flag再./flag.txt里，那么我们只需要让action里有write和read字符，然后利用param上传flag.txt，这样就能将flag.txt内容写入并且读出。  

接下来要解决的就是如何通过Exec()函数的第一个检测了，我们已经知道要使md5(secert_key + param + action)==sign，而param我们要使其为flag.txt,action要为write+read。而这个sign的值是我们通过get方式上传的，所以上传的sign的值应该是这样的： 

	md5(secert_keyflag.txtscanread)//加号可省略  

那么哪里还用到了getsign()函数呢？没错，就是第一个路由：  

	@app.route("/geneSign", methods=['GET', 'POST'])
	def geneSign():
	param = urllib.unquote(request.args.get("param", ""))
	action = "scan"
	return getSign(action, param)  

可以看到，这个同样是将action和param进行getsign的操作。**但是它可以返回加密后的结果，**这不就能解决sign值的问题了吗？不同的就是这里action的值是固定的scan，但是param可以控制，那么我们可以拼接一下让action的值变成scanread:  

我们这样上传：/geneSign?param=flag.txtread（注意这里用到的是/geneSign路由），这样由于md5()函数中+号可以省略，最后的函数就是

	md5(secert_key+flag.txtread+scan)
	也就是md5(secert_keyflag.txtreadscan)

这样就返回了我们需要的sign的值：  

[![](https://img.imgdb.cn/item/6069578e8322e6675c34bad9.png)](https://img.imgdb.cn/item/6069578e8322e6675c34bad9.png)  

接下来就是通过/De1ta路由写入和读出flag.txt了，我们抓包来构造：  

[![](https://img.imgdb.cn/item/606957f98322e6675c3514bb.png)](https://img.imgdb.cn/item/606957f98322e6675c3514bb.png)  

可以看到，我们通过/De1ta路由上传param的参数为flag.txt。
  
通过cookie上传action=readscan。  

sign为md5(secert_key+flag.txtread+scan)的值。  

这样就满足了刚开始的检测，同时又满足了action为scanread，以此来写入和读取文件。从而得到flag.txt内容。得到flag。  

# 小结  

这道题目其实挺新颖的，说是考察了ssrf，但是其实也就是通过特定的路由来执行文件的读取，和一般的ssrf通过image参数构造?image=www...有相同之处。  

但是这道题目主要考察的还是代码审计能力，也就是对于flask框架下的代码的审计和理解，虽然一眼看到的代码很长，但是只要抓住关键，**也就是先审计路由，再由路由跟进函数，最后找到可以执行的漏洞函数，**通过不同路由的联系最后构造payload，得到flag。  

总的来说，对于这种题，一是自己的知识面要广，认识不同的代码，二是代码的审计能力要强。  

附上关于flask框架的简单介绍：  

[Flask框架快速入门学习（1）](https://blog.csdn.net/qq_38664371/article/details/80352102)