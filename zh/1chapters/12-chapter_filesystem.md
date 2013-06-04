# 文件系统 #

## 简介 ##

RT-Thread 的文件系统采用了三层的结构，如图 ***文件系统结构图*** 所示：

![文件系统结构图](figures/rt_fs_structure.png)

最顶层的是一套面向嵌入式系统，专门优化过的虚拟文件系统（接口）。通过它，RT-thread 操作系统能够适配下层不同的文件系统格式，例如个人电脑上常使用的FAT 文件系统，或者是嵌入式设备中常见的flash 文件系统（YAFFS2、JFFS2 等）。

接下来中间的一层是各种文件系统的实现，例如支持FAT文件系统的DFS-ELM、支持NandFlash 的YAFFS2，只读文件系统ROMFS 等。（RT-Thread 1.0.0版本中包含了ELM FatFS，ROMFS以及网络文件系统NFS v3实现，YAFFS2等flash 文件系统则包含在了RT-Thread 1.1.0 版本中）

最底层的是各类存储驱动，例如SD 卡驱动，IDE 硬盘驱动等。RT-Thread 
1.1.0 版本也将在NandFlash 上构建一层转换层(FTL)，以使得NandFlash
能够支持Flash 文件系统。 

RT-Thread 的文件系统对上层提供的接口主要以POSIX 标准接口为主，这样也能够保证程序可以在PC 上编写、调试，然后再移植到RT-Thread 操作系统上。 


##文件系统、文件与文件夹##

文件系统是一套实现了数据的存储、分级组织、访问和获取等操作的抽象数据类型(Abstract data type)，是一种用于向用户提供底层数据访问的机制。文件系统通常存储的基本单位是文件，即数据是按照一个个文件的方式进行组织。当文件比较多时，将导致文件繁多，不易分类、重名的问题。而文件夹作为一个容纳多个文件的容器而存在。

在 RT-Thread 中，文件系统名称使用上类似UNIX 文件、文件夹的风格，例如如图 ***目录结构*** 的目录结构：

![目录结构](figures/rt_fs_eg.png)

在RT-Thread 操作系统中，文件系统有统一的根目录，使用’/’来表示。而在根目录下的f1.bin 文件则使用’/f1.bin’来表示，2011 目录下的f1.bin目录则使用’/data/2011/f1.bin’来表示。即目录的分割符号是’/’，这与UNIX/Linux 完全相同的，与Windows 则不相同（Windows 操作系统上使用’\’来作为目录的分割符）。 

默认情况下，RT-Thread 操作系统为了获得较小的内存占用，宏定义DFS_USING_WORKDIR 并不会被定义。当它不定义时，那么在使用文件、目录
接口进行操作时应该使用绝对目录进行（因为此时系统中不存在当前工作的目录）。如果需要使用当前工作目录以及相对目录，可以在rtconfig.h头文件中定义DFS_USING_WORKDIR 宏。

##文件系统接口##

###打开文件###

打开或创建一个文件可以调用下面的open 函数接口：

	int open(const char *pathname, int oflag, int mode);

参数： 

+ pathname  - 打开或创建的文件名；

+ oflag   - 指定打开文件的方式，当前支持的打开方式有：


+--------------+--------------------------------------------+
|      参数    |        描述                                |
+==============+============================================+
|   O_RDONLY   |  只读方式打开文件                          |
+--------------+--------------------------------------------+
|   O_WRONLY   |  只写方式打开文件                          |
+--------------+--------------------------------------------+
|   O_RDWR     |  以读写方式打开文件                        |
+--------------+--------------------------------------------+
|   O_CREAT    |  如果要打开的文件不存在，则建立该文件。    |
+--------------+--------------------------------------------+
|   O_APPEND   |  当读写文件时会从文件尾开始移动，也就是所  |
|              |  写入的数据会以附加的方式添加到文件的尾部。|
+--------------+--------------------------------------------+

+ mode    - 与POSIX 标准接口像兼容的参数（目前没有意义，传入0即可）。

