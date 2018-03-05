[TOC]
# RT-Thread package 创建教程

---

## 1.package 软件包的添加
### 1.1如何制作一个软件包
### 1.2如何制作一个软件包下载索引
#### 1.2.1 制作一个git形式的软件包下载索引
使用命令`pkgs --wizard`开始制作软件包下载索引：

![image](./figures/pkgs_wizard.png)

接下来需要输入软件包的名字，请使用小写字母组合的形式来给软件包取名。本次示例我们制作一个`pahomqtt`的软件包。

![image](./figures/pkgs_wizard_make.png)

分为五步操作，首先输入软件包的名称，注意使用小写字母的组合，menuconfig选项名，版本号和第三个keyword在第一次制作的时候都可以按下回车使用默认值，最后根据提示为这次制作的软件包选择一个类别。因为本次制作的是iot类的软件包，所以输入1后按下回车，包下载索引就制作完成了。

在使用`pkgs --wizard`命令的目录找到生成的文件夹pahomqtt。

![image](./figures/pahomqtt_pkgs.png)

进入pkginfo目录：

![image](./figures/paho_pkinfo.png)

这里的Kconfig和package.json就是我们重点需要关注的文件了。kconfig内容文件如下：

![image](./figures/pkgs_mqtt_kconfig.png)

package.json文件内容如下：

![image](./figures/pkgs_mqtt_json.png)

本次示例所用的软件包已经制作好并上传到了git上，地址如下：

![image](./figures/pkgs_mqtt_git.png)

然后找到标识这个版本的SHA值，填入json文件。

![image](./figures/pkgs_mqtt_getSHA.png)

修改好的json文件如下：

![image](./figures/pkgs_mqtt_complete_SHA.png)

到此软件包下载索引就制作完成了，接下来将pkginfo文件夹改名为软件包名：

![image](./figures/pkgs_ready2commit.png)

接下来我们需要将软件包索引通过PR流程推送到`https://github.com/RT-Thread/packages`，我们需要将我们的软件包放在packages相应的文件夹下后再进行推送。

![image](./figures/pkgs_mqtt_add_dir.png)

使用git进行PR的方法请参考：
    https://github.com/RT-Thread/rtthread-manual-doc/blob/master/zh/9appendix/03_github.md

#### 1.2.2 制作一个压缩包形式的软件包下载索引

制作一个压缩包形式的软件包下载索引大体上和上面的操作步骤是相同的。唯一不同的地方在于json文件，一个压缩包形式的软件包json文件如下：

![image](./figures/pkgs_comon_json.png)

这样我们就制作好一个下载索引包了，如果还有不懂的地方可以参考已有的软件包，或在rt-thread论坛中提出来，我们会第一时间回答你的疑问，并且对env中存在的问题进行优化。谢谢你的参与。
