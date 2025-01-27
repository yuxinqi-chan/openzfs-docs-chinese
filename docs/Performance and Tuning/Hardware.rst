硬件
********

.. contents:: 目录
  :local:

介绍
============

在ZFS之前，存储依赖于相当昂贵的硬件，这些硬件无法防止静默数据损坏，并且扩展性较差。ZFS的引入使得人们能够使用比行业中以前使用的硬件便宜得多的硬件，并且具有更好的扩展性。本页面试图为购买用于基于ZFS的服务器和工作站的硬件提供一些基本指导。

遵循此指导的硬件将使ZFS能够充分发挥其性能和可靠性潜力。不遵循此指导的硬件将成为一个障碍。除非另有说明，否则这些障碍适用于所有存储堆栈，并非ZFS特有。使用竞争存储堆栈构建的系统也将从这些建议中受益。

.. _bios_cpu_microcode_updates:

BIOS / CPU 微码更新
============================

强烈建议运行最新的BIOS和CPU微码。

背景
----------

计算机微处理器是非常复杂的设计，通常会有一些错误，称为“勘误表”。现代微处理器设计为使用微码。这使得部分硬件设计可以放入准软件中，无需更换整个芯片即可进行修补。勘误表通常通过CPU微码更新来解决。这些更新通常包含在BIOS更新中。在某些情况下，BIOS通过机器寄存器与CPU的交互可以被修改以修复相同微码的问题。如果较新的微码未作为BIOS更新的一部分捆绑，通常可以通过操作系统引导加载程序或操作系统本身加载。

.. _ecc_memory:

ECC 内存
==========

位翻转对所有计算机文件系统都可能产生相当严重的后果，ZFS也不例外。ZFS（或任何其他文件系统）中使用的技术都无法防止位翻转。因此，强烈建议使用ECC内存。

.. _background_1:

背景
----------

普通的背景辐射会随机翻转计算机内存中的位，导致未定义的行为。这些被称为“位翻转”。每个位翻转可能产生四种可能的后果，具体取决于翻转的位：

- 位翻转可能没有影响。

   - 未使用的内存中的位翻转可能没有影响。

- 位翻转可能导致运行时故障。

   - 当从磁盘读取的内容发生位翻转时，会出现这种情况。
   - 通常当程序代码被修改时会观察到故障。
   - 如果位翻转发生在系统内核或/sbin/init中的例程中，系统可能会崩溃。否则，重新加载受影响的数据可以清除它。通常通过重启来实现。

- 它可能导致数据损坏。

   - 当位被用于写入磁盘的数据时，会出现这种情况。
   - 如果位翻转发生在ZFS的校验和计算之前，ZFS将不会意识到数据已损坏。
   - 如果位翻转发生在ZFS的校验和计算之后，但在写入之前，ZFS将检测到它，但可能无法纠正它。

- 它可能导致元数据损坏。

   - 当位翻转发生在写入磁盘的磁盘结构时，会出现这种情况。
   - 如果位翻转发生在ZFS的校验和计算之前，ZFS将不会意识到元数据已损坏。
   - 如果位翻转发生在ZFS的校验和计算之后，但在写入之前，ZFS将检测到它，但可能无法纠正它。
   - 从此类事件中恢复将取决于损坏的内容。在最坏的情况下，池可能变得无法导入。

      - 所有文件系统在其最坏的位翻转故障场景中都具有较差的可靠性。这些场景应被视为极其罕见。

.. _drive_interfaces:

驱动器接口
================

.. _sas_versus_sata:

SAS 与 SATA
---------------

ZFS依赖于块设备层进行存储。因此，ZFS受到影响其他文件系统的相同因素的影响，例如驱动程序支持和非工作硬件。因此，有几点需要注意：

- 切勿在没有SAS转接器的情况下将SATA磁盘放入SAS扩展器中。

   - 如果你这样做并且它确实有效，那是个例外，而不是规则。

- 不要期望SAS控制器与SATA端口倍增器兼容。

   - 通常不会测试此配置。
   - 磁盘可能无法识别。

- 对SATA端口倍增器的支持在OpenZFS平台上不一致

   - Linux驱动程序通常支持它们。
   - Illumos驱动程序通常不支持它们。
   - FreeBSD驱动程序的支持介于Linux和Illumos之间。

.. _usb_hard_drives_andor_adapters:

USB 硬盘和/或适配器
-------------------------------

