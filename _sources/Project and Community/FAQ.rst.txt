FAQ
===

.. contents:: 目录
   :local:

什么是 OpenZFS
---------------

OpenZFS 是一个出色的存储平台，它集成了传统文件系统、卷管理等功能，并在所有发行版中提供一致的可靠性、功能和性能。有关 OpenZFS 的更多信息，请参阅 `OpenZFS 维基百科文章 <https://en.wikipedia.org/wiki/OpenZFS>`__。

硬件要求
---------------------

由于 ZFS 最初是为 Sun Solaris 设计的，因此长期以来它被认为是为大型服务器和能够负担得起最强大硬件的公司设计的文件系统。但随着 ZFS 被移植到多个开源平台（BSD、Illumos 和 Linux——在“OpenZFS”组织下），这些要求已经降低。

建议的硬件要求如下：

-  ECC 内存。这并不是硬性要求，但强烈推荐。
-  8GB 以上的内存以获得最佳性能。使用 2GB 或更少的内存也可以运行（并且有人这样做），但如果你使用去重功能，则需要更多内存。

ZFS 必须使用 ECC 内存吗？
------------------------------------

在企业环境中，强烈建议为 OpenZFS 使用 ECC 内存，以确保最强的数据完整性保证。如果没有 ECC 内存，宇宙射线或故障内存可能导致罕见的随机位翻转，而这些错误可能无法被检测到。如果发生这种情况，OpenZFS（或任何其他文件系统）会将损坏的数据写入磁盘，并且无法自动检测到损坏。

不幸的是，消费级硬件并不总是支持 ECC 内存。即使支持，ECC 内存也会更昂贵。对于家庭用户来说，ECC 内存带来的额外安全性可能无法证明其成本的合理性。你需要根据数据的保护需求来决定是否使用 ECC 内存。

安装
------------

OpenZFS 可用于 FreeBSD 和所有主要的 Linux 发行版。请参阅 wiki 的 :doc:`入门 <../Getting Started/index>` 部分以获取安装说明的链接。如果你的发行版/操作系统未列出，你始终可以从最新的官方 `tarball <https://github.com/openzfs/zfs/releases>`__ 构建 OpenZFS。

支持的架构
-----------------------

OpenZFS 定期为以下架构编译：aarch64、arm、ppc、ppc64、x86、x86_64。

支持的 Linux 内核
-----------------------

每个 OpenZFS 版本的 `发布说明 <https://github.com/openzfs/zfs/releases>`__ 会包含支持的内核范围。点版本将根据需要标记，以支持从 `kernel.org <https://www.kernel.org/>`__ 获取的 *稳定* 内核。由于其在企业 Linux 发行版中的普及，支持的最旧内核是 2.6.32。

.. _32-bit-vs-64-bit-systems:

32 位与 64 位系统
------------------------

强烈建议使用 64 位内核。OpenZFS 可以构建用于 32 位系统，但可能会遇到稳定性问题。

ZFS 最初是为 Solaris 内核开发的，该内核与某些 OpenZFS 平台在几个重要方面有所不同。对于 ZFS 来说，最重要的是 Solaris 内核通常大量使用虚拟地址空间。然而，在 Linux 内核中强烈不建议使用虚拟地址空间。这在 32 位架构上尤其明显，因为虚拟地址空间默认限制为 100M。在 64 位 Linux 内核上使用虚拟地址空间也不推荐，但由于地址空间比物理内存大得多，因此问题较小。

如果你在 32 位系统上遇到虚拟内存限制，你将在系统日志中看到以下消息。你可以通过引导选项 ``vmalloc=512M`` 来增加虚拟地址空间大小。

::

   vmap allocation for size 4198400 failed: use vmalloc=<size> to increase size.

然而，即使进行了此更改，你的系统也可能不会完全稳定。对 32 位系统的适当支持取决于 OpenZFS 代码摆脱对虚拟内存的依赖。这需要一些时间才能正确完成，但这是 OpenZFS 的计划。此更改预计还将提高 OpenZFS 管理 ARC 缓存的效率，并允许与标准 Linux 页面缓存更紧密地集成。

从 ZFS 启动
----------------

在 Linux 上从 ZFS 启动是可能的，许多人已经这样做了。有优秀的教程可用于 :doc:`Debian <../Getting Started/Debian/index>`、:doc:`Ubuntu <../Getting Started/Ubuntu/index>` 和 `Gentoo <https://github.com/pendor/gentoo-zfs-install/tree/master/install>`__。

