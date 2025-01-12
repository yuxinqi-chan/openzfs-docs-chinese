构建 ZFS
============

GitHub 仓库
~~~~~~~~~~~

OpenZFS 的官方源码由 GitHub 上的 `openzfs <https://github.com/openzfs/>`__ 组织维护。该项目的主要 git 仓库是 `zfs <https://github.com/openzfs/zfs>`__ 仓库。

该仓库包含两个主要组件：

-  **ZFS**: ZFS 仓库包含上游 OpenZFS 代码的副本，这些代码已经为 Linux 和 FreeBSD 进行了适配和扩展。绝大部分核心 OpenZFS 代码是自包含的，可以在不修改的情况下使用。

-  **SPL**: SPL 是一个薄薄的垫片层，负责实现 OpenZFS 所需的基本接口。正是这一层使得 OpenZFS 能够在多个平台上使用。SPL 曾经在一个单独的仓库中维护，但在 ``0.8`` 大版本发布时被合并到了 `zfs <https://github.com/openzfs/zfs>`__ 仓库中。

安装依赖
~~~~~~~~~~~~~~~~~~~~~~~

首先，您需要准备环境，安装完整的开发工具链。此外，内核和以下包的开发头文件也必须可用。需要注意的是，如果当前运行内核的开发头文件没有安装，模块将无法正确编译。

以下依赖项应安装以构建最新的 ZFS 2.1 版本。

-  **RHEL/CentOS 7**:

.. code:: sh

   sudo yum install epel-release gcc make autoconf automake libtool rpm-build libtirpc-devel libblkid-devel libuuid-devel libudev-devel openssl-devel zlib-devel libaio-devel libattr-devel elfutils-libelf-devel kernel-devel-$(uname -r) python python2-devel python-setuptools python-cffi libffi-devel git ncompress libcurl-devel
   sudo yum install --enablerepo=epel python-packaging dkms

-  **RHEL/CentOS 8, Fedora**:

.. code:: sh

   sudo dnf install --skip-broken epel-release gcc make autoconf automake libtool rpm-build libtirpc-devel libblkid-devel libuuid-devel libudev-devel openssl-devel zlib-devel libaio-devel libattr-devel elfutils-libelf-devel kernel-devel-$(uname -r) python3 python3-devel python3-setuptools python3-cffi libffi-devel git ncompress libcurl-devel
   sudo dnf install --skip-broken --enablerepo=epel --enablerepo=powertools python3-packaging dkms

-  **Debian, Ubuntu**:

.. code:: sh

   sudo apt install alien autoconf automake build-essential debhelper-compat dh-autoreconf dh-dkms dh-python dkms fakeroot gawk git libaio-dev libattr1-dev libblkid-dev libcurl4-openssl-dev libelf-dev libffi-dev libpam0g-dev libssl-dev libtirpc-dev libtool libudev-dev linux-headers-generic parallel po-debconf python3 python3-all-dev python3-cffi python3-dev python3-packaging python3-setuptools python3-sphinx uuid-dev zlib1g-dev

-  **FreeBSD**:

.. code:: sh

   pkg install autoconf automake autotools git gmake python devel/py-sysctl sudo
    
构建选项
~~~~~~~~~~~~~

构建 OpenZFS 有两种选择；正确的选择主要取决于您的需求。

-  **包**: 通常从 git 构建自定义包并安装到系统上会很有用。这是与 systemd、dracut 和 udev 进行集成测试的最佳方式。使用包的缺点是大大增加了构建、安装和测试更改所需的时间。

-  **树内**: 开发可以完全在 SPL/ZFS 源码树中进行。这通过允许开发人员快速迭代补丁来加速开发。在树内工作时，开发人员可以利用增量构建、加载/卸载内核模块、执行实用程序，并使用 ZFS 测试套件验证所有更改。

本页的其余部分将重点介绍 **树内** 选项，这是大多数更改推荐的开发方法。有关构建自定义包的更多信息，请参阅 :doc:`自定义包 <./Custom Packages>` 页面。

树内开发
~~~~~~~~~~~~~~~~~~

从 GitHub 克隆
^^^^^^^^^^^^^^^^^

首先从 GitHub 克隆 ZFS 仓库。该仓库有一个用于开发的 **master** 分支和一系列用于标记发布的 **\*-release** 分支。检出仓库后，您的克隆将默认使用 master 分支。可以通过检出具有匹配版本号的 zfs-x.y.z 标签或匹配的发布分支来构建标记的发布。

::

   git clone https://github.com/openzfs/zfs

配置和构建
^^^^^^^^^^^^^^^^^^^

对于开发人员来说，在进行更改时，始终基于 master 创建一个新的主题分支。这将使以后更容易使用您的更改打开拉取请求。通过广泛的 `回归测试 <http://build.zfsonlinux.org/>`__，master 分支在每次拉取请求合并前后都保持稳定。我们尽一切努力尽早发现缺陷并将其排除在树外。开发人员应习惯于频繁地将其工作与最新的 master 分支进行变基。

