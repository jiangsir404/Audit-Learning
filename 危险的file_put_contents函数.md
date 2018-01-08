在EIS上遇到一道文件上传的题，发现过滤了`<`,这样基本很多姿势都无效了，想了很久没做出来这题，赛后才知道是利用数组来绕过, 这里分析了下原理

来看下`file_put_contents`函数第二个参数data的官网定义:
```
data
要写入的数据。类型可以是 string，array 或者是 stream 资源（如上面所说的那样）。

如果 data 指定为 stream 资源，这里 stream 中所保存的缓存数据将被写入到指定文件中，这种用法就相似于使用 stream_copy_to_stream() 函数。

参数 data 可以是数组（但不能为多维数组），这就相当于 file_put_contents($filename, join('', $array))。
```


可以看到，data参数可以是数组, 会自动做join('',$array)转换为字符串的

但我们字符串过滤函数一般是用`preg_match`函数来过滤的,如:
```
if(preg_match('/\</',$data)){
	die('hack');
}
```


我们知道，很多处理字符串的函数如果传入数组会出错返回NULL, 如strcmp,strlen,md5等，　但`preg_match` 函数出错返回false, 这里我们可以通过`var_dump(preg_match('/\</',$data));` 来验证，　这样的话，preg_match 的正则过滤就失效了

因此，猜测文件上传的代码是这样写的

```
<?php 

if(isset($_POST['content']) && isset($_POST['ext'])){
	$data = $_POST['content'];
	$ext = $_POST['ext'];

	//var_dump(preg_match('/\</',$data));
	if(preg_match('/\</',$data)){
		die('hack');
	}
	$filename = time();
	file_put_contents($filename.$ext, $data);
}

?>
```

于是我么可以传入`content[]=<?php phpinfo();?>&ext=php` 这样来绕过


#### 修复方法
修复方法是使用`fwrite` 函数来代替危险的`file_put_contents`函数,fwrite函数只能传入字符串，如果是数组会出错返回false

```
<?php 

if(isset($_POST['content']) && isset($_POST['ext'])){
	$data = $_POST['content'];
	$ext = $_POST['ext'];

	//var_dump(preg_match('/\</',$data));
	if(preg_match('/\</',$data)){
		die('hack');
	}
	$filename = time();
	// file_put_contents($filename.$ext, $data);
	$f = fopen($filename.$ext);
	var_dump(fwrite($f,$data));
}

?>
```