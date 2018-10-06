## strpos
```
if(stripos($url,'.')){
	die("unknow url");
}

if(stripos($url,'.') == False){
	die('unkonw url');
}
```

这段代码是存在漏洞的，因为当.出现在第一位时候返回值为0

正常写法:

```
if(stripo($url,'.') === False){
	die('unkonwn url');
}


## 注入

	preg_match("/\b(select|insert|update|delete)\b/i",$message)

然后使用/*!00000select*/ 来bypass 语义分析的绕过

> \b”匹配单词边界，不匹配任何字符

/*!*/ 只在mysql中有用，在别的数据库中这只是注释，但是在mysql，/*!select 1*/可以成功执行，在语句前可以加上5位数字，代表版本号，表示只有在大于该版本的mysql中不作为注释