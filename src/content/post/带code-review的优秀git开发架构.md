---
title: 带Code Review的优秀Git开发架构
author: Johnson
type: post
date: 2016-07-04T11:57:06+00:00
categories:
  - Git
tags:
  - C/C++
  - Git
  - 项目管理工具

---
大部分公司虽然都用 Git 等版本管理系统管理开发过程，但这并不代表就能达到高效可靠的开发效率和质量，开发人员的水平是层次不齐的，除了加大在招聘方面的投入外，code review 是提高代码质量的一大法器。

错误且糟糕的开发架构特点：

  * 没有 branch 模型，大家的 commit 都提交在很随意创建的 branch，时间一长就变成一大堆branch，而且谁都知道大部分 branch 都是没用的，但谁也没有十足把握说能把它们删掉。 
  * 没有 coding style，代码千奇百怪，还谁也说服不了谁。 
  * 严重两级分化，忙的忙死，闲的闲死，最后经常忙的人中能力强的人跳槽走掉，闲的却依然那么闲，好像走了的那个人原来都是在瞎忙一样（很神奇的现象）。 
  * Code review 反复被提起，又反复被遗忘，通常采用同步方式进行即开会，效率不高，而且大部分最后演变成给不懂的人反复无聊的讲解，或讨论其他问题完全忘记了主题。 
  * 新人不知道该怎么进行每天的工作，稀里糊涂就把 code 提交上去了（其实老人也一样），极其恶心的低级 bug 被隐藏到集成测试时发现，把那些能力强的人气的半死。 
  * 没有规范成型的单元测试，你的代码今天能跑，明天也许就被别人给搞的不能跑了，直到你下次做这部分功能时才发现，然后 debug，然后奔溃的心里。费了九牛二虎的力气重构了极其可怕的烂代码后，发现程序跑不起来，造成人人都不会去想重构这个事儿，都是插来插去的添加新逻辑，最后造成更烂的代码。 

今天总结的开发架构由几部分组成：成熟优秀的 branch 模型 + Branch 模型的保护 + Phabricator 结合 Git 实现的 Pre-Push 高效 Code Review + 普通开发人员开发流程 + Unit Test + Coding Style + 自动化构建及持续集成。本文所讲架构不适用于开发者分散在全球各地互不相识的开源项目，这样的项目更适合用 GitHub 的 Pull-Request 的那套架构，本文适用于开发人员经常在一起的公司项目的开发。

<!--more-->

本文不会很容易阅读，因为会由大量文字堆砌而成，画图太费时间了，主要是记录给自己看的，所以估计好多概念不会将太细，本人另一篇博文“[Git各概念深入学习]({{< relref "Git各概念深入学习.md" >}})”中有可能会有你需要的细节。

  * ##### 成熟优秀的branch模型

<a href="http://nvie.com/posts/a-successful-git-branching-model/" target="_blank"><img title="http://nvie.com/posts/a-successful-git-branching-model/" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; float: none; padding-top: 0px; padding-left: 0px; margin-left: auto; display: block; padding-right: 0px; border-top-width: 0px; margin-right: auto" border="0" alt="http://nvie.com/posts/a-successful-git-branching-model/" src="/images/12.png" width="181" height="240" /></a> 

这里的模型就是 Vincent Driessen 提出的 branch 模型，详情请见<a href="http://nvie.com/posts/a-successful-git-branching-model/" target="_blank">原文</a>。

  * ##### Branch模型的保护

上面了解了需要用什么模型了，也对此模型有所了解了，那么下面的问题就该是具体怎么实现的问题了，实现的好也许能获得人家这个模型80%的好处，实现的不好有可能还不如不知道这事儿呢。

Vincent Driessen（下文简称VD）branch 模型长期存在的只有两个branch：master and develop。其他的 feature，release，hotfixes（下文简称FRH) branch 都只会存在一段时间。VD branch model 的健康在于不能违反 model 中的原则，那么问题来了，怎么保证？人是不可靠的，如果大家都有权利对 repository 里的 branch 进行 create，rename，delete，那几乎不可能保证VD branch model 在长期开发中是健康的。

