自定义包
============

以下说明假设您是从官方发布的 `release tarball <https://github.com/zfsonlinux/zfs/releases/latest>`__（版本 0.8.0 或更新版本）或直接从 `git 仓库 <https://github.com/zfsonlinux/zfs>`__ 构建的。大多数用户不需要这样做，应该优先使用发行版提供的包。一般来说，发行版提供的包会更紧密地集成、经过更广泛的测试，并且得到更好的支持。然而，如果您选择的发行版没有提供包，或者您是开发人员并希望自己构建包，以下是具体方法。

首先需要注意的是，构建系统能够生成几种不同类型的包。您选择哪种类型的包取决于您的平台支持什么以及您的具体需求。

-  **DKMS** 包仅包含源代码和用于重新构建内核模块的脚本。安装 DKMS 包后，将为所有可用的内核构建内核模块。此外，当内核升级时，将自动为该内核构建新的内核模块。这对于频繁接收内核更新的桌面系统特别方便。缺点是，由于 DKMS 包从源代码构建内核模块，因此需要完整的开发环境，这可能不适合大规模部署。

-  **kmods** 包是针对特定版本内核编译的二进制内核模块。这意味着如果您更新内核，则必须编译并安装新的 kmod 包。如果您不经常更新内核，或者您正在管理大量系统，那么 kmod 包是一个不错的选择。

-  **kABI-tracking kmod** 包与标准的二进制 kmods 类似，可用于像 Red Hat 和 CentOS 这样的企业 Linux 发行版。这些发行版提供了稳定的 kABI（内核应用程序二进制接口），允许相同的二进制模块与新版本的发行版内核一起使用。

默认情况下，构建系统将生成用户包以及 DKMS 和 kmod 风格的内核包（如果可能）。用户包可以与任何一组内核包一起使用，并且在内核更新时不需要重新构建。您还可以通过仅构建 DKMS 或 kmod 包来简化构建过程，如下所示。

请注意，当直接从 git 仓库构建时，您必须首先运行 *autogen.sh* 脚本来创建 *configure* 脚本。这将需要为您的发行版安装 GNU autotools 包。要执行任何构建，您必须安装所有必要的开发工具和头文件。

需要注意的是，如果当前运行内核的开发内核头文件未安装，模块将无法正确编译。

-  `Red Hat、CentOS 和 Fedora <#red-hat-centos-and-fedora>`__
-  `Debian 和 Ubuntu <#debian-and-ubuntu>`__

RHEL、CentOS 和 Fedora
-----------------------

确保已安装构建最新 ZFS 2.1 版本所需的包：

-  **RHEL/CentOS 7**:

.. code:: sh

   sudo yum install epel-release gcc make autoconf automake libtool rpm-build libtirpc-devel libblkid-devel libuuid-devel libudev-devel openssl-devel zlib-devel libaio-devel libattr-devel elfutils-libelf-devel kernel-devel-$(uname -r) python python2-devel python-setuptools python-cffi libffi-devel ncompress
   sudo yum install --enablerepo=epel dkms python-packaging
 
**注意：** RHEL/CentOS 7 已停止支持。请使用 yum 而不是 dnf 进行安装。

-  **RHEL/CentOS 8**:

.. code:: sh

   sudo dnf install --skip-broken epel-release gcc make autoconf automake libtool rpm-build kernel-rpm-macros libtirpc-devel libblkid-devel libuuid-devel libudev-devel openssl-devel zlib-devel libaio-devel libattr-devel elfutils-libelf-devel kernel-devel-$(uname -r) kernel-abi-stablelists-$(uname -r | sed 's/\.[^.]\+$//') python3 python3-devel python3-setuptools python3-cffi libffi-devel ncompress
   sudo dnf install --skip-broken --enablerepo=epel --enablerepo=powertools python3-packaging dkms

-  **RHEL/CentOS 9**:

.. code:: sh

   sudo dnf config-manager --set-enabled crb
   sudo dnf install --skip-broken epel-release gcc make autoconf automake libtool rpm-build kernel-rpm-macros libtirpc-devel libblkid-devel libuuid-devel libudev-devel openssl-devel zlib-devel libaio-devel libattr-devel elfutils-libelf-devel kernel-devel-$(uname -r) kernel-abi-stablelists-$(uname -r | sed 's/\.[^.]\+$//') python3 python3-devel python3-setuptools python3-cffi libffi-devel
   sudo dnf install --skip-broken --enablerepo=epel python3-packaging dkms

