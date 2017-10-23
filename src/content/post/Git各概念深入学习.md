---
title: Git各概念深入学习
author: Johnson
type: post
date: 2016-07-01T03:32:16+00:00
categories:
  - Git
tags:
  - Git
  - 项目管理工具

---
平时用TortoiseGit多了，只会用Git来做做简单的版本控制管理，其实这样管理后的项目基本就是野路子，当当地痞还行，上不了战场。最近比较有兴趣，网上的资料结合着命令行Git的练习，对很多概念有了清晰的认识，也找到了一套成熟的，高效，能发挥Git真正实力的用法。

最常用命令：

<table cellspacing="0" cellpadding="2" width="590" border="0">
  <tr>
    <td valign="top" width="80">
      git clone
    </td>
    
    <td valign="top" width="508">
      从远程仓库拉代码到本地，其实内部主要是将整个git内部资料拉下来。
    </td>
  </tr>
  
  <tr>
    <td valign="top" width="80">
      git checkout
    </td>
    
    <td valign="top" width="508">
      表面上:主要是切换branch，加 –b 指如果当前没有此branch那么先创建它。 <br />实质上:从某个区域取出某些文件，所以它也可以用于取以前版本的文件。
    </td>
  </tr>
  
  <tr>
    <td valign="top" width="80">
      git status
    </td>
    
    <td valign="top" width="508">
      查看当前代码的状态，如果修改了代码就会出现让你add到stage区的提示。
    </td>
  </tr>
  
  <tr>
    <td valign="top" width="80">
      git add
    </td>
    
    <td valign="top" width="508">
      添加一个修改到stage区，这之后才能commit。
    </td>
  </tr>
  
  <tr>
    <td valign="top" width="80">
      git commit
    </td>
    
    <td valign="top" width="508">
      将stage区的修改提交到当前branch的HEAD区。
    </td>
  </tr>
  
  <tr>
    <td valign="top" width="80">
      git log
    </td>
    
    <td valign="top" width="508">
      看commit信息的history
    </td>
  </tr>
  
  <tr>
    <td valign="top" width="80">
      git fetch
    </td>
    
    <td valign="top" width="508">
      将远程仓库的最新代码拉下来但不自动与本地代码merge。将远程仓库的代码看作是另一个branch（只是也叫当前branch的名字），更容易理解fetch和pull的区别。
    </td>
  </tr>
  
  <tr>
    <td valign="top" width="80">
      git pull
    </td>
    
    <td valign="top" width="508">
      将远程仓库的最新代码拉下来并merge到当前分支上，当前分支是主动者。
    </td>
  </tr>
  
  <tr>
    <td valign="top" width="80">
      git merge
    </td>
    
    <td valign="top" width="508">
      将目标branch merge到当前branch上。
    </td>
  </tr>
  
  <tr>
    <td valign="top" width="80">
      git push
    </td>
    
    <td valign="top" width="508">
      将本地代码推到远程仓库上去。
    </td>
  </tr>
</table>

这些命令是最基本的东西，熟练使用这些命令只能好比一个产线工人，用正确的方式使用这些基本命令，从而提高项目的质量才能算是一个工程师，再进一步改善设计出更符合实际应用环境的方法才能算是构架师。

<div align="right">
  <!--more-->
</div>

<h3 align="center">
  基本原理
</h3>

  * ##### 四角色Git系统理解

<a href="http://stackoverflow.com/questions/2745076/what-are-the-differences-between-git-commit-and-git-push/2745097#2745097" target="_blank"><img title="http://stackoverflow.com/questions/2745076/what-are-the-differences-between-git-commit-and-git-push/2745097#2745097" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; float: none; padding-top: 0px; padding-left: 0px; margin-left: auto; display: block; padding-right: 0px; border-top-width: 0px; margin-right: auto" border="0" alt="http://stackoverflow.com/questions/2745076/what-are-the-differences-between-git-commit-and-git-push/2745097#2745097" src="/images/14.png" width="481" height="455" /></a> 