也许有人狐疑说有那么严重么，偶尔弄错改过来不就行了，偶尔的小任务弄错就弄错呗，反正也没啥事儿。但我觉得这就像木桶的短板效应一样，只要开了口，就没好儿~忘了从哪里看的了，一直觉得很精辟的一句话，大概意思是说有人违反规则犯了错误，比犯错的人更应该反省的是制定或引进这套规则的人，是他给犯错的人留了机会。

所以，VD branch model 保护的方式为在 Git 中对此 repository 进行权限配置，只允许特定的某些人拥有 branch 的 create，rename，delete 的权限。所谓特定的某些人是那些对 VD branch model 有深入了解的高级工程师，这些人一定得是你们团队中的<img class="wlEmoticon wlEmoticon-thumbsup" style="border-top-style: none; border-left-style: none; border-bottom-style: none; border-right-style: none" alt="太棒了" src="/images/15.png" />这个。

  * ##### Branch模型更进一步的保护

上面的策略只能实现 VD branch model 整体结构的保护，光做到这一点还不够。VD branch model中，开发者应该在某个 FRH branch 上提交工作，当工作完成后才应该按照 VD branch model 中 feature，release，hotfixes branch 各自的策略，尽量以 rebase 且 no-fast-forward 的方式 merge 到 master 或 develop branch 上，不应该直接向 master 或 develop branch 提交commit。这一点怎么保证呢？

**本人认为不好的方法：**类似 Pull-Request 或 Merge-Request 的方法。

Pull-Request 和 Merge-Request 其实是一样的，GitHub 叫 Pull-Request， GitLab 叫 Merge-Request，下文统一叫 Pull-Request，这个设计其实非常好，当然不能独立的讨论 Pull-Request，因为它和 Fork 结合在一起，才能说是一套机制。这套机制完美的解决了开发者不固定，分散在全球各地且大多互不相识的开源项目的问题，普通开发者想开发一个新功能先通过 Fork 获得一个指向目标项目 repository 的自己的 repository，然后在这个repository上提交代码，测试通过觉得是时候合进原始 repository 了，就发一个 Pull-Request 给原始 repository 的开发团队。原始 repository 的开发团队对其修改进行 review 后，如果觉得不错就可以 merge 到原始 repository。

让我觉得 GitHub 这些设计的 Pull-Request 机制很好的一个原因要从良好的 Git 使用习惯来说，最好的 Git 习惯是 clone 远程 repository 到 local 后，先从要进行开发的 branch（假如叫 feature\_a）上 create 一个 private 的 branch（假如叫 feature\_a\_johnson），然后在 feature\_a\_johnson branch 上开发完成工作，feature\_a branch 可以随时 pull remote repository 的新变化到本地而且一定不会产生 merge，如果 feature\_a\_johnson branch 上的工作还没做完，突然有一个很紧急的小 feature 必须马上做，那就能看出这套做法的好处了，直接从 feature\_a 上再 create 一个 private 的 branch（假如叫 feature\_a\_fuck），这样就可以从最新的完全干净的 code 基础上开发插进来的任务了，而且这样在 merge feature\_a\_johnson branch 到 feature\_a branch 上时，使用 &#8211;squash，rebase 也会感觉很直观好理解。不会像现在的做法这样，再找一个文件夹把整个 remote repository 全部再 clone 一份到 local，而且 commit merge 乱七八糟。

可是以上的这个好习惯怎么能强制实行呢？你又不能监督着谁。Pull-Request 机制就悄悄的实现了让我们强制使用这个好习惯的目的，Fork 一个 repository 其实相对于原始的 repository 来说你的这个 repository 不就相当于上面那个 feature\_a branch 的角色么，然后我们再将我们自己的这个 repository clone 到 local，其实不就是又创建了一个 private 的 repository，不就相当于上面那个 feature\_a_johnson branch 的角色么。

Pull-Request 介绍完了，该回到更进一步保护 VD branch model 方法上了，我认为不好的方法就是通过禁止普通开发人员拥有向 master and develop branch 上提交 commit 的权限，来达到保护的目的，所以本质上与 Pull-Request 机制一样。

