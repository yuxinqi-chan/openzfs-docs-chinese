Git 和 GitHub 初学者指南 (ZoL 版)
==========================================

这是一个非常基础的指南，介绍如何使用 Git 和 GitHub 进行更改。

推荐阅读: `ZFS on Linux
CONTRIBUTING.md <https://github.com/zfsonlinux/zfs/blob/master/.github/CONTRIBUTING.md>`__

首次设置
----------------

如果你从未使用过 Git，你需要进行一些设置才能开始。

::

   git config --global user.name "我的名字"
   git config --global user.email myemail@noreply.non

克隆初始仓库
------------------------------

最简单的方法是点击主仓库页面顶部的 fork 图标。然后你需要将 fork 的仓库下载到你的电脑上：

::

   git clone https://github.com/<你的账户名>/zfs.git

这将把你的 fork 仓库设置为 "origin"。这在创建拉取请求时会很有用。为了在更改时从 "upstream" 仓库拉取，将上游仓库添加为另一个远程仓库非常有用（参见 man git-remote）：

::

   cd zfs
   git remote add upstream https://github.com/zfsonlinux/zfs.git

准备和进行更改
----------------------------

为了进行更改，建议创建一个分支，这可以让你同时处理多个不相关的更改。除非你拥有该仓库，否则不建议在 master 分支上进行更改。

::

   git checkout -b 我的新分支

从这里开始，你可以进行更改并继续下一步。

推荐阅读: `C 风格和编码标准
SunOS <https://www.cis.upenn.edu/~lee/06cse480/data/cstyle.ms.pdf>`__,
`ZFS on Linux 开发者资源 <https://github.com/zfsonlinux/zfs/wiki/Developer-Resources>`__,
`OpenZFS 开发者资源 <https://openzfs.org/wiki/Developer_resources>`__

推送前测试你的补丁
-----------------------------------

在提交和推送之前，你可能想测试你的补丁。你可以对你的分支运行多种测试，例如风格检查和功能测试。所有拉取请求在推送到主仓库之前都会经过这些测试，但在本地测试可以减轻构建/测试服务器的负担。此步骤是可选的，但强烈推荐，不过测试套件应在虚拟机或当前未使用 ZFS 的主机上运行。你可能需要安装 ``shellcheck`` 和 ``flake8`` 以正确运行 ``checkstyle``。

::

   sh autogen.sh
   ./configure
   make checkstyle

推荐阅读: `构建 ZFS <https://github.com/zfsonlinux/zfs/wiki/Building-ZFS>`__, `ZFS 测试套件
README <https://github.com/zfsonlinux/zfs/blob/master/tests/README.md>`__

提交你的更改以推送
------------------------------------

当你完成对分支的更改后，在创建拉取请求之前还有几个步骤。

::

   git commit --all --signoff

此命令会打开一个编辑器并添加你分支中所有未暂存的文件。在这里你需要描述你的更改并添加一些内容：

::


   # 请输入你的更改的提交信息。以 '#' 开头的行将被忽略，
   # 空消息将中止提交。
   # 位于分支 我的新分支
   # 要提交的更改：
   #   (使用 "git reset HEAD <文件>..." 取消暂存)
   #
   #   修改:   hello.c
   #

我们需要添加的第一件事是提交信息。这是显示在 git log 中的内容，应该是对更改的简短描述。根据风格指南，此信息必须少于 72 个字符。

在提交信息下方，你可以为提交添加更详细的描述。此部分的行数必须少于 72 个字符。

完成后，提交应如下所示：

::

   添加 hello 命令

   这是一个带有描述性提交信息的测试提交。
   此信息可以像这里显示的那样多行。

   Signed-off-by: 我的名字 <myemail@noreply.non>
   Closes #9998
   Issue #9999
   # 请输入你的更改的提交信息。以 '#' 开头的行将被忽略，
   # 空消息将中止提交。
   # 位于分支 我的新分支
   # 要提交的更改：
   #   (使用 "git reset HEAD <文件>..." 取消暂存)
   #
   #   修改:   hello.c
   #

如果你正在为现有问题提交拉取请求，你还可以引用问题和拉取请求，如上所示。完成后保存并退出编辑器。

推送并创建拉取请求
-------------------------------------

最后一步。你已经完成了更改并提交了。现在是时候推送它了。

::

   git push --set-upstream origin 我的新分支

这将要求你输入 GitHub 凭据并将你的更改上传到你的仓库。

最后一步是转到你的仓库或上游仓库的 GitHub 页面，你应该会看到一个按钮，用于为你最近提交的分支创建新的拉取请求。

修正拉取请求中的问题
----------------------------------------

有时事情并不总是按计划进行，你可能需要更新你的拉取请求以修正提交信息或更改。这可以通过重新推送你的分支来完成。如果你需要进行代码更改或 ``git add`` 文件，你现在可以进行这些操作，然后执行以下命令：

::

   git commit --amend
   git push --force

这将返回到提交编辑器屏幕，并将你的更改推送到旧更改之上。请注意，这将重新启动任何正在运行的构建/测试服务器的进程，频繁推送可能会导致所有拉取请求的处理延迟。

维护你的仓库
---------------------------

当你希望在未来进行更改时，你将需要一个最新的上游仓库副本来进行更改。以下是保持更新的方法：

::

   git checkout master
   git pull upstream master
   git push origin master

这将确保你在仓库的 master 分支上，从上游获取更改，然后将它们推回你的仓库。

最后的建议
-----------

这是一个非常基础的 Git 和 GitHub 介绍，但应该能让你开始为许多开源项目做出贡献。并非所有项目都有风格要求，有些项目可能有不同的提交更改流程，因此请参考他们的文档以查看是否需要做任何不同的事情。我们尚未涉及的一个主题是 ``git rebase`` 命令，这对于本维基文章来说有点高级。

额外资源: `Github 帮助 <https://help.github.com/>`__,
`Atlassian Git 教程 <https://www.atlassian.com/git/tutorials>`__