-  **Fedora 41**:

.. code:: sh

  sudo dnf install gcc make autoconf automake libtool rpm-build kernel-rpm-macros libtirpc-devel libblkid-devel libuuid-devel systemd-devel openssl-devel zlib-ng-compat-devel libaio-devel libattr-devel libffi-devel libunwind-devel kernel-devel-$(uname -r) python3 python3-devel openssl ncompress
  sudo dnf install python3-packaging dkms



`获取源代码 <#get-the-source-code>`__。

DKMS
~~~~

构建基于 RPM 的 DKMS 和用户包可以按以下步骤进行：

.. code:: sh

   $ cd zfs
   $ ./configure
   $ make -j1 rpm-utils rpm-dkms
   $ sudo dnf install *.$(uname -m).rpm *.noarch.rpm

kmod
~~~~

构建 kmod 包时，关键是要知道必须指定一个特定的 Linux 内核。在配置时，构建系统会猜测您要针对哪个内核进行构建。然而，如果配置无法找到您的内核开发头文件，或者您想针对不同的内核进行构建，则必须使用 *--with-linux* 和 *--with-linux-obj* 选项指定确切路径。

.. code:: sh

   $ cd zfs
   $ ./configure
   $ make -j1 rpm-utils rpm-kmod
   $ sudo dnf install *.$(uname -m).rpm *.noarch.rpm

**注意：** Fedora 41 Workstation 包含 rpm 包 zfs-fuse，这会阻止您安装自己的包。在 dnf install 之前删除该包：

.. code:: sh

   $ sudo rpm -e --nodeps zfs-fuse

kABI-tracking kmod
~~~~~~~~~~~~~~~~~~

构建 kABI-tracking kmods 的过程与构建普通 kmods 几乎相同。但是，它只会生成可用于多个内核的二进制文件，前提是发行版支持稳定的 kABI。为了请求 kABI-tracking 包，必须在配置时传递 *--with-spec=redhat* 选项。

**注意：** 这种类型的包不适用于 Fedora。

.. code:: sh

   $ cd zfs
   $ ./configure --with-spec=redhat
   $ make -j1 rpm-utils rpm-kmod
   $ sudo dnf install *.$(uname -m).rpm *.noarch.rpm

Fedora 41 安全启动与 kmod
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

在使用 UEFI 和安全启动的现代计算机上，zfs 内核模块将无法加载：

.. code::

   $ sudo modprobe zfs
   modprobe: ERROR: could not insert 'zfs': Key was rejected by service

要么禁用安全启动，要么创建一次自定义机器所有者密钥 (MOK) 并使用该密钥手动签名当前和未来的模块：

.. code:: sh

   $ sudo mkdir /etc/pki/mok
   $ cd /etc/pki/mok
   $ sudo openssl req -new -x509 -newkey rsa:2048 -keyout LOCALMOK.priv -outform DER -out LOCALMOK.der -nodes -days 36500 -subj "/CN=LOCALMOK/"
   $ sudo mokutil --import LOCALMOK.der

Mokutil 会要求您创建并记住一个密码，然后重新启动您的机器，UEFI 将要求导入您的密钥：

.. code::

   选择 "Enroll MOK", "Continue", "Yes", 输入 mokutil 的密码, "Reboot"

然后可以使用此 MOK 手动签名您的 zfs 内核模块：

.. code::

   $ rpm -ql kmod-zfs-$(uname -r) | grep .ko
   /lib/modules/6.11.8-300.fc41.x86_64/extra/zfs/spl.ko
   /lib/modules/6.11.8-300.fc41.x86_64/extra/zfs/zfs.ko

.. code:: sh

   $ sudo /usr/src/kernels/$(uname -r)/scripts/sign-file sha256 /etc/pki/mok/LOCALMOK.priv /etc/pki/mok/LOCALMOK.der /lib/modules/$(uname -r)/extra/zfs/spl.ko
   $ sudo /usr/src/kernels/$(uname -r)/scripts/sign-file sha256 /etc/pki/mok/LOCALMOK.priv /etc/pki/mok/LOCALMOK.der /lib/modules/$(uname -r)/extra/zfs/zfs.ko