这种方法的<font color="#ff0000">坏处</font>在于只有上面说的“特定的某些人”能向 master and develop branch 上提交 commit，这样普通开发人员在 FRH branch 上进行完开发后，需要“特定的某些人”去将 FRH branch merge 到 master 或 develop branch 上。通常情况下“特定的某些人”一定是比较少的而且他们都身担重职，另外还有开不尽的会，很忙，让他们去 merge 没什么问题，关键是这种 merge 十有八九会产生很多 conflict 要去解决，他又不是这部分 code 的第一开发者，就很麻烦，结果是既耽误了“特定的某些人”的宝贵时间，又没给普通开发者省去工作量，因为最后一定演变成这个“特定的某些人”去开会了，跟那个第一开发者说：“来，你来我电脑上用我的账号 merge~“（瞬间回到解放前的节奏，哈哈哈）

**本人认为好的方法：**通过 Pre-Push Code Review 达到目的。

先来回顾在更进一步保护 VD branch model 中我们要达到的目的：master and develop branch 上的 commit 只能是 merge FRH branch 产生的，不应该存在直接向 master 或 develop branch 提交的commit。再想想我们现在拥有的条件：Pre-Push Code Review（具体实现后面介绍）。那么既然每一个要 push 进 repository 的commit 比然是经过 reviewer 确认过的，那保证开发人员不能直接向 master 或 develop branch 提交 commit 岂不是相对简单，如果有这种情况发生而且所有reviewer 都没发现，那只能说你们团队给你个飞机你们都能开成驴车，没救儿~

此方法中“特定的某些人”所需要做的就只有：建立某个 FRH branch，对普通开发者进行的开发进行 review，删除这个已正确 merge 到 master 或（和）develop branch 的 FRH branch。这些事物也确实符合“特定的某些人”的职位属性。

也许有人狐疑 FRH branch 的 merge 动作由普通开发者进行，那会不会有这种情况，普通开发者的工作 commit 是被 reviewers 确认没问题了，可是他在 merge 的时候出现了问题，然后这个问题就被悄悄的带进了 repository？其实不会的，因为我们都知道 merge 的时候我们是以 no-fast-forward 的方式进行的，所以必然会出现 commit ，又因为我们是用的 Pre-Push 的 Code Review，所以这个 merge 产生的 commit 也是需要被 reviewers 确认的。又有人狐疑说那要是那小子没用 no-fast-forward 呢？其实也没问题，我们想如果是这样的话，那无非就两种情况，一种是没有 conflict，一种是有 conflict。要是没有 conflict，那确实 code 就直接进 repository 了，可这不是肯定没问题么，哈哈哈。要是有 conflict 那他必然又逃不过 reviewers 这一层。

  * ##### <font color="#555555">Branch模型尚未被保护部分</font><font color="#ff0000">（已想到完美解法）</font>

经过以上的策略，还有一点的保护力度和可靠读不够，那就是像 release 和 hotfixes 这样的branch，他们不仅需要被 merge 到 develop branch 上，还需要被正确 merge 到 master branch，怎么保证普通的开发者没有漏 merge 而且 reviewer 也刚好没发现呢？本人尚且没有想到很好的办法，感觉也就能在这种 commit 的 review 标题啊，信息啊上做点文章了。

<font color="#ff0000">完美解法：</font>

VD branch model 的作者为了方便开发人员采用 VD branch model，使用 shell 脚本开发了一个名为 git flow 的辅助工具，此工具可以使开发人员不用时时刻刻的想自己有没有违反 VD branch model 的规则，将 VD branch model 的实现细节即各项规则使用程序自动完成，开发者只需站在更高的抽象概念层次上。比如当开发完成一个 hotfixes 后，你不用记住需要先将它 merge 到 master，而且还必须 merge 到 develop 上，最后还需要删除此 hotfixes branch，在此过程中还需要考虑本地这些 branch 一定要与 remote 对应的 branch 同步，你只需要使用 git flow hotfixes finish hotfixes_name 即可。显然这是一个卓越的工具，我们应该毫无疑问的使用它。

可是参杂着 Branch 模型的保护，Code Review，普通开发者，“特定的某些人”，我本以为估计这个东西够呛能被用上了，还小可惜一把，结果没想到在思考上面昨天觉得暂时解决不了的问题的时候，突然想到 git flow 工具可以完美的解决这个问题，而且同时还能使用上 git flow 这个卓越的工具更进一步简化“特定的某些人”的工作。

