Buildbot 选项
================

在提交级别上，有多种方式可以控制 ZFS Buildbot。本页提供了 ZFS Buildbot 支持的各种选项的摘要及其对测试的影响。有关其实现的更多详细信息，请访问 `ZFS Buildbot Github 页面 <https://github.com/zfsonlinux/zfs-buildbot>`__。

选择构建器
-----------------

默认情况下，ZFS 拉取请求中的所有提交都会由 BUILD 构建器进行编译。此外，ZFS 拉取请求的顶部提交会由 TEST 构建器进行测试。然而，您可以选择在每个提交的基础上覆盖应使用的构建器类型。在这种情况下，您可以在提交消息中添加
``Requires-builders: <none|all|style|build|arch|distro|test|perf|coverage|unstable>``。可以提供逗号分隔的选项列表。支持的选项有：

-  ``all``：此提交应由所有可用的构建器构建
-  ``none``：此提交不应由任何构建器构建
-  ``style``：此提交应由 STYLE 构建器构建
-  ``build``：此提交应由所有 BUILD 构建器构建
-  ``arch``：此提交应由标记为“架构”的 BUILD 构建器构建
-  ``distro``：此提交应由标记为“发行版”的 BUILD 构建器构建
-  ``test``：此提交应由 TEST 构建器构建和测试（不包括 Coverage TEST 构建器）
-  ``perf``：此提交应由 PERF 构建器构建和测试
-  ``coverage``：此提交应由 Coverage TEST 构建器构建和测试
-  ``unstable``：此提交应由 Unstable TEST 构建器构建和测试（目前仅限 Fedora Rawhide TEST 构建器）

以下是一些如何在提交消息中使用 ``Requires-builders:`` 的示例。

.. _防止提交被构建和测试:

防止提交被构建和测试
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   这是一个提交消息

   此文本是提交消息正文的一部分。

   Signed-off-by: 贡献者 <contributor@email.com>
   Requires-builders: none

.. _将提交仅提交给 STYLE 和 TEST 构建器:

将提交仅提交给 STYLE 和 TEST 构建器
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   这是一个提交消息

   此文本是提交消息正文的一部分。

   Signed-off-by: 贡献者 <contributor@email.com>
   Requires-builders: style test

要求 SPL 版本
----------------------

目前，ZFS Buildbot 尝试根据拉取请求的基础分支选择正确的 SPL 分支进行构建。在需要构建特定 SPL 版本的情况下，ZFS Buildbot 支持为拉取请求测试指定 SPL 版本。通过打开针对 ZFS 的拉取请求并在提交消息中添加 ``Requires-spl:``，您可以指示 Buildbot 使用特定的 SPL 版本。以下是指定 SPL 版本的提交消息示例。

从特定拉取请求构建 SPL
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   这是一个提交消息

   此文本是提交消息正文的一部分。

   Signed-off-by: 贡献者 <contributor@email.com>
   Requires-spl: refs/pull/123/head

从 ``zfsonlinux/spl`` 仓库构建 SPL 分支 ``spl-branch-name``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   这是一个提交消息

   此文本是提交消息正文的一部分。

   Signed-off-by: 贡献者 <contributor@email.com>
   Requires-spl: spl-branch-name

要求内核版本
------------------------

目前，Kernel.org 构建器将克隆并构建 Linux 的主分支。在需要构建特定版本的 Linux 内核的情况下，ZFS Buildbot 支持通过提交消息指定要构建的 Linux 内核。通过打开针对 ZFS 的拉取请求并在提交消息中添加 ``Requires-kernel:``，您可以指示 Buildbot 使用特定的 Linux 内核。以下是指定特定 Linux 内核标签的提交消息示例。

.. _构建 Linux 内核版本 4.14:

构建 Linux 内核版本 4.14
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   这是一个提交消息

   此文本是提交消息正文的一部分。

   Signed-off-by: 贡献者 <contributor@email.com>
   Requires-kernel: v4.14

构建步骤覆盖
---------------------

每个构建器将根据其默认偏好执行或跳过构建步骤。在某些情况下，可能会跳过各种构建步骤。ZFS Buildbot 支持在提交消息中覆盖所有构建器的默认设置。可用的覆盖选项包括：

-  ``Build-linux: <Yes|No>``：所有构建器应为此提交构建 Linux
-  ``Build-lustre: <Yes|No>``：所有构建器应为此提交构建 Lustre
-  ``Build-spl: <Yes|No>``：所有构建器应为此提交构建 SPL
-  ``Build-zfs: <Yes|No>``：所有构建器应为此提交构建 ZFS
-  ``Built-in: <Yes|No>``：所有 Linux 构建应内置 SPL 和 ZFS
-  ``Check-lint: <Yes|No>``：所有构建器应为此提交执行 lint 检查
-  ``Configure-lustre: <options>``：构建 Lustre 时提供 ``<options>`` 作为配置标志
-  ``Configure-spl: <options>``：构建 SPL 时提供 ``<options>`` 作为配置标志
-  ``Configure-zfs: <options>``：构建 ZFS 时提供 ``<options>`` 作为配置标志

以下是一些如何在提交消息中使用覆盖的示例。

