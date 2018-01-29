### escapeshellarg

php手册上的解释为:

> escapeshellarg() 将给字符串增加一个单引号并且能引用或者转码任何已经存在的单引号，这样以确保能够直接将一个字符串传入 shell 函数，shell 函数包含 exec(), system() 执行运算符(反引号)



错误的使用:

经过 escapeshellarg 函数处理过的参数被拼凑成 shell 命令 并且被双引号包裹这样就会造成漏洞

主要在于bash中双引号和单引号解析变量是有区别的

在解析单引号的时候 , 被单引号包裹的内容中如果有变量 , 这个变量名是不会被解析成值的
但是双引号不同 , bash 会将变量名解析成变量的值再使用

```
lj@lj:/usr/share/virtualbox$ echo `date`
2018年 01月 29日 星期一 11:38:08 CST
lj@lj:/usr/share/virtualbox$ echo '`date`'
`date`
lj@lj:/usr/share/virtualbox$ echo "`date`"
2018年 01月 29日 星期一 11:38:21 CST
lj@lj:/usr/share/virtualbox$ echo "'`date`'"
'2018年 01月 29日 星期一 11:38:27 CST'
lj@lj:/usr/share/virtualbox$ echo '"`date`"'
"`date`"
```


如上可知， 如果参数用了 escapeshellarg 函数过滤单引号，但参数在拼接命令的时候用了双引号的话还是会导致命令执行的漏洞。


具体漏洞参考: https://www.jianshu.com/p/162844f45f1d


### escapeshellcmd

php手册上的解释为:

escapeshellcmd() 对字符串中可能会欺骗 shell 命令执行任意命令的字符进行转义。 此函数保证用户输入的数据在传送到 exec() 或 system() 函数，或者 执行操作符 之前进行转义。

反斜线（\）会在以下字符之前插入： &#;`|*?~<>^()[]{}$\, \x0A 和 \xFF。 ' 和 " 仅在不配对儿的时候被转义。 在 Windows 平台上，所有这些字符以及 % 和 ! 字符都会被空格代替。


```
<?php 

$s = "`ls`";
echo escapeshellcmd($s);

>>>
\`ls\`
```