上图中的workspace，index，local repository，remote repository就是Git系统中最重要的四个角色，其中index也经常被叫成stage，也就是上面说到的stage区，此图来源于网络资源，了解详情请阅读<a href="http://stackoverflow.com/questions/2745076/what-are-the-differences-between-git-commit-and-git-push/2745097#2745097" target="_blank">原文</a>。

  * ##### 两部分Git系统理解

<a href="http://selfcontroller.iteye.com/blog/1786644" target="_blank"><img title="http://selfcontroller.iteye.com/blog/1786644" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; float: none; padding-top: 0px; padding-left: 0px; margin-left: auto; display: block; padding-right: 0px; border-top-width: 0px; margin-right: auto" border="0" alt="http://selfcontroller.iteye.com/blog/1786644" src="/images/11.png" width="775" height="387" /></a> 

此图更贴近Git的实现，在实现中数据都保存在objects里，而index（stage），HEAD，master这些都是一些索引信息。直观概念中，我们大部分人都只能感觉出来工作区和版本库之间的差别，所以我觉得这个图更直观些。从这里可以看出checkout的本质是根据index或者HEAD里面的索引信息从objects中取出数据，并覆盖当前目录中的同名文件，所以还是比较危险的命令。reset命令的默认是soft模式的，所以比较安全，所以的更改还都存在。

<a href="http://selfcontroller.iteye.com/blog/1786644" target="_blank"><img title="http://selfcontroller.iteye.com/blog/1786644" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; float: none; padding-top: 0px; padding-left: 0px; margin-left: auto; display: block; padding-right: 0px; border-top-width: 0px; margin-right: auto" border="0" alt="http://selfcontroller.iteye.com/blog/1786644" src="/images/6.jpg" width="728" height="312" /></a>从上图也能比较清晰的看出，reset是对应于commit的命令，当我们有几个commit提交的有问题，想撤回来时边需要用到它，同时soft模式下的reset可以保证所在的修改都被完整的保存着，此两图来源于网络资源，了解详情请阅读<a href="http://selfcontroller.iteye.com/blog/1786644" target="_blank">原文</a>。

<h3 align="center">
  分支模型及开发流程
</h3>

  * <h5 align="left">
      基本介绍
    </h5>

<a href="http://nvie.com/posts/a-successful-git-branching-model/" target="_blank"><img title="http://nvie.com/posts/a-successful-git-branching-model/" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; float: none; padding-top: 0px; padding-left: 0px; margin-left: auto; display: block; padding-right: 0px; border-top-width: 0px; margin-right: auto" border="0" alt="http://nvie.com/posts/a-successful-git-branching-model/" src="/images/12.png" width="580" height="768" /></a>

Vincent Driessen 2010年发布于其博客的这套分支模型，被很多公司及个人使用，其中永久存在的branch只有两个，一个master，一个develop，其他的都是存在一部分时间。这个合理的分支模型能让开发团队有序高效的进行开发，不像很多公司的开发完全是胡来，最后大把的时间都被浪费在了debug本不该出现的bug了。此图来源于网络资源，欲了解此分支模型请阅读<a href="http://nvie.com/posts/a-successful-git-branching-model/" target="_blank">原文</a>。

  * <h5 align="left">
      相关工具及改进
    </h5>

Vincent Driessen 的分支模型及由此分支模型决定出来的开发流程被广泛接收，为了方便应用此分支模型，他还开发了git-flow工具并开源于<a href="https://github.com/nvie/gitflow/" target="_blank">nvie/gitflow</a>，使用git-flow工具可以非常直观的使用此开发流程，而且很多集成开发环境也集成了此工具，其实它是一个shell脚本，所以在windows上运行的很慢，我测试了下，对各种情况处理的都很好，完全可以在开发中用。