+ 返回值：

打开成功时返回打开文件的描述符序号，否则返回负数。可以参考 @@以下@@ 代码，看看如何去打开一个文件。

~~~{.c}
#include <rtthread.h> 
#include <dfs_posix.h> /* 当需要使用文件操作时，需要包含这个头文件 */ 
 
/* 假设文件操作是在一个线程中完成*/ 
void file_thread() 
{ 
    int fd, size; 
    char s[] = "RT-Thread Programmer!\n", buffer[80]; 

    /* 打开/text.txt 作写入，如果该文件不存在则建立该文件*/ 
    fd = open("/text.txt", O_WRONLY | O_CREAT); 
    if (fd >= 0) 
    { 
        write(fd, s, sizeof(s)); 
        close(fd); 
    } 
    
    /* 打开/text.txt 准备作读取动作*/ 
    fd = open("/text.txt", O_RDONLY); 
    if (fd >= 0) 
    { 
        size=read(fd, buffer, sizeof(buffer)); 
        close(fd); 
    } 
    
    rt_kprintf("%s", buffer); 
}
~~~

###关闭文件###

当使用完文件后若不再需要使用则可使用close()函数接口关闭该文件，而close()会让数据写回磁盘，并释放该文件所占用的资源。关闭文件的函数接口如下：

	int close(int fd); 

+ 参数： 

    fd - open()函数所返回的文件描述字。 

+ 返回值：
无

###读取数据###

读取数据可使用下面的函数接口：

    ssize_t read(int fd, void *buf, size_t count);

+ 参数： 

    fd    - 文件描述词； 
    buf  - 内存指针； 
    count - 预读取文件的字节数。

+ 返回值： 

实际读取到的字节数。read()函数接口会把参数fd 所指的文件的count 个字节传送到buf 指针所指的内存中。返回值为实际读取到的字节数，有两种情况会返回0 值，一是读取数据已到达文件结尾，二是无可读取的数据（例如设定count为0），此外，文件的读写位置会随读取到的字节移动。

###写入数据###

写入数据可使用下面的函数接口：

	size_t write(int fd, const void *buf, size_t count); 

+ 参数： 

    fd    - 文件描述词； 
    buf   - 内存指针； 
    count  - 预写入文件的字节数。 

+ 返回值： 
    实际写入的字节数。 

write()函数接口会把buf 指针所指向的内存中count 个字节写入到参数fd 所指的文件内。返回值为实际写入文件的字节数，返回值为0 时表示写入出错，错误代码存入当前线程的errno中，此外，文件的读写位置会写入的字节移动。

可以参考 @@以下@@ 代码，看看一个完整的文件读写流程：

~~~{.c}
/* 
 * 代码清单：文件读写例子 
 * 这个例子演示了如何读写一个文件，特别是写的时候应该如何操作。 
 */ 
 
#include <rtthread.h> 
#include <dfs_posix.h> /* 当需要使用文件操作时，需要包含这个头文件 */ 
 
#define TEST_FN    "/test.dat" 
 
/* 测试用的数据和缓冲 */ 
static char test_data[120], buffer[120]; 
 