只要能保证“特定的某些人”一定通过 git flow 这个工具来执行不需要的 FRH branch 的删除工作，就能检查到普通开发者漏 merge 的问题。如使用 git flow hotfixes finish hotfixes\_name 来完成一个 hotfixes 工作，这时候 git flow 会按照 VD branch model 的规则进行 branch 的 merge 动作，具体是首先切换到 master branch 上然后 merge hotfixes\_name 到 master 上，如果普通程序员漏 merge master branch 了，那么马上就被 “特定的某些人”发现这个问题了，即使这时 master branch 已经又进来更新的 commit 了那也不会有问题，因为 hotfixes\_name branch 上已经停止了开发，因为是将 hotfixes\_name branch merge into master branch，所以 master branch 上有新 commit 也没事儿，倒是有可能出现 conflict，不过那也比没发现强太多倍了。如果普通开发人员已经正确将 hotfixes\_name branch merge into master branch 了，那“特定的某些人”执行 git flow 也是安全的，只不过是没有进行 merge 而已，不会有问题。之后将 hotfixes\_name branch merge into develop branch 的过程及原理同上。

最后可以归结为保证“特定的某些人”总是使用 git flow 工具完成一个 FRH 任务就可以了。

  * ##### Pre-Push 高效 Code Review

Code Review 的工具有很多，本人选择的是 Phabricator，也许有人知道这个一开始是 Facebook 内部的工具，后来开发这个的工程师自己出来创业开公司了，这个也是开源的可以免费使用。简单说下其他工具的相对弱点，Gerrit 也是一个用的比较广的，但本人觉得①安装巨复杂②配置烦死人③界面丑痛心④用法也不直观。安装和配置就不说了，界面主要的痛点是它还需要Gitolite 作为 Git repository 的网页浏览，给人一种拼装的感觉不是很好，而且界面的易用性和Phabricator 没法比。Gerrit 开发人员提交 Code Review 的做法是先将 commit push 到一个其他的ref，对于不是很了解 Git 的开发人员使用难度还是比较大，如果 Code Review 没有通过，修改后再提交还涉及到一个 Change-Id 的概念，即使有相应的工具简化操作对于有些开发者可能依然有些吃力。

Phabricator 是 PHP 开发的，Pre-Push Code Review 的实现有一个要求，就是 Git repository 必须得是由它管理的，因为是通过 Git-Hook 的方式实现的，这种情况下如果一个 commit 没有被 review 通过是 push 不到 repository 的。但这个是需要设置的，需要设置一条 Herald rule 如下图步骤1所示，具体设置方法可以参考<a href="http://stackoverflow.com/questions/25662723/phabricator-restrict-git-push" target="_blank">原文</a>。

<a href="http://stackoverflow.com/questions/25662723/phabricator-restrict-git-push" target="_blank"><img title="http://stackoverflow.com/questions/25662723/phabricator-restrict-git-push" style="border-left-width: 0px; border-right-width: 0px; background-image: none; border-bottom-width: 0px; float: none; padding-top: 0px; padding-left: 0px; margin-left: auto; display: block; padding-right: 0px; border-top-width: 0px; margin-right: auto" border="0" alt="http://stackoverflow.com/questions/25662723/phabricator-restrict-git-push" src="/images/13.png" width="640" height="456" /></a>

Phabricator 需要我们提供一个 diff 文件用于 review，所以当开发人员完成开发准备让 reviewers 进行 review 时，需要先将当前的修改和原始代码进行一下 diff 然后用这个 diff 在 Phabricator 里创建一个 review 事务，review 没通过那就继续修改，再产生一个 diff，但这次提交时需要和上一个 review 关联在一起，review 通过了进行 push 动作时也需要附加一些 review 相关的信息，所以使用 Pre-Push Code Review 的话就必须使用他们提供的工具 Arcanist 了。亲测这个工具使用很简单，而且可以设置执行指定的 Lint 工具，开发人员不用了解什么新概念，只需熟悉了之后，每次都那样运行固定的命令就可以了。

Arcanist 不够好的地方也许就是它是用 PHP 写的，所以我们想使用它还得先装 PHP，下载并解压 PHP 到一个目录，然后将这个目录添加到 PATH 环境变量里，之后复制 php.ini-development 文件为 php.ini，并且把“extension\_dir= "ext"”，“extension=php\_curl.dll”，“extension=php_mbstring.dll” 这几行行首的分号去掉。Arcanist 本身的安装其实就是下载下来然后将 arc.bat 所在目录加到 PATH 环境变量里。