我依然过了盲目崇拜的阶段，现在看什么东西都会保留批判的想法，在网上也看到了此模型在实际使用中，会与图中呈现的有一些小小的不一致，有兴趣可以阅读<a href="http://www.jiangyouxin.net/2013/02/12/git_flow_1.html" target="_blank">此文</a>及其作者的相关文章。我打算以后实际使用时遇到此类问题再去研究，所以先放在这里备忘。

<h3 align="center">
  开发流程进阶思考
</h3>

  * <h5 align="left">
      Git Flow就是万能的么？
    </h5>

<p align="left">
  我的答案显然是否定的，虽然现在对这方面没有丰富的经验，但这段时间的学习，看到有人将其与GitHub的Flow比较，说GitHub的Flow更好，我们都知道GitHub的pull request，虽然也许我们从来没有用过，但也不难理解它的流程，尤其在无组织的开源项目中这种模式应该说是分布式开发的必选。
</p>

<p align="left">
  比如我在GitHub上看到一个项目，我用他们的成果感觉很好，但是唯独有一部分功能没有，我想去自己实现它，那么我可以直接fork一下这个项目到我的repository，然后开始开发，等我开发测试完成后，我想让原项目吸收我这个功能，那我就create一个pull request，原项目的开发人员看到我的pull request便去review我提交的code，通过后merge到原项目。在此过程中我无需加入到原项目的开发组，而且很可能我只想做这一点儿的开发，之后我也不做这个项目的开发了，所以将我加入到项目开发组里也不是很合适，所以这种模式就工作的很好。
</p>

<p align="left">
  所以我认为公司或者固定团队适合用Git Flow去开发，分布式的开源项目确实需要GitHub Flow的帮助，另外之前看一本讲facebook内部的书，其中描述的开发及部署方式感觉更加先进，而且web开发在release这件事上其实跟通常软件开发还有点区别，web更讲究持续集成，持续部署，所以感觉也不太适合用Git Flow。看那书里描述的，facebook的一个新功能可以按百分比的方式部署，可以指定将新功能暴露给那类用户，关键只需要用他们自己的内部工具点几个按钮就可以完成，可见应该将最牛的人放到内部工具组。
</p>

  * <h5 align="left">
      估计不错的commit策略
    </h5>

说”估计“是因为看了说commit策略的文章，但也没怎么实践过，感觉也没有理解的十分透彻，所以只是感觉百分之八九十应该是对的，但不像上述分支模型那么让我感觉到醍醐灌顶。

作者大致讲了两个概念，一是说我们进行开发时应该区分public和private branch，先在自己的parivate branch上开发，之后切换到public branch上，然后将private branch的修改merge进来，然后将public branch push到remote repository里。二是根据不同类型的工作的不同commit建议。

作者将工作分为三类：短期（一天内完成），中期（持续了几天），长期（有可能隔了好多天）。

  * 短期工作：作者主要强调在自己的private branch上开发时提交的一系列commit，完成工作后其实不需要这些凌乱的commit，所以在把private branch的修改merge到public branch时使用merge &#8211;squash方式，将这些凌乱的commit合成一个commit并加上详细的说明，这样只有一个commit节点也不会有merge节点。 
  * 中期工作：持续几天的开发，别人很可能已经向repository提交了一些commit，这样比然造成需要与remote repository中对应的branch进行merge，但其实大部分时候这种merge都没有conflict，所以作者建议先将public branch更新到最新，然后在private branch上对public branch进行rebase操作，然后在切换到public branch将private branch的修改merge进去，而且这种情况大部分都可以进行fast forward了，这样commit history会非常清晰明了。 
  * 长期工作：隔了一段时间，再次捡起来之前改了半截的工作继续，我们肯定希望能基于最新的code继续进行开发，作者建议先将public branch更新到最新，然后基于它在create出来一个cleaned branch，之后将private branch的修改merge &#8211;squash到cleaned branch上，然后进行下reset，这样所有修改都保留了，而且处于unstage状态（最后reset的作用）。 

欲了解详情请阅读<a href="https://sandofsky.com/blog/git-workflow.html" target="_blank">原文</a>。

