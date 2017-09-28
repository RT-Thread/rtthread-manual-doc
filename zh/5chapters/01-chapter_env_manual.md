# RT-Thread 软件包环境

RT-Thread软件包环境是RT-Thread源代码的软件包开发环境，包管理系统等。提供了RT-Thread
下所需要到配置，编译等环境。

## 1.准备工作

* env环境编译器默认配置为GNU GCC，工具链目录指向 `env\tools\gnu_gcc`，如果不使用GCC编译器则无需配置。请将交叉编译工具链放置于 `env\tools\gnu_gcc`目录下,或者使用以下命令设置工具链地址：


    set RTT_EXEC_PATH=your_Cross_Compilation_tool_chain_path
* 在电脑上装好git，git的下载地址为`https://git-scm.com/downloads`,根据向导正确安装git。一些组件包是通过git下载管理的。

* 注意在工作环境中，所有的路径都不可以有中文字符或者空格。


## 2.打开控制台

RT-Thread 软件包环境主要以命令行控制台为主，同时以字符型界面来进行辅助，使得尽量减少修改配置文件的方式即可搭建好RT-Thread开发环境的方式。

![image](./figures/console_bat.png)

进入env目录，可以运行本目录下的 `console.bat `程序，它会配置一些环境变量，然后弹出控制台窗口。接下来对软件包的操作都是在控制台环境下进行的，下图为控制台窗口：

![image](./figures/console_window.png)

## 3.针对BSP的配置

### 3.1 环境变量的设置

#### 第一步：切换到工程目录

打开控制台后，可以在命令行模式下使用cd命令切换到你想要配置的工程目录。
例如工程目录为`F:\git_repositories\ART_wifi\firmware\app`，先进入工程根目录。

#### 第二步：设置RTT_ROOT根目录
可以通过命令：

    set RTT_ROOT=your_rtthread

的方式设置RT-Thread目录（其中your_rtthread请填写你的RT-Thread根目录位置，记住RT-Thread不要放于带空格或中文字符的目录路径下）。

![image](./figures/set_rtt_root.png)

#### 第三步：复制KConfig_bsp到工程目录并改名为KConfig

在使用env的`menuconfig`命令配置功能前，如果设备工程目录(bsp)中没有相应的`KConfig`文件，可将`env`目录下的`KConfig_bsp`文件复制到bsp根目录中并改名为`KConfig`。

![image](./figures/copy2bsp.png)

在bsp根目录中改名为：

![image](./figures/renamekconfig.png)

#### 第四步：更新env的在线组件包仓库列表

在使用`menuconfig`命令之前还需要使用

    pkgs --upgrade

命令来更新env的组件包仓库列表

![image](./figures/upgrade_from_gitpackages.png)

#### 第五步：使用menuconfig开始配置项目

现在就可以在设备工程目录中使用`menuconfig`命令开始进行项目配置，如果没有出错接下来就可以看到`menuconfig`的界面了，如下图：

![image](./figures/menuconfig_window.png)

### 3.2 错误提示
如果没有使用`pkgs --upgrade`来更新列表就使用`menuconfig`命令会出现如下错误：

![image](./figures/no_pkgs_upgrade_error.png)

重新使用`pkgs --upgrade`命令即可解决这个问题。

### 3.3 menuconfig的简单使用方法：

- 上下键：选择不同的行，移动到不同的（每一行的）选项上。

- 空格键：用于在选择该选项，取消选择该选项，之间来回切换。

  - 选择该（行所在的）选项：对应的该选项前面就变成了，中括号里面一个星号，即 **[ \* ]**，表示被选中了。
  - 如果是取消该选项，就变成了，只有一个中括号，里面是空的，即：[   ]

- 左右键：用于在Select/Exit/Help之前切换

- 回车键：左右键切换到了某个键上，此时回车键，就执行相应的动作：

  - Select：此时一般都是所在（的行的）选项，后面有三个短横线加上一个右箭头，即 —>，表示此项下面还有子选项，即进入子菜单

  - Exit：直接退出当前的配置，当你更改了一些配置，但是又没有保存，此时会询问你是否要保存当前（已修改后的最新的）配置，然后再退出。

  - Help：针对你当前所在某个（行的）选项，查看其帮助信息。如果你对某个选项的功能不是很清楚就可以查看其Help，也可以可能查到写出到配置文件中的宏。

## 4.包管理器

### 4.1包管理器介绍

包管理器是一个组件包平台，用户可以通过包管理器来获取，添加或者删除自己所需要的组件包。还可以提供自己使用的组件包到平台来。通过包管理器的`pkgs --wizard`可以制作组件包下载索引，提交组件包下载索引到如下地址即可将自己制作的组件包发布出来：

    https://github.com/RT-Thread/packages.git

![image](./figures/rtt_packages.png)