在此示例中，我们将使用 master 分支并逐步完成标准的 **树内** 构建。首先检出所需的分支，然后以传统的 autotools 方式构建 ZFS 和 SPL 源码。

::

   cd ./zfs
   git checkout master
   sh autogen.sh
   ./configure
   make -s -j$(nproc)

| **提示:** ``--with-linux=PATH`` 和 ``--with-linux-obj=PATH`` 可以传递给 configure 以指定安装在内核非默认位置的路径。
| **提示:** ``--enable-debug`` 可以传递给 configure 以启用所有 ASSERT 和额外的正确性测试。

**可选** 构建包

::

   make rpm #构建 CentOS/Fedora 的 RPM 包
   make deb #构建 Debian/Ubuntu 的 RPM 转换 DEB 包
   make native-deb #构建 Debian/Ubuntu 的原生 DEB 包

| **提示:** 原生 Debian 包使用为 Debian 和 Ubuntu 预配置的路径构建。最好不要在配置期间覆盖路径。
| **提示:** 对于原生 Debian 包，可以导出 ``KVERS``、``KSRC`` 和 ``KOBJ`` 环境变量以指定安装在内核非默认位置的路径。

.. note::
   从 openzfs-2.2 版本开始，将支持原生 Debian 打包。

安装
^^^^^^^

您可以在不安装 ZFS 的情况下运行 ``zfs-tests.sh``，请参见下文。如果您有理由在构建后安装 ZFS，请注意您的发行版如何处理内核模块。例如，在 Ubuntu 上，此仓库中的模块安装在 ``extra`` 内核模块路径中，该路径不在标准的 ``depmod`` 搜索路径中。因此，在测试期间，编辑 ``/etc/depmod.d/ubuntu.conf`` 并将 ``extra`` 添加到搜索路径的开头。

然后您可以使用 ``sudo make install; sudo ldconfig; sudo depmod`` 进行安装。您可以使用 ``sudo make uninstall; sudo ldconfig; sudo depmod`` 进行卸载。您可以使用 ``sudo make -C modules/ install`` 仅安装内核模块。

.. _running-zloopsh-and-zfs-testssh:

运行 zloop.sh 和 zfs-tests.sh
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

如果您希望运行 ZFS 测试套件 (ZTS)，则必须安装 ``ksh`` 和一些额外的实用程序。

-  **RHEL/CentOS 7:**

.. code:: sh

   sudo yum install ksh bc bzip2 fio acl sysstat mdadm lsscsi parted attr nfs-utils samba rng-tools pax perf
   sudo yum install --enablerepo=epel dbench

-  **RHEL/CentOS 8, Fedora:**

.. code:: sh

   sudo dnf install --skip-broken ksh bc bzip2 fio acl sysstat mdadm lsscsi parted attr nfs-utils samba rng-tools pax perf
   sudo dnf install --skip-broken --enablerepo=epel dbench

-  **Debian:**

.. code:: sh

   sudo apt install ksh bc bzip2 fio acl sysstat mdadm lsscsi parted attr dbench nfs-kernel-server samba rng-tools pax linux-perf selinux-utils quota

-  **Ubuntu:**

.. code:: sh

   sudo apt install ksh bc bzip2 fio acl sysstat mdadm lsscsi parted attr dbench nfs-kernel-server samba rng-tools pax linux-tools-common selinux-utils quota

-  **FreeBSD**:

.. code:: sh

   pkg install base64 bash checkbashisms fio hs-ShellCheck ksh93 pamtester devel/py-flake8 sudo


在顶级脚本目录中提供了一些辅助脚本，旨在帮助开发人员使用树内构建。

-  **zfs-helper.sh:** 某些功能（即 /dev/zvol/）依赖于系统上安装的 ZFS 提供的 udev 辅助脚本。此脚本可用于在系统上创建从安装位置到树内辅助脚本的符号链接。这些链接必须到位才能成功运行 ZFS 测试套件。**-i** 和 **-r** 选项可用于安装和删除符号链接。

::

   sudo ./scripts/zfs-helpers.sh -i

-  **zfs.sh:** 可以使用 ``zfs.sh`` 加载新构建的内核模块。稍后可以使用 **-u** 选项卸载内核模块。

::

   sudo ./scripts/zfs.sh

-  **zloop.sh:** 一个包装器，用于使用随机参数重复运行 ztest。ztest 命令是一个用户空间压力测试，旨在通过并发运行一组随机测试用例来检测正确性问题。如果遇到崩溃，ztest 日志、任何相关的 vdev 文件和核心文件（如果存在）将被收集并移动到输出目录以供分析。

::

   sudo ./scripts/zloop.sh

-  **zfs-tests.sh:** 一个包装器，可用于启动 ZFS 测试套件。在 ``/var/tmp/`` 中位于稀疏文件顶部的三个回环设备被创建并用于回归测试。ZFS 测试套件的详细说明可以在顶级测试目录中的 `README <https://github.com/openzfs/zfs/tree/master/tests>`__ 中找到。

::

    ./scripts/zfs-tests.sh -vx

**提示:** 除非在 zfs 目录及其父目录上设置了组读取权限，否则将跳过 **delegate** 测试。