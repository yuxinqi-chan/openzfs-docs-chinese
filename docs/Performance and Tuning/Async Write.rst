异步写入
============

异步写入 I/O 类的并发操作数量遵循由几个可调整点定义的分段线性函数。

::

          |              o---------| <-- zfs_vdev_async_write_max_active
     ^    |             /^         |
     |    |            / |         |
   active |           /  |         |
    I/O   |          /   |         |
   count  |         /    |         |
          |        /     |         |
          |-------o      |         | <-- zfs_vdev_async_write_min_active
         0|_______^______|_________|
          0%      |      |       100% of zfs_dirty_data_max
                  |      |
                  |      `-- zfs_vdev_async_write_active_max_dirty_percent
                  `--------- zfs_vdev_async_write_active_min_dirty_percent

在脏数据量超过池中允许的脏数据的最小百分比之前，I/O 调度程序将限制并发操作的数量为最小值。当超过该阈值时，并发操作的数量将线性增加，直到达到池中允许的脏数据的最大百分比。

理想情况下，繁忙池中的脏数据量应保持在 zfs_vdev_async_write_active_min_dirty_percent 和 zfs_vdev_async_write_active_max_dirty_percent 之间的函数斜率部分。如果超过最大百分比，则表明传入数据的速率大于后端存储可以处理的速率。在这种情况下，必须进一步限制传入的写入操作，如下一节所述。