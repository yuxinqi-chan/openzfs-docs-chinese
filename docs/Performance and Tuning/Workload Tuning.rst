工作负载调优
===============

以下是针对各种工作负载的调优建议。

.. contents:: 目录
  :local:

.. _basic_concepts:

基本概念
--------

以下是影响应用程序性能的ZFS内部机制描述。

.. _adaptive_replacement_cache:

自适应替换缓存 (Adaptive Replacement Cache)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

几十年来，操作系统一直使用RAM作为缓存，以避免等待磁盘IO，因为磁盘IO非常慢。这个概念称为页面替换。在ZFS之前，几乎所有文件系统都使用最近最少使用（LRU）页面替换算法，其中最近最少使用的页面首先被替换。不幸的是，LRU算法容易受到缓存刷新的影响，即偶尔发生的短暂工作负载变化会从缓存中移除所有频繁使用的数据。ZFS中实现了自适应替换缓存（ARC）算法来替代LRU。它通过维护四个列表来解决这个问题：

#. 最近缓存条目的列表。
#. 最近缓存且被访问超过一次的条目列表。
#. 从列表1中驱逐的条目列表。
#. 从列表2中驱逐的条目列表。

数据从第一个列表中被驱逐，同时努力将数据保留在第二个列表中。通过这种方式，ARC能够通过提供更高的命中率来优于LRU。

此外，可以向池中添加专用的缓存设备（通常是SSD），使用命令``zpool add POOLNAME cache DEVICENAME``。缓存设备由L2ARC管理，L2ARC扫描即将被驱逐的条目并将其写入缓存设备。ARC和L2ARC中存储的数据可以通过``primarycache``和``secondarycache``属性分别控制，这些属性可以在zvol和数据集上设置。可能的设置为``all``、``none``和``metadata``。当zvol或数据集托管执行自己缓存的应用程序时，可以通过仅缓存元数据来提高性能。一个例子是使用ZFS的虚拟机。另一个例子是管理自己缓存的数据库系统（例如Oracle）。相比之下，PostgreSQL依赖于操作系统级文件缓存来进行大部分缓存。

.. _alignment_shift_ashift:

对齐偏移 (ashift)
~~~~~~~~~~~~~~~~~

顶级vdev包含一个称为ashift的内部属性，表示对齐偏移。它在vdev创建时设置，并且是不可变的。可以使用``zdb``命令读取它。它计算为任何子vdev的物理扇区大小的最大以2为底的对数，并更改磁盘格式，使得写入始终按照它进行。这使得2^ashift成为vdev上可能的最小IO。正确配置ashift很重要，因为部分扇区写入会导致性能下降，必须先读取扇区到缓冲区才能写入。ZFS假设驱动器报告的扇区大小是正确的，并基于此计算ashift。

在理想情况下，物理扇区大小总是正确报告的，因此这不需要特别注意。不幸的是，情况并非如此。在基于闪存的固态硬盘出现之前，所有存储设备的扇区大小都是512字节。一些操作系统，如Windows XP，是在此假设下编写的，当驱动器报告不同的扇区大小时将无法正常工作。

基于闪存的固态硬盘大约在2007年上市。这些设备报告512字节扇区，但实际的闪存页（大致对应于扇区）从不是512字节。早期型号使用4096字节页，而较新的型号已转向8192字节页。此外，还创建了使用4096字节扇区大小的“高级格式”硬盘。部分页写入会导致与部分扇区写入类似的性能下降。在某些情况下，NAND闪存的设计使性能下降更加严重，但这超出了本描述的范围。

报告正确的扇区大小是块设备层的责任。不幸的是，这使得处理误报驱动器的设备在不同平台上有所不同。各自的方法如下：

- 在illumos上使用`sd.conf <http://wiki.illumos.org/display/illumos/ZFS+and+Advanced+Format+disks#ZFSandAdvancedFormatdisks-OverridingthePhysicalBlockSize>`__
- 在FreeBSD上使用`gnop(8) <https://www.freebsd.org/cgi/man.cgi?query=gnop&sektion=8&manpath=FreeBSD+10.2-RELEASE>`__；例如参见`FreeBSD on 4K sector drives <http://web.archive.org/web/20151022020605/http://ivoras.sharanet.org/blog/tree/2011-01-01.freebsd-on-4k-sector-drives.html>`__（2011-01-01）
- 在ZFS on Linux上使用`ashift= <https://openzfs.github.io/openzfs-docs/Project%20and%20Community/FAQ.html#advanced-format-disks>`__
- -o ashift= 也适用于MacZFS（池版本8）和ZFS-OSX（池版本5000）。