在 FreeBSD 13+ 上，ZFS 启动是开箱即用的。

创建池时选择 /dev/ 名称（Linux）
--------------------------------------------------

创建 ZFS 池时可以使用不同的 /dev/ 名称。每个选项都有其优缺点，选择适合你的 ZFS 池的选项取决于你的需求。对于开发和测试，使用 /dev/sdX 命名是快速且简单的。典型的家庭服务器可能更喜欢 /dev/disk/by-id/ 命名，因为它简单且易于阅读。而具有多个控制器、机箱和交换机的大型配置可能更喜欢 /dev/disk/by-vdev 命名，以获得最大的控制权。但最终，如何选择识别你的磁盘取决于你。

-  **/dev/sdX, /dev/hdX:** 最适合开发/测试池

   -  摘要：顶级 /dev/ 名称是与其他 ZFS 实现保持一致的默认选项。它们在所有 Linux 发行版下都可用，并且常用。然而，由于它们不是持久的，因此应仅用于 ZFS 的开发/测试池。
   -  优点：这种方法适用于快速测试，名称简短，并且在所有 Linux 发行版上都可用。
   -  缺点：名称不是持久的，并且会根据磁盘检测的顺序而变化。添加或移除硬件很容易导致名称更改。然后你需要删除 zpool.cache 文件并使用新名称重新导入池。
   -  示例：``zpool create tank sda sdb``

-  **/dev/disk/by-id/:** 最适合小型池（少于 10 个磁盘）

   -  摘要：此目录包含更具可读性的磁盘标识符。磁盘标识符通常由接口类型、供应商名称、型号、设备序列号和分区号组成。这种方法更友好，因为它简化了识别特定磁盘的过程。
   -  优点：适用于具有单个磁盘控制器的小型系统。由于名称是持久的且保证不会更改，因此磁盘如何连接到系统并不重要。你可以将它们全部取出，随机混合，然后放回系统中的任何位置，你的池仍将自动正确导入。
   -  缺点：基于物理位置的冗余组配置变得困难且容易出错。在许多个人虚拟机设置上不可靠，因为软件默认情况下不会生成持久的唯一名称。
   -  示例：``zpool create tank scsi-SATA_Hitachi_HTS7220071201DP1D10DGG6HMRP``

-  **/dev/disk/by-path/:** 适用于大型池（超过 10 个磁盘）

   -  摘要：此方法使用包含系统中物理电缆布局的设备名称，这意味着特定磁盘绑定到特定位置。名称描述了 PCI 总线号、机箱名称和端口号。这允许在配置大型池时获得最大的控制权。
   -  优点：将存储拓扑编码在名称中不仅有助于在大型安装中定位磁盘，还允许你明确地在多个适配器或机箱上布局冗余组。
   -  缺点：这些名称冗长、繁琐且难以人工管理。
   -  示例：``zpool create tank pci-0000:00:1f.2-scsi-0:0:0:0 pci-0000:00:1f.2-scsi-1:0:0:0``

-  **/dev/disk/by-vdev/:** 最适合大型池（超过 10 个磁盘）

   -  摘要：此方法通过配置文件 /etc/zfs/vdev_id.conf 提供对设备命名的管理控制。JBOD 中的磁盘名称可以自动生成以反映其物理位置（通过机箱 ID 和插槽号）。名称也可以基于现有的 udev 设备链接手动分配，包括 /dev/disk/by-path 或 /dev/disk/by-id 中的链接。这允许你为磁盘选择自己独特且有意义的名称。这些名称将由所有 zfs 实用程序显示，因此可以用于简化大型复杂池的管理。有关更多详细信息，请参阅 vdev_id 和 vdev_id.conf 手册页。
   -  优点：此方法的主要优点是允许你选择有意义且易于阅读的名称。除此之外，优点取决于所采用的命名方法。如果名称是从物理路径派生的，则可以实现 /dev/disk/by-path 的优点。另一方面，基于驱动器标识符或 WWN 的别名具有与使用 /dev/disk/by-id 相同的优点。
   -  缺点：此方法依赖于为你的系统正确配置 /etc/zfs/vdev_id.conf 文件。要配置此文件，请参阅 `设置 /etc/zfs/vdev_id.conf 文件 <#setting-up-the-etc-zfs-vdev-id-conf-file>`__ 部分。与优点一样，缺点也取决于所采用的命名方法。
   -  示例：``zpool create tank mirror A1 B1 mirror A2 B2``