加载模块并验证其是否处于活动状态：

.. code::

   $ sudo modprobe zfs

   $ lsmod | grep zfs
   zfs                  6930432  0
   spl                   155648  1 zfs

Debian 和 Ubuntu
----------------

确保已安装所需的包：

.. code:: sh

   sudo apt install build-essential autoconf automake libtool gawk alien fakeroot dkms libblkid-dev uuid-dev libudev-dev libssl-dev zlib1g-dev libaio-dev libattr1-dev libelf-dev linux-headers-generic python3 python3-dev python3-setuptools python3-cffi libffi-dev python3-packaging debhelper-compat dh-python po-debconf python3-all-dev python3-sphinx libpam0g-dev

`获取源代码 <#get-the-source-code>`__。

.. _kmod-1:

kmod
~~~~

构建 kmod 包时，关键是要知道必须指定一个特定的 Linux 内核。在配置时，构建系统会猜测您要针对哪个内核进行构建。然而，如果配置无法找到您的内核开发头文件，或者您想针对不同的内核进行构建，则必须使用 *--with-linux* 和 *--with-linux-obj* 选项指定确切路径。

要构建转换为 RPM 的 Debian 包：

.. code:: sh

   $ cd zfs
   $ ./configure --enable-systemd
   $ make -j1 deb-utils deb-kmod
   $ sudo apt-get install --fix-missing ./*.deb

从 openzfs-2.2 版本开始，可以按以下方式构建原生 Debian 包：

.. code:: sh

   $ cd zfs
   $ ./configure
   $ make native-deb-utils native-deb-kmod
   $ rm ../openzfs-zfs-dkms_*.deb
   $ rm ../openzfs-zfs-dracut_*.deb  # deb-based 系统通常使用 initramfs
   $ sudo apt-get install --fix-missing ../*.deb

原生 Debian 包使用为 Debian 和 Ubuntu 预配置的路径构建。最好不要在配置期间覆盖路径。可以导出 ``KVERS``、``KSRC`` 和 ``KOBJ`` 环境变量以指定安装在非默认位置的内核。

.. _dkms-1:

DKMS
~~~~

构建转换为 RPM 的基于 deb 的 DKMS 和用户包可以按以下步骤进行：

.. code:: sh

   $ cd zfs
   $ ./configure --enable-systemd
   $ make -j1 deb-utils deb-dkms
   $ sudo apt-get install --fix-missing ./*.deb

从 openzfs-2.2 版本开始，可以按以下方式构建基于 deb 的原生 DKMS 和用户包：

.. code:: sh

   $ sudo apt-get install dh-dkms
   $ cd zfs
   $ ./configure
   $ make native-deb-utils
   $ rm ../openzfs-zfs-dracut_*.deb  # deb-based 系统通常使用 initramfs
   $ sudo apt-get install --fix-missing ../*.deb

获取源代码
-----------

发布的 Tarball
~~~~~~~~~~~~~~

发布的 tarball 包含最新经过全面测试和发布的 ZFS 版本。这是生产系统中使用的首选源代码位置。如果您想使用官方发布的 tarball，请使用以下命令获取并准备源代码。

.. code:: sh

   $ wget http://archive.zfsonlinux.org/downloads/zfsonlinux/zfs/zfs-x.y.z.tar.gz
   $ tar -xzf zfs-x.y.z.tar.gz

Git 主分支
~~~~~~~~~~~

Git *master* 分支包含软件的最新版本，并且可能包含由于某种原因未包含在发布 tarball 中的修复。这是打算修改 ZFS 的开发人员的首选源代码位置。如果您想使用 git 版本，可以从 Github 克隆并准备源代码，如下所示。

.. code:: sh

   $ git clone https://github.com/zfsonlinux/zfs.git
   $ cd zfs
   $ ./autogen.sh

准备好源代码后，您需要决定要构建哪种类型的包，并跳转到上面的相应部分。请注意，并非所有平台都支持所有包类型。