/* 文件读写测试 */ 
void readwrite(const char* filename) 
{ 
    int fd; 
    int index, length; 
    
    /* 只写 & 创建 打开 */ 
    fd = open(TEST_FN, O_WRONLY | O_CREAT | O_TRUNC, 0); 
    if (fd < 0) 
    { 
        rt_kprintf("open file for write failed\n"); 
        return; 
    } 
    
    /* 准备写入数据 */ 
    for (index = 0; index < sizeof(test_data); index ++) 
    { 
        test_data[index] = index + 27; 
    } 
    
    /* 写入数据 */ 
    length = write(fd, test_data, sizeof(test_data)); 
    if (length != sizeof(test_data)) 
    { 
        rt_kprintf("write data failed\n"); 
        close(fd); 
        return; 
    } 
    
    /* 关闭文件 */ 
    close(fd); 
    
    /* 只写并在末尾添加打开 */ 
    fd = open(TEST_FN, O_WRONLY | O_CREAT | O_APPEND, 0); 
    if (fd < 0) 
    { 
        rt_kprintf("open file for append write failed\n"); 
        return; 
    } 
    
    length = write(fd, test_data, sizeof(test_data)); 
    if (length != sizeof(test_data)) 
    { 
        rt_kprintf("append write data failed\n"); 
        close(fd); 
        return; 
    } 
    /* 关闭文件 */ 
    close(fd); 
    
    /* 只读打开进行数据校验 */ 
    fd = open(TEST_FN, O_RDONLY, 0); 
    if (fd < 0) 
    { 
        rt_kprintf("check: open file for read failed\n"); 
        return; 
    } 
    
    /* 读取数据(应该为第一次写入的数据) */ 
    length = read(fd, buffer, sizeof(buffer)); 
    if (length != sizeof(buffer)) 
    { 
        rt_kprintf("check: read file failed\n"); 
        close(fd); 
        return;
    } 
    
    /* 检查数据是否正确 */ 
    for (index = 0; index < sizeof(test_data); index ++) 
    { 
        if (test_data[index] != buffer[index]) 
        { 
            rt_kprintf("check: check data failed at %d\n", index); 
            close(fd); 
            return; 
        } 
    } 
    
    /* 读取数据(应该为第二次写入的数据) */ 
    length = read(fd, buffer, sizeof(buffer)); 
    if (length != sizeof(buffer)) 
    { 
        rt_kprintf("check: read file failed\n"); 
        close(fd); 
        return;
    }
    
    /* 检查数据是否正确 */ 
    for (index = 0; index < sizeof(test_data); index ++) 
    { 
        if (test_data[index] != buffer[index]) 
        { 
            rt_kprintf("check: check data failed at %d\n", index); 
            close(fd); 
            return; 
        } 
    } 
    
    /* 检查数据完毕，关闭文件 */ 
    close(fd); 
    /* 打印结果 */ 
    rt_kprintf("read/write done.\n"); 
}

#ifdef RT_USING_FINSH 
#include <finsh.h> 
/* 输出函数到finsh shell 命令行中 */ 
FINSH_FUNCTION_EXPORT(readwrite, perform file read and write test); 
#endif 
~~~

###更改名称###

更改文件的名称可使用下面的函数接口：

	int rename(const char *oldpath, const char *newpath); 

+ 参数： 
    oldpath   - 需更改的文件名； 
    newpath   - 更改成的文件名。 

+ 返回值：
    无 

rename()会将参数oldpath 所指定的文件名称改为参数newpath 所指的文件名称。若newpath 所指定的文件已经存在，则该文件将会被覆盖。可以参考 @@以下@@ 代码，如何进行文件名改名。

~~~{.c}
#include <dfs_posix.h> 

void file_thread(void* parameter) 
{
    rt_kprintf("%s => %s ", "/text1.txt", "/text2.txt"); 

    if(rename("/text1.txt", "/text2.txt") <0 ) 
        rt_kprintf("[error!]\n"); 
    else 
        rt_kprintf("[ok!]\n"); 
}
~~~

这个示例函数会把文件’/text1.txt’改名成’/text2.txt’。

###取得状态###

获取文件状态可使用下面的stat 函数接口：

	int stat(const char *file_name, struct stat *buf); 

stat()函数用来将参数file_name 所指向的文件状态，复制到buf 指针所指的结构中(struct stat)。 

+ 参数： 
    file_name  - 文件名； 
    buf     - 结构指针，指向获取文件状态的结构。 

返回值：
  无 
可以参考 @@以下@@ 代码了解如何使用stat 函数。

~~~{.c}
void file_thread(void* parameter) 
{ 
    struct stat buf; 
    stat("/text.txt", &buf); 
    rt_kprintf("text.txt file size = %d\n", buf.st_size); 
}
~~~

##目录操作接口##

###创建目录###