-  **/dev/disk/by-uuid/:** 不是一个好选择

  -   摘要：有人可能认为使用“UUID”将是一个理想的选择——然而，在实践中，这最终会列出每个 **池** ID 的一个设备，这对于导入具有多个磁盘的池并不是很有用。

-  **/dev/disk/by-partuuid/**/**by-partlabel:** 仅适用于现有分区

  -   摘要：分区 UUID 是在创建时生成的，因此使用受到限制。
  -   缺点：你无法在未分区的磁盘上引用分区的唯一 ID 以进行 ``zpool replace``/``add``/``attach``，并且如果没有提前写下映射，你将无法轻松找到故障磁盘。

设置 /etc/zfs/vdev_id.conf 文件
-----------------------------------------

为了使用 /dev/disk/by-vdev/ 命名，必须配置 ``/etc/zfs/vdev_id.conf``。此文件的格式在 vdev_id.conf 手册页中有描述。以下是几个示例。

具有直接连接的 SAS 机箱和任意插槽重新映射的非多路径配置。

::

               multipath     no
               topology      sas_direct
               phys_per_port 4

               #       PCI_SLOT HBA PORT  CHANNEL NAME
               channel 85:00.0  1         A
               channel 85:00.0  0         B

               #    Linux      Mapped
               #    Slot       Slot
               slot 0          2
               slot 1          6
               slot 2          0
               slot 3          3
               slot 4          5
               slot 5          7
               slot 6          4
               slot 7          1

SAS 交换机拓扑。请注意，在此示例中，channel 关键字仅接受两个参数。

::

               topology      sas_switch

               #       SWITCH PORT  CHANNEL NAME
               channel 1            A
               channel 2            B
               channel 3            C
               channel 4            D

多路径配置。请注意，通道名称有多个定义——每个物理路径一个。

::

               multipath yes

               #       PCI_SLOT HBA PORT  CHANNEL NAME
               channel 85:00.0  1         A
               channel 85:00.0  0         B
               channel 86:00.0  1         A
               channel 86:00.0  0         B

使用设备链接别名的配置。

::

               #     by-vdev
               #     name     fully qualified or base name of device link
               alias d1       /dev/disk/by-id/wwn-0x5000c5002de3b9ca
               alias d2       wwn-0x5000c5002def789e

定义新的磁盘名称后，运行 ``udevadm trigger`` 以提示 udev 解析配置文件。这将生成一个新的 /dev/disk/by-vdev 目录，其中包含指向 /dev/sdX 名称的符号链接。按照上面的第一个示例，你可以使用以下命令创建新的镜像池：

::

   $ zpool create tank \
       mirror A0 B0 mirror A1 B1 mirror A2 B2 mirror A3 B3 \
       mirror A4 B4 mirror A5 B5 mirror A6 B6 mirror A7 B7

   $ zpool status
     pool: tank
    state: ONLINE
    scan: none requested
   config:

       NAME        STATE     READ WRITE CKSUM
       tank        ONLINE       0     0     0
         mirror-0  ONLINE       0     0     0
           A0      ONLINE       0     0     0
           B0      ONLINE       0     0     0
         mirror-1  ONLINE       0     0     0
           A1      ONLINE       0     0     0
           B1      ONLINE       0     0     0
         mirror-2  ONLINE       0     0     0
           A2      ONLINE       0     0     0
           B2      ONLINE       0     0     0
         mirror-3  ONLINE       0     0     0
           A3      ONLINE       0     0     0
           B3      ONLINE       0     0     0
         mirror-4  ONLINE       0     0     0
           A4      ONLINE       0     0     0
           B4      ONLINE       0     0     0
         mirror-5  ONLINE       0     0     0
           A5      ONLINE       0     0     0
           B5      ONLINE       0     0     0
         mirror-6  ONLINE       0     0     0
           A6      ONLINE       0     0     0
           B6      ONLINE       0     0     0
         mirror-7  ONLINE       0     0     0
           A7      ONLINE       0     0     0
           B7      ONLINE       0     0     0

   errors: No known data errors

更改现有池的 /dev/ 名称
----------------------------------------

可以通过简单地导出池并使用 -d 选项重新导入池来更改现有池的 /dev/ 名称，以指定应使用的新名称。例如，要使用 /dev/disk/by-vdev 中的自定义名称：

::

   $ zpool export tank
   $ zpool import -d /dev/disk/by-vdev tank

.. _the-etczfszpoolcache-file:

/etc/zfs/zpool.cache 文件
-----------------------------