这些设备在扇区大小报告、SMART透传、设置ERC的能力等方面存在问题。ZFS在这些设备上的表现将取决于它们的能力，但尽量避免使用它们。不应期望它们具有与SAS和SATA驱动器相同的正常运行时间，应视为不可靠。

控制器
===========

理想的ZFS存储控制器具有以下属性：

- 在主要OpenZFS平台上的驱动程序支持

   - 稳定性很重要。

- 高每端口带宽

   - PCI Express接口带宽除以端口数

- 低成本

   - 不需要支持RAID、电池备份单元和硬件写缓存。

Marc Bevand的博客文章 `从32到2端口：ZFS和Linux MD RAID的理想SATA/SAS控制器 <http://blog.zorinaq.com/?e=10>`__ 包含一个满足这些标准的存储控制器的优秀列表。随着新控制器的出现，他会定期更新。

.. _hardware_raid_controllers:

硬件 RAID 控制器
-------------------------

不应将硬件RAID控制器与ZFS一起使用。虽然ZFS在硬件RAID上可能比其他文件系统更可靠，但它不会像在独立使用时那样可靠。

- 硬件RAID将限制ZFS在校验和故障时执行自我修复的机会。当ZFS执行RAID-Z或镜像时，一个磁盘上的校验和故障可以通过将包含该扇区的磁盘视为坏盘来纠正原始信息。当RAID控制器处理冗余时，除非ZFS存储了副本，否则无法完成此操作。如果设置了副本标志或RAID是ZFS内的镜像/raid-z vdev的一部分，则元数据损坏可能是可修复的。

- 硬件RAID在RAID 1上不一定能正确传递扇区大小信息。在RAID 5/6上无法正确传递扇区大小信息。
   硬件RAID 1更可能因部分扇区写入而产生读-修改-写开销，而硬件RAID 5/6几乎肯定会因部分条带写入（即RAID写入漏洞）而受到影响。ZFS原生使用磁盘允许它获取磁盘报告的扇区大小信息，以避免扇区的读-修改-写，而ZFS通过设计使用写时复制避免了RAID-Z上的部分条带写入。

   - 当驱动器错误报告其扇区大小时，ZFS可能会出现扇区对齐问题。此类驱动器通常是基于NAND闪存的固态硬盘和Windows XP EoL之前高级格式（4K扇区大小）过渡期间的旧SATA驱动器。可以在vdev创建时手动纠正此问题。
   - RAID头可能会导致RAID 1上的扇区写入未对齐，因为阵列从实际驱动器的扇区内开始，因此在vdev创建时手动纠正扇区对齐无法解决此问题。

- RAID控制器故障可能需要用相同型号的控制器替换，或者在不太极端的情况下，用同一制造商的型号替换。单独使用ZFS允许使用任何控制器。

- 如果使用硬件RAID控制器的写缓存，则会引入一个额外的故障点，只能通过增加复杂性来部分缓解，例如添加闪存以在电源丢失事件中保存数据。如果电池在需要时失效或没有闪存且电源未及时恢复，缓存中的数据仍可能丢失。写缓存中的数据丢失可能会严重损坏存储在RAID阵列上的任何数据，尤其是在缓存中有许多未完成的写入时。此外，所有写入都存储在缓存中，而不仅仅是需要写缓存的同步写入，这是低效的，并且写缓存相对较小。ZFS允许将同步写入直接写入闪存，这应提供与硬件RAID类似的加速能力，并且能够加速更多的在途操作。

- 在RAID重建期间，当静默数据损坏损坏数据时，行为未定义。有报告称，当控制器遇到静默数据损坏时，RAID 5和6阵列在重建过程中丢失。ZFS的校验和允许它通过确定是否存在足够的信息来重建数据来避免这种情况。如果没有，文件将在zpool状态中列为损坏，系统管理员有机会从备份中恢复它。

- 每当操作系统在IO操作上阻塞时，IO响应时间将增加（即性能降低），因为系统CPU在RAID控制器中使用的嵌入式CPU上阻塞。这降低了相对于ZFS本可以实现IOPS。

- 控制器的固件是一个额外的复杂性层，无法由任意第三方检查。ZFS源代码是开源的，任何人都可以检查。