组件包代码本身可以存储在自己的网络存储空间，也可以提交到The packages repositories of RT-Thread中审核并管理，地址如下：

    https://github.com/RT-Thread-packages

![image](./figures/pkgs_repo_rtt.png)

目前支持的组件包格式有`.zip,.rar.gz,rar.bz2`。同时支持托管在git上，并且附带有submudule的组件包。比如mqtt组件包的地址为`https://github.com/RT-Thread-packages/paho-mqtt.git`。

### 4.2包管理器命令

包管理器的操作主要使用pkgs命令，可以使用 `pkgs -h`来获取使用帮助。 注意：在使用menuconfig选择在线包之前，需要先使用` pkgs --upgrade` 命令更新env的packages文件夹。请预先在电脑上装好git工具。

    pkgs --list            列出当前使用的组件包列表
    pkgs --update          读取目前menuconfig对项目的配置，和旧的项目配置做对比，进而更新包
    pkgs --wizard          组件包制作向导，根据提示输入来制作包向导文件夹
    pkgs --upgrade         从reposource更新env的本地packages文件夹
    example：
    使用pkgs --upgrade命令后，env环境会自动从默认git地址： https://github.com/RT-Thread/packages.git 来更新本地包。后续会支持更新源列表。

使用menuconfig来配置项目所需要的组件包，然后通过pkgs --update命令来更新项目中的组件包。如果不想要某个组件包，可以在menuconfig的配置中去掉包选项，然后再次使用`pkgs --update`命令更新即可。

如果解压出的组件包被人为修改，那么在删除组件包的时候会提示是否要删除被修改的文件。如果选择保留文件，那么请自行保存好这些文件，避免在下次更新包时被覆盖。

支持在线下载的组件包在RT-thread online packages选项中，根据项目需要来选择所需的组件。目前提供了不同类型的组件包以供测试。

## 5.如何制作一个组件包
## 6.如何制作一个组件包下载索引

### 6.1 制作一个git形式的组件包下载索引
使用命令`pkgs --wizard`开始制作组件包下载索引：

![image](./figures/pkgs_wizard.png)

接下来需要输入组件包的名字，请使用小写字母组合的形式来给组件包取名。本次示例我们制作一个`pahomqtt`的组件包。

![image](./figures/pkgs_wizard_make.png)

分为五步操作，首先输入组件包的名称，注意使用小写字母的组合，menuconfig选项名，版本号和第三个keyword在第一次制作的时候都可以按下回车使用默认值，最后根据提示为这次制作的软件包选择一个类别。因为本次制作的是iot类的组件包，所以输入1后按下回车，包下载索引就制作完成了。

在使用`pkgs --wizard`命令的目录找到生成的文件夹pahomqtt。

![image](./figures/pahomqtt_pkgs.png)

进入pkginfo目录：

![image](./figures/paho_pkinfo.png)

这里的Kconfig和package.json就是我们重点需要关注的文件了。kconfig内容文件如下：

![image](./figures/pkgs_mqtt_kconfig.png)

package.json文件内容如下：

![image](./figures/pkgs_mqtt_json.png)

本次示例所用的组件包已经制作好并上传到了git上，地址如下：

![image](./figures/pkgs_mqtt_git.png)

然后找到标识这个版本的SHA值，填入json文件。

![image](./figures/pkgs_mqtt_getSHA.png)

修改好的json文件如下：

![image](./figures/pkgs_mqtt_complete_SHA.png)

到此组件包下载索引就制作完成了，接下来将pkginfo文件夹改名为组件包名：

![image](./figures/pkgs_ready2commit.png)

接下来我们需要将组件包索引通过PR流程推送到`https://github.com/RT-Thread-packages`，我们需要将我们的组件包放在packages相应的文件夹下后再进行推送。

![image](./figures/pkgs_mqtt_add_dir.png)

使用git进行PR的方法请参考：
    https://github.com/RTThread/rtthreadmanualdoc/blob/master/zh/9appendix/03_github.md

### 6.2 制作一个压缩包形式的组件包下载索引

制作一个压缩包形式的组件包下载索引大体上和上面的操作步骤是相同的。唯一不同的地方在于json文件，一个压缩包形式的组件包json文件如下：

![image](./figures/pkgs_comon_json.png)

这样我们就制作好一个下载索引包了，如果还有不懂的地方可以参考已有的组件包，或者及时在群里提出来，我们会第一时间回答你的疑问，并且对env中存在的问题进行优化。谢谢你的参与。

## 7.编译RT-Thread

### 7.1 scons编译工具链的配置

### 7.2 开始编译工程吧

RT-Thread 软件包环境也携带了Python & scons环境，所以只需要在设备工程目录中运行：

    scons

就可以编译RT-Thread了。一般来说，工程所需要的环境变量都会在控制台环境中已经配置好。