每当在系统上导入池时，它将被添加到 ``/etc/zfs/zpool.cache`` 文件中。此文件存储池配置信息，例如设备名称和池状态。如果在运行 ``zpool import`` 命令时存在此文件，则将使用它来确定可用于导入的池列表。当池未列在缓存文件中时，需要使用 ``zpool import -d /dev/disk/by-id`` 命令检测并导入池。

.. _generating-a-new-etczfszpoolcache-file:

生成新的 /etc/zfs/zpool.cache 文件
------------------------------------------

当你的池配置更改时，``/etc/zfs/zpool.cache`` 文件将自动更新。然而，如果由于某种原因它变得过时，你可以通过在池上设置 cachefile 属性来强制生成新的 ``/etc/zfs/zpool.cache`` 文件。

::

   $ zpool set cachefile=/etc/zfs/zpool.cache tank

相反，可以通过设置 ``cachefile=none`` 来禁用缓存文件。这对于故障转移配置非常有用，其中池应始终由故障转移软件显式导入。

::

   $ zpool set cachefile=none tank

发送和接收流
-----------------------------

hole_birth 错误
~~~~~~~~~~~~~~~

`hole_birth` 功能存在（或曾经存在）一些错误，导致的结果是，如果你从一个受影响的数据集执行 `zfs send -i`（或 `-R`，因为它使用了 `-i`），接收方不会看到任何校验和或其他错误，但生成的目标快照将与源快照不匹配。

ZoL 版本 0.6.5.8 和 0.7.0-rc1（及以上版本）默认忽略导致此问题的错误元数据 *在发送方*。

有关更多详细信息，请参阅 :doc:`hole_birth FAQ <./FAQ hole birth>`。

发送大块数据
~~~~~~~~~~~~~~~~~~~~

当发送包含大块数据（>128K）的增量流时，必须指定 ``--large-block`` 标志。在增量发送之间不一致地使用此标志可能导致文件在接收时被错误地清零。原始加密的发送/接收自动隐含 ``--large-block`` 标志，因此不受影响。

有关更多详细信息，请参阅 `问题 6224 <https://github.com/zfsonlinux/zfs/issues/6224>`__。

CEPH/ZFS
--------

根据 CEPH/ZFS 上的工作负载，可以进行大量调整，以及一些一般性指南。以下是一些建议：

ZFS 配置
~~~~~~~~~~~~~~~~~

CEPH 文件存储后端严重依赖 xattrs，为了获得最佳性能，所有 CEPH 工作负载都将受益于以下 ZFS 数据集参数：

-  ``xattr=sa``
-  ``dnodesize=auto``

除此之外，通常 rbd/cephfs 工作负载受益于较小的 recordsize（16K-128K），而 objectstore/s3/rados 工作负载受益于较大的 recordsize（128K-1M）。

.. _ceph-configuration-cephconf:

CEPH 配置 (ceph.conf)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

此外，CEPH 根据底层文件系统设置了各种处理 xattrs 的值。由于 CEPH 仅正式支持/检测 XFS 和 BTRFS，对于所有其他文件系统，它会回退到相当 `有限的“安全”值 <https://github.com/ceph/ceph/blob/4fe7e2a458a1521839bc390c2e3233dd809ec3ac/src/common/config_opts.h#L1125-L1148>`__。在新版本中，对较大 xattrs 的需求将阻止 OSD 启动。

官方推荐的解决方法（`参见此处 <https://tracker.ceph.com/issues/16187#note-3>`__）有一些严重的缺点，特别是针对具有“有限”xattr 支持的文件系统（如 ext4）。

ZFS 内部没有 xattrs 长度的限制，因此我们可以像 CEPH 处理 XFS 一样处理它。我们可以设置覆盖以将三个内部值设置为与 XFS 使用的值相同（`参见此处 <https://github.com/ceph/ceph/blob/9b317f7322848802b3aab9fec3def81dddd4a49b/src/os/filestore/FileStore.cc#L5714-L5737>`__ 和 `此处 <https://github.com/ceph/ceph/blob/4fe7e2a458a1521839bc390c2e3233dd809ec3ac/src/common/config_opts.h#L1125-L1148>`__），并允许它在没有“官方”解决方法的严重限制的情况下使用。

::

   [osd]
   filestore_max_inline_xattrs = 10
   filestore_max_inline_xattr_size = 65536
   filestore_max_xattr_value_size = 65536

其他一般性指南
~~~~~~~~~~~~~~~~~~~~~~~~

