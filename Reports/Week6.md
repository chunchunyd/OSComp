# Week6进展记录

## 1. 初赛代码框架/环境配置

## 2. Syscall list

### 2.1 文件系统相关

|                |                                      |      |      |
| -------------- | ------------------------------------ | ---- | ---- |
| SYS_getcwd     | 获取当前工作目录                     | 17   | X    |
| SYS_pipe2      | 创建管道                             | 59   | O    |
| SYS_dup        | 复制文件描述符                       | 23   | O    |
| SYS_dup3       | 复制文件描述符，并指定新的文件描述符 | 24   | X    |
| SYS_chdir      | 需要切换到的目录                     | 49   | X    |
| SYS_openat     | 打开或创建一个文件                   | 56   | O    |
| SYS_close      | 关闭一个文件描述符                   | 57   | O    |
| SYS_getdents64 | 获取目录的条目                       | 61   | X    |
| SYS_read       | 从一个文件描述符中读取               | 63   | O    |
| SYS_write      | 从一个文件描述符中写入               | 64   | O    |
| SYS_linkat     | 创建文件的链接                       | 37   | O    |
| SYS_unlinkat   | 移除指定文件的链接(可用于删除文件)   | 35   | O    |
| SYS_mkdirat    | 创建目录                             | 34   | X    |
| SYS_umount2    | 卸载文件系统                         | 39   | X    |
| SYS_mount      | 挂载文件系统                         | 40   | X    |
| SYS_fstat      | 获取文件状态                         | 80   | O    |

### 2.2 进程管理相关

|             |                        |      |      |
| ----------- | ---------------------- | ---- | ---- |
| SYS_clone   | 创建一个子进程         | 220  | O    |
| SYS_execve  | 执行一个指定的程序     | 221  | O    |
| SYS_wait4   | 等待进程改变状态       | 260  | O    |
| SYS_exit    | 触发进程终止，无返回值 | 93   | O    |
| SYS_getppid | 获取父进程ID           | 173  | X    |
| SYS_getpid  | 获取进程ID             | 172  | O    |

### 2.3 内存管理相关

|            |                              |      |      |
| ---------- | ---------------------------- | ---- | ---- |
| SYS_brk    | 修改数据段的大小             | 214  | O    |
| SYS_munmap | 将文件或设备取消映射到内存中 | 215  | O    |
| SYS_mmap   | 将文件或设备映射到内存中     | 222  | O    |

### 2.4 其他

|                  |                                           |      |      |
| ---------------- | ----------------------------------------- | ---- | ---- |
| SYS_times        | 获取进程时间                              | 153  | O    |
| SYS_uname        | 打印系统信息                              | 160  | X    |
| SYS_sched_yield  | 让出调度器                                | 124  | O    |
| SYS_gettimeofday | 获取时间                                  | 169  | O    |
| SYS_nanosleep    | 执行线程睡眠，sleep()库函数基于此系统调用 | 101  | X    |



## 3. fat32文件系统

FAT文件系统由4个region组成：

1. Reserved sectors ：

   1. Boot Sector: 

      1. **at logical sector 0.**

      2. usually contains the operating system's [boot loader](https://en.wikipedia.org/wiki/Boot_loader) code.

      3. includes an area called the BIOS Parameter Block which contains some basic file system information

      4. stores the total count of reserved sectors

         > **Total sectors in volume (including VBR):**
         >
         > ```
         > total_sectors = (fat_boot->total_sectors_16 == 0)? fat_boot->total_sectors_32 : fat_boot->total_sectors_16;
         > ```
         >
         > **FAT size in sectors:**
         >
         > ```
         > fat_size = (fat_boot->table_size_16 == 0)? fat_boot_ext_32->table_size_16 : fat_boot->table_size_16;
         > ```
         >
         > **The size of the root directory (unless you have FAT32, in which case the size will be 0):**
         >
         > ```
         > root_dir_sectors = ((fat_boot->root_entry_count * 32) + (fat_boot->bytes_per_sector - 1)) / fat_boot->bytes_per_sector;
         > ```
         >
         > This calculation will round up. 32 is the size of a FAT directory in bytes.
         >
         > 
         > **The first data sector (that is, the first sector in which directories and files may be stored):**
         >
         > ```
         > first_data_sector = fat_boot->reserved_sector_count + (fat_boot->table_count * fat_size) + root_dir_sectors;
         > ```
         >
         > 
         > **The first sector in the File Allocation Table:**
         >
         > ```
         > first_fat_sector = fat_boot->reserved_sector_count;
         > ```
         >
         > 
         > **The total number of data sectors:**
         >
         > ```
         > data_sectors = fat_boot->total_sectors - (fat_boot->reserved_sector_count + (fat_boot->table_count * fat_size) + root_dir_sectors);
         > ```
         >
         > 
         > **The total number of clusters:**
         >
         > ```
         > total_clusters = data_sectors / fat_boot->sectors_per_cluster;
         > ```

   2. FS Information Sector (FAT32 only): 

      1. **at logical sector 1**

   3. More

2. FAT Region ：
   1. File Allocation Table #1
      1. FAT 32 uses 28 bits to address the clusters on the disk. The highest 4 bits are reserved. This means that they should be ignored when read and unchanged when written. 
   2. File Allocation Table #2（Optional）
      1. Set for redundancy checking but rarely used.

