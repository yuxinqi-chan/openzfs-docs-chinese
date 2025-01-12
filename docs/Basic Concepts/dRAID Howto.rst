dRAID
=====

.. note::
   本页面描述了 OpenZFS 2.1.0 版本中添加的功能，该功能未包含在 OpenZFS 2.0.0 版本中。

简介
~~~~~~~~~~~~

`dRAID`_ 是 raidz 的一种变体，它提供了集成的分布式热备盘，能够在保留 raidz 优点的同时实现更快的重建速度。dRAID vdev 由多个内部的 raidz 组构成，每个组包含 D 个数据设备和 P 个校验设备。这些组分布在所有子设备上，以充分利用可用的磁盘性能。这被称为奇偶校验分散（parity declustering），它一直是研究的热点领域。下图是简化的，但它有助于说明 dRAID 和 raidz 之间的关键区别。

|draid1|

此外，dRAID vdev 必须以某种方式对其子 vdev 进行洗牌，以确保无论哪个驱动器发生故障，重建 IO（包括读取和写入）都会均匀分布在所有幸存的驱动器上。这是通过使用精心选择的预计算排列映射来实现的。这样做的好处是既保持了池创建的快速性，又确保了映射不会被损坏或丢失。

dRAID 与 raidz 的另一个区别是它使用固定的条带宽度（必要时用零填充）。这使得 dRAID vdev 可以按顺序重建，但固定条带宽度会显著影响可用容量和 IOPS。例如，默认的 D=8 和 4k 磁盘扇区的最小分配大小为 32k。如果使用压缩，这种相对较大的分配大小可能会降低有效压缩比。当使用 ZFS 卷和 dRAID 时，默认的 volblocksize 属性会增加以考虑分配大小。如果 dRAID 池将存储大量小数据块，建议添加一个镜像的特殊 vdev 来存储这些数据块。

在 IO/s 方面，性能与 raidz 相似，因为对于任何读取操作，都必须访问所有 D 个数据磁盘。交付的随机 IOPS 可以合理地近似为 floor((N-S)/(D+P))*<单盘 IOPS>。

总之，dRAID 可以提供与 raidz 相同级别的冗余和性能，同时还提供了快速的集成分布式热备盘。

创建 dRAID vdev
~~~~~~~~~~~~~~~~~~~

通过使用 ``zpool create`` 命令并枚举应使用的磁盘，可以像创建其他 vdev 一样创建 dRAID vdev。

::

   # zpool create <pool> draid[1,2,3] <vdevs...>

与 raidz 类似，奇偶校验级别在 ``draid`` vdev 类型后立即指定。然而，与 raidz 不同的是，可以指定额外的冒号分隔选项。其中最重要的是 ``:<spares>s`` 选项，它控制要创建的分布式热备盘的数量。默认情况下，不创建热备盘。可以指定 ``:<data>d`` 选项来设置每个 RAID 条带中使用的数据设备数量（D+P）。如果未指定，则会选择合理的默认值。

::

   # zpool create <pool> draid[<parity>][:<data>d][:<children>c][:<spares>s] <vdevs...>

- **parity** - 奇偶校验级别（1-3）。默认为 1。

- **data** - 每个冗余组中的数据设备数量。通常，较小的 D 值会增加 IOPS，提高压缩比，并加快重建速度，但会牺牲总可用容量。默认为 8，除非 N-P-S 小于 8。

- **children** - 预期的子设备数量。在列出大量设备时作为交叉检查很有用。当提供的子设备数量不同时，会返回错误。

- **spares** - 分布式热备盘的数量。默认为零。

例如，要创建一个具有 4+1 冗余和单个分布式热备盘的 11 磁盘 dRAID 池，命令如下：

::

   # zpool create tank draid:4d:1s:11c /dev/sd[a-k]
   # zpool status tank

     pool: tank
    state: ONLINE
   config:

           NAME                  STATE     READ WRITE CKSUM
           tank                  ONLINE       0     0     0
             draid1:4d:11c:1s-0  ONLINE       0     0     0
               sda               ONLINE       0     0     0
               sdb               ONLINE       0     0     0
               sdc               ONLINE       0     0     0
               sdd               ONLINE       0     0     0
               sde               ONLINE       0     0     0
               sdf               ONLINE       0     0     0
               sdg               ONLINE       0     0     0
               sdh               ONLINE       0     0     0
               sdi               ONLINE       0     0     0
               sdj               ONLINE       0     0     0
               sdk               ONLINE       0     0     0
           spares
             draid1-0-0          AVAIL

请注意，dRAID vdev 名称 ``draid1:4d:11c:1s`` 完全描述了配置，并且列出了所有属于 dRAID 的磁盘。此外，逻辑分布式热备盘显示为可用的备用磁盘。

重建到分布式热备盘
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

dRAID 的主要优势之一是它支持顺序重建和传统修复重建。当执行顺序重建到分布式热备盘时，性能会随着磁盘数量除以条带宽度（D+P）而扩展。这可以大大减少重建时间，并在通常时间的一小部分内恢复完全冗余。例如，下图显示了基于 90 个 HDD 的 dRAID 在 90% 容量下的顺序重建时间（以小时为单位）。

|draid-resilver|

