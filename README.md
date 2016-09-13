# HW
RMAN备份和恢复相关概念
（1）	逻辑备份和物理备份
Oracle数据库的备份按类型可以分为逻辑备份和物理备份。逻辑备份的核心是复制数据，而不管数据库中具体是哪些文件存储数据，只通过相关 命令将数据保存到其他位置，如exp、expdp命令。物理备份是将相关物理文件（如数据文件、控制文件、归档日志文件、启动参数文件等）复制到其他路径。Oracle数据库提供RMAN（Recovery Manager）工具执行相关的物理备份，其实现原理是基于数据块的拷贝，备份恢复速度比逻辑备份快许多。
（2）	归档模式和非归档模式
Oracle数据库有联机重做日志，这个日志是记录对数据库所做的修改，比如插入，删除，更新数据等，对这些操作都会记录在联机重做日志里。一般数据库至少要有2个联机重做日志组。当一个联机重做日志组被写满的时候，就会发生日志切换，这时联机重做日志组2成为当前使用的日志，当联机重做日志组2写满的时候，又会发生日志切换，去写联机重做日志组1，就这样反复进行。
　　如果数据库处于非归档模式,联机日志在切换时就会丢弃. 而在归档模式下，当发生日志切换的时候，被切换的日志会进行归档。比如，当前在使用联机重做日志1，当1写满的时候，发生日志切换，开始写联机重做日志2，这时联机重做日志1的内容会被拷贝到另外一个指定的目录下。这个目录叫做归档目录，拷贝的文件叫归档重做日志。
　　数据库使用归档方式运行时才可以进行灾难性恢复。
非归档模式只能做冷备份,并且恢复时只能做完全备份.最近一次完全备份到系统出错期间的数据不能恢复.
    归档模式可以做热备份,并且可以做增量备份,可以做部分恢复.
