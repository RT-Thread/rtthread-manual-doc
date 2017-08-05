首先解释一下pullrequest这个词，意思是推送请求，实际意思是我发现你的git仓库代码中有不完善的bugs，我想修改这些bugs,我就把你的代码下载到我的本地进行修改，修改过后我想把我的代码提交给你，让你把我修改的代码更新到你的git仓库中，这时候我就需要发起一个pullrequest。当然仓库的管理者可以决定是否接受我的请求，如果同意，那么这份代码就会被merge（合并）到原来的git仓库中。如果不同意，那么这次请求就作废了。继续努力做出新的贡献吧。

下面是我摘自知乎https://www.zhihu.com/question/21682976网友的一段解释：
我尝试用类比的方法来解释一下pullreqeust。想想我们中学考试，老师改卷的场景吧。你做的试卷就像仓库，你的试卷肯定会有很多错误，就相当于程序里的bug。老师把你的试卷拿过来，相当于先fork。在你的卷子上做一些修改批注，相当于gitcommit。最后把改好的试卷给你，相当于发pullrequest，你拿到试卷重新改正错误，相当于merge。

当你想更正别人仓库里的错误时，要按照下面的流程进行：
1. 先 fork 别人的仓库，相当于拷贝一份别人的资料。因为不能保证你的修改一定是正确的，对项目有利的，所以你不能直接在别人的仓库里修改，而是要先fork到自己的git仓库中。
2. clone到自己的本地分支，做一些bugfix，然后发起pullrequest给原仓库，让原仓库的管理者看到你提交的修改。 
3. 原仓库的管理者 review这个bug，如果是正确的话，就会merge到他自己的项目中。merge的意思就是合并，将你修改的这部分代码合并到原来的仓库中添加代码或者替换掉原来的代码。至此，整个 pull request 的过程就结束了，原来仓库中就有了你贡献的代码啦。

现在以我们的rt-thread的仓库为例：

1. 首先要将rt-thread仓库fork到自己的git仓库中。

![avatar](https://github.com/SummerGGift/rtthread-manual-doc/raw/master/figures/fork.png)
2. 首先要将rt-thread仓库clone到本地。

![image](https://github.com/SummerGGift/rtthread-manual-doc/raw/master/figures/cloneformgit.png)
![image](https://github.com/SummerGGift/rtthread-manual-doc/raw/master/figures/cloneformgit2.png)
![image](https://github.com/SummerGGift/rtthread-manual-doc/raw/master/figures/cloneformgit3.png)

3.在本地修改后发起pull request.
![image](https://github.com/SummerGGift/rtthread-manual-doc/raw/master/figures/pullrequest.png)

4.发起请求成功后，仓库的管理者就可以看到你的修改了，如果同意你的代码就会被合并到仓库中，这样一次pull request就结束啦。

这就是向rt-thread提交代码的方法啦，希望对你有所帮助，小伙伴们快快行动起来~