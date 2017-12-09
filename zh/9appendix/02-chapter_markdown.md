# 电子书markdown入门 #

本章节附录主要描述电子书环境下书写markdown文件的规则，包括markdown本身的规则，也包括用于电子书而扩展的一些规则。

## 标题、段落、区块代码 ##

电子书中的每个章节，章节标题行首都由一个 `#` 包围，例如：
    # 章节名称 #

每个章节文件有且仅有一个章节标题，其余的都是它的子标题。每个子标题由两个到六个 `#` 包围而成，从而形成标题2到标题6阶。

一个段落是由一个以上的连接的行句组成，而一个以上的空行则会划分出不同的段落（空行的定义是显式上看起来是空行，就被视为空行，例如有一行只有空白和 tab，那该行也会被视为空行），一般的段落不需要用空白或换行缩进。

在markdown电子书中不存在引用的情况，相替换的，建议使用代码方式风格来替换引用。代码风格可以在文本前加入4个空格，例如：

    引用的区块#1
    应用的区块#2

## 修辞和强调 ##

Markdown 使用星号和底线来标记需要强调的区段。例如：

    这是一个 **强调** 的文本，这是一个加 __底线__ 的文本。

这是一个 **强调** 的文本，这是一个加 __底线__ 的文本。

## 列表 ##

无序列表使用星号、加号和减号来做为列表的项目标记，这些符号是都可以使用的，使用星号：

    * Candy.
	* Gum.
	* Booze.

加号：

	+ Candy.
	+ Gum.
	+ Booze.

和减号

	- Candy.
	- Gum.
	- Booze.

有序的列表则是使用一般的数字接着一个英文句点作为项目标记：

	1. Red
	2. Green
	3. Blue

如果你在项目之间插入空行，那项目的内容会形成段落，可以在一个项目内放上多个段落，只要在它前面缩排 4 个空白或 1 个 tab 。

	* A list item.
	With multiple paragraphs.

	* Another item in the list.

* A list item.
With multiple paragraphs.

* Another item in the list.

## 链接 ##

Markdown 支援两种形式的链接语法： *行内* 和 *参考* 两种形式，两种都是使用角括号来把文字转成连结。

行内形式是直接在后面用括号直接接上链接：

	This is an [example link](http://example.com/).

实际效果如：

This is an [example link](http://example.com/).

你也可以选择性的加上 title 属性：

	This is an [example link](http://example.com/ "With a Title").

实际效果如：

This is an [example link](http://example.com/ "With a Title").

参考形式的链接让你可以为链接定一个名称，之后你可以在文件的其他地方定义该链接的内容：

	I get 10 times more traffic from [Google][1] than from
	[Yahoo][2] or [MSN][3].
	
	[1]: http://google.com/ "Google"
	[2]: http://search.yahoo.com/ "Yahoo Search"
	[3]: http://search.msn.com/ "MSN Search"

实际效果如：

I get 10 times more traffic from [Google][1] than from
[Yahoo][2] or [MSN][3].
	
[1]: http://google.com/ "Google"
[2]: http://search.yahoo.com/ "Yahoo Search"
[3]: http://search.msn.com/ "MSN Search"

title 属性是选择性的，链接名称可以用字母、数字和空格，但是不分大小写：

	I start my morning with a cup of coffee and
	[The New York Times][NY Times].

	[ny times]: http://www.nytimes.com/

实际效果如：

I start my morning with a cup of coffee and
[The New York Times][NY Times].

[ny times]: http://www.nytimes.com/

## 图片 ##

图片的语法和链接很像，同时图也可以选择一个标题，标题序号会在生成PDF时自动加上序号，例如：

	![标题](../../figures/logo.png)

![标题](../../figures/logo.png)

其中，请确保指向的图形文件在figures目录下存在，否则在生成PDF文件时会报错。

## 代码 ##

在电子书中，当转换成PDF时，可以支持代码的语法高亮，可以使用如下的形式（也可以根据实际排版情况，在代码前加入4个或2个空格）：

	```c
	#include <stdio.h>
	
	int main(int argc, char** argv)
	{
		printf("hello\n");
		return 0;
	}
	```

它的效果类似于这样：

```c
#include <stdio.h>

int main(int argc, char** argv)
{
    printf("hello\n");
    return 0;
}
```

## API说明 ##

可以使用下面的格式来说明系统中的API接口。例如：

    int func(int a, int b);

这个函数用于什么目的，完成了什么事。

**函数参数**

--------------------------------------------------------------------------
      参数  描述
----------  ------------------------------------------------------------
     int a  a的意义是balabala

     int b  b的意义是balabala
--------------------------------------------------------------------------


**函数返回**

函数执行成功返回0；失败返回 -1


**注意事项**

这个函数能用于什么，不能用于什么，是否有什么注意事项。


**使用范例**

这是一个用于什么的例子，例子的大致描述。

```c
int result;

/* 调用函数 */
result = function(1, 2);
```

例子是否需要详细解释。


## 注意事项 ##

在markdown向PDF转换过程中，pandoc会生成中间的LaTex文件，如果markdown文本中（不是引用、代码中使用）使用了反斜杠，会导致转化PDF文件报错。所以最好的方式是使用双反斜杠，如下所示：

    C:\\Python27路径balabala
