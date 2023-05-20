## sys_writev()

sys_writev()是Linux内核中的一个系统调用,用于从多个缓冲区(scatter/gather buffers)写入数据到文件描述符。它的原型如下:

```c
ssize_t sys_writev(unsigned long fd, const struct iovec *vec, unsigned long vlen)
```

\- fd: 待写入的文件描述符
\- *vec: iovec结构数组的指针,指定写入数据的缓冲区列表
\- vlen: vec数组的长度,指定有多少个缓冲区iovec结构定义如下:

```c
struct iovec 
{
  void *iov_base;  /* Starting address */
  size_t iov_len;  /* Number of bytes to transfer */
};
```

\- iov_base: 缓冲区起始地址
\- iov_len: 缓冲区长度(字节数)所以,sys_writev()可以从多个缓冲区(由vec数组指定)向fd写入数据。它会依次取vec数组中每个iovec结构指定的缓冲区,写入iov_len个字节的数据,直到全部写入完成或发生错误。这与write()系统调用不同,后者只能从单一缓冲区写入数据。sys_writev()提供了一种从片段化的、处于不同地址的多缓冲区高效写入数据的机制(sendfile()也有类似功能)。



## sys_readv()

sys_readv()是Linux内核中的一个系统调用,用于从文件描述符读取数据到多个缓冲区(scatter/gather buffers)。它的原型如下:

```c
ssize_t sys_readv(unsigned long fd, const struct iovec *vec, unsigned long vlen)
```

\- fd: 待读取的文件描述符
\- *vec: iovec结构数组的指针,指定读取数据的缓冲区列表
\- vlen: vec数组的长度,指定有多少个缓冲区iovec结构与sys_writev()中的定义相同:

```c
struct iovec 
{
  void *iov_base;  /* Starting address */
  size_t iov_len;  /* Number of bytes to transfer */
};
```

\- iov_base: 缓冲区起始地址 
\- iov_len: 缓冲区长度(字节数)所以,sys_readv()可以从fd读取数据到多个缓冲区(由vec数组指定)。它会依次取vec数组中每个iovec结构指定的缓冲区,读取iov_len个字节的数据,直到全部读取完成或发生错误。这与read()系统调用不同,后者只能读取数据到单一缓冲区。sys_readv()提供了一种从文件高效读取数据到片段化的、处于不同地址的多缓冲区的机制。

## sys_access

sys_access()是Linux内核中的一个系统调用,用于判断对一个文件的访问权限。它的原型如下:

```c
int sys_access(const char *filename, int mode) 
```

\- filename: 要判断访问权限的文件名
\- mode: 判断的访问权限,可以是F_OK、R_OK、W_OK和X_OK中的一个或多个。
  \- F_OK: 判断文件是否存在
  \- R_OK: 判断是否有可读权限
  \- W_OK: 判断是否有可写权限
  \- X_OK: 判断是否有可执行权限sys_access()会检查当前调用进程对filename指定的文件是否具有mode参数指定的访问权限。返回值:- 如果mode指定的所有访问权限都有,则返回0,表示成功
\- 如果任何一种访问权限没有,则返回-1,并设置错误号为EACCES

举个例子:

```c
int ret;
ret = sys_access("/etc/passwd", R_OK);  //检查是否可读
if (ret == -1) 
    printf("No read access!\n");

ret = sys_access("/etc/shadow", W_OK); //检查是否可写
if (ret == -1)
    printf("No write access!\n");
```

这两个示例分别 checks 如果当前进程对/etc/passwd是否有可读权限,和对/etc/shadow是否有可写权限。sys_access()系统调用通常在进行某个操作前,判断当前进程是否具有必要的文件访问权限,避免因为权限不足而失败。

## sys_utimensat()

sys_utimensat()是Linux内核中的一个系统调用,用于修改文件或目录的时间戳(timestamp)。它的原型如下:

```c
int sys_utimensat(int dirfd, const char *pathname, struct timespec times[2], int flags)
```

\- dirfd: 文件描述符或AT_FDCWD(-100),用于指定修改哪个目录下文件的时间戳
\- pathname: 要修改时间戳的文件路径名
\- times: 时间戳数组,包含修改后的atime和mtime
\- flags: 标志位,指定是修改文件还是目录的时间戳times数组包含两个timespec结构,分别指定最新的访问时间atime和修改时间mtime:

```c
struct timespec {
    __kernel_time_t    tv_sec;        /* seconds */   
    long        tv_nsec;    /* nanoseconds */ 
};
```

