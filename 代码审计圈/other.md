## strpos
```
if(stripos($url,'.')){
	die("unknow url");
}
```

这段代码是存在漏洞的，因为当.出现在第一位时候返回值为0