Arcanist 安装完后还需要配置，主要是三件事情：

  * 设置你们 Phabricator 服务器的网址，让 Arcanist 能找到并自动在 Phabricator 上帮你完成工作，这里可以通过 arc set-config default http://phabricator_host/ 来设置一个默认的Phabricator 服务器的网址，也可以在允许 arc 命令的目录下创建一个 .arcconfig 文件，将一些信息写在这个里面，具体用法请参考<a href="https://secure.phabricator.com/book/phabricator/article/arcanist_windows/" target="_blank">官方文档</a>。 
  * 安装证书，arcinstall-certificate，依照提示访问 Arcanist 说的那个网址，打开网址后复制内容粘贴回来即可。 
  * 设置编辑器，windows 下它支持 Notepad++，具体方法参考<a href="https://secure.phabricator.com/book/phabricator/article/arcanist_windows/" target="_blank">官方文档</a>。亲测有一点问题，使用 Arcanist 时需要 Notepad++ 是没有启动的状态，也就是说如果你现在用 Notepad++ 正在打开着一个其他的文件，那么会出错的。 

虽然 Phabricator 创建 review 的时候是让你发送 diff 而不是完整文档，但 reviewer 在 review 的时候是完全可以查看到完整文件的，而且界面做的和 GitHub 似的，非常好用，针对每一行进行注释，搜索，通知，查看仓库，任务管理，bug track，wiki，activity 都有，所以感觉很棒。

  * ##### 普通开发人员开发流程

经过以上 branch model 保护策略 + Pre-Push Code Review，确定了普通开发人员一定是在两个前提下进行开发的：①一定是在已有的某个 FRH branch 上进行开发。②可能多个开发人员工作在同一个 FRH branch 上，都会向这个 FRH branch push commit。

本人觉得最好的流程为：

  1. Clone remote repository to local。 
  2. Checkout 到 相应的 FRH branch 上。 
  3. Checkout –b 建立一个 private 的 branch。 
  4. 进行开发，也许需要进行多次 commit，完成开发。 
  5. Checkout 回相应的 FRH branch 上。 
  6. Pull（一定不会有 merge 发生）。 
  7. Checkout 回 private branch上。 
  8. 对相应的 FRH branch 进行 rebase（如果失败就放弃 rebase，继续下一步）。 
  9. 再 Checkout 回相应的 FRH branch 上。 
 10. . Merge（可以优先考虑用 –squash 方式）private branch，并 commit。 
 11. . 使用 Arcanist 进行 Pre-Push Code Review 并等待通过。 
 12. . Push 到 remote repository，并删除 private branch，如果 remote repository 已经更新了，此 pull 被拒绝，我们还需要再进行 pull，这时尽量要用 pull 的 rebase 方式，如果不行再用默认的 pull，完成后到 11 继续循环。 

很多开发人员懒得再建立一个 private branch，对于这样的开发者，以上流程简化为：

  1. Clone remote repository to local。 
  2. Checkout 到 相应的 FRH branch 上。 
  3. 进行开发，也许需要进行多次 commit，完成开发。 
  4. 使用 Arcanist 进行 Pre-Push Code Review 并等待通过。 
  5. Push 到 remote repository，如果 remote repository 已经更新了，此 pull 被拒绝，我们还需要再进行 pull，这时尽量要用 pull 的 rebase 方式，如果不行再用默认的 pull，完成后到 4 继续循环。 

  * ##### Unit Test