当使用 dRAID 和分布式热备盘时，处理故障磁盘的过程与使用传统热备盘的 raidz 几乎相同。当检测到磁盘故障时，ZFS 事件守护进程（ZED）将开始重建到备用磁盘（如果有）。唯一的区别是，对于 dRAID，会启动顺序重建，而对于 raidz，必须使用修复重建。

::

   # echo offline >/sys/block/sdg/device/state
   # zpool replace -s tank sdg draid1-0-0
   # zpool status

     pool: tank
    state: DEGRADED
   status: 一个或多个设备当前正在重建中。池将继续运行，可能处于降级状态。
   action: 等待重建完成。
     scan: 重建 (draid1:4d:11c:1s-0) 自 2020 年 11 月 24 日 星期二 14:34:25 开始
           3.51T 已扫描，速度为 13.4G/s，1.59T 已发出，速度为 6.07G/s，总计 6.13T
           326G 已重建，完成 57.17%，预计剩余时间 00:03:21
   config:

           NAME                  STATE     READ WRITE CKSUM
           tank                  DEGRADED     0     0     0
             draid1:4d:11c:1s-0  DEGRADED     0     0     0
               sda               ONLINE       0     0     0  (重建中)
               sdb               ONLINE       0     0     0  (重建中)
               sdc               ONLINE       0     0     0  (重建中)
               sdd               ONLINE       0     0     0  (重建中)
               sde               ONLINE       0     0     0  (重建中)
               sdf               ONLINE       0     0     0  (重建中)
               spare-6           DEGRADED     0     0     0
                 sdg             UNAVAIL      0     0     0
                 draid1-0-0      ONLINE       0     0     0  (重建中)
               sdh               ONLINE       0     0     0  (重建中)
               sdi               ONLINE       0     0     0  (重建中)
               sdj               ONLINE       0     0     0  (重建中)
               sdk               ONLINE       0     0     0  (重建中)
           spares
             draid1-0-0          INUSE     当前正在使用

虽然两种类型的重建都能达到相同的目标，但值得花点时间总结一下关键区别。

- 传统的修复重建会扫描整个块树。这意味着每个块的校验和在修复时可用，并且可以立即验证。缺点是这会创建一个随机读取工作负载，不利于性能。

- 顺序重建则扫描空间映射，以确定哪些空间已分配以及哪些空间必须修复。此重建过程不受块边界限制，可以顺序读取磁盘并使用较大的 I/O 进行修复。这种性能改进的代价是在重建期间无法验证块校验和。因此，在顺序重建完成后会启动一次 scrub 来验证校验和。

有关顺序重建和修复重建之间差异的更深入解释，请查看这些在 OpenZFS 开发者峰会上展示的 `顺序重建`_ 幻灯片。

重新平衡
~~~~~~~~~~~

通过简单地用新驱动器替换任何故障驱动器，可以再次使分布式备用空间可用。此过程称为重新平衡，本质上是一次重建。在执行重新平衡时，建议使用修复重建，因为池不再处于降级状态。这确保在重建到新磁盘时验证所有校验和，并消除了随后对池执行 scrub 的需要。

::

   # zpool replace tank sdg sdl
   # zpool status

     pool: tank
    state: DEGRADED
   status: 一个或多个设备当前正在重建中。池将继续运行，可能处于降级状态。
   action: 等待重建完成。
     scan: 重建自 2020 年 11 月 24 日 星期二 14:45:16 开始
           6.13T 已扫描，速度为 7.82G/s，6.10T 已发出，速度为 7.78G/s，总计 6.13T
           565G 已重建，完成 99.44%，预计剩余时间 00:00:04
   config:

           NAME                  STATE     READ WRITE CKSUM
           tank                  DEGRADED     0     0     0
             draid1:4d:11c:1s-0  DEGRADED     0     0     0
               sda               ONLINE       0     0     0  (重建中)
               sdb               ONLINE       0     0     0  (重建中)
               sdc               ONLINE       0     0     0  (重建中)
               sdd               ONLINE       0     0     0  (重建中)
               sde               ONLINE       0     0     0  (重建中)
               sdf               ONLINE       0     0     0  (重建中)
               spare-6           DEGRADED     0     0     0
                 replacing-0     DEGRADED     0     0     0
                   sdg           UNAVAIL      0     0     0
                   sdl           ONLINE       0     0     0  (重建中)
                 draid1-0-0      ONLINE       0     0     0  (重建中)
               sdh               ONLINE       0     0     0  (重建中)
               sdi               ONLINE       0     0     0  (重建中)
               sdj               ONLINE       0     0     0  (重建中)
               sdk               ONLINE       0     0     0  (重建中)
           spares
          draid1-0-0          INUSE     当前正在使用

重建完成后，分布式热备盘再次可用，池已恢复到正常的健康状态。

.. |draid1| image:: /_static/img/raidz_draid.png
.. |draid-resilver| image:: /_static/img/draid-resilver-hours.png
.. _dRAID: https://docs.google.com/presentation/d/1uo0nBfY84HIhEqGWEx-Tbm8fPbJKtIP3ICo4toOPcJo/edit
.. _sequential resilver: https://docs.google.com/presentation/d/1vLsgQ1MaHlifw40C9R2sPsSiHiQpxglxMbK2SMthu0Q/edit#slide=id.g995720a6cf_1_39
.. _custom packages: https://openzfs.github.io/openzfs-docs/Developer%20Resources/Custom%20Packages.html#