flags可以是:- AT_SYMLINK_NOFOLLOW:修改符号链接本身的时间戳,而非其指向的文件
\- AT_SYMLINK_FOLLOW:修改符号链接指向的文件的时间戳(默认)如果dirfd为AT_FDCWD,则修改当前工作目录下的文件时间戳。否则修改dirfd指定目录描述符所指目录下的文件时间戳。sys_utimensat()允许精确到纳秒的时间粒度来修改文件时间戳。这比旧的utime()和utimes()系统调用更为精确和高级。

举个例子:

```c
struct timespec times[2];
times[0].tv_sec = time(NULL);   // Set access time to current time
times[1].tv_sec = time(NULL);   // Set modification time to current time
sys_utimensat(AT_FDCWD, "/tmp/file", times, 0);
```

这会将/tmp/file访问时间和修改时间都设置为当前时间。

## sys_lseek()

sys_lseek()是Linux内核中的一个系统调用,用于移动文件描述符的读写指针。它的原型如下:

```c
off_t sys_lseek(unsigned int fd, off_t offset, unsigned int whence)
```

\- fd: 文件描述符
\- offset: 移动的偏移量
\- whence: 从哪个位置开始移动whence可以取以下值:- SEEK_SET: 相对文件开始位置移动
\- SEEK_CUR: 相对文件当前位置移动
\- SEEK_END: 相对文件结束位置移动sys_lseek()移动fd指定的文件描述符的读写指针。offset表示移动的字节数,可以是正数或负数。当ce参数指定从哪个位置开始移动。成功时,sys_lseek()返回新的文件指针位置。失败时返回-1,错误代码设置为EINVAL或ESPIPE。举个例子:

```c
int fd = open("file.txt", O_RDWR);

//移动到文件开始
sys_lseek(fd, 0, SEEK_SET);  

//移动10个字节
sys_lseek(fd, 10, SEEK_CUR);  

//移动到文件结束
sys_lseek(fd, 0, SEEK_END);
```

这个示例打开一个文件,并通过sys_lseek()移动文件描述符fd的读写指针到文件开始、当前位置之后10字节、文件结束这三个位置。sys_lseek()常用于读取文件的特定区域,实现随机访问的效果。通过它可以自由控制读写指针在文件中的位置。

## sys_umask()

sys_umask()是Linux内核中的一个系统调用,用于设置文件模式创建屏蔽字(file mode creation mask)。它的原型如下:

```c
mode_t sys_umask(mode_t mask)
```

\- mask: 设置的新文件模式创建屏蔽字sys_umask()用来设置进程的文件模式创建屏蔽字,它会禁止某些文件模式位对新创建的文件生效。文件模式包含文件类型(regular、directory等)以及文件访问权限(read、write、execute)等信息。文件模式创建屏蔽字的设置对进程新创建的文件生效。它是一个八进制数,每一位禁止对应的文件模式位,1表示禁止,0表示不禁止。例如:
\- 设置umask为022,禁止group写权限和其他用户的所有权限,则进程新建文件的默认权限为775
\- 设置umask为077,禁止其他用户的所有权限,则进程新建文件的默认权限为700umask也可用来设置目录的默认权限,规则类似。sys_umask()返回旧的umask值。通常在程序启动时调用它一次来设置所需要的umask。举个例子:

```c
//设置umask为022 
sys_umask(022);  

//新建文件默认权限为775
int fd = open("file.txt", O_CREAT, 0666);
```

这里umask设置为022,然后新建文件file.txt,文件模式为0666,与umask作与运算得0755,所以file.txt的最终文件权限为775。

## sys_fcntl64()

sys_fcntl64()是Linux内核中的一个系统调用,用于对打开的文件描述符进行各种控制操作。它的原型如下:

```c
long sys_fcntl64(unsigned int fd, unsigned int cmd, unsigned long arg)
```

\- fd: 文件描述符
\- cmd: 操作命令,用来指定对fd进行什么样的控制操作
\- arg: 根据cmd的不同,传递不同的参数cmd可以是以下值:- F_DUPFD: 复制文件描述符,arg指定复制出的描述符最小值
\- F_GETFD: 获取文件描述符标志,arg忽略
\- F_SETFD: 设置文件描述符标志,arg可以是FD_CLOEXEC
\- F_GETFL: 获取文件状态标志和访问模式,arg忽略
\- F_SETFL: 设置文件状态标志和访问模式,arg可以是O_APPEND、O_NONBLOCK等
\- F_GETLK: 获取文件锁定信息,arg指向struct flock结构
\- F_SETLK: 设置文件锁定(非阻塞),arg指向struct flock结构
\- F_SETLKW: 设置文件锁定(阻塞),arg指向struct flock结构
\- ...sys_fcntl64()实现了对已打开文件的各种控制,包括获取和设置文件状态标志、文件描述符标志、文件锁定等操作。举个例子:

```c
int fd = open("file.txt", O_RDWR);

//设置文件描述符fd为非阻塞  
sys_fcntl64(fd, F_SETFL, O_NONBLOCK);

//获取文件描述符3以上的一个可用描述符 
int new_fd = sys_fcntl64(fd, F_DUPFD, 3);  

struct flock lock;
lock.l_type = F_RDLCK;  //读锁
sys_fcntl64(fd, F_SETLKW, &lock);  //设置文件读锁,阻塞等待
```

这个示例打开一个文件,设置其为非阻塞,然后复制出一个新的文件描述符,最后对文件设置一个阻塞的读锁。

## sys_sendfile64()

sys_sendfile64()是Linux内核中的一个系统调用,用于高效传输文件内容到socket或其他文件描述符。它的原型如下:

```c
ssize_t sys_sendfile64(int out_fd, int in_fd, __off64_t *offset, size_t count)
```

\- out_fd: 输出文件描述符,通常为socket
\- in_fd: 输入文件描述符,文件的文件描述符
\- offset: 输入文件中的偏移地址,NULL表示从当前位置开始发送
\- count: 要发送的字节数sys_sendfile64()从in_fd指定的输入文件发送count个字节的数据到out_fd指定的输出文件描述符。如果offset不为NULL,则它指定在输入文件中的起始偏移地址。与直接读文件再写入socket相比,sys_sendfile64()有以下优点:1. 零拷贝:数据直接从内核空间的页面缓存发送到socket,减少数据拷贝次数,提高效率。2. 协议栈自动处理:发送到socket的数据会自动加入socket发送缓存,并由网络协议栈处理,程序不需要考虑分片与窗口大小等问题。3. 发送速度提高:由于减少数据拷贝和不需要用户态参与,发送速度可以显著提高。sys_sendfile64()通常用在实现文件下载等功能上,可以大幅提高传输效率。举个例子:

```c
int in_fd = open("input.file", O_RDONLY);
int out_fd = socket(AF_INET, SOCK_STREAM, 0);

//从输入文件描述符in_fd发送100000个字节的数据到socket out_fd
sys_sendfile64(out_fd, in_fd, NULL, 100000);
```

这个示例打开一个输入文件和一个socket,然后使用sys_sendfile64()将100000字节的数据从文件发送到socket。

## sys_renameat2()

sys_renameat2()是Linux内核中的一个系统调用,用于重命名文件或目录。与renameat()系统调用不同,它允许指定标志位来控制文件重命名的行为。它的原型如下:

```c
int sys_renameat2(int olddirfd, const char *oldpath, int newdirfd, const char *newpath, unsigned int flags)
```

\- olddirfd: 源文件路径名的目录文件描述符
\- oldpath: 源文件路径名
\- newdirfd: 目标文件路径名的目录文件描述符
\- newpath: 目标文件路径名
\- flags: 标志位,控制重命名行为flags可以设置以下值:- RENAME_NOREPLACE: 重命名失败,如果目标文件已经存在
\- RENAME_EXCHANGE: 重命名交换两个文件 
\- RENAME_WHITEOUT: 创建目标文件并插入whiteout标记如果newpath已经存在,并且未指定RENAME_NOREPLACE标志,则新文件会替换旧文件。成功重命名文件后,sys_renameat2()返回0。失败时返回-1,并设置错误码。举个例子:

```c
//重命名/tmp/file1为/tmp/file2,如果file2已存在则失败 
sys_renameat2(AT_FDCWD, "/tmp/file1", AT_FDCWD, "/tmp/file2", RENAME_NOREPLACE);

//交换/tmp/file1和/tmp/file2
sys_renameat2(AT_FDCWD, "/tmp/file1", AT_FDCWD, "/tmp/file2", RENAME_EXCHANGE); 
```

这个示例演示了如何使用RENAME_NOREPLACE防止目标文件被替换,和使用RENAME_EXCHANGE交换两个文件的重命名。sys_renameat2()提供了更加灵活的文件重命名机制,通过标志位可以实现只有当目标文件不存在时才重命名、交换两个文件的重命名等功能。