（3）	RMAN备份原理
原理：   RMAN基于备份算法规则来编译要备份的数据文件列表。基于通道数和同时备份的数据文件数，RMAN在ORACLE共享内存段中创建一些内存缓冲区,一般是在PGA中,不过有时候内存缓冲区会被推入SGA。通道服务进程随后就开始读取数据文件，并在RMAN缓冲中填充这些数据块。一个缓冲区被填满时，输入缓冲区的数据就会推出到输出缓冲区。数据文件中的数据块都会都会发生这种memery-to-monery write的过程，如果数据块符合备份的标准，并且memery-to-monery write操作没有检查到数据corruption, 则该数据块会被保存到输出数据缓冲区中，直到输出缓冲区被填满。一旦输出缓冲区被填满，输出缓冲区的内容就会被推到备份位置（磁盘或者磁带）
RMAN备份数据库过程：
RMAN发出备份全库命令后，RMAN生成到目标数据库的bequeath连接，也就是说会检查ORACLA_SID变量中的实例名，并在该实例上产生一个服务器进程，然后作为sysdba登陆，然后会产生一个作为备份的通道，（在PGA或者是在SGA分配存储）。随后RMAN调用SYS.DBMS_RCVMAN请求数据库结构信息，包括控制文件的信息（当前序列号，创建时间……）， 由于指定了备份全库，所以RMAN会请求数据库中数据文件信息，并判断是否存在offline数据文件（包括所在的位置和工作方式）。
RMAN开始备份，为了保持数据一致性RMAN必须构建控制文件快照，接下来RMAN调用DBMS_BACKUP_RESTORE数据包，该调用可以创建备份片。RMAN拥有文件列表，所以它为数据文件读取操作分配内存缓冲区，分配缓冲区后RMAN初始化备份片。一旦初始化了备份片，
RMAN会判断是否使用了服务器参数文件，如果使用了则会做为备份的一部分，还要备份控制文件，之后才开始备份数据文件，并将其推至内存。为了实现这一功能，通道进程在磁盘上执行预读取操作，并且将多个数据文件读入内存中，RMAN会判断数据块头信息是否仍然为零，如果数据块没有被使用过，就不会发生到输出缓冲区的写操作，同时会丢弃这个数据块（这就是RMAN为什么会只备份使用过的数据的原因，也是它的优点）。RMAN还会执行检查数据块有没有corruption操作。当检查通过了就被写入到输出缓冲区。一旦输出缓冲区填满了，就被推至备份文件位置。
在备份数据块的时候，RMAN影子进程会得到备份状态信息。并将它传给V$session_longops视图。查询它能得到信息。
当数据文件的所有数据块都被读入输入缓冲区并确定了状态之后, RMAN就会通过将这个数据文件写入备份片来结束该文件的备份操作。所有数据文件写入备份片之后，RMAN生成最后一个对SYS DBMS BACKUP RESTORE数据包的调用，该调用在控制文件中写入备份信息（包括备份片名，启动备份操作时的检查点的SCN和完成备份的时间）至此完成备份。
（4）	用户管理备份与恢复备份
alter tablespace users begin backup 的时候是锁定了users表空间对应的数据文件头的change scn.首先考虑一下数据库怎么用日志文件做恢复： 查找不一致的数据文件（根据文件头中旧的scn）如果锁定了文件头，这个文件头中的scn就不会改变（当然了， 数据块还是会变化的，还可以做读写）。 然后就会应用这个scn到现在的日志。那我锁定了scn，不管你后边怎么修改，总之做恢复的时候是应用锁定的时候的scn一直到现在的日志（完全恢复的话）.
select tablespace_name,checkpoint_change#,checkpoint_count,name from v$datafile_header；
alter system checkpoint；
热备份归档增加的原因：  热备份的时候redo log会增长较快，归档较平时增多，是由于在begin backup之后，如果正在备份（也就是OS命令拷贝cp）的数据块恰好又在被用户修改(因为是热备份，用户可以操作)，那么可能会产生split block的情况(split block被oracle认为是corrupt block)，也就是说，一个Oracle Block可能包含多个OS Block, OS Level的拷贝可能正拷贝的是一个Oracle Block的一部分(比如Header)，而另一部分(比如尾端)被用户更新，发生变化，这样导致一个Oracle Block内部的不一致(不是consistent version),可能出现一个数据块包含了几个不同版本的操作系统块,被称为Split Block(注意，这里split block是Oracle Block不是OS Block，是因为一个Oracle Block中不同版本的OS Block才导致产生Split Oracle Block的)。Oracle处理Split block的方法是将整个当前Oracle split block(变更后的)写入online redo log中,恢复的时候如果发现datafile中某个Oracle Block中有不同版本(的OS Block)，就从redo把这个变更后的镜像拷贝回来，在这个版本一致的镜像上开始恢复。 不是像原来那样只写入更新部分到redo log,所以热备期间redo log会激增 。
RMAN备份处理split block与热备份不一样，它不存在这样的问题，是因为它在执行备份每个数据块的时候会判断这个数据块是否是split的，如果是，它会重新读这个数据块直到得到一个consistent version。
RMAN结合带库的备份是OS拷贝无法比的，从备份的速度上而言可能有很大差异, RMAN可以多个进程进行备份和恢复，?9i可以进行block  recover，可以进行增量备份，10G中结合新增的日志更有效的进行增量备份和可以回退整个数据库。 RMAN的缺点就是复杂，不使用catalog数据库的话控制文件丢失问题比较严重，解决很麻烦。
注意：  9i, 10g RMAN备份机制不太一样。9i的rman增量备份实质上是将所有的db block做了一次遍历，比较scn号是否发生变化，然后重新备份更新的OS块直到得到一个一致版本的Oracle Block；10g是做了一个scn变化表，增量备份的时候直接从改变读取变化的块 。

（5）	热备份和冷备份
冷备份数据库是将数据库关闭后备份所有关键文件包括数据文件、控制文件、联机Redolog文件，将其拷贝到另外的位置。冷备实际也是一种物理备份，是一个备份数据库物理文件的过程。冷备也被称为完全的数据库备份。
优点：快速，只需拷贝文件即可恢复到某一时间点。维护量少，相对安全。
缺点：必须关闭数据库
热备份是在数据库运行的情况下，采用archive log mode方式备份数据库的方法，其需要大量的档案空间。当执行备份时，只能在数据文件级或表空间进行。
优点：备份时间段，可秒级恢复，不需要关闭数据库。
缺点：难以维护，失败后恢复困难。