-o ashift= 很方便，但它有一个缺陷，即创建包含具有多个最佳扇区大小的顶级vdev的池需要使用多个命令。已经讨论了一种新的语法，它将依赖于实际的扇区大小，并可能在未来作为跨平台替代方案实现。

此外，ZFS on Linux项目有一个`已知误报扇区大小的驱动器数据库 <https://github.com/openzfs/zfs/blob/master/cmd/zpool/os/linux/zpool_vdev_os.c#L98>`__。它用于自动调整ashift，而无需系统管理员的协助。这种方法无法完全补偿驱动器标识符使用不明确的情况（例如虚拟机、iSCSI LUN、一些罕见的SSD），但它做了很多好事。该格式大致与illumos的sd.conf兼容，预计其他实现将在未来版本中集成该数据库。严格来说，这个数据库不属于ZFS，但在Linux内核（尤其是旧版本）中修补的困难使得在ZFS本身中实现这一点成为必要。MacZFS也是如此。然而，FreeBSD和illumos都能够在正确的层中实现这一点。

压缩
~~~~

在内部，ZFS使用设备扇区大小的倍数分配数据，通常为512字节或4KB（见上文）。启用压缩后，可以为每个块分配较少的扇区。未压缩的块大小由``recordsize``（默认为128KB）或``volblocksize``（自v2.2以来默认为16KB）属性设置（用于文件系统与卷）。

以下压缩算法可用：

- LZ4

  - 在创建功能标志后添加的新算法。它在所有测试指标上显著优于LZJB。它是OpenZFS中的`新默认压缩算法 <https://github.com/illumos/illumos-gate/commit/db1741f555ec79def5e9846e6bfd132248514ffe>`__（compression=on）。截至2020年，它在所有平台上都可用。

- LZJB

  - ZFS的原始默认压缩算法（compression=on）。它创建是为了满足文件系统中使用的压缩算法的需求。具体来说，它提供了公平的压缩率，具有高压缩速度、高解压缩速度，并能快速检测不可压缩的数据。

- GZIP（1到9）

  - 经典的Lempel-Ziv实现。它提供了高压缩率，但通常会使IO成为CPU瓶颈。

- ZLE（零长度编码）

  - 一种非常简单的算法，仅压缩零。

- ZSTD（Zstandard）

  - Zstandard是一种现代的高性能通用压缩算法，它提供了与GZIP相似或更好的压缩级别，但性能更好。Zstandard提供了非常广泛的性能/压缩权衡，并由一个极快的解码器支持。它从`OpenZFS 2.0版本 <https://github.com/openzfs/zfs/pull/10278>`__开始可用。

如果您想使用压缩但不确定使用哪种算法，请使用LZ4。它的平均压缩比为2.1:1，而gzip-1的平均压缩比为2.7:1，但gzip要慢得多。这两个数字均来自`LZ4项目的测试 <https://github.com/lz4/lz4>`__，测试数据为Silesia语料库。gzip的更高压缩率通常只对很少访问的数据有价值。

.. _raid_z_stripe_width:

RAID-Z 条带宽度
~~~~~~~~~~~~~~~

根据您的IOPS需求和愿意为奇偶校验信息分配的空间量选择RAID-Z条带宽度。如果您需要更多的IOPS，请使用每条带较少的磁盘。如果您需要更多的可用空间，请使用每条带较多的磁盘。尝试根据确切数字优化RAID-Z条带宽度在几乎所有情况下都是无关紧要的。有关更多详细信息，请参阅此`博客文章 <https://www.delphix.com/blog/delphix-engineering/zfs-raidz-stripe-width-or-how-i-learned-stop-worrying-and-love-raidz/>`__。

.. _dataset_recordsize:

数据集记录大小
~~~~~~~~~~~~~~~