跳过构建 SPL 并在没有 ldiskfs 的情况下构建 Lustre
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

::

   这是一个提交消息

   此文本是提交消息正文的一部分。

   Signed-off-by: 贡献者 <contributor@email.com>
   Build-lustre: Yes
   Configure-lustre: --disable-ldiskfs
   Build-spl: No

仅构建 ZFS
~~~~~~~~~~~~~~

::

   这是一个提交消息

   此文本是提交消息正文的一部分。

   Signed-off-by: 贡献者 <contributor@email.com>
   Build-lustre: No
   Build-spl: No

使用 TEST 文件配置测试
------------------------------------

在 ZFS 源代码树的顶层，有一个 `TEST 文件 <https://github.com/zfsonlinux/zfs/blob/master/TEST>`__，其中包含控制特定测试是否以及如何运行的变量。以下是每个变量的列表及其控制的简要描述。

-  ``TEST_PREPARE_WATCHDOG`` - 启用 Linux 内核看门狗
-  ``TEST_PREPARE_SHARES`` - 启动 NFS 和 Samba 服务器
-  ``TEST_SPLAT_SKIP`` - 确定是否跳过 ``splat`` 测试
-  ``TEST_SPLAT_OPTIONS`` - 提供给 ``splat`` 的命令行选项
-  ``TEST_ZTEST_SKIP`` - 确定是否跳过 ``ztest`` 测试
-  ``TEST_ZTEST_TIMEOUT`` - ``ztest`` 应运行的时长
-  ``TEST_ZTEST_DIR`` - ``ztest`` 创建 vdevs 的目录
-  ``TEST_ZTEST_OPTIONS`` - 传递给 ``ztest`` 的选项
-  ``TEST_ZTEST_CORE_DIR`` - ``ztest`` 存储核心转储的目录
-  ``TEST_ZIMPORT_SKIP`` - 确定是否跳过 ``zimport`` 测试
-  ``TEST_ZIMPORT_DIR`` - ``zimport`` 期间使用的目录
-  ``TEST_ZIMPORT_VERSIONS`` - 要测试的源版本
-  ``TEST_ZIMPORT_POOLS`` - ``zimport`` 用于测试的池名称
-  ``TEST_ZIMPORT_OPTIONS`` - 提供给 ``zimport`` 的命令行选项
-  ``TEST_XFSTESTS_SKIP`` - 确定是否跳过 ``xfstest`` 测试
-  ``TEST_XFSTESTS_URL`` - 下载 ``xfstest`` 的 URL
-  ``TEST_XFSTESTS_VER`` - 从 ``TEST_XFSTESTS_URL`` 下载的 tarball 名称
-  ``TEST_XFSTESTS_POOL`` - 为 ``xfstest`` 创建和使用的池名称
-  ``TEST_XFSTESTS_FS`` - ``xfstest`` 使用的数据集名称
-  ``TEST_XFSTESTS_VDEV`` - ``xfstest`` 使用的 vdev 名称
-  ``TEST_XFSTESTS_OPTIONS`` - 提供给 ``xfstest`` 的命令行选项
-  ``TEST_ZFSTESTS_SKIP`` - 确定是否跳过 ``zfs-tests`` 测试
-  ``TEST_ZFSTESTS_DIR`` - 存储文件和回环设备的目录
-  ``TEST_ZFSTESTS_DISKS`` - ``zfs-tests`` 允许使用的磁盘列表（以空格分隔）
-  ``TEST_ZFSTESTS_DISKSIZE`` - ``zfs-tests`` 使用的基于文件的 vdev 的文件大小
-  ``TEST_ZFSTESTS_ITERS`` - ``test-runner`` 应执行其测试集的次数
-  ``TEST_ZFSTESTS_OPTIONS`` - 提供给 ``zfs-tests`` 的选项
-  ``TEST_ZFSTESTS_RUNFILE`` - 运行 ``zfs-tests`` 时使用的运行文件
-  ``TEST_ZFSTESTS_TAGS`` - 提供给 ``test-runner`` 的标签列表
-  ``TEST_ZFSSTRESS_SKIP`` - 确定是否跳过 ``zfsstress`` 测试
-  ``TEST_ZFSSTRESS_URL`` - 下载 ``zfsstress`` 的 URL
-  ``TEST_ZFSSTRESS_VER`` - 从 ``TEST_ZFSSTRESS_URL`` 下载的 tarball 名称
-  ``TEST_ZFSSTRESS_RUNTIME`` - 运行 ``runstress.sh`` 的时长
-  ``TEST_ZFSSTRESS_POOL`` - 为 ``zfsstress`` 测试创建和使用的池名称
-  ``TEST_ZFSSTRESS_FS`` - ``zfsstress`` 测试期间使用的数据集名称
-  ``TEST_ZFSSTRESS_FSOPT`` - 提供给 ``zfsstress`` 的文件系统选项
-  ``TEST_ZFSSTRESS_VDEV`` - ``zfsstress`` 测试期间使用的 vdev 存储目录
-  ``TEST_ZFSSTRESS_OPTIONS`` - 提供给 ``runstress.sh`` 的命令行选项