创建目录可使用下面的函数接口： 

	int mkdir(const char *path, mode_t mode); 

mkdir()函数用来创建一个目录，参数path 为目录名，参数mode 在当前版本未启用，输入0x777 即可。 

+ 参数： 
    path  - 目录名； 
    mode  - 创建模式。 
+ 返回值： 
    创建成功返回0，创建失败返回-1。 

可以参考 @@以下@@ 代码了解如何使用mkdir 函数：

~~~{.c}
void file_thread(void* parameter) 
{ 
    int ret; 
    
    /* 创建目录*/ 
    ret = mkdir("/web", 0x777); 
    if(ret < 0) 
    { 
        /* 创建目录失败*/ 
        rt_kprintf("mkdir error!\n"); 
    } 
    else 
    {  
        /* 创建目录成功*/ 
        rt_kprintf("mkdir ok!\n"); 
    } 
}
~~~


###打开目录###

打开目录可使用下面的函数接口： 

	DIR* opendir(const char* name); 

opendir()函数用来打开一个目录，参数name 为目录路径名。若读取目录成功，返回该目录结构，若读取目录失败，返回RT_NULL。 

+ 参数： 
    name  - 目录路径名。 

+ 返回值： 
    打开文件成功，返回指向目录的DIR 结构指针，否则返回RT_NULL。 

    可以参考 @@以下@@ 代码了解如何使用opendir()函数：
    
~~~{.c}
#include <dfs_posix.h> 
 
void dir_operation(void* parameter) 
{
    int result; 
    DIR *dirp; 
    
    /* 打开/web 目录*/ 
    dirp = opendir("/web"); 
    if(dirp == RT_NULL) 
    { 
        rt_kprintf("open directory error!\n"); 
    } 
    else 
    {
        /* 在这儿进行读取目录相关操作*/ 
        /* ...... */ 
            
        /* 关闭目录 */ 
          closedir(dirp); 
    } 
}
~~~ 


###读取目录###
读取目录可使用下面的函数接口： 

	struct dirent* readdir(DIR *d); 

readdir()函数用来读取目录，参数d 为目录路径名。返回值为读到的目录项结构，如果返回值为RT_NULL，则表示已经读到目录尾；此外，每读取一次目录，目录流的指针位置将自动往后递推1 个位置。 

+ 参数： 

    d  - 目录路径名。 

+ 返回值： 

    读取成功返回指向目录entry 的结构指针，否则返回RT_NULL。 

可以参考 @@以下@@ 代码了解如何使用readdir 函数： 

~~~{.c}
void dir_operation(void* parameter) 
{ 
    int result; 
    DIR *dirp; 
    struct dirent *d; 

    /* 打开/web 目录*/ 
    dirp = opendir("/web"); 
    if(dirp == RT_NULL) 
    { 
        rt_kprintf("open directory error!\n"); 
    } 
    else 
    { 
        /* 读取目录*/ 
        while ((d = readdir(dirp)) != RT_NULL) 
        { 
            rt_kprintf("found %s\n", d->d_name); 
        } 

        /* 关闭目录 */ 
        closedir(dirp); 
    } 
} 
~~~

###取得目录流的读取位置###

获取目录流的读取位置可使用下面的函数接口：

	off_t telldir(DIR *d); 

+ 参数： 

    d  - 目录路径名。 
+ 返回值： 

    无 

###设置下次读取目录的位置###

设置下次读取目录的位置可使用下面的函数接口： 

	void seekdir(DIR *d, off_t offset); 

+ 参数： 

    d    - 目录路径名； 
    offset  - 偏移值，距离本次目录的位移。 
+ 返回值： 

    无 
可以参考 @@以下@@代码了解如何使用seekdir 函数：

