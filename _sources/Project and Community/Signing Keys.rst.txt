签名密钥
============

所有标记的 ZFS on Linux
`版本 <https://github.com/zfsonlinux/zfs/releases>`__ 都由该分支的官方维护者签名。这些签名由 GitHub 自动验证，并且可以通过下载维护者的公钥在本地进行检查。

维护者
-----------

发布分支 (spl/zfs-\*-release)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

| **维护者:** `Ned Bass <https://github.com/nedbass>`__
| **下载:**
  `pgp.mit.edu <http://pgp.mit.edu/pks/lookup?op=vindex&search=0xB97467AAC77B9667&fingerprint=on>`__
| **密钥 ID:** C77B9667
| **指纹:** 29D5 610E AE29 41E3 55A2 FE8A B974 67AA C77B 9667

| **维护者:** `Tony Hutter <https://github.com/tonyhutter>`__
| **下载:**
  `pgp.mit.edu <http://pgp.mit.edu/pks/lookup?op=vindex&search=0x6ad860eed4598027&fingerprint=on>`__
| **密钥 ID:** D4598027
| **指纹:** 4F3B A9AB 6D1F 8D68 3DC2 DFB5 6AD8 60EE D459 8027

主分支 (master)
~~~~~~~~~~~~~~~~~~~~~~

| **维护者:** `Brian Behlendorf <https://github.com/behlendorf>`__
| **下载:**
  `pgp.mit.edu <http://pgp.mit.edu/pks/lookup?op=vindex&search=0x0AB9E991C6AF658B&fingerprint=on>`__
| **密钥 ID:** C6AF658B
| **指纹:** C33D F142 657E D1F7 C328 A296 0AB9 E991 C6AF 658B

检查 Git 标签的签名
-----------------------------------

首先将上面列出的公钥导入你的密钥环。

::

   $ gpg --keyserver pgp.mit.edu --recv C6AF658B
   gpg: 从 hkp 服务器 pgp.mit.edu 请求密钥 C6AF658B
   gpg: 密钥 C6AF658B: "Brian Behlendorf <behlendorf1@llnl.gov>" 未更改
   gpg: 总共处理了 1 个密钥
   gpg:              未更改: 1

导入公钥后，可以按如下方式验证 Git 标签的签名。

::

   $ git tag --verify zfs-0.6.5
   object 7a27ad00ae142b38d4aef8cc0af7a72b4c0e44fe
   type commit
   tag zfs-0.6.5
   tagger Brian Behlendorf <behlendorf1@llnl.gov> 1441996302 -0700

   ZFS 版本 0.6.5
   gpg: 签名于 2015 年 9 月 11 日 星期五 11:31:42 AM PDT，使用 DSA 密钥 ID C6AF658B
   gpg: 来自 "Brian Behlendorf <behlendorf1@llnl.gov>" 的良好签名
   gpg:                 即 "Brian Behlendorf (LLNL) <behlendorf1@llnl.gov>"