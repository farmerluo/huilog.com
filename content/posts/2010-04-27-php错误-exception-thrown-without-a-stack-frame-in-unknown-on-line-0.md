---
title: 'PHP错误: Exception thrown without a stack frame in Unknown on line 0'
author: 阿辉
date: 2010-04-27T11:03:00+00:00
categories:
- Php
tags:
- Php
keywords:
- Php
comments: true
showTags: true
showPagination: true
showSocial: false
showDate: true
showMeta: true
---
就目前我的了解，在两种情况下，PHP会报 Exception thrown without a stack frame in Unknown on line 0这种错误：

1）异常捕捉用了set_exception_handler导向,Exception里面执行另一个Exception

如下面这段代码，就会出现这种问题：

http://de.php.net/manual/de/function.set-exception-handler.php#88082

<!--more-->

```php
function error_handler(code, message, file, line)
{
if (0 == error_reporting())
{
return;
}
throw new ErrorException(message, 0, code, file, line);
}
function exception_handler(e)
{
// ... normal exception stuff goes here
print undefined; // This is the underlying problem
}
set_error_handler("error_handler");
set_exception_handler("exception_handler");
throw new Exception("Just invoking the exception handler");
?>
```

exception_handler函数内print undefined;这行本身会抛出一个异常，而他又去调用set_exception_handler的exception_handler函数，死循环了。

解决办法：不要在一个Exception里面执行另一个Exception

上面的问题可以用try ... catch的方式，如exception_handler改成下面这样：
```php
function exception_handler(e)
{
try
{
// ... normal exception stuff goes here
print undefined; // This is the underlying problem
}
catch (Exception e)
{
print get_class(e)." thrown within the exception handler. Message: ".e->getMessage()." on line ".e->getLine();
}
}
?>
```

2) 在析构函数抛出异常

参考这个bug:http://bugs.php.net/bug.php?id=33598

下面的代码就会报这个错误：
```php
class test {
function __construct() {
echo “Constructn”;
}

function greet() {
echo “Hello Worldn”;
}

function __destruct() {
echo “Destructn”;
throw new Exception( ‘test’ );
}
}

test = new test();
test->greet();
```
目前的解决办法：

1.不要在析构函数中抛出异常.

2.由于析构函数是在退出时执行的，所以要手动unset这种类，并catch该异常。

比如上面的例子，在最后加一行unset(test),这时程序就会报throw new Exception( ‘test’ ); 这行有错误，再catch这个异常就行了。

上面两种情况在php 5.2.11版本上都会出现，至于原因我认为PHP可能就是这样处理的，php bug 33598 2005年就报上去了，bug Status为Closed，说明官方并不认为这是一个bug,或不当一个bug处理了。