3. Root Directory Region ：

   1. a Directory Table that stores information about the files and directories located in the root directory

   > 在FAT32中被并入Data Region.

4. Data Region：
   1. This is where the actual file and directory data is stored and takes up most of the partition. 
   2. The size of files and subdirectories can be increased arbitrarily (as long as there are free clusters) by simply adding more links to the file's chain in the FAT. Files are allocated in units of clusters, so if a 1 KB file resides in a 32 KB cluster, 31 KB are wasted.
   3. FAT32 typically commences the Root Directory Table in cluster number 2: the first cluster of the Data Region.

> ## FAT32 的文件存储
>
> 平常操作文件的时候，例如你打开一个 doc 文件，增加一些内容然后保存，或者删除某个文件到回收站，它们的内部操作是如何实现的呢？不同的文件系统有不同的实现方式。但所有的操作都离不开存储作为基础，问题来了：如何设计一个文件系统，让它既能高效读写文件，又能快速对文件定位？
>
> 我们来看看最原始的想法：直接连续添加，也就是把文件一个挨着一个地加到储存空间（硬盘）中去。但是，这样实现，既不利于查找，也不利于删除、添加与修改。想一想，如果把一些文件删除，就会产生缺口，再次添加文件的时候，单独的缺口可能不足以容纳新的文件，从而产生浪费。而且只要查找某个文件，就需要遍历所有的文件结构，这个是要花相当长的时间。
>
> 我们来看一看 FAT32 的实现方式：它将储存空间分成了一个个小块( cluster ),存储文件的时候，会把文件切分成对应长度的小块，然后填充到硬盘中去：
>
> 这样一来，我们就不用担心文件太大以至于不能放进缺口中，因为我们可以把一部分小块放在一个缺口，把另一部分小块放在另外的地方，这样很高效地利用了磁盘的空间。
>
> 第二个概念，FAT32 采用了链表的数据结构。也就是说，磁盘中的每一个 cluster 都是链表中的一个节点，它们记录着下一个 cluster 的位置（next pointer）。什么叫下一个 cluster？如果一个文件被放在了储存空间中，如果他所占用了超过一个cluster，那么我们就需要把这些cluster连接起来。FAT32中，只记录了每一个文件开始的cluster，所以我们需要用链表来完成访问整个文件的操作。
>
> 用来存储这个链表信息的表格叫做 FAT ( FILE ALLOCATE TABLE ) ,真正存放数据的地方与FAT是相互分离的。FAT的作用就是方便查找。
>
> 接下来我们看看，删除的操作。这会引出另一个专有结构：FILE ENTRY
>
> 首先你来回想一下，删除文件和写入一个新的文件（比如复制粘贴），哪个更快些？删除。几乎是互逆过程，为何时间不同？实际上，在你删除文件的时候，文件系统并没有真正地把数据从磁盘上抹去（这也是为什么我们有希望恢复删除文件的原因），而只是修改了它的FILE ENTRY信息。
>
> 何谓 FILE ENTRY？ 简单些讲，就是记录文件属性的一个小结构。它们被统一存储在 ROOT DIRECTORY 中。我们先看一下 FAT32 的磁盘整体面貌
>
> [![img](https://github.com/KuangjuX/FAT32/raw/8d5e2e756c4d88b25034d17260e5a1e49b7cb09d/static/fat32_structure_1.png)](https://github.com/KuangjuX/FAT32/blob/8d5e2e756c4d88b25034d17260e5a1e49b7cb09d/static/fat32_structure_1.png)
>
> 我们先忽略最前面的几个 sector，从 FAT 看起。一个 FAT 系统有若干个 FAT 结构（因为磁盘大小不同，所需要的链表节点数也不同），紧挨FAT区域的是ROOT DIRECTORY,它是整个磁盘的目录结构，而这之中存储的就是我们说的FILE ENTRY,也就是每个文件的属性。ROOT DIRECTORY后，才是真正地DATA FIELD，用来存储真正地文件内容。
>
> 在我们查看某个文件信息而非打开它时，我们并不需要直接访问文件的数据。文件系统会在ROOT DIRECTORY找到相应的FILE ENTRY，然后把相关信息显示出来。这包括：文件名，创建、修改时间，文件大小，文件的第一个cluster的位置，只读/隐藏等等。请注意，文件夹在文件系统中也表示成一个文件，也有相应的FILE ENTRY，只是他们储存的是一批文件而已 ( FILE ENTRY 中会有相应的标志显示是否是文件夹)。
>
> 回到我们删除文件的话题，当一个文件被删除的时候，系统会找到相应的 FILE ENTRY，把文件名第一个字符改为 0xE5 ——完毕。就是这么简单，只是把文件属性修改，一点内部数据都没有改动。这时候如果我们再添加一个文件进去，由于系统会通过查找 ROOT DIRECTORY 来确定可用的空间，因此如果发现一些 FILE ENTRY 文件名是未分配或者已经删除的标志，那么对应的 cluster 就会被占用。但是在被覆盖之前，这些删除的文件依然存在在你的硬盘里（只是你丢失了如何获取他们信息的渠道）。这就是为什么删除要更快些。

## 4. 下周安排

完成初赛内容