ZFS数据集的默认内部记录大小为128KB。数据集记录大小是用于文件内部写时复制的基本数据单元。部分记录写入需要从ARC（廉价）或磁盘（昂贵）读取数据。recordsize可以设置为从512字节到1兆字节的任何2的幂次方。以固定记录大小写入的软件（例如数据库）将受益于使用匹配的记录大小。

更改数据集上的recordsize仅对新文件生效。如果您因为应用程序在另一个记录大小下表现更好而更改recordsize，则需要重新创建其文件。对每个文件执行cp后跟mv就足够了。或者，当执行完整接收时，send/recv应重新创建具有正确记录大小的文件。

.. _larger_record_sizes:

更大的记录大小
^^^^^^^^^^^^^^^

支持高达16M的记录大小，使用large_blocks池功能，该功能在支持它的系统上的新池中默认启用。

在openZFS v2.2之前，大于1M的记录大小默认被禁用，除非设置了zfs_max_recordsize内核模块参数以允许大于1M的大小。

\`zfs send\`操作必须指定-L以确保发送大于128KB的块，并且接收池必须支持large_blocks功能。

.. _zvol_volblocksize:

zvol 卷块大小
~~~~~~~~~~~~~

zvol有一个``volblocksize``属性，类似于``recordsize``。当前默认值（自v2.2以来为16KB）平衡了元数据开销、压缩机会和大多数池配置上的良好空间效率，由于4KB磁盘物理块舍入（尤其是在RAIDZ和DRAID上），同时在使用较小块大小的客户机文件系统上产生一些写放大[#VOLBLOCKSIZE]_。

建议用户测试他们的场景，看看是否需要更改``volblocksize``以偏向于以下之一：

- 客户机文件系统的扇区对齐至关重要
- 大多数客户机文件系统使用4-8KB的默认块大小，因此：

  - 较大的``volblocksize``有助于主要顺序工作负载，并将获得压缩效率

  - 较小的``volblocksize``有助于随机工作负载并最小化IO放大，但会使用更多元数据（例如，ZFS将生成更多小IO），并且可能具有较差的空间效率（尤其是在RAIDZ和DRAID上）

  - 将``volblocksize``设置为小于客户机文件系统的块大小或:ref:`ashift <alignment_shift_ashift>`是没有意义的

  - 有关更多信息，请参阅:ref:`数据集记录大小 <dataset_recordsize>`

去重
~~~~

去重使用磁盘上的哈希表，使用`可扩展哈希 <http://en.wikipedia.org/wiki/Extensible_hashing>`__，如ZAP（ZFS属性处理器）中实现的那样。每个缓存条目使用略多于320字节的内存。DDT代码依赖于ARC来缓存DDT条目，因此没有双重缓存或内核内存分配器的内部碎片。每个池都有一个全局去重表，共享所有启用了去重的数据集和zvol。哈希表中的每个条目都是池中唯一块的记录。（其中块大小由``recordsize``或``volblocksize``属性设置。）

哈希表（也称为DDT或去重表）必须为每个写入或释放的去重块访问（无论它是否有多个引用）。如果没有足够的内存将DDT缓存在内存中，每次缓存未命中都需要从磁盘读取一个随机块，导致性能下降。例如，如果在单个7200RPM驱动器上操作，该驱动器可以执行100 io/s，未缓存的DDT读取将限制整体写入吞吐量为每秒100个块，或4KB块时为400KB/s。

后果是，需要足够的内存来存储去重数据以获得良好的性能。去重数据被视为元数据，因此如果``primarycache``或``secondarycache``属性设置为``metadata``，则可以缓存。此外，去重表将与其他元数据竞争元数据存储，这可能对性能产生负面影响。可以使用zdb的-D选项模拟给定池所需的去重表条目数。然后可以通过乘以320字节来获得近似的内存需求。或者，您可以通过将计划在每个数据集上使用的存储量（考虑到部分记录每个都计为去重目的的完整记录大小）除以记录大小，每个zvol除以卷块大小，求和然后乘以320字节来估计唯一块数的上限。

.. _metaslab_allocator:

元数据块分配器
~~~~~~~~~~~~~~~

ZFS顶级vdev被划分为元数据块，可以从中独立分配块，以允许并发IO执行分配而不会相互阻塞。目前，`Linux和Mac OS X端口上存在一个回归 <https://github.com/zfsonlinux/zfs/pull/3643>`__，导致序列化发生。