~~~{.c}
void dir_operation(void* parameter) 
{ 
    DIR * dirp; 
    int save3 = 0; 
    int cur; 
    int i = 0; 
    struct dirent *dp; 
    
    /* 打开根目录 */ 
    dirp = opendir ("/"); 
    for (dp = readdir (dirp); dp != RT_NULL; dp = readdir (dirp)) 
    { 
        /* 保存第三个目录项的目录指针*/ 
        if (i++ == 3) 
            save3 = telldir (dirp); 
        
        rt_kprintf ("%s\n", dp->d_name); 
    } 
    
    /* 回到刚才保存的第三个目录项的目录指针*/ 
    seekdir (dirp, save3); 
    
    /* 检查当前目录指针是否等于保存过的第三个目录项的指针. */ 
    cur = telldir (dirp); 
    if (cur != save3) 
    { 
        rt_kprintf ("seekdir (d, %ld); telldir (d) == %ld\n", save3, cur); 
    } 
    
    /* 从第三个目录项开始打印*/ 
    for (dp = readdir(dirp); dp != NULL; dp = readdir (dirp)) 
        rt_kprintf ("%s\n", dp->d_name); 
    
    /* 关闭目录*/ 
    closedir (dirp); 
}
~~~


###重设读取目录的位置为开头位置###

重设读取目录为开头位置可使用下面的函数接口： 

	void rewinddir(DIR *d); 

+ 参数： 

    d  - 目录路径名； 
+ 返回值： 

    无 

###关闭目录###
关闭目录可使用下面的函数接口： 

	int closedir(DIR* d); 

closedir()函数用来关闭一个目录。该函数必须和opendir()函数成对出现。 

+ 参数： 

    d  - 目录路径名； 
+ 返回值： 

    关闭成功返回0，否则返回-1； 


###删除目录###
删除目录可使用下面的函数接口： 

	int rmdir(const char *pathname); 

+ 参数： 

    d  - 目录路径名； 

+ 返回值： 

    删除目录成功返回0，否则返回-1。


##底层驱动接口##

RT-Thread DFS 文件系统针对下层媒介使用的是RT-Thread 的设备IO系统，其中主要包括设备读写等操作。但是有些文件系统并不依赖于RT-Thread 的设备系统，例如1.0.x分支引入的只读文件系统、网络文件系统等。对于常使用的FAT 文件系统，下层驱动必须用块设备的形式来实现。


##文件系统初始化##

在使用文件系统接口前，需要对文件系统进行初始化，代码如下：

~~~{.c}
#ifdef RT_USING_DFS 
/* 包含DFS 的头文件 */ 
#include <dfs_fs.h> 
#include <dfs_elm.h> 
#endif 
 
/* 初始化线程 */ 
void rt_init_thread_entry(void *parameter) 
{ 
    /* 文件系统初始化 */ 
    #ifdef RT_USING_DFS 
    { 
    /* 初始化设备文件系统 */ 
    dfs_init(); 
    #ifdef RT_USING_DFS ELMFAT 
    /* 如果使用的是ELM 的FAT 文件系统，需要对它进行初始化 */ 
    elm_init(); 

    /* 调用dfs_mount 函数对设备进行装载 */ 
    if (dfs_mount("sd0", "/", "elm", 0, 0) == 0) 
        rt_kprintf("File System initialized!\n"); 
    else 
        rt_kprintf("File System init failed!\n"); 
    #endif 
    } 
    #endif 
}
~~~

其主要包括的函数接口为： 

	int dfs_mount(const char* device_name, const char* path, const char* filesystemtype, 
	              rt_uint32_t rwflag, const void* data); 

dfs_mount 函数用于把以device_name 为名称的设备挂接到path 路径中。filesystemtype 指定了设备上的文件系统的类型（如上面代码所述的elm、rom、nfs 等文件系统）。data参数对某些文件系统是有意义的，如nfs，对elm 类型系统则没有意义。 

+ 参数：

    device_name   - 设备名； 
    path      - 挂接路径； 
    filesystemtype  - 文件系统的类型； 
    rwflag      - 文件系统的标志； 
    data      - 文件系统的数据。 

+ 返回值：

    装载成功将返回0，否则返回-1。具体的错误需要查看errno。