单元测试之重要，非简短的小程序所能提现的，如果开发人员只有你自己或者很少的几个人，也许你们会觉得它既耽误时间又起不到集成测试的效果。多部门合作式的开发一款稍复杂些的项目就必然会遇到如下问题：

  * 测试组同事说程序被测出了 bug，开发组同事去一看发现不是 bug，是缺一个设置。 
  * 测试组同事设置好了之后又测，说这次确实有 bug 了，开发组同事一看说，这也不是 bug，测这个的时候需要将刚刚那个设置设为另一个值。 
  * 测试组同事只能每次改一下，再测一下，很苦恼，说希望能让他自由一点，因为他有的时候需要将所有的 test case 都测，有的时候又只想测某一个或某几个，开发组同事废了九牛二虎之力实现了，结果等下一个项目的时候这套实现又不适用了，还需要再根据那个项目实现一次。几次之后，开发组同事不鸟测试组同事了。 
  * 功能 A 测试的差不多了，大家开始把主要精力放在了 功能 B 上，不管是测试组同事还是开发组同事都能熟练的调整功能 B 的测试参数，高效的完成测试，就像他们曾经测试功能 A 那样。功能 B 快接近尾声了，功能 A 在这期间又不多不少的添加了几个修改，但大家都觉得这么小的修改肯定没事儿，不可能有错就没测。为了集成功能 A 和 B，现在要再跑一次完整的 A 的测试，然后和 B 进行集成，结果不管是测试组同事还是开发组同事都记不太清功能 A 的完整测试需要跑那些了。&#160; 
  * 对一个之前仓促实现的功能进行重构，但谁都不想接收这个任务，因为大家都知道重构不是事儿，关键是你不知道怎么能保证你重构完后的代码和以前的逻辑一样，没有单元测试留下来，重构了后会陷入到茫茫 bug 中。 
  * 开发人员 A 修改了一段代码，自己的任务完成的很好，测试通过，高兴的提交了。开发人员 B 开发一个新功能，但这个新功能基于他之前的一个旧 API，这个 API 已经经过了完整的测试并多次被开发人员 B 使用在其开发中。结果这个新功能怎么也出不来正确的结果，开发人员 B 调啊调，终于发现用的那个 API 的输出就不对，疑惑，不可能啊，我根本没动过它。结果发现是开发人员 A 的那次修改影响了这个 API 的功能。 

单元测试必须能让每个开发人员非常轻松的一键式的运行程序中所有的 test case，而不只是他自己开发的那部分 test case，要不然的话，跟没有一样。就拿本人工作来说，我们就没有一种方式能很容易的运行所有的 test case，虽然我们开会都会说大家开发完后，一定要把所有 test case 都跑一下，这样才可靠，然而大家也都是当时点点头。刚入职我还对这些老人很不屑，心想这是必须要做的，怎么能形式化的走这么个过场就完了呢？后来我发现我也不想运行所有的 test case 因为我都不知道怎么运行另一个同事的 test case，甚至看到他写的那恶心代码我都想赶快点儿下班。

所以 Unit Test 不是简单的测试函数，它起码得是个框架，如果没法用现有的优秀框架，可以参考优秀框架实现一个自己的版本，但一定不要让你们团队里面的二流选手做，要不然~还不如不知道这个事儿呢。

本人平时自己开发喜欢使用 C++，所以关注 C++ 的 Unit Test 框架多一些，现选择 Google 开源的 gtest 使用，一般的 C 语言程序也用它，感觉很棒，而且 gtest 还有与之搭配的 gmock，如有兴趣使用请见其 <a href="https://github.com/google/googletest" target="_blank">GitHub 仓库</a>。

  * ##### Coding Style

Coding Style 是个极其容易引起争议的东西，统一的 Coding Style 的好处我想根本不用在这里废话，大家都心知肚明。以前我也觉得这样的 Coding Style 不好，那样的有部分好，所以自己会根据一些参考自己弄一个小笔记一点一点总结，从命名规范到注释规范，结果随着时间的推移会发现，你以前觉得你自己的规范简洁明了，等把所有的东西都收罗在一起时，你的这个版本还不如一些成熟的大公司的规范来的更好。

但这并不代表我们就应该随便跟一些大公司的风，因为有很多所谓的大公司，其实他们的技术很烂，国内的公司大多不爱开源，也没有让我看到很有水平的 Coding Style 规范，之前主要参见微软的 Coding Style，但显然它更适合 C#。知道我知道 Google 的 Coding Style，让我觉得没有必要再自己记小册子了，直接用 Google 的来吧，经过我的仔细阅读后，发现直接完全遵循就可以了，每一条都是经过千锤百炼的。而且最近在看 Code Review 的知识的时候也看到一个原 Google 的高级开发人员给队员讲解为什么直接遵循就好了的原因，说那些命名规则什么的不是简单的争论出来的，都是经过美学实验的。

