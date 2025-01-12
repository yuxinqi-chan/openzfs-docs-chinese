OpenZFS 补丁
=============

ZFS on Linux 项目是对上游 `OpenZFS 仓库 <https://github.com/openzfs/openzfs/>`__ 的适配，旨在在 Linux 环境中工作。这个上游仓库是一个集成了来自所有 OpenZFS 平台的新功能、错误修复和性能改进的地方。每个平台负责跟踪 OpenZFS 仓库，并将相关改进合并到他们的版本中。

对于 ZFS on Linux 项目，这种跟踪是通过一个 `OpenZFS 跟踪 <http://build.zfsonlinux.org/openzfs-tracking.html>`__ 页面来管理的。该页面会定期更新，并显示 OpenZFS 提交列表及其在 ZFS on Linux 主分支中的状态。

本页面描述了将未完成的 OpenZFS 提交应用到 ZFS on Linux 并提交这些更改以包含的过程。作为开发者，这是熟悉 ZFS on Linux 并快速为项目做出有价值贡献的好方法。以下指南假设您已经有一个 `GitHub 账户 <https://help.github.com/articles/signing-up-for-a-new-github-account/>`__，熟悉 git，并且习惯在 Linux 环境中开发。

将 OpenZFS 更改移植到 ZFS on Linux
---------------------------------------

环境设置
~~~~~~~~~~~~~

**克隆源代码。** 首先，克隆 `spl <https://github.com/zfsonlinux/spl>`__ 和 `zfs <https://github.com/zfsonlinux/zfs>`__ 仓库到本地。

::

   $ git clone -o zfsonlinux https://github.com/zfsonlinux/spl.git
   $ git clone -o zfsonlinux https://github.com/zfsonlinux/zfs.git

**添加远程仓库。** 使用 GitHub 网页界面 `fork <https://help.github.com/articles/fork-a-repo/>`__ `zfs <https://github.com/zfsonlinux/zfs>`__ 仓库到您的个人 GitHub 账户。将您的新 zfs fork 和 `openzfs <https://github.com/openzfs/openzfs/>`__ 仓库添加为远程仓库，然后获取这两个仓库。OpenZFS 仓库很大，初始获取在慢速连接上可能需要一些时间。

::

   $ cd zfs 
   $ git remote add <your-github-account> git@github.com:<your-github-account>/zfs.git
   $ git remote add openzfs https://github.com/openzfs/openzfs.git
   $ git fetch --all

**编译源代码。** 编译 spl 和 zfs 的主分支。这些分支始终保持稳定，这是验证您是否安装了完整的构建环境以及所有必需的依赖项是否可用的有用方法。这也可能加快后续小补丁的编译时间，因为增量构建是一个选项。

::

   $ cd ../spl
   $ sh autogen.sh && ./configure --enable-debug && make -s -j$(nproc)
   $
   $ cd ../zfs
   $ sh autogen.sh && ./configure --enable-debug && make -s -j$(nproc)

选择一个补丁
~~~~~~~~~~~~

查阅 `OpenZFS 跟踪 <http://build.zfsonlinux.org/openzfs-tracking.html>`__ 页面，并选择一个尚未应用的补丁。对于您的第一个补丁，您应该选择一个小补丁以熟悉该过程。

移植补丁
~~~~~~~~~~~~~

有两种方法：

-  `cherry-pick（更简单） <#cherry-pick>`__
-  `手动合并 <#manual-merge>`__

请先阅读 `手动合并 <#manual-merge>`__ 以了解整个过程。

Cherry-pick
^^^^^^^^^^^

您可以自己开始 `cherry-pick <https://git-scm.com/docs/git-cherry-pick>`__，但我们提供了一个特殊的 `脚本 <https://github.com/zfsonlinux/zfs-buildbot/blob/master/scripts/openzfs-merge.sh>`__，它会尝试自动 `cherry-pick <https://git-scm.com/docs/git-cherry-pick>`__ 补丁并生成描述。

0) 准备环境：

必需的 git 设置（添加到 ``~/.gitconfig``）：

