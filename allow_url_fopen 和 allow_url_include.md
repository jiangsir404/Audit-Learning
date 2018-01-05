## allow_url_fopen

只要在php.ini 文件中激活了allow_url_fopen选项， 就可以在大多数需要用文件名作为参数的函数中适用httpftp的url来代替文件名.

比如: fopen,file_get_contents,copy,readfile等函数都可以接收一个stream作为文件名来读取stream里的文件内容, imagecreatefromxxx等图像处理函数也可以传入一个url图片作为文件名

但也有一些函数不能接收一个url作为文件名，比如include,include_once,require,require_once 函
.

```
imagecreatefrombmp — 由文件或 URL 创建一个新图象。
imagecreatefromgd2 — 从 GD2 文件或 URL 新建一图像
imagecreatefromgd2part — 从给定的 GD2 文件或 URL 中的部分新建一图像
imagecreatefromgd — 从 GD 文件或 URL 新建一图像
imagecreatefromgif — 由文件或 URL 创建一个新图象。
imagecreatefromjpeg — 由文件或 URL 创建一个新图象。
imagecreatefrompng — 由文件或 URL 创建一个新图象。
imagecreatefromstring — 从字符串中的图像流新建一图像
imagecreatefromwbmp — 由文件或 URL 创建一个新图象。
imagecreatefromwebp — 由文件或 URL 创建一个新图象。
imagecreatefromxbm — 由文件或 URL 创建一个新图象。
imagecreatefromxpm — 由文件或 URL 创建一个新图象。
```



如果关闭allow_url_fopen, 那么这些函数是无法获取远程文件内容


需要注意的一点

1.  出于安全性考虑， 此选项只能在php.ini中设置，无法直接在脚本中通过`ini_set('allow_url_fopen',1)` 这样来修改

原文：

`ini_set() cannot be used for allow_url_fopen and allow_url_include (note that they are marked "PHP_INI_SYSTEM")`

这里要明白php_ini_system的意思, 配置可被修改的设定范围通常有五种:
```
php_ini_user: 该配置选项可在用户的php脚本中修改

php_ini_perdir:该配置选项可在php.ini ， .htaccess 或 httpd.conf 中设

php_ini_system: 该配置选项可在php.ini或httpd.conf中设置

php.ini_all: 该配置选项可在任何地方设置

php.ini only: 该配置选项可仅可在php.ini 中设置
```


在php.ini中这两个tag的范围为:

```
allow_url_fopen	"1"	PHP_INI_SYSTEM	在 PHP <= 4.3.4 时是 PHP_INI_ALL。
allow_url_include	"0"	PHP_INI_ALL	在 PHP 5 时是 PHP_INI_SYSTEM。 从 PHP 5.2.0 起可用。(貌似有误)
```

参考: http://www.php.net/manual/zh/ini.list.php



## allow_url_include

该值如果设定为1,那么include,require,include_once,require_once 就可以把url当作文件名参数来远程包含了。

> Note:
This setting requires allow_url_fopen to be on.


### 支持的协议和封装协议

PHP 带有很多内置 URL 风格的封装协议，可用于类似 fopen()、 copy()、 file_exists() 和 filesize() 的文件系统函数。 除了这些封装协议，还能通过 stream_wrapper_register() 来注册自定义的封装协议。

```
file:// — 访问本地文件系统
http:// — 访问 HTTP(s) 网址
ftp:// — 访问 FTP(s) URLs
php:// — 访问各个输入/输出流（I/O streams）
zlib:// — 压缩流
data:// — 数据（RFC 2397）
glob:// — 查找匹配的文件路径模式
phar:// — PHP 归档
ssh2:// — Secure Shell 2
rar:// — RAR
ogg:// — 音频流
expect:// — 处理交互式的流
```




参考： http://www.php.net/manual/zh/filesystem.configuration.php

