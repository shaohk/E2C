# MySQL备份最佳实践
## 原文地址：
[best-practices-for-mysql-backups](https://www.percona.com/blog/2020/05/27/best-practices-for-mysql-backups/)
## 原文翻译
在Percona的培训部门，我们认为我们指导学习者进行所有MySQL相关事情的最佳实践，除性能调优、查询优化和同步配置，另一个重要的主题就是备份了，让我们围绕MySQL备份，深入的讨论一些基础和最佳的实践。
### MySQL逻辑备份
对于MySQL，可能会进行两个不同表之间的备份，第一个表是逻辑表是很普遍的，实质上，你创建所有需要INSERT语句来重新填充表数据。在这方便，有两个最受欢迎的工具，分别是mysqldump、mydumper
#### mysqldump
这个工具从一开始以来就支持多种使用方式，在这篇文章中会讨论几种。

接下来是一个简单的对members database进行逻辑表备份的例子，同时对结果进行了压缩。
```
mysqldump --single-transaction members | gzip - >members.sql.gz
```
如果你想对所有DB的所有表进行完整备份，你可以这样操作
```
mysqldump --single-transaction --all-databases | gzip - >full_backup.sql.gz
```
注意--single-transaction标志的用法，这标志”为备份的所有table创建一个一致性快照在一个事务里“，如果你不使用这个标志，一个完整的备份可能包含不一致的情况在相关表之间。
##### 数据不一致
mysqldump的一个最大的缺点就是缺少并行机制。这个工具从第一个DB开始，按照字母顺序，一次一个的备份里面的每一张表。思考再一个事务里，同时向'alpha'和'zeta'表插入数据，现在开启一个备份，当备份进行到表'delta'时，另一个事务开始删除‘zeta’表的数据，你的备份在alpha和zeta表之间就存在数据不一致。

这就是为什么–single-transaction非常重要。

#### mydumper
mydumper是开源的，第三方的工具，起初是Domas Mituzas开发，现在是Max Bube维护。这个工具的功能和mysqldump类似，但是它提供了很多改进，像并行备份，一致性读，内置压缩。mydumper的另一个好处是每一个每一个单独的表会备份到一个单独的文件中。这在恢复一个表的时候比mysqldump具有极大的优势（mysqldump备份所有的东西到同一个文件中）。
```
mydumper --compress
```
上面的命令将会链接你本地的MySQL服务，开始对所有DB所有表开始进行一致性备份，像上面说的，每一个表都会在备份的目录下面创建一个单独的文件，被命名为‘export-YYYYMMDD-HHMMSS’的格式。每一个备份文件会被使用gzip进行单独的压缩。
一个配套的工具，myloader，也包含在内。这个工具允许并行恢复所有的table，这个mysqldump是不支持的。

### MySQL物理备份
一个物理备份拷贝表数据文件从本地到另一个地方，理想的，使用一个在线的、一致性的方式。[Percona XtraBackup](https://www.percona.com/software/mysql-database/percona-xtrabackup)符合这个描述。也支持Oracle发行的收费版企业级MySQL备份，设备快照。值得注意的是Percona XtraBackup是免费的，开源的，可以做企业级的任何操作，或者更多。
#### Percona XtraBackup
```
xtrabackup --backup --parallel 4 --compress --target-dir /var/backup/
```
上面的命令会链接你的MySQL服务器，执行一个压缩的，并行的备份，存储结果数据文件到/var/backup/目录。查看这个目录，你会发现所有熟悉的文件像ibdata1,mysql.ibd,剩下的都是.ibd文件，表示你的数据。

你也可以"stream"你的物理备份到一个包：
```
xtrabackup --backup --parallel 4 --stream=xbstream > mybackup.xbstream
  OR
xtrabackup --backup --stream=tar > mybackup.tar
```
（注意：'tar'包不支持并发）

并且，感谢Procona的xbcloud工具，你可以直接将备份流导入到S3的兼容桶里：
```
xtrabackup --backup --stream=xbstream | xbcloud put --storage=s3 <S3 parameters> mybackup_s3.blob
```

#### 卷快照(Volume Snapshots)
在某些情况下，大量数据需要备份，太大以至于不能导出一个物理备份（更别说逻辑备份了）。考虑一个MySQL服务，数据量超过1TB，即使磁盘速度很快，执行一个物理备份也会花费很多小时。

在这些情况下，使用底层文件系统或磁盘驱动的快照特性执行一个物理备份是非常快的。

LVM和ZFS都支持本机快照，在做快照之前，确保MySQL已经停止写操作，并且内存中的数据也都刷新到了磁盘。这是Yves Trudeau提供的一个使用ZFS的进程：
```
mysql -e 'FLUSH TABLES WITH READ LOCK; SHOW MASTER STATUS; ! zfs snapshot -r mysqldata/mysql@my_first_snapshot'
```
上面的命令会锁住MySQL所有的表，阻塞其他事物的写操作，打印当前binlog位置信息，然后另起一个shell执行ZFS快照命令。一旦快照结束，连接的MySQL链接会断开，并且释放所有表锁。

大多数的云厂商针对他们的块存储支持本地快照特性(ie: EC2, GPD)。命令类似于上面的ZFS示例。

### 验证MySQL备份
因此，通过上面这些最佳实践，你会获得一个很好的备份过程。你怎么知道备份是成功的呢？你看文件大小了吗？你只检测文件是否存在？也许你只检测了你使用的备份工具的退出码？

Shlomi Noach在之前的一个Percona Live会议告诉我“你都没有做过一个备份直到你对那些备份做验证。”非常好的忠告。换一种方式，你做的每一个备份可以考虑作为DB的备份，直到你验证它，并且正常工作。

这里的最佳实践就是直接使用你创建的备份来恢复一个MySQL服务；然而，你创建了它。处理还原任务的机器不需要像源机器一样强大；一个简单的VM就可以执行这个任务并且可以自动完成。

你可以使用MySQL自己的客户端还原一个mysqldump
```
zcat my_full_backup.sql.gz | mysql
```

使用mydumper/myloader:
```
myloader --directory dump_dir --overwrite-tables --verbose=3
```

Percona XtraBackup:
```
# Prepare the backup
xtrabackup --prepare --parallel 4 --use-memory 4G --target-dir /var/backup

# Copy backup to original location (ie: /var/lib/mysql), assuming backup taken on same host
xtrabackup --copy-back --target-dir /var/backup

# Fix file permissions if necessary
chown -R mysql:mysql /var/lib/mysql

# Start MySQL
systemctl start mysql
```

是的，Percona XtraBackup需要更多的步骤，但是物理备份一直是备份最快的方法和恢复最快的方法。

### 总结
有多种不同的方法来进行MySQL备份。希望这篇文章能给你一些最佳实践的洞察，当你选择你的备份方法的时候。

还有一些事需要考虑：
* 上面的大多数工具都支持加密备份，确认对你的加密key进行备份。
* Point-in-time-recovery(PITR)没有被本文覆盖。然而，要达到PITR，你需要备份你的binlog日志。rsync对这个有帮助，或者mysqlbinlog工具可以是一个在线的MySQL binlog备份工具。
* 确保对你的my.cnf进行备份
* 这篇文章没有覆盖增量备份，它本身可以是一篇单独的文章。这也是Percona XtraBackup本机支持的内容。

