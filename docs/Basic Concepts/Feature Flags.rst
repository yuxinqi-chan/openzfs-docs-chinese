功能标志
=============

ZFS 的磁盘格式最初是通过单一数字进行版本控制的，每当格式发生变化时，该数字就会增加。当 ZFS 的开发由单一组织推动时，这种编号方法是合适的。

对于 OpenZFS 的分布式开发，版本编号不再适用。任何对编号的更改都需要在所有实现之间达成一致，以确保每次对磁盘格式的更改都能被接受。

OpenZFS 功能标志——传统版本编号的替代方案——允许**为每次磁盘格式的更改分配一个唯一的池属性名称**。这种方法支持：

- 独立的格式更改
- 相互依赖的格式更改

兼容性
-------------

如果一个池使用的所有*功能*都被多个 OpenZFS 实现支持，那么该磁盘格式在这些实现之间是可移植的。

启用时具有排他性的功能应定期移植到所有发行版中。

参考资料
-------------------

`ZFS 功能标志 <http://web.archive.org/web/20160419064650/http://blog.delphix.com/csiden/files/2012/01/ZFS_Feature_Flags.pdf>`_
（Christopher Siden，2012-01，存档于互联网档案馆 Wayback Machine），特别是：“……池版本 1-28 的旧版版本号仍然存在……”。

`zpool-features(7) 手册页 <../man/7/zpool-features.7.html>`_ - OpenZFS

`zpool-features <http://illumos.org/man/5/zpool-features>`__ (5) – illumos

各操作系统的功能标志实现
-----------------------------------

.. raw:: html

   <div class="man_container">

.. raw:: html
   :file: ../_build/zfs_feature_matrix.html

.. raw:: html

   </div>