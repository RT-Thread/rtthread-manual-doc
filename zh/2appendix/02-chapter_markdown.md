# 电子书markdown入门 #

本章节附录主要描述电子书环境下书写markdown文件的规则，包括markdown本身的规则，也包括用于电子书而扩展的一些规则。

## 标题、段落、区块代码 ##

电子书中的每个章节，章节标题行首都由一个 `#` 包围，例如：
    # 章节名称 #

每个章节文件有且仅有一个章节标题，其余的都是它的子标题。每个子标题由两个到六个 `#` 包围而成，从而形成标题2到标题6阶。

一个段落是由一个以上的连接的行句组成，而一个以上的空行则会划分出不同的段落（空行的定义是显式上看起来是空行，就被视为空行，例如有一行只有空白和 tab，那该行也会被视为空行），一般的段落不需要用空白或换行缩进。

区块引用则使用 email 形式的 '`>`' 角括号。例如以下的文本：

    > 引用的区块#1
    > 引用的区块#2

> 引用的区块#1
> 应用的区块#2

### 修辞和强调 ###

Markdown 使用星号和底线来标记需要强调的区段。例如：

    这是一个 *强调* 的文本，这是一个加 __底线__ 的文本。

这是一个 *强调* 的文本，这是一个加 __底线__ 的文本。

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

### 链接 ###

Markdown 支援两种形式的链接语法： *行内* 和 *参考* 两种形式，两种都是使用角括号来把文字转成连结。

行内形式是直接在后面用括号直接接上链接：

	This is an [example link](http://example.com/).

你也可以选择性的加上 title 属性：

	This is an [example link](http://example.com/ "With a Title").

参考形式的链接让你可以为链接定一个名称，之后你可以在文件的其他地方定义该链接的内容：

	I get 10 times more traffic from [Google][1] than from
	[Yahoo][2] or [MSN][3].
	
	[1]: http://google.com/ "Google"
	[2]: http://search.yahoo.com/ "Yahoo Search"
	[3]: http://search.msn.com/ "MSN Search"

title 属性是选择性的，链接名称可以用字母、数字和空格，但是不分大小写：

	I start my morning with a cup of coffee and
	[The New York Times][NY Times].

	[ny times]: http://www.nytimes.com/

### 图片 ###

图片的语法和链接很像。

行内形式（title 是选择性的）：

	![alt text](/path/to/img.jpg "Title")

参考形式：

	![alt text][id]

	[id]: /path/to/img.jpg "Title"

上面两种方法都会输出 HTML 为：

	<img src="/path/to/img.jpg" alt="alt text" title="Title" />

### 代码 ###
在一般的段落文字中，你可以使用反引号 `` ` `` 来标记代码区段，区段内的 `&`、`<` 和 `>` 都会被自动的转换成 HTML 实体，这项特性让你可以很容易的在代码区段内插入 HTML 码：

	I strongly recommend against using any `<blink>` tags.

	I wish SmartyPants used named entities like `&mdash;`
	instead of decimal-encoded entites like `&#8212;`.


如果要建立一个已经格式化好的代码区块，只要每行都缩进 4 个空格或是一个 tab 就可以了，而 `&`、`<` 和 `>` 也一样会自动转成 HTML 实体。

Markdown 语法:

	If you want your page to validate under XHTML 1.0 Strict,
	you've got to put paragraph tags in your blockquotes:

	<blockquote>
	<p>For example.</p>
	</blockquote>