::

   [merge]
       renameLimit = 999999
   [user]
       email = mail@yourmail.com
       name = Your Name

下载脚本：

::

   wget https://raw.githubusercontent.com/zfsonlinux/zfs-buildbot/master/scripts/openzfs-merge.sh

1) 运行：

::

   ./openzfs-merge.sh -d path_to_zfs_folder -c openzfs_commit_hash

此命令将获取所有仓库，创建一个新分支 ``autoport-ozXXXX``（XXXX - OpenZFS 问题编号），尝试 cherry-pick，编译并在成功时检查 cstyle。

如果它成功且没有任何合并冲突 - 转到 ``autoport-ozXXXX`` 分支，它将有一个准备提交的提交。恭喜，您可以转到步骤 7！

否则，您应该转到步骤 2。

2) 手动解决所有合并冲突。简单的方法 - 安装 `Meld <http://meldmerge.org/>`__ 或任何其他差异工具并运行 ``git mergetool``。

3) 检查所有编译和 cstyle 错误（参见 `测试补丁 <#testing-a-patch>`__）。

4) 提交您的更改并附上任何描述。

5) 更新提交描述（最后一次提交将被更改）：

::

   ./openzfs-merge.sh -d path_to_zfs_folder -g openzfs_commit_hash

6) 添加任何移植说明（如果您修改了某些内容）：``git commit --amend``

7) 将您的提交推送到 GitHub：``git push <your-github-account> autoport-ozXXXX``

8) 创建一个拉取请求到 ZoL 主分支。

9) 转到 `测试补丁 <#testing-a-patch>`__ 部分。

手动合并
^^^^^^^^^^^^

**创建一个新分支。** 为每个移植到 ZFS on Linux 的提交创建一个新分支非常重要。这将允许您轻松地将您的工作作为 GitHub 拉取请求提交，并使您能够同时处理多个 OpenZFS 更改。所有开发分支都需要基于 ZFS 主分支，并且以您正在处理的问题编号命名分支会很有帮助。

::

   $ git checkout -b openzfs-<issue-nr> master

**生成补丁。** 您首先会注意到的一件事是，ZFS on Linux 仓库的布局与 OpenZFS 仓库不同。在组织上，它要扁平得多，这是因为它只包含 OpenZFS 的代码，而不是整个操作系统。这意味着，为了应用来自 OpenZFS 的补丁，补丁中的路径名称必须更改。提供了一个名为 zfs2zol-patch.sed 的脚本来执行此转换。使用 ``git format-patch`` 命令和此脚本来生成补丁。

::

   $ git format-patch --stdout <commit-hash>^..<commit-hash> | \
       ./scripts/zfs2zol-patch.sed >openzfs-<issue-nr>.diff

**应用补丁。** 在许多情况下，生成的补丁将干净地应用到仓库。但是，请记住，zfs2zol-patch.sed 脚本仅转换路径。通常还有其他原因导致补丁可能无法应用。在某些情况下，补丁的部分内容可能不适用于 Linux，应该删除。在其他情况下，补丁可能依赖于必须先应用的其他更改。这些更改也可能与 Linux 特定的修改冲突。在所有这些情况下，都需要手动修改补丁以干净地应用，同时保留其原始意图。

::

   $ git am ./openzfs-<commit-nr>.diff

**更新提交消息。** 通过使用 ``git format-patch`` 生成补丁，然后使用 ``git am`` 应用它，原始评论和作者身份将得以保留。但是，由于 OpenZFS 提交的格式，您可能会发现整个提交评论都被压缩到了主题行中。使用 ``git commit --amend`` 清理评论，并小心遵循 `这些标准指南 <http://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html>`__。

OpenZFS 提交的摘要行通常很长，您应该将其截断为 50 个字符。这很有用，因为它保留了 ``git log --pretty=oneline`` 命令的正确格式。确保在摘要和提交正文之间留一个空行。然后包括完整的 OpenZFS 提交消息，换行任何超过 72 个字符的行。最后，添加一个 ``Ported-by`` 标签，包含您的联系信息，以及一个 ``OpenZFS-issue`` 和 ``OpenZFS-commit`` 标签，并附上适当的链接。您需要验证您的提交包含以下所有信息：