-  使用单独的日志设备。如果可能，不要将 CEPH 日志与 ZFS 数据集放在一起，这将很快导致严重的碎片化，甚至在碎片化之前就会导致性能极差（CEPH 日志对每次写入都进行 dsync）。
-  使用 SLOG 设备，即使有单独的 CEPH 日志设备。对于某些工作负载，跳过 SLOG 并设置 ``logbias=throughput`` 可能是可以接受的。
-  使用高质量的 SLOG/CEPH 日志设备。消费级 SSD 甚至 NVMe 都不适合（如三星 830、840、850 等），原因有很多。CEPH 会很快杀死它们，而且在这种使用中性能相当低。通常推荐的设备是 [Intel DC S3610、S3700、S3710、P3600、P3700] 或 [Samsung SM853、SM863] 或更好的设备。
-  如果使用高质量的 SSD 或 NVMe 设备（如上所述），你可以在单个设备上共享 SLOG 和 CEPH 日志以获得良好的效果。4 个 HDD 与 1 个 SSD（Intel DC S3710 200GB）的比例，每个 SSD 分区（记得对齐！）为 4x10GB（用于 ZIL/SLOG）+ 4x20GB（用于 CEPH 日志）已被报告为效果良好。

再次强调——CEPH + ZFS 会很快杀死消费级 SSD。即使忽略缺乏电源丢失保护和耐久性评级，你也会对消费级 SSD 在这种工作负载下的性能感到非常失望。

性能考虑
--------------------------

要实现良好的池性能，应遵循一些简单的最佳实践。

-  **将磁盘均匀分布在控制器上：** 通常性能的瓶颈不是磁盘，而是控制器。通过将磁盘均匀分布在控制器上，通常可以提高吞吐量。
-  **使用整个磁盘创建池：** 在运行 zpool create 时使用整个磁盘名称。这将允许 ZFS 自动分区磁盘以确保正确对齐。它还将提高与其他 OpenZFS 实现的互操作性，这些实现遵循 wholedisk 属性。
-  **有足够的内存：** 建议 ZFS 至少使用 2GB 内存。当启用压缩和去重功能时，强烈建议增加内存。
-  **通过设置 ashift=12 提高性能：** 你可以通过设置 ``ashift=12`` 来提高某些工作负载的性能。此调整只能在块设备首次添加到池时设置，例如在首次创建池或向池添加新 vdev 时。此调整参数可能会导致 RAIDZ 配置的容量减少。

高级格式磁盘
---------------------

高级格式（AF）是一种新的磁盘格式，它原生使用 4,096 字节而不是 512 字节的扇区大小。为了与旧系统保持兼容，许多 AF 磁盘模拟 512 字节的扇区大小。默认情况下，ZFS 会自动检测驱动器的扇区大小。这种组合可能导致磁盘访问未对齐，从而大大降低池性能。

因此，zpool 命令中添加了设置 ashift 属性的功能。这允许用户在设备首次添加到池时（通常在池创建时或向池添加 vdev 时）显式分配扇区大小。ashift 值的范围从 9 到 16，默认值 0 表示 zfs 应自动检测扇区大小。此值实际上是位移值，因此 512 字节的 ashift 值为 9（2^9 = 512），而 4,096 字节的 ashift 值为 12（2^12 = 4,096）。

要在池创建时强制池使用 4,096 字节扇区，你可以运行：

::

   $ zpool create -o ashift=12 tank mirror sda sdb

要在向池添加 vdev 时强制池使用 4,096 字节扇区，你可以运行：

::

   $ zpool add -o ashift=12 tank mirror sdc sdd

ZVOL 使用空间大于预期
------------------------------------

| 根据 zvol 上使用的文件系统（例如 ext4）和使用情况（例如删除和创建许多文件），zvol 报告的 ``used`` 和 ``referenced`` 属性可能大于消费者报告的“实际”使用空间。
| 这可能是因为某些文件系统的工作方式，它们更喜欢在未使用的新块中分配文件，而不是标记为空闲的碎片化块。这迫使 zfs 引用底层文件系统曾经接触过的所有块。
| 这本身并不是一个大问题，因为当 ``used`` 属性达到配置的 ``volsize`` 时，底层文件系统将开始重用块。但如果需要对 zvol 进行快照，问题就会出现，因为快照引用的空间将包含未使用的块。