<h3 align="center">
  上文中Git特殊命令理解
</h3>

  * <h5 align="left">
      merge 和 rebase
    </h5>

merge和rebase产生的最终code都是一样的，只是rebase强制让commit history呈现一条直线的样子，好像所有的commit都是有序提交的，直接没有同时修改的时候，merge就更贴近真实情况，记录了代码修改最真实的样子。欲更详细的了解merge和rebase的不同，请阅读<a href="http://blog.csdn.net/jixiuffff/article/details/5970891" target="_blank">此文</a>。

  * <h5 align="left">
      rebase 和 merge &#8211;no-ff
    </h5>

rebase一般被大家认为也是一个危险的命令，因为在rebase中出现conflict没有merge时那么好处理，因为rebase会将你的commit拆分成好几个小的path然后一个个的打到另一个branch上，所以出现conflict的时候你会发现你看不懂conflict的文件，因为已经不是你修改的时候的样子了，所以我觉得这种情况下就直接merge就完了。

<a href="http://hungyuhei.github.io/2012/08/07/better-git-commit-graph-using-pull---rebase-and-merge---no-ff.html" target="_blank"><img title="http://hungyuhei.github.io/2012/08/07/better-git-commit-graph-using-pull---rebase-and-merge---no-ff.html" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; float: none; padding-top: 0px; padding-left: 0px; margin-left: auto; display: block; padding-right: 0px; border-top-width: 0px; margin-right: auto" border="0" alt="http://hungyuhei.github.io/2012/08/07/better-git-commit-graph-using-pull---rebase-and-merge---no-ff.html" src="/images/10.jpg" width="512" height="291" /></a>

merge &#8211;no-ff是为了实现上图的效果，这样能很清楚的看出来这一系列commit时为一个事儿做的，这在Vincent Driessen 的分支模型里也是被大力推荐，确实挺好。此图来源于网络资源，欲更详细的了解pull &#8211;rebase和merge &#8211;no-ff，请阅读<a href="http://hungyuhei.github.io/2012/08/07/better-git-commit-graph-using-pull---rebase-and-merge---no-ff.html" target="_blank">原文</a>。

  * <h5 align="left">
      fetch 和 pull
    </h5>

<a href="http://blog.csdn.net/a19881029/article/details/42245955" target="_blank"><img title="http://blog.csdn.net/a19881029/article/details/42245955" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; float: none; padding-top: 0px; padding-left: 0px; margin-left: auto; display: block; padding-right: 0px; border-top-width: 0px; margin-right: auto" border="0" alt="http://blog.csdn.net/a19881029/article/details/42245955" src="/images/7.png" width="296" height="324" /></a>

看上图就很容易理解pull和fetch的过程了，fetch做的事情很简单就是将remote reposity中对应的branch最新版下载下来。而pull又多做了一件事，那就是将它与本地的branch进行了merge。欲更详细的了解fetch和pull，请阅读<a href="http://blog.csdn.net/a19881029/article/details/42245955" target="_blank">原文</a>。

  * <h5 align="left">
      blame，cherry-pick 和 patch
    </h5>

<table cellspacing="0" cellpadding="2" width="238" border="0">
  <tr>
    <td valign="top" width="106">
      blame
    </td>
    
    <td valign="top" width="130">
      详情请见<a href="http://blog.csdn.net/feelang/article/details/25649273" target="_blank">此文</a>。
    </td>
  </tr>
  
  <tr>
    <td valign="top" width="106">
      cherry-pick
    </td>
    
    <td valign="top" width="130">
      详情请见<a href="http://blog.csdn.net/hudashi/article/details/7669462" target="_blank">此文</a>。
    </td>
  </tr>
  
  <tr>
    <td valign="top" width="106">
      patch
    </td>
    
    <td valign="top" width="130">
      详情请见<a href="http://www.cnblogs.com/y041039/articles/2411600.html" target="_blank">此文</a>。
    </td>
  </tr>
</table>
