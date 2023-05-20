# Week13 Report

## 1. 本地通过了所有测试

- brk

  ```
  ========== START test_brk ==========
  Before alloc,heap pos: 16384
  After alloc,heap pos: 16448
  Alloc again,heap pos: 16512
  ========== END test_brk ==========
  
  ```

  ......

  ......

- pipe

  ```
  ========== START test_pipe ==========
  cpid: 6
  cpid: 0
    Write to pipe successfully.
  
  ========== END test_pipe ==========
  
  ```

- dup

  ```
  ========== START test_dup ==========
    new fd is 3.
  ========== END test_dup ==========
  ```

- dup2

  ```
  ========== START test_dup2 ==========
    from fd 100
  ========== END test_dup2 ==========
  ```

- mkdir_

  ```
  ========== START test_mkdir ==========
  mkdir ret: 0
    mkdir success.
  ========== END test_mkdir ==========
  ```

- chdir

  ```
  ========== START test_chdir ==========
  chdir ret: 0
    current working dir : /test_chdir/
  ========== END test_chdir ==========
  
  ```

- unlink

  ```
  ========== START test_unlink ==========
    unlink success!
  ========== END test_unlink ==========
  ```

- mount

  ```
  ========== START test_mount ==========
  Mounting dev:/dev/vda2 to ./mnt
  mount return:0
  mount successfully
  umount return: 0
  ========== END test_mount ==========
  ```

- umount

  ```
  ========== START test_umount ==========
  Mounting dev:/dev/vda2 to ./mnt
  mount return:0
  umount success.
  return: 0
  ========== END test_umount ==========
  ```

但是实际上很多系统调用的实现还很简陋，只能通过目前的测例，例如fstat里很多信息在底层并没有实现。

## 2. 研究并尝试实现了一些后续的文件系统调用

例如sys_renameat2()：

> ### sys_renameat2()
>
> sys_renameat2()是Linux内核中的一个系统调用,用于重命名文件或目录。与renameat()系统调用不同,它允许指定标志位来控制文件重命名的行为。它的原型如下:
>
> ```c
> int sys_renameat2(int olddirfd, const char *oldpath, int newdirfd, const char *newpath, unsigned int flags)
> ```
>
> \- olddirfd: 源文件路径名的目录文件描述符
> \- oldpath: 源文件路径名
> \- newdirfd: 目标文件路径名的目录文件描述符
> \- newpath: 目标文件路径名
> \- flags: 标志位,控制重命名行为flags可以设置以下值:- RENAME_NOREPLACE: 重命名失败,如果目标文件已经存在
> \- RENAME_EXCHANGE: 重命名交换两个文件 
> \- RENAME_WHITEOUT: 创建目标文件并插入whiteout标记如果newpath已经存在,并且未指定RENAME_NOREPLACE标志,则新文件会替换旧文件。成功重命名文件后,sys_renameat2()返回0。失败时返回-1,并设置错误码。

```Rust
pub fn syscall_renameat2(old_dirfd: usize, old_path: *const u8, new_dirfd: usize, new_path: *const u8, flags: usize) -> isize {
    let process = current_process();
    let mut process_inner = process.inner.lock();

    let old_path = process_inner.memory_set.lock().translate_str(old_path);
    let new_path = process_inner.memory_set.lock().translate_str(new_path);

    // 处理path
    let mut old_path_ = "".to_string();
    if !old_path.starts_with('/') {
        ......
    }

    if flags == RENAME_NOREPLACE {
        if api::metadata(new_path_.as_str()).is_ok() {
            debug!("new_path_ already exist");
            return -1;
        }
    }

    if flags == RENAME_EXCHANGE {
        let old_metadata = api::metadata(old_path_.as_str()).unwrap();
        let new_metadata = api::metadata(new_path_.as_str()).unwrap();
        if old_metadata.is_dir() != new_metadata.is_dir() {
            debug!("old_path_ and new_path_ is not the same type");
            return -1;
        }
    }

    if flags == RENAME_WHITEOUT {
        let new_metadata = api::metadata(new_path_.as_str()).unwrap();
        if new_metadata.is_dir() {
            debug!("new_path_ is a directory");
            return -1;
        }
    }

    if flags != RENAME_NOREPLACE && flags != RENAME_EXCHANGE && flags != RENAME_WHITEOUT {
        debug!("flags is not valid");
        return -1;
    }

    if api::rename(old_path_.as_str(), new_path_.as_str()).is_err() {
        debug!("rename failed");
        return -1;
    }

    0
}
```

## 3. 在线评测尝试

- 修改makefile
- 搭建本地docker服务器







