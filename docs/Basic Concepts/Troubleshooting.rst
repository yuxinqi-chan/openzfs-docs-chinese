故障排除
===============

.. todo::
   本页面为草稿。

本页面包含有关在Linux上排除ZFS故障的提示，以及开发人员在错误分类时可能需要的相关信息。

-  `关于日志文件 <#about-log-files>`__

   -  `通用内核日志 <#generic-kernel-log>`__
   -  `ZFS内核模块调试信息 <#zfs-kernel-module-debug-messages>`__

-  `无法终止的进程 <#unkillable-process>`__
-  `ZFS事件 <#zfs-events>`__

--------------

关于日志文件
---------------

日志文件对于故障排除非常有用。在某些情况下，相关信息会存储在多个日志文件中，并与系统事件相关联。

专业提示：像 *elasticsearch*、*fluentd*、*influxdb* 或 *splunk* 这样的日志基础设施工具可以简化日志分析和事件关联。

通用内核日志
~~~~~~~~~~~~~~~~~~

通常，Linux内核日志消息可以通过 ``dmesg -T``、``/var/log/syslog`` 或内核日志消息发送的位置（例如通过 ``rsyslogd``）获取。

ZFS内核模块调试信息
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

ZFS内核模块使用内部日志缓冲区来记录详细的日志信息。在ZFS构建中，此日志信息可通过伪文件 ``/proc/spl/kstat/zfs/dbgmsg`` 获取，前提是ZFS模块参数 `zfs_dbgmsg_enable = 1 <https://github.com/zfsonlinux/zfs/wiki/ZFS-on-Linux-Module-Parameters#zfs_dbgmsg_enable>`__。

--------------

无法终止的进程
------------------

症状：``zfs`` 或 ``zpool`` 命令似乎挂起，无法返回，并且无法终止。

可能原因：内核线程挂起或崩溃。

相关日志文件：`通用内核日志 <#generic-kernel-log>`__、`ZFS内核模块调试信息 <#zfs-kernel-module-debug-messages>`__。

重要信息：如果内核线程卡住，则日志中可能会有卡住线程的回溯信息。在某些情况下，卡住的线程直到死机计时器到期才会被记录。另请参阅 `调试可调参数 <https://github.com/zfsonlinux/zfs/wiki/ZFS-on-Linux-Module-Parameters#debug>`__。

--------------

ZFS事件
----------

ZFS使用基于事件的消息接口与系统上运行的其他消费者进行重要事件的通信。ZFS事件守护程序（zed）是一个用户态守护程序，用于监听这些事件并处理它们。zed是可扩展的，因此您可以编写shell脚本或其他程序来订阅事件并采取行动。例如，通常安装在 ``/etc/zfs/zed.d/all-syslog.sh`` 的脚本会将格式化的事件消息写入 ``syslog``。有关更多信息，请参阅 ``zed(8)`` 的手册页。

事件历史记录也可以通过 ``zpool events`` 命令获取。此历史记录从ZFS内核模块加载开始，并包括来自任何池的事件。这些事件存储在RAM中，数量受内核可调参数 `zfs_event_len_max <https://github.com/zfsonlinux/zfs/wiki/ZFS-on-Linux-Module-Parameters#zfs_zevent_len_max>`__ 的限制。``zed`` 具有内部节流机制，以防止处理ZFS事件时过度消耗系统资源。

使用 ``zpool events -v`` 可以观察到更详细的事件信息。详细事件的内容可能会根据事件和事件发生时可用信息的变化而变化。

每个事件都有一个用于过滤事件类型的类标识符。常见的事件是与池管理相关的事件，类为 ``sysevent.fs.zfs.*``，包括导入、导出、配置更新和 ``zpool history`` 更新。

与错误相关的事件报告为类 ``ereport.*``。这些信息对于故障排除非常宝贵。某些故障可能会导致多个ereport，因为软件的各个层都在处理该故障。例如，在没有奇偶校验保护的简单池中，故障磁盘可能会导致读取磁盘时出现 ``ereport.io``，进而在池级别产生 ``erport.fs.zfs.checksum``。这些事件也反映在 ``zpool status`` 中观察到的错误计数器中。如果您在 ``zpool status`` 中看到校验和或读/写错误，那么在 ``zpool events`` 输出中应该有一个或多个相应的ereport。