Google 的 Coding Style 没有很多公司的 Coding Style 的明显错误，这就能看出来制定这套 style 的人不是一般的开发人员，所以你依赖于它，可信度和可靠度都是很高的，不会让你白费力气的。比如很多公司的 style 都有的一个问题，那就是给变量的名字前加上一个类型前缀，如 unsinged char ucUserName = ‘A’; 如果你有比较深的功力，然后站在当代这个时间点上好好思考下这个东西有什么用，你会得出屁用没有的结论。好多没水平只会人云亦云的人只会不假思索地的说，这样我一下就能看出它是个什么类型。他们永远不会把问题再深入的想一步，再深入想一步你就会问，so what？知道类型干什么？其实这种写法大部分人是从微软失败的垃圾产品 MFC 学来的，殊不知人家早已经弃用了，如果在上个世纪，那它还是有好处的，为什么呢？那时候的开发工具多简陋呀，你看到一个变量，想知道它是什么类型你得自己去找，所以加上这些前缀很方便，但现在几乎是个编辑器就能非常只能的跳转到定义，根本就不需要这个东西。另外设计这种 Coding Style 的那哥们儿的初衷也不是为了看的出一个变量的类型，而是说因为有这个类型前缀，所以你很容易发现一个类型不相符的赋值错误，然而现在这种错误在你敲完那行代码就能被编辑器发现并提示出来。

再好的 Coding Style 也不能光靠一个文档和人眼来保证一致性，这种强规则的工作必然是工具比人更胜任，在精确的工具面前，人显得太不可靠了。Google C++ Coding Style 有非常好的 lint 工具，可以帮助我们检查我们的代码是否符合了 Google C++ Coding Style，叫 cpplint.py 显然它是一个 python 程序。发现一个 cpplint.py 工具与 Google C++ Coding Style 不一致的地方，Google C++ Coding Style 中规定 #define Guard 的命名需要以 project name 开头，但亲测 cpplint.py 缺要求不加 project name，有别人也遇到了这个问题，详情可以参见<a href="https://github.com/google/styleguide/issues/138#issuecomment-230085211" target="_blank">此 issue</a>。

Code Review 的时候，绝不能成为单纯的 Coding Style 的 人工检查，要将精力放在代码质量上，所以我们需要用 lint 工具帮助我们完成 Code Review 中的 Coding Style 的检查，Phabricator 可以配置 lint 工具，这刚好能帮助我们，可见本文的这一套架构中的各工具非常好的进行了相互协作。Google 还有很多别的语言的 Coding Style，都放在 Google Style Guides 项目上，详情请见其 <a href="https://github.com/google/styleguide" target="_blank">GitHub 仓库</a>。

  * ##### 自动化构建及持续集成

自动化构建和持续集成的做法一般是发现有新的 code 被 push 到 repository 就开始用这份新 code 编译，连接，运行测试。这种做法能很快的发现 repository 内 code 的问题，code 一旦进入到 repository，对于 Pre-Push 的 Code Review 已经就无能为力了，那么使用自动化技术进行持续集成，就能再进一步的提高我们 code 的质量。对于大型的项目，有可能对所有配置进行一次构建需要很多步骤很麻烦，开发人员和 Code Review 人员很可能只是对某一种配置进行了构建及测试，所以必须要使用持续集成来保证新提交的 code 没有污染 repository。

对这部分的知识并没有做太多功课，现阶段只是选定 CMake 做 C++ 项目的构建工具，CMake 可以自动生成出绝大部分其他构建工具比如 Visual Studio 所需的文件，这样用 CMake 作为项目的构建工具，如果想在 Windows 上想用 Visual Studio 开发，就可以使用 CMake 直接生成出 Visual Studio 的工程，很方便，同时在可以在 linux server 上直接用 CMake 进行自动化构建，减轻了开发者很多的工作。相比单纯的 make 或 Autoconf，CMake 将开发人员从那些恶心的设计中解脱了出来，尤其看到 Autoconf 那一大推，真的感觉太蠢了。

持续集成我知道的就是 Jenkins 但没有详细研究过，不过相信不错，使用 Jenkins 配合 CMake 进行持续集成还有待学习。

  * ##### 缺少的部分

到目前为止，让一个开发团队健康的运转起来所需的大部分东西都有了，不过测试这部分还是一个缺失项，成熟的优秀公司是怎么进行测试的？测试整个这个事情与开发之间是怎么结合的？这部分还有很多待学习研究的东西。