| 可以通过发出所谓的 trim（例如 Linux 上的 ``fstrim`` 命令）来防止此问题，以允许内核指定 zfs 哪些块未使用。
| 在快照之前发出 trim 将确保最小的快照大小。
| 对于 Linux，在 ``/etc/fstab`` 中为挂载的 ZVOL 添加 ``discard`` 选项可以有效地使内核持续发出 trim 命令，而无需按需执行 fstrim。

在 Linux 上使用 zvol 作为交换设备
---------------------------------------

你可以使用 zvol 作为交换设备，但需要适当配置。

**警告：** 目前 zvol 上的交换可能导致死锁，如果发生这种情况，请将日志发送到 `此处 <https://github.com/zfsonlinux/zfs/issues/7734>`__。

-  将卷块大小设置为与系统的页面大小匹配。此调整可防止 ZFS 在系统内存不足时对较大的块执行读-修改-写操作。
-  设置 ``logbias=throughput`` 和 ``sync=always`` 属性。写入卷的数据将立即刷新到磁盘，以尽快释放内存。
-  设置 ``primarycache=metadata`` 以避免通过 ARC 将交换数据保留在 RAM 中。
-  禁用交换设备的自动快照。

::

   $ zfs create -V 4G -b $(getconf PAGESIZE) \
       -o logbias=throughput -o sync=always \
       -o primarycache=metadata \
       -o com.sun:auto-snapshot=false rpool/swap

在 Xen Hypervisor 或 Xen Dom0 上使用 ZFS（Linux）
-----------------------------------------------

通常建议将虚拟机存储和 Hypervisor 池分开。尽管有些人已成功部署并运行 OpenZFS，使用同一台机器配置为 Dom0。有几个注意事项：

-  在 grub.conf 中为 Dom0 设置合理的内存量。

   -  dom0_mem=16384M,max:16384M

-  在 ``/etc/modprobe.d/zfs.conf`` 中为 Dom0 的内存分配不超过 30-40% 给 ZFS。

   -  options zfs zfs_arc_max=6442450944

-  在 ``/etc/xen/xl.conf`` 中禁用 Xen 的自动气球功能。
-  注意任何 Xen 错误，例如与气球相关的 `此错误 <https://github.com/zfsonlinux/zfs/issues/1067>`__。

udisks2 为 zvol 创建 /dev/mapper/ 条目（Linux）
------------------------------------------------------

为防止 udisks2 创建必须手动删除或维护的 /dev/mapper 条目，请在 zvol 删除/重命名期间创建 udev 规则，例如 ``/etc/udev/rules.d/80-udisks2-ignore-zfs.rules``，内容如下：

::

   ENV{ID_PART_ENTRY_SCHEME}=="gpt", ENV{ID_FS_TYPE}=="zfs_member", ENV{ID_PART_ENTRY_TYPE}=="6a898cc3-1dd2-11b2-99a6-080020736631", ENV{UDISKS_IGNORE}="1"

许可
---------

许可信息可以在 `此处 <https://openzfs.github.io/openzfs-docs/License.html>`__ 找到。

报告问题
-------------------

你可以使用公共 `问题跟踪器 <https://github.com/zfsonlinux/zfs/issues>`__ 打开新问题并搜索现有问题。问题跟踪器用于组织未解决的错误报告、功能请求和其他开发任务。任何人在注册 github 帐户后都可以发表评论。

请确保你实际看到的是错误而不是支持问题。如果有疑问，请先在邮件列表中询问，如果要求你提交问题，请照做。

打开新问题时，请在问题顶部包含以下信息：

-  你使用的发行版及其版本。
-  你使用的 spl/zfs 包及其版本。
-  描述你观察到的问题。
-  描述如何重现问题。
-  包括系统日志中的任何警告/错误/回溯。

当打开新问题时，开发人员通常会要求提供有关问题的更多信息。一般来说，你分享的细节越多，开发人员解决问题的速度就越快。例如，提供一个简单的测试用例总是非常有帮助的。准备好与开发人员合作，以便解决问题。他们可能会要求提供以下信息：

-  你的池配置，如 ``zdb`` 或 ``zpool status`` 报告的那样。
-  你的硬件配置，例如

   -  CPU 数量。
   -  内存量。
   -  你的系统是否具有 ECC 内存。
   -  是否在 VMM/Hypervisor 下运行。
   -  内核版本。
   -  spl/zfs 模块参数的值。

-  可能记录到 ``dmesg`` 的堆栈跟踪。

OpenZFS 有行为准则吗？
------------------------------------

是的，OpenZFS 社区有行为准则。有关详细信息，请参阅 `行为准则 <https://openzfs.org/wiki/Code_of_Conduct>`__。