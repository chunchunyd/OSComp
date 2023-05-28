# Week14 Report

## 1. 完成初赛
![image](https://github.com/chunchunyd/OSComp/assets/80908379/27ccd4ae-0ed8-465a-a7b5-89fd4c39db7f)

1. 解决所有已知bug；
2. 依赖库本地化并采用--offline构建，适应测试平台环境，修改makefile，在容器内生成内核；
3. 起草初赛报告文档；
4. 对已有的代码做了一些优化，例如link与unlink的逻辑、一些使用频次较高的结构中含有String属性更换为&str并引入生命周期、尝试实际挂载文件系统；

## 2. 准备决赛

1. 构建镜像，尝试libc测例,

```shell
cd kernel
make clean
DISK_DIR=libc make testcases-img
```

2. 添加了系统调用writev,readv,access,lseek,renameat。

## 3. 其他

1. 尝试用已有的syscall实现一个简单的用户程序shell

   ![image-20230528053739721](C:\Users\hawan\AppData\Roaming\Typora\typora-user-images\image-20230528053739721.png)

   ![image-20230528053915595](C:\Users\hawan\AppData\Roaming\Typora\typora-user-images\image-20230528053915595.png)

在shell里实现了echo,ls,exit三个简单的功能。
