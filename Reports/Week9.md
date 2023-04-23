# Week9进展报告

## 1. 基本了解了Arceos的axfs组件

1. 添加了一些文档和注释

2. 总结了它提供的一些大概率接口;

   ```
   1. 文件操作接口:
   - File - 文件类型
   - File::open() - 打开文件,返回`File`对象
   - OpenOptions - 文件打开选项,用来配置打开方式
   - File::read() - 读文件
   - File::write() - 写文件
   - File::seek() - 文件指针操作
   - File::flush() - 将缓冲区中的数据写入磁盘
   - File::metadata() - 获取文件元数据
   - File::set_len() - 设置文件大小
   2. 目录操作接口: 
   - read_dir() - 读取目录条目
   - DirEntry - 目录条目,包含文件名,文件类型等
   - DirBuilder - 目录创建器,可以递归创建目录
   3. 路径操作接口:
   - canonicalize() - 路径标准化
   - current_dir() - 获取当前工作目录
   - set_current_dir() - 设置当前工作目录
   4. 其它接口:
   - read() - 读取整个文件到Vec<u8>
   - read_to_string() - 读取整个文件到String
   - write() - 将数据写入文件
   - create_dir() - 创建目录
   - create_dir_all() - 递归创建目录 
   - remove_dir() - 删除空目录
   - remove_file() - 删除文件
   - symlink() - 创建符号链接
   - symlink_metadata() - 获取符号链接元数据
   - read_link() - 读取符号链接目标
   ```

3. 发现了一些可能的bug；
例如目录的lookup方法里递归调用时发生了死循环，添加了额外的判断条件。
```Rust
pub(crate) fn lookup(dir: Option<&VfsNodeRef>, path: &str) -> AxResult<VfsNodeRef> {
    if path.is_empty() {
        return ax_err!(NotFound);
    }
    let node = parent_node_of(dir, path).lookup(path)?;
    if path.ends_with('/') && !node.get_attr()?.is_dir() {
        ax_err!(NotADirectory)
    } else {
        Ok(node)
    }
}
```
```
pub(crate) fn lookup(dir: Option<&VfsNodeRef>, path: &str) -> AxResult<VfsNodeRef> {
     // 首先判断绝对路径,如果是,直接在根目录查找
     if path.starts_with('/') { 
         return _ROOT_DIR_.lookup(path); 
     }
     // ...
 }
 ```


4. 尝试实现了学长留下的一些TODO，比如用字典树记录、查找挂载点;
```Rust
struct PathTrie {
     children: BTreeMap<char, PathTrie>,
     mount_point: Option<MountPoint>,
 }
 
impl PathTrie {
     fn insert(&mut self, path: &str, mp: MountPoint) {
         let mut node = self;
         for c in path.chars() {
             if let Some(child) = node.children.get_mut(&c) {
                 node = child;
             } else {
                 let child = PathTrie::default();
                 node.children.insert(c, child);
                 node = &mut child;
             }
         }
         node.mount_point = Some(mp);
     }
 
     fn longest_prefix(&self, path: &str) -> Option<(usize, &MountPoint)> {
         let mut node = self;
         let mut len = 0;
         for c in path.chars() {
             if let Some(child) = node.children.get(&c) {
                 node = child;
                 len += 1;
                 if node.mount_point.is_some() {
                     return Some((len, node.mount_point.as_ref().unwrap()));
                 }
             } else {
                 break;
             }
         }
         None
     }
 }
```

5. 正在分析学习它依赖的fatfs库。

## 2. 实现了用户态与内核态的分离

要实现文件相关的系统调用，还是要先支持最基础的用户态分离。

## 3. 实现了open, write，close
### 0. 文件描述符表
```Rust
struct FileDesc {
     file: fs::File,
     flags: i32,
 }

struct FileTable {
     table: Vec<FileDesc>,
 }
```

### 1. 在用户库添加对应的系统调用

```Rust
pub fn sys_open(path: &str, flags: usize, mode: usize) -> isize {
    let ret =syscall(
        SYSCALL_OPEN,
        [path.as_ptr() as usize, path.len(), flags, mode, 0, 0],
    );
}

pub fn sys_close(fd: usize) -> isize {
    syscall(SYSCALL_CLOSE, [fd, 0, 0, 0, 0, 0])
}

```

### 2. 在axfs为close添加支持

在File的Drop trait方法中修改，添加fflush。

```Rust
impl Drop for Directory {
    fn drop(&mut self) {
        unsafe { self.node.access_unchecked().release().ok() };
    }
}
```

```Rust
impl Drop for File {
    fn drop(&mut self) {
        //fflush
        if let Err(err) = self.flush() {
            panic!("File flush failed: {:?}", err);
        }
        if let Err(err) = unsafe { self.node.access_unchecked().release() } {
            panic!("File close failed: {:?}", err);
        }
    }
}
```

到这里发现vfs里的fsync似乎还没实现, 看上去只在trait里声明了，写回磁盘并未实际进行。

### 3.在axhal里对syscall进行分发。
### 4.将syswrite的参数改为特征对象`dyn File+Send+Sync`，统一标准输入输出流和文件输入输出流。