- 如果同一控制器形成多个RAID阵列并且其中一个失败，则暴露给操作系统的阵列提供的标识符可能会变得不一致。直接将驱动器交给操作系统可以通过映射到唯一端口或唯一驱动器标识符的命名来避免这种情况。

   - 例如，如果你有阵列A、B、C和D；阵列B死亡，硬件RAID控制器与操作系统之间的交互可能会将阵列C和D重命名为看起来像阵列B和C。这可能会使从缓存文件逐字导入的池出错。
   - 并非所有RAID控制器都这样表现。在Linux和FreeBSD上，当系统管理员使用单驱动器RAID 0阵列时，已经观察到此问题。此外，不同供应商的控制器也观察到了此问题。

有人可能倾向于尝试使用单驱动器RAID 0阵列来尝试将RAID控制器用作HBA，但由于许多列出的硬件RAID类型的原因，这不推荐。最好使用HBA而不是RAID控制器，以获得更好的性能和可靠性。

.. _hard_drives:

硬盘
===========

.. _sector_size:

扇区大小
-----------

历史上，所有硬盘都有512字节的扇区，除了一些可以修改以支持稍大扇区的SCSI驱动器。2009年，行业从512字节扇区迁移到4096字节的“高级格式”扇区。由于Windows XP与4096字节扇区或大于2TB的驱动器不兼容，一些最早的高级格式驱动器实施了黑客手段以保持Windows XP兼容性。

- 市场上最早的高级格式驱动器错误地将其扇区大小报告为512字节以保持Windows XP兼容性。截至2013年，据信此类硬盘已不再生产。在此期间或之后制造的高级格式硬盘应报告其真实的物理扇区大小。
- 存储2TB及更小的驱动器可能有一个跳线，可以设置为将所有扇区偏移1。这是为了为Windows XP提供适当的对齐，Windows XP从第63个扇区开始其第一个分区。在使用此类驱动器与ZFS时，应关闭此跳线设置。

截至2014年，市场上仍有512字节和4096字节的驱动器，但它们已知会正确识别自己，除非在USB到SATA控制器后面。在vdev中使用512字节扇区驱动器创建的4096字节扇区驱动器替换将不利地影响性能。用512字节扇区驱动器替换4096字节扇区驱动器不会对性能产生负面影响。

.. _error_recovery_control:

错误恢复控制
----------------------

据说ZFS能够使用便宜的驱动器。这在ZFS引入时是正确的，当时硬盘支持错误恢复控制。自ZFS引入以来，某些制造商（最著名的是西部数据）已从低端驱动器中删除了错误恢复控制。一致的性能需要支持错误恢复控制的硬盘。

.. _background_2:

背景
~~~~~~~~~~

硬盘使用磁表面上的小极化区域存储数据。从该表面读取和/或写入数据会带来一些可靠性问题。一个是表面上的缺陷可能会损坏位。另一个是振动可能导致驱动器磁头错过目标。因此，硬盘扇区由三个区域组成：

- 扇区号
- 实际数据
- ECC

扇区号和ECC使硬盘能够检测并响应此类事件。当读取过程中发生任一事件时，硬盘将多次重试读取，直到成功或得出结论认为无法读取数据。后一种情况可能需要大量时间，因此驱动器的IO将停滞。

企业硬盘和一些消费级硬盘实现了称为时间限制错误恢复（TLER）的功能（西部数据）、错误恢复控制（ERC）（希捷）以及命令完成时间限制（日立和三星），该系统管理员可以限制驱动器愿意在此类事件上花费的时间。

缺乏此类功能的驱动器可以预期具有任意高的限制。几分钟并非不可能。具有此功能的驱动器通常默认为7秒。ZFS目前不会调整驱动器上的此设置。但是，建议编写脚本将错误恢复时间设置为较低的值，例如0.1秒，直到ZFS修改为控制它。这必须在每次启动时完成。

.. _rpm_speeds:

转速
----------

高转速驱动器具有较低的寻道时间，这在历史上被认为是可取的。它们增加了成本并牺牲了存储密度，以实现通常不超过其较低转速对应物的6倍改进。

提供一些数字，主要制造商的15k RPM驱动器额定为3.4毫秒的平均读取和3.9毫秒的平均写入。据推测，此数字假设目标扇区最多为驱动器磁道数的一半和磁盘的一半。离得更远是最坏情况下慢2倍。7200 RPM驱动器的制造商数字不可用，但根据经验测量，它们的平均为13到16毫秒。5400 RPM驱动器可以预期更慢。