-  原始 OpenZFS 补丁的主题行，格式为："OpenZFS <issue-nr> - 简短描述"。
-  原始补丁的作者身份应保留。
-  OpenZFS 提交消息。
-  以下标签：

   -  **Authored by:** 原始补丁作者
   -  **Reviewed by:** 原始补丁的所有 OpenZFS 审阅者。
   -  **Approved by:** 原始补丁的所有 OpenZFS 审阅者。
   -  **Ported-by:** 您的姓名和电子邮件地址。
   -  **OpenZFS-issue:** https ://www.illumos.org/issues/issue
   -  **OpenZFS-commit:** https
      ://github.com/openzfs/openzfs/commit/hash

-  **移植说明:** 可选部分，描述移植时所需的任何更改。

例如，OpenZFS 问题 6873 已 `应用到 Linux <https://github.com/zfsonlinux/zfs/commit/b3744ae>`__，来自此上游 `OpenZFS 提交 <https://github.com/openzfs/openzfs/commit/ee06391>`__。

::

   OpenZFS 6873 - zfs_destroy_snaps_nvl 泄漏 errlist
      
   Authored by: Chris Williamson <chris.williamson@delphix.com>
   Reviewed by: Matthew Ahrens <mahrens@delphix.com>
   Reviewed by: Paul Dagnelie <pcd@delphix.com>
   Ported-by: Denys Rtveliashvili <denys@rtveliashvili.name>
       
   lzc_destroy_snaps() 返回一个 nvlist 在 errlist 中。
   zfs_destroy_snaps_nvl() 应该在返回之前 nvlist_free() 它。
       
   OpenZFS-issue: https://www.illumos.org/issues/6873
   OpenZFS-commit: https://github.com/openzfs/openzfs/commit/ee06391

测试补丁
~~~~~~~~~~~~~

**编译源代码。** 验证修补后的源代码可以无错误地编译，并解决所有警告。

::

   $ make -s -j$(nproc)

**运行样式检查器。** 验证修补后的源代码通过样式检查器，命令应返回而不打印任何输出。

::

   $ make cstyle

**打开拉取请求。** 当您的补丁干净地构建并通过样式检查时，`打开一个新的拉取请求 <https://help.github.com/articles/creating-a-pull-request/>`__。拉取请求将排队进行 `自动化测试 <https://github.com/zfsonlinux/zfs-buildbot/>`__。作为测试的一部分，该更改将针对广泛的 Linux 发行版进行构建，并运行一系列功能和压力测试以检测回归。

::

   $ git push <your-github-account> openzfs-<issue-nr>

**修复任何问题。** 测试大约需要 2 小时才能完全完成，结果将发布在 GitHub `拉取请求 <https://github.com/zfsonlinux/zfs/pull/4594>`__ 中。所有测试都应通过，您应该调查并解决任何测试失败。`测试脚本 <https://github.com/zfsonlinux/zfs-buildbot/tree/master/scripts>`__ 都可用，并且设计为在本地运行以重现问题。一旦您解决了问题，强制更新拉取请求以触发新一轮测试。迭代直到所有测试都通过。

::

   # 修复问题，修改提交，强制更新分支。
   $ git commit --amend
   $ git push --force <your-github-account> openzfs-<issue-nr>

合并补丁
~~~~~~~~~~~~~~~~~

**审查。** 最后，ZFS on Linux 的维护者之一将对补丁进行最终审查，并可能请求额外的更改。一旦维护者对补丁的最终版本感到满意，他们将添加他们的签名，将其合并到主分支，在跟踪页面上标记为完成，并感谢您对项目的贡献！

将 ZFS on Linux 更改移植到 OpenZFS
---------------------------------------

通常，问题会首先在 ZFS on Linux 中修复或开发新功能。非 Linux 特定的更改应提交到上游 OpenZFS GitHub 仓库进行审查。此过程在 `OpenZFS README <https://github.com/openzfs/openzfs/>`__ 中有描述。