默认情况下，元数据块的选择偏向于较低的LBA，以提高旋转磁盘的性能，但这在固态介质上没有意义。可以通过将ZFS模块的全局metaslab_lba_weighting_enabled可调参数设置为0来全局调整此行为。此可调参数仅建议在仅使用固态介质的系统上使用。

元数据块分配器将在元数据块具有大于或等于4％空闲空间时以首次适应方式分配块，而在元数据块具有小于4％空闲空间时以最佳适应方式分配块。前者比后者快得多，但无法从池的空闲空间判断何时发生此行为。但是，命令``zdb -mmm $POOLNAME``将提供此信息。

.. _pool_geometry:

池几何
~~~~~~

如果小随机IOPS是主要关注点，镜像vdev将优于raidz vdev。镜像上的读取IOPS将随每个镜像中的驱动器数量扩展，而raidz vdev将受限于最慢驱动器的IOPS。

如果顺序写入是主要关注点，raidz将优于镜像vdev。顺序写入吞吐量随raidz中的数据磁盘数量线性增加，而写入受限于镜像vdev中最慢的驱动器。顺序读取性能在每种情况下应大致相同。

无论它们是raidz还是镜像，IOPS和吞吐量都将随每个顶级vdev的IOPS和吞吐量的总和增加。

.. _whole_disks_versus_partitions:

整盘与分区
~~~~~~~~~~~

ZFS在不同平台上在给定整盘时的行为会有所不同。

在illumos上，ZFS尝试在整盘上启用写缓存。illumos UFS驱动程序无法确保启用写缓存的完整性，因此默认情况下，使用UFS文件系统启动的Sun/Solaris系统（很久以前，当Sun还是一个独立公司时）出厂时驱动器写缓存被禁用。为了在illumos上的安全起见，如果ZFS没有获得整盘，它可能与UFS共享，因此ZFS启用写缓存是不合适的。在这种情况下，写缓存设置不会更改，并将保持不变。今天，大多数供应商出厂时驱动器写缓存默认启用。

在Linux上，Linux IO电梯在很大程度上是多余的，因为ZFS有自己的IO电梯。

ZFS在illumos上的x86/amd64和Linux上给定整盘时还会创建一个GPT分区表自己的分区。这主要是为了使通过UEFI启动成为可能，因为UEFI需要一个小FAT分区才能启动系统。ZFS驱动程序将能够通过标签中的whole_disk字段区分池是否获得了整个磁盘。

在FreeBSD上不这样做。由FreeBSD创建的池将始终将whole_disk字段设置为true，因此在另一个平台上导入的池将始终被视为ZFS获得了整盘。

.. _OS_specific:

操作系统/发行版特定建议
------------------------

.. _linux_specific:

Linux
~~~~~

init_on_alloc
^^^^^^^^^^^^^
一些Linux发行版（至少Debian、Ubuntu）默认启用``init_on_alloc``选项作为安全预防措施。此选项可以帮助[#init_on_alloc]_：

  防止可能的信息泄露
  使依赖于未初始化值的控制流错误更具确定性。

不幸的是，它可能会显著降低ARC吞吐量（参见`bug <https://github.com/openzfs/zfs/issues/9910>`__）。

如果您准备好应对这些安全风险[#init_on_alloc]_，可以通过在GRUB内核启动参数中设置``init_on_alloc=0``来禁用它。

.. _general_recommendations:

一般建议
--------

.. _alignment_shift:

对齐偏移
~~~~~~~~

确保创建池时vdev具有正确的对齐偏移以适应存储设备的大小。如果处理闪存介质，这将是12（4K扇区）或13（8K扇区）。对于Amazon EC2上的SSD临时存储，正确的设置是12。

.. _atime_updates:

访问时间更新
~~~~~~~~~~~~

设置relatime=on或atime=off以最小化用于更新访问时间戳的IO。为了与支持它的一小部分软件向后兼容，relatime在可用时应优先设置在整个池上。atime=off应更有选择性地使用。

.. _free_space:

空闲空间
~~~~~~~~

保持池的空闲空间高于10％，以避免许多元数据块达到4％空闲空间阈值，从而从首次适应切换到最佳适应分配策略。当达到阈值时，:ref:`元数据块分配器 <metaslab_allocator>`变得非常CPU密集，以保护自己免受碎片化的影响。这会降低IOPS，尤其是当更多元数据块达到4％阈值时。

建议是10％而不是5％，因为元数据块选择考虑位置和空闲空间，除非全局metaslab_lba_weighting_enabled可调参数设置为0。当该可调参数为0时，ZFS将仅考虑空闲空间，因此通过保持空闲空间高于5％可以避免最佳适应分配器的开销。该设置应仅用于由固态驱动器组成的池，因为它会降低机械磁盘上的顺序IO性能。

.. _lz4_compression:

LZ4压缩
~~~~~~~

在池的根数据集上设置compression=lz4，以便所有数据集继承它，除非您有理由不启用它。LZ4压缩不可压缩数据的单线程用户空间测试表明，它可以处理10GB/秒，因此即使在不可压缩数据上也不太可能成为瓶颈。此外，不可压缩数据将不压缩存储，因此启用压缩的不可压缩数据读取不会受到解压缩的影响。写入速度如此之快，以至于不可压缩数据不太可能因使用LZ4压缩而受到性能损失。LZ4减少的IO通常会是性能上的胜利。

请注意，较大的记录大小将通过允许压缩算法一次处理更多数据来提高可压缩数据的压缩比。

.. _nvme_low_level_formatting_link:

NVMe低级格式化
~~~~~~~~~~~~~~

参见:ref:`nvme低级格式化 <nvme_low_level_formatting>`。

.. _pool_geometry_1:

池几何
~~~~~~

不要在raidz中放置超过约16个磁盘。当池满时，机械磁盘上的重建时间将过长。

.. _synchronous_io:

同步IO
~~~~~~

如果工作负载涉及fsync或O_SYNC，并且池由机械存储支持，请考虑添加一个或多个SLOG设备。具有多个SLOG设备的池将在它们之间分配ZIL操作。SLOG设备的最佳选择可能是Optane / 3D XPoint SSD。有关它们的描述，请参见:ref:`optane_3d_xpoint_ssds`。如果Optane / 3D XPoint SSD是一个选项，则无需阅读本节其余部分。如果Optane / 3D XPoint SSD不是选项，请参见:ref:`nand_flash_ssds`以获取NAND闪存SSD的建议，并阅读以下信息。

为了确保基于NAND闪存SSD的SLOG设备上的最大ZIL性能，您还应过度配置备用区域以增加IOPS[#ssd_iops]_。仅需要约4GB，因此其余部分可以留作过度配置的存储。4GB的选择有些随意。大多数系统在事务组提交之间不会向ZIL写入接近4GB的内容，因此过度配置所有存储超过4GB分区应该是可以的。如果工作负载需要更多，则使其不超过最大ARC大小。即使在极端工作负载下，ZFS也不会从超过最大ARC大小的SLOG存储中受益。在Linux上是系统内存的一半，在illumos上是系统内存的3/4。

.. _overprovisioning_by_secure_erase_and_partition_table_trick:

通过安全擦除和分区表技巧进行过度配置
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

您可以通过安全擦除和分区表技巧的组合来实现这一点，例如以下步骤：

#. 在NAND闪存SSD上运行安全擦除。
#. 在NAND闪存SSD上创建分区表。
#. 创建一个4GB分区。
#. 将该分区提供给ZFS用作日志设备。

如果使用安全擦除和分区表技巧，请*不要*将未分区的空间用于其他用途，即使是暂时的。这将通过将页面标记为脏来减少或消除过度配置。

或者，一些设备允许您更改它们报告的大小。这也将起作用，尽管在更改报告大小之前应进行安全擦除以确保SSD识别额外的备用区域。可以使用\`hdparm -N \`在支持它的驱动器上更改报告大小，前提是系统安装了laptop-mode-tools。

.. _nvme_overprovisioning:

NVMe过度配置
^^^^^^^^^^^^

在NVMe上，您可以使用命名空间来实现过度配置：

#. 作为预防措施，执行清理命令以确保设备完全干净。
#. 删除默认命名空间。
#. 创建一个大小为4GB的新命名空间。
#. 将该命名空间提供给ZFS用作日志设备。例如，zfs add tank log /dev/nvme1n1

.. _whole_disks:

整盘
~~~~

应将整盘提供给ZFS而不是分区。如果必须使用分区，请确保分区正确对齐以避免读-修改-写开销。有关正确对齐的描述，请参见:ref:`对齐偏移 (ashift) <alignment_shift_ashift>`部分。此外，请参见:ref:`整盘与分区 <whole_disks_versus_partitions>`部分以了解在分区上操作时ZFS行为的变化。

来自RAID控制器的单盘RAID 0阵列不等同于整盘。:ref:`硬件RAID控制器 <hardware_raid_controllers>`页面详细解释了这一点。

.. _bit_torrent:

BitTorrent
----------

BitTorrent执行16KB随机读取/写入。16KB写入会导致读-修改-写开销。当写入的数据量超过系统内存时，读-修改-写开销可能会使性能降低16倍（使用128KB记录大小时）。可以通过使用recordsize=16KB的专用数据集来避免这种情况。

当通过HTTP服务器顺序读取文件时，文件生成时的随机性会导致碎片化，这在7200RPM硬盘上观察到会使顺序读取性能降低一半。如果性能是一个问题，可以通过以下两种方式之一顺序重写文件来消除碎片化：

第一种方法是配置客户端将文件下载到临时目录，然后在下载完成后将它们复制到最终位置，前提是您的客户端支持此操作。

第二种方法是使用send/recv顺序重新创建数据集。

实际上，仅当通过BitTorrent获得的文件存储在磁性存储上并在创建后受到显著的顺序读取工作负载时，对文件进行碎片整理才应提高性能。

.. _database_workloads:

数据库工作负载
--------------

设置``redundant_metadata=most``可以通过消除间接块树最低级别的冗余元数据来至少提高几个百分点的IOPS。这带来的警告是，如果指向数据块的元数据块损坏且没有重复副本，则会发生数据丢失，但这在生产环境中通常不是问题，尤其是在镜像或raidz vdev上。

MySQL
~~~~~

InnoDB
^^^^^^

为InnoDB的数据文件和日志文件创建单独的数据集。在InnoDB的数据文件上设置``recordsize=16K``以避免昂贵的部分记录写入，并在日志文件上保留recordsize=128K。在两者上设置``primarycache=metadata``以优先使用InnoDB的缓存[#mysql_basic]_。在数据上设置``logbias=throughput``以阻止ZIL写入两次。

在my.cnf中设置``innodb_doublewrite=0``以防止innodb写入两次。双写是一个数据完整性功能，旨在防止部分写入记录导致的损坏，但在ZFS上这是不可能的。应该注意的是，`Percona的博客曾提倡 <https://www.percona.com/blog/2014/05/23/improve-innodb-performance-write-bound-loads/>`__使用ext4配置，其中双写被关闭以获得性能提升，但后来撤回了这一建议，因为它导致了数据损坏。在适时的电源故障后，ext4等就地文件系统可能会有一半的8KB记录是旧的，而另一半是新的。这将导致Percona撤回其建议的损坏。然而，ZFS的写时复制设计将导致它在电源故障后返回旧的正确数据（无论时间如何）。这防止了双写功能旨在防止的损坏发生。因此，在ZFS上双写功能是不必要的，可以安全地关闭以获得更好的性能。

在Linux上，驱动程序的AIO实现是一个兼容性垫片，仅勉强通过POSIX标准。InnoDB在使用其默认AIO代码路径时性能会受到影响。在my.cnf中设置``innodb_use_native_aio=0``和``innodb_use_atomic_writes=0``以禁用AIO。必须同时禁用这两个设置才能禁用AIO。

PostgreSQL
~~~~~~~~~~

为PostgreSQL的数据和WAL创建单独的数据集。在两者上设置``compression=lz4``和``recordsize=32K``（64K也适用，128K默认值也适用）。为PostgreSQL配置``full_page_writes = off``，因为ZFS永远不会提交部分写入。对于具有大量更新的数据库，可以在PostgreSQL的数据上尝试``logbias=throughput``以避免写入两次，但请注意，使用此设置较小的更新可能会导致严重的碎片化。

SQLite
~~~~~~

为数据库创建单独的数据集。将记录大小设置为64K。将SQLite页面大小设置为65536字节[#sqlite_ps]_。

请注意，SQLite数据库通常不会受到足够的锻炼以进行特殊调优，但这将提供它。请注意SQLite.org上提到的缓存大小变化的副作用[#sqlite_ps_change]_。

.. _file_servers:

文件服务器
----------

为正在提供的文件创建专用数据集。

参见:ref:`顺序工作负载 <sequential_workloads>`以获取配置建议。

Samba
~~~~~
Windows/DOS客户端不支持区分大小写的文件名。如果主要工作负载不需要其他支持的客户端的区分大小写，请使用``zfs create -o casesensitivity=insensitive``创建数据集，以便Samba将来可以更快地搜索文件名[#FS_CASEFOLD_FL]_。

参见``smb.conf(5) <https://www.samba.org/samba/docs/current/man-html/smb.conf.5.html>`__中的``case sensitive``选项。

.. _sequential_workloads:

顺序工作负载
------------

在受顺序工作负载影响的数据集上设置``recordsize=1M``。在设置1M记录大小之前，请阅读:ref:`更大的记录大小 <larger_record_sizes>`以了解应知事项。

按照:ref:`LZ4压缩 <lz4_compression>`的一般建议设置``compression=lz4``。

.. _video_games_directories:

视频游戏目录
------------

创建专用数据集，使用chown使其对用户可访问（或在其下创建目录并使用chown），然后配置游戏下载应用程序将游戏放置在那里。有关如何配置各种应用程序的具体信息如下。

在安装游戏之前，请参见:ref:`顺序工作负载 <sequential_workloads>`以获取配置建议。

请注意，此调优的性能提升可能很小，仅限于加载时间。然而，1M记录和LZ4的组合将允许存储更多游戏，这就是为什么尽管性能提升有限，仍记录此调优的原因。一个包含300个游戏（主要来自humble bundle）的Steam库应用了这些调整后，看到了20％的空间节省。当此调优完成后，可压缩游戏的加载时间更快和显著的空间节省是可能的。已经压缩的游戏的资产将看到很少或没有好处。

Lutris
~~~~~~

通过左键单击右上角的三条线图标打开上下文菜单。转到“Preferences”，然后选择“System options”选项卡。更改默认安装目录并单击保存。

Steam
~~~~~

转到“Settings” -> “Downloads” -> “Steam Library Folders”并使用“Add Library Folder”设置Steam用于存储游戏的目录。在关闭对话框之前，请确保通过右键单击它并单击“Make Default Folder”将其设置为默认目录。

如果您将使用Proton运行非原生游戏，请使用``zfs create -o casesensitivity=insensitive``创建数据集，以便Wine将来可以更快地搜索文件名[#FS_CASEFOLD_FL]_。

.. _wine:

Wine
----

Windows文件系统的标准行为是不区分大小写。使用``zfs create -o casesensitivity=insensitive``创建数据集，以便Wine将来可以更快地搜索文件名[#FS_CASEFOLD_FL]_。

.. _virtual_machines:

虚拟机
------

ZFS上的虚拟机映像应使用zvol或原始文件存储，以避免不必要的开销。可以配置recordsize/volblocksize和客户机文件系统以匹配，以避免部分记录修改的开销，请参见:ref:`zvol volblocksize <zvol_volblocksize>`。如果使用原始文件，应使用单独的数据集以便轻松配置recordsize，独立于ZFS上存储的其他内容。

.. _qemu_kvm_xen:

QEMU / KVM / Xen
~~~~~~~~~~~~~~~~

使用文件作为客户机存储时，应使用AIO以最大化IOPS。

.. rubric:: 脚注

.. [#ssd_iops] <http://www.anandtech.com/show/6489/playing-with-op>
.. [#mysql_basic] <https://www.patpro.net/blog/index.php/2014/03/09/2617-mysql-on-zfs-on-freebsd/>
.. [#sqlite_ps] <https://www.sqlite.org/pragma.html#pragma_page_size>
.. [#sqlite_ps_change] <https://www.sqlite.org/pgszchng2016.html>
.. [#FS_CASEFOLD_FL] <https://github.com/openzfs/zfs/pull/13790>
.. [#init_on_alloc] <https://patchwork.kernel.org/project/linux-security-module/patch/20190626121943.131390-2-glider@google.com/#22731857>
.. [#VOLBLOCKSIZE] <https://github.com/openzfs/zfs/pull/12406>