ARC和ZIL能够减轻较低寻道时间的大部分好处。通过增加RAM用于ARC、L2ARC设备和SLOG设备，可以获得更大的IOPS性能提升。通过完全用固态存储替换硬盘，可以获得更高的性能提升。在考虑IOPS时，这些通常比高转速驱动器更具成本效益。

.. _command_queuing:

命令队列
---------------

具有命令队列的驱动器能够重新排序IO操作以增加IOPS。这在SATA上称为本机命令队列（NCQ），在PATA/SCSI/SAS上称为标记命令队列（TCQ）。ZFS将对象存储在元数据片中，并且它可以同时使用多个元数据片。因此，ZFS不仅设计为利用命令队列，而且良好的ZFS性能需要命令队列。几乎所有在过去10年内制造的驱动器都可以预期支持命令队列。例外情况是：

- 消费级PATA/IDE驱动器
- 2003年至2004年使用IDE到SATA转换芯片的第一代SATA驱动器。
- 在系统BIOS中配置为IDE仿真的SATA驱动器。

每个OpenZFS系统都有不同的方法来检查是否支持命令队列。在Linux上，使用``hdparm -I /path/to/device \| grep Queue``。在FreeBSD上，使用``camcontrol identify $DEVICE``。

.. _nand_flash_ssds:

NAND 闪存 SSD
===============

截至2014年，固态存储主要由NAND闪存主导，大多数关于固态存储的文章都专门关注它。截至2014年，与ZFS一起使用的最流行的闪存存储形式是带有SATA接口的驱动器。具有SAS接口的企业型号开始出现。

截至2017年，使用NAND闪存并具有PCI-E接口的固态存储已在市场上广泛可用。它们主要是企业驱动器，使用NVMe接口，其开销低于SATA中使用的ATA或SAS中使用的SCSI。还有一种称为M.2的接口，主要用于消费级SSD，但不一定限于它们。它可以为多个总线（如SATA、PCI-E和USB）提供电气连接。M.2 SSD似乎使用SATA或NVME。

.. _nvme_low_level_formatting:

NVMe 低级格式化
-------------------------

许多NVMe SSD支持512字节扇区和4096字节扇区。它们通常出厂时使用512字节扇区，其性能不如4096字节扇区。一些还支持T10/DIF CRC的元数据以提高可靠性，尽管这在ZFS中是不必要的。

NVMe驱动器应 `格式化 <https://filers.blogspot.com/2018/12/how-to-format-nvme-drive.html>`__ 为使用4096字节扇区且不带元数据，然后交给ZFS以获得最佳性能，除非它们指示512字节扇区与4096字节扇区性能相同，尽管这不太可能。在``smartctl -a /dev/$device_namespace``（例如``smartctl -a /dev/nvme1n1``）的“支持的LBA大小”中的“Rel_Perf”较低的数字表示更高级别的低级格式，0为最佳。当前格式将在格式Fmt下用加号标记。

您可以使用``nvme format /dev/nvme1n1 -l $ID``格式化驱动器。$ID对应于“支持的LBA大小”SMART信息中的Id字段值。

.. _power_failure_protection:

电源故障保护
------------------------

.. _background_3:

背景
~~~~~~~~~~

闪存上的数据结构非常复杂，传统上非常容易损坏。在过去，此类损坏将导致*所有*驱动器数据丢失，诸如PSU故障之类的事件可能导致多个驱动器同时故障。由于驱动器固件不可供审查，传统的结论是，所有缺乏避免电源故障事件的硬件功能的驱动器都不可信，这在过去多次被发现 [#ssd_analysis]_ [#ssd_analysis2]_ [#ssd_analysis3]_。
关于电源故障导致NAND闪存SSD损坏的讨论在2015年后从文献中消失。SSD制造商现在声称固件电源丢失保护足够强大，可以提供与硬件电源丢失保护相当的保护。 `Kingston是一个例子 <https://www.kingston.com/us/solutions/servers-data-centers/ssd-power-loss-protection>`__。
固件电源丢失保护用于保证刷新数据和驱动器自身元数据的保护，这是ZFS等文件系统所需的全部。

然而，那些需要或希望确保固件错误不太可能在电源丢失事件后导致驱动器损坏的人应继续使用提供硬件电源丢失保护的驱动器。硬件电源故障保护的基本概念已由 `Intel记录 <https://www.intel.com/content/dam/www/public/us/en/documents/technology-briefs/ssd-power-loss-imminent-technology-brief.pdf>`__ 供那些希望了解细节的人阅读。截至2020年，硬件电源丢失保护的使用现在仅是企业SSD的功能，这些SSD试图保护未刷新数据以及驱动器元数据和刷新数据。这种超出保护刷新数据和驱动器元数据的额外保护对ZFS没有额外的好处，但也没有坏处。

还应注意的是，数据中心和笔记本电脑中的驱动器不太可能经历电源丢失事件，从而降低了硬件电源丢失保护的实用性。在数据中心中，冗余电源、UPS电源和使用IPMI进行强制重启应防止大多数驱动器经历电源丢失事件。

以下列出了提供硬件电源丢失保护的驱动器，供那些需要/想要的人使用。由于ZFS与其他文件系统一样，只需要电源故障保护来保护刷新数据和驱动器元数据，因此仅保护这些内容的旧驱动器也包含在列表中。

.. _nvme_drives_with_power_failure_protection:

具有电源故障保护的NVMe驱动器
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

以下是非详尽的具有电源故障保护的NVMe驱动器列表：

-  Intel 750
-  Intel DC P3500/P3600/P3608/P3700
-  Kingston DC1000B (M.2 2280 外形，适合大多数笔记本电脑)
-  Micron 7300/7400/7450 PRO/MAX
-  Samsung PM963 (M.2 外形)
-  Samsung PM1725/PM1725a
-  Samsung XS1715
-  Toshiba ZD6300
-  Seagate Nytro 5000 M.2 (XP1920LE30002 测试；**购买前请阅读以下说明**)

   -  使用消费级MLC的廉价22110 M.2企业驱动器，针对读取为主的工作负载进行了优化。它不是写入为主的SLOG设备的良好选择。
   -  该驱动器的 `手册 <https://www.seagate.com/www-content/support-content/enterprise-storage/solid-state-drives/nytro-5000/_shared/docs/nytro-5000-mp2-pm-100810195d.pdf>`__ 指定了气流要求。如果驱动器未从机箱风扇获得足够的气流，它将在空闲时过热。其热节流将严重降低性能，使得写入吞吐量性能将限制为规格的1/10，读取延迟将达到数百毫秒。在持续负载下，设备将继续变热，直到发生“可靠性降低”事件，其中至少一个NVMe命名空间上的所有数据丢失。然后，NVMe命名空间在安全擦除之前无法使用。即使在正常情况下有足够的气流，在企业环境中风扇故障后，负载下仍可能发生数据丢失。任何在企业环境中将其部署到生产环境中的人都应注意此故障模式。
   -  那些希望在低气流情况下使用此驱动器的人可以通过在NAND闪存控制器上放置被动散热器（如 `此 <https://smile.amazon.com/gp/product/B07BDKN3XV>`__）来解决此故障模式。它是靠近电容器的贴纸下的芯片。通过将散热器放在贴纸上（因为移除它被认为是不希望的）进行了测试。散热器将防止驱动器过热到数据丢失的程度，但在没有主动气流的情况下，负载下的过热情况不会完全缓解。擦洗将在读取几百GB后导致其过热。然而，热节流将迅速将驱动器从76摄氏度冷却到74摄氏度，恢复性能。

      -  可能可以在企业环境中使用散热器来提供风扇故障后数据丢失的保护。然而，这尚未评估。此外，消费级NAND闪存的工作温度应长期保持在40摄氏度或以上。因此，在企业环境中使用散热器来提供风扇故障后数据丢失的保护应在将驱动器部署到生产环境中之前进行评估，以确保驱动器不会过冷。

.. _sas_drives_with_power_failure_protection:

具有电源故障保护的SAS驱动器
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

以下是非详尽的具有电源故障保护的SAS驱动器列表：

-  Samsung PM1633/PM1633a
-  Samsung SM1625
-  Samsung PM853T
-  Toshiba PX05SHB***/PX04SHB***/PX04SHQ**\*
-  Toshiba PX05SLB***/PX04SLB***/PX04SLQ**\*
-  Toshiba PX05SMB***/PX04SMB***/PX04SMQ**\*
-  Toshiba PX05SRB***/PX04SRB***/PX04SRQ**\*
-  Toshiba PX05SVB***/PX04SVB***/PX04SVQ**\*

.. _sata_drives_with_power_failure_protection:

具有电源故障保护的SATA驱动器
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

以下是非详尽的具有电源故障保护的SATA驱动器列表：

-  Crucial MX100/MX200/MX300
-  Crucial M500/M550/M600
-  Intel 320

   -  早期报告称330和335也具有电源故障保护，`但它们没有 <https://engineering.nordeus.com/power-failure-testing-with-ssds>`__。

-  Intel 710
-  Intel 730
-  Intel DC S3500/S3510/S3610/S3700/S3710
-  Kingston DC500R/DC500M
-  Micron 5210 Ion

   -  列表中的第一个QLC驱动器。高容量，每GB价格低。

-  Samsung PM863/PM863a
-  Samsung SM843T (不要与SM843混淆)
-  Samsung SM863/SM863a
-  Samsung 845DC Evo
-  Samsung 845DC Pro

   -  `高持续写入IOPS <http://www.anandtech.com/show/8319/samsung-ssd-845dc-evopro-preview-exploring-worstcase-iops/5>`__

-  Toshiba HK4E/HK3E2
-  Toshiba HK4R/HK3R2/HK3R

.. _criteriaprocess_for_inclusion_into_these_lists:

列入这些列表的标准/过程
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

这些列表由OpenZFS贡献者（主要是Richard Yao）从可信信息源自愿编制。这些列表旨在保持供应商中立，并不旨在使任何特定制造商受益。对任何制造商的任何感知偏见是由于缺乏意识和缺乏时间研究其他选项。由可靠来源确认存在足够的电源丢失保护是列入此列表的唯一要求。足够的电源丢失保护意味着驱动器必须保护其自身的内部元数据和所有刷新数据。未刷新数据的保护无关紧要，因此不是要求。ZFS只期望存储保护刷新数据。因此，仅保护刷新数据的固态驱动器足以确保ZFS数据安全。

任何认为未列出的驱动器提供足够电源故障保护的人可以联系 :ref:`mailing_lists` 并提供包含请求和电源故障保护提供的证明。证明的例子包括显示电容器存在的驱动器内部图片、Anandtech等知名独立评论网站的声明以及制造商规格表。后者在制造商被发现错误陈述驱动器自身内部元数据结构和/或刷新数据保护之前，基于诚信系统接受。迄今为止，所有制造商都是诚实的。

.. _flash_pages:

闪存页
-----------

NAND芯片上可写入的最小单元是闪存页。市场上第一个NAND闪存SSD具有4096字节的页。进一步复杂化的是，自那时以来，页大小已经翻倍两次。NAND闪存SSD **应** 将这些页报告为扇区，但到目前为止，所有SSD都错误地报告512字节扇区以保持Windows XP兼容性。结果是我们遇到了与早期高级格式硬盘类似的情况。

截至2014年，市场上大多数NAND闪存SSD具有8192字节的页大小。然而，某些制造商的128-Gbit NAND型号具有16384字节的页大小。最大性能要求使用正确的ashift值创建vdev（8192字节为13，16384字节为14）。然而，并非所有OpenZFS平台都支持这一点。Linux端口支持ashift=13，而其他平台仅限于ashift=12（4096字节）。

截至2017年，NAND闪存SSD针对4096字节IO进行了优化。匹配闪存页大小是不必要的，ashift=12通常是正确的选择。关于闪存页大小的公开文档也几乎不存在。

.. _ata_trim_scsi_unmap:

ATA TRIM / SCSI UNMAP
---------------------

应注意，这与zvol上的丢弃或文件系统上的打孔是分开的情况。无论是否向实际块设备发送ATA TRIM / SCSI UNMAP，这些操作都有效。

.. _ata_trim_performance_issues:

ATA TRIM 性能问题
~~~~~~~~~~~~~~~~~~~~~~~~~~~

SATA 3.0及更早版本中的ATA TRIM命令是非队列命令。在符合SATA 3.0或更早版本的SATA驱动器上发出TRIM命令将导致驱动器耗尽其IO队列并停止服务请求，直到完成，这会损害性能。SATA 3.1移除了此限制，但市场上很少有SATA驱动器符合SATA 3.1，并且很难将它们与SATA 3.0驱动器区分开。同时，SCSI UNMAP没有此类问题。

.. _optane_3d_xpoint_ssds:

Optane / 3D XPoint SSD
=======================

这些是具有比NAND闪存SSD更好的延迟和写入耐久性的SSD。它们是字节可寻址的，因此ashift=9适用于它们。与NAND闪存SSD不同，它们不需要任何特殊的电源故障保护电路来保证可靠性。也不需要在其上运行TRIM。然而，它们的每GB成本比NAND闪存更高（截至2020年）。企业型号是优秀的SLOG设备。以下是一些已知表现良好的型号：

-  `Intel DC
   P4800X <https://www.servethehome.com/intel-optane-hands-on-real-world-benchmark-and-test-results/>`__   

-  `Intel DC
   P4801X <https://www.servethehome.com/intel-optane-dc-p4801x-review-100gb-m-2-nvme-ssd-log-option/>`__
   
-  `Intel DC
   P1600X <https://www.servethehome.com/intel-optane-p1600x-small-capacity-ssd-for-boot-launched/>`__

请注意，SLOG设备在任何给定时间很少使用超过4GB，因此较小容量的设备通常是成本最佳选择，较大容量不会带来任何好处。根据性能需求和成本考虑，较大容量可能是其他vdev类型的不错选择。

电源
=====

强烈建议确保计算机正确接地。在用户家中，有机器在插入没有接地线（即完全没有接地线）的电源插座时出现随机故障的情况。这可能导致任何计算机系统出现随机故障，无论它是否使用ZFS。

电源也应相对稳定。最好通过使用UPS单元或线路调节器来避免电压大幅下降的电压波动。受不稳定电源影响的系统如果不直接关闭，可能会表现出未定义的行为。具有较长保持时间的PSU应能够提供部分保护，但保持时间通常未记录，并且不能替代UPS或线路调节器。

.. _pwr_ok_signal:

PWR_OK 信号
-------------

PSU应取消断言PWR_OK信号以指示提供的电压不再在额定规格内。这应强制立即关闭。然而，在开发人员工作站上，在一系列约1秒的电压波动期间，系统时钟显著偏离预期值。当时该机器未使用UPS。然而，PWR_OK机制应保护免受此影响。观察到PWR_OK信号未能强制关闭并导致不良后果（在此情况下是系统时钟）表明PWR_OK机制不是严格的保证。

.. _psu_hold_up_times:

PSU 保持时间
-----------------

PSU保持时间是指在输入电源丢失后，PSU可以在标准电压容差内继续以最大输出输出电源的时间量。这对于支持UPS单元很重要，因为 `转换时间 <https://www.sunpower-uk.com/glossary/what-is-transfer-time/>`__ 由标准UPS从其电池供电所需的时间可能使机器在“5-12毫秒”内没有电源。 `Intel的ATX电源设计指南 <https://paginas.fe.up.pt/~asousa/pc-info/atxps09_atx_pc_pow_supply.pdf>`__ 指定在最大连续输出下保持时间为17毫秒。保持时间是PSU输出功率的倒数函数，较低的输出功率会增加保持时间。

PSU中的电容器老化将降低保持时间，使其低于新时的水平，这可能导致设备老化时的可靠性问题。因此，使用保持时间低于规格的劣质PSU的机器需要更高端的UPS单元来保护，以确保转换时间不超过保持时间。在转换到电池电源期间，保持时间低于转换时间可能导致未定义的行为，如果PWR_OK信号未取消断言以强制机器关闭。

如果有疑问，请使用双转换UPS单元。双转换UPS单元始终从电池运行，因此转换时间为0。除非它们是标准UPS单元和双转换UPS单元之间的混合高效型号，尽管这些型号的转换时间比标准PSU低得多。您还可以联系您的PSU制造商以获取保持时间规格，但如果多年的可靠性是要求，您应使用具有低转换时间的高端UPS。

请注意，双转换单元最多只有94%的效率，除非它们支持高效模式，这会增加转换到电池电源的时间。

.. _ups_batteries:

UPS 电池
-------------

UPS单元中的铅酸电池通常需要定期更换，以确保在停电期间提供电源。对于家庭系统，这是每3到5年一次，尽管这随温度变化 [#ups_temp]_。对于企业系统，请联系您的供应商。


.. rubric:: 脚注

.. [#ssd_analysis] <http://lkcl.net/reports/ssd_analysis.html>
.. [#ssd_analysis2] <https://www.usenix.org/system/files/conference/fast13/fast13-final80.pdf>
.. [#ssd_analysis3] <https://engineering.nordeus.com/power-failure-testing-with-ssds>
.. [#ups_temp] <https://www.apc.com/us/en/faqs/FA158934/>