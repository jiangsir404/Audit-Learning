# Audit-Learning

开新坑了，准备花一个月的时间对学过的代码审计知识好好总结一下，持续更新，欢迎各位师傅star支持一下。

## Todo
- [x] open_basedir 绕过研究
- [x] allow_url_fopen 和 allow_url_include
- [x] 宽字节注入及数据库编码分析
- [x] 通用代码审计思路
- [x] 危险的file_put_contents函数
- [x] escapeshellarg 和 escapeshellcmd 函数.md
- [x] parse_url 函数研究
- [x] 其他
- [x] 特殊的文件名写入技巧(move_uploaded_file, file_put_contents,copy，readfile,file,fopen 都存在) 
- [x] mail函数命令执行
- [ ] disable_functions 绕过研究
- [ ] curl 函数研究
- [ ] addslashes 函数绕过研究
- [ ] move_uploaded_file
- [ ] 其他 [php函数默认配置引发的安全问题](http://skysec.top/2018/08/17/php%E5%87%BD%E6%95%B0%E9%BB%98%E8%AE%A4%E9%85%8D%E7%BD%AE%E5%BC%95%E5%8F%91%E7%9A%84%E5%AE%89%E5%85%A8%E9%97%AE%E9%A2%98/#openssl-verify-%E5%87%BD%E6%95%B0)
- [ ] 误用htmlentities函数引发的漏洞 (http://sec-redclub.com/archives/964/)
- [x] filter_var函数缺陷 (http://sec-redclub.com/archives/925/)


## 一些资源

### 代码审计练习题

https://github.com/CHYbeta/Code-Audit-Challenges

wonderkun师傅的ctf web练习题: https://github.com/wonderkun/CTF_web

https://github.com/bowu678/php_bugs

RIPS2017 代码审计练习oj: https://www.ripstech.com/php-security-calendar-2017/

红日安全 RIPS oj 里题解: https://github.com/hongriSec/PHP-Audit-Labs

### 漏洞exp
推荐一波自己的仓库： https://github.com/jiangsir404/PHP-code-audit

各种开源CMS 各种版本的漏洞以及EXP： https://github.com/Mr5m1th/0day

CMS漏洞测试用例集合： https://github.com/SecWiki/CMS-Hunter


### 乌云 

Xyntax师傅整理的乌云1000个PHP代码审计案例： https://github.com/Xyntax/1000php

乌云Drops文章备份： https://github.com/SecWiki/wooyun_articles

php_code_audit_project： https://github.com/SukaraLin/php_code_audit_project

### 思维导图，资料集合

cheybeta师傅的Web学习资料整理: https://github.com/CHYbeta/Web-Security-Learning

https://github.com/CHYbeta/phith0n-Mind-Map

https://github.com/bit4woo/code2sec.com

python 代码审计: https://github.com/bit4woo/python_sec

高级PHP应用程序漏洞审核技术(by黑哥）https://github.com/Jyny/pasc2at


### 博客
离别歌:https://www.leavesongs.com/

漏洞时代: http://0day5.com/

lorexxar师傅: http://lorexxar.cn/

知道创宇paper: https://paper.seebug.org/


### 书籍
《代码审计》

《PHP7内核剖析》 https://github.com/pangudashu/php7-internal

《深入理解PHP内核》https://github.com/reeze/tipi

### 代码审计工具

cobra： https://github.com/wufeifei/cobra

Seay源代码审计系统2.1: http://www.cnseay.com/

rips: https://github.com/ripsscanner/rips


