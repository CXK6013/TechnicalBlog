# 重新认识IO(二) 探秘零拷贝

[TOC]

## 前言

回忆一下我们上面讲的《深入Java IO：文件读写原理（一）》里面讲的内容，我们主要介绍了字节流FileOutputStream和FileOutputStream, 缓冲流BufferedOutputStream和BufferedInputStream。最基础的还是字节流FileOutputStream和FileOutputStream，缓冲流还是在字节流之上做了一层封装:

![](https://a.a2k6.com/gerald/i/2025/01/16/m7g6.png)

而字节流读写数据本质上都是借助操作系统的能力，也就是调用操作系统的api:

```java
// FileOutputStream
public void write(byte b[]) throws IOException {
        writeBytes(b, 0, b.length, append);
}
private native void writeBytes(byte b[], int off, int len, boolean append) throws IOException;
```

```java
// FileInputStream
public int read() throws IOException {
     return read0();
}
private native int read0() throws IOException;
```

在Linux上调用的是：

```c
//read()函数尝试从文件描述符fd中读取最多count个字节的数据，并将其存储到以buf为起始地址的缓冲区中。
size_t read(int fd, void buf[.count], size_t count);
// write() 函数尝试将最多count个字节从buf指向的缓冲区写入到文件描述符fd所引用的文件中
// 成功调用并不承诺数据到达磁盘
size_t write(int fd, const void buf[.count], size_t count);
```

这其实是一个系统调用，所谓系统调用是指应用程序调用操作系统的函数，这些操作系统提供的函数是进程在用户态没有特权和能力完成的操作，当应用进程发起系统调用的时候，会进入到内核态，由操作系统来操作系统资源，完成之后再返回到进程。我们在《深入Java IO：文件读写原理（一）》提到write调用其实是将数据刷新到了page cache中去了，并把对应页面标记为脏页（dirty page）, 然后添加到链表(dirty list), 内核会定期把这个链表里面的内容刷到磁盘上，保证磁盘和缓存的一致性。

![](https://a.a2k6.com/gerald/i/2025/01/16/3a6.png)

这么设计的好处是提升了写数据的性能，但是劣势是磁盘和缓存的一致性降低了。放松一致性的要求来提升性能，这种例子并不少见。为了提升读数据的性能，Linux在page cache里面引入了两个链表：

- active list：活跃链表
- inactive list：不活跃链表



![](https://a.a2k6.com/gerald/i/2024/12/21/5kvf2.jpg)



当内核需要分配内存或者加载磁盘上的文件时，会触发缺页中断(page fault)。新加入的页面会被放入Inactive链表。在内存页面里面有两个标志位：PG_referenced、PG_active。PG_active为1则当前页面在活跃链表里面，PG_active = 0 , 则该页面在非活跃链表里面。如果inactive List上的页面被访问两次，则该页面会被晋升到活跃链表中。

## read/write调用的流程

现在让我们分析一下程序中常见的文件复制流程：本机文件传输、网络文件传输。

### 本机的文件复制

```java
try(FileInputStream fileInputStream  = new FileInputStream("");
  FileOutputStream fileOutputStream = new FileOutputStream("")
) {
    byte[] readData = new byte[4096];
    int bytesRead = -1;
    while( (bytesRead = fileInputStream.read(readData)) != -1){
        fileOutputStream.write(readData, 0, bytesRead);
    }
}
```



![](https://a.a2k6.com/gerald/i/2025/01/17/ypv6.png)





### 网络文件复制





## 零拷贝

事实上在JDK下面还有更高效的复制文件的方式，这两种方式都依赖于操作系统，也就是mmap和transferTo，我们将分别介绍这两种文件复制方式和其高效的原理。

### mmap简介

所谓mmap是Memory-mapped 的缩写也就是内存映射,在Linux中对mmap这个方法的介绍如下:

> mmap()函数在调用进程的虚拟地址空间中创建一个新的映射。新映射的起始地址在addr参数中指定。length参数指定映射的长度(必须大于0)

所谓映射是一种对应关系，定义函数的时候用到了映射:

> 设A,B是两个非空集合，若对A中的任一元素x，依照某种规律或法则f，恒有B中唯一确定的元素y与之对应，则称此对应规律或法则f为一个从A到B的映射。

那么这么来看我们需要进程的虚拟地址空间开辟一段区域和我们需要操作的文件进行映射，那这样我就有一个疑问，我认为开辟区域的这块内存和文件大小是一对一的，也就是说操纵的文件有一个字节，在进程的地址空间也同样需要一个字节，那这样会很消耗内存，如果是操作大文件，这会不会将操作系统的内存都消耗殆尽？  

实际中，操作系统采取了一种延迟(lazy)分配的方式，当你通过mmap调用一个匿名的页面，内核会狡猾的立即返回一个指针。然后它会等待,直到你通过"触碰"该内存而触发一个缺页异常(page fault),才会进行真正的内存分配工作。这种机制被称为"按需分页"(demand paging)。

这是一种高效利用内存的策略，如果你还没触及对应的内存，否则就不会分配，这意味着你可以分配超过物理内存的虚拟内存，所谓的触及指的是读取或者写入，对新的映射区域的写入操作需要内核执行完整的内存分配，你需要内存，现在就需要，不能再推迟了。

那看来这种按需分配不会撑爆进程，操作系统相当狡猾，紧接着我又提出了第二个问题，那一次映射多少呢？ 映射全部估计还是会有风险，让我们来看下源码，在FileChannel里面我们可以构造出来mmap， 像下面这样:

```java
try (RandomAccessFile randomAccessFile = new RandomAccessFile("D:\\tmp\\1.txt", "rw");
    FileChannel writeChannel = randomAccessFile.getChannel();) {
  	// 设置文件大小
    randomAccessFile.setLength(4096);
    // 创建一个内存映射
    MappedByteBuffer mbb = writeChannel.map(FileChannel.MapMode.READ_WRITE, 0, writeChannel.size());
    // 写入值
    mbb.put("hello world".getBytes());
    mbb.position(0);
    byte[] readBytes = new byte[3];
    // 读取
    mbb.get(readBytes);
    String readString = new String(readBytes);
    System.out.println("Read String: " + readString);
}
```

我们主要看map的时候做了什么，注意在Oracle JDK里面这部分源码是闭源的，在OpenJDK里面我们能看到对应的实现:

```java
// 省略无关代码
public MappedByteBuffer map(MapMode mode, long position, long size)
    throws IOException
{
    long addr = -1;
    int ti = -1;
    try {
        beginBlocking();
        ti = threads.add();
        if (!isOpen())
            return null;
        long mapSize;
        int pagePosition;
        synchronized (positionLock) {
            long filesize;
            do {
                filesize = nd.size(fd);
            } while ((filesize == IOStatus.INTERRUPTED) && isOpen());
            if (!isOpen())
                return null;

            if (filesize < position + size) { // Extend file size
                if (!writable) {
                    throw new IOException("Channel not open for writing " +
                        "- cannot extend file to required size");
                }
                int rv;
                do {
                    rv = nd.truncate(fd, position + size);
                } while ((rv == IOStatus.INTERRUPTED) && isOpen());
                if (!isOpen())
                    return null;
            }

            if (size == 0) {
                addr = 0;
                // a valid file descriptor is not required
                FileDescriptor dummy = new FileDescriptor();
                if ((!writable) || (imode == MAP_RO))
                    return Util.newMappedByteBufferR(0, 0, dummy, null);
                else
                    return Util.newMappedByteBuffer(0, 0, dummy, null);
            }
			// 
            pagePosition = (int)(position % allocationGranularity);
            long mapPosition = position - pagePosition;
            mapSize = size + pagePosition;
            try {
               	// mapPosition 是0 ,mapSize 是我们填入的大小              
                addr = map0(imode, mapPosition, mapSize);
            } catch (OutOfMemoryError x) {
                // An OutOfMemoryError may indicate that we've exhausted
                // memory so force gc and re-attempt map
                System.gc();
                try {
                    Thread.sleep(100);
                } catch (InterruptedException y) {
                    Thread.currentThread().interrupt();
                }
                try {
                    addr = map0(imode, mapPosition, mapSize);
                } catch (OutOfMemoryError y) {
                    // After a second OOME, fail
                    throw new IOException("Map failed", y);
                }
            }
        } // synchronized

        // On Windows, and potentially other platforms, we need an open
        // file descriptor for some mapping operations.
        FileDescriptor mfd;
        try {
            mfd = nd.duplicateForMapping(fd);
        } catch (IOException ioe) {
            unmap0(addr, mapSize);
            throw ioe;
        }

        assert (IOStatus.checkAll(addr));
        assert (addr % allocationGranularity == 0);
        int isize = (int)size;
        Unmapper um = new Unmapper(addr, mapSize, isize, mfd);
        if ((!writable) || (imode == MAP_RO)) {
            return Util.newMappedByteBufferR(isize,
                                             addr + pagePosition,
                                             mfd,
                                             um);
        } else {
            // 打断点是走到了这里
            return Util.newMappedByteBuffer(isize,
                                            addr + pagePosition,
                                            mfd,
                                            um);
        }
    } finally {
        threads.remove(ti);
        endBlocking(IOStatus.checkAll(addr));
    }
}
```

```java
 static MappedByteBuffer newMappedByteBuffer(int size, long addr,
                                                FileDescriptor fd,
                                                Runnable unmapper)
    {
        MappedByteBuffer dbb;
        if (directByteBufferConstructor == null)
            initDBBConstructor();
        try {
            dbb = (MappedByteBuffer)directByteBufferConstructor.newInstance(
              new Object[] { size,
                             addr,
                             fd,
                             unmapper });
        } catch (InstantiationException |
                 IllegalAccessException |
                 InvocationTargetException e) {
            throw new InternalError(e);
        }
        return dbb;
    }
```

这里的构造函数其实是java.nio.DirectByteBuffer的构造函数:

```java
protected DirectByteBuffer(int cap, long addr, FileDescriptor fd,Runnable unmapper)
```

这里简单的介绍一下FileDescriptor：

> Instances of the file descriptor class serve as an opaque handle to the underlying machine-specific structure representing an open file, an open socket, or another source or sink of bytes. The main practical use for a file descriptor is to create a FileInputStream or FileOutputStream to contain it.
> Applications should not create their own file descriptors.
>
> FileDescriptor的实例充当底层机器特定结构的匿名句柄，表示打开的文件、打开的套接字(socket)、字节源(RandomAccessFile)。在创建FileInputStream 或FileOutputStream 会包含它，应用程序不应该自己去操纵它。

### 浅浅总结一下

总结一下FileChannel在map的时候做了什么:

1. 发起一个mmap系统调用，这回返回一个地址.
2. 根据1返回的地址, 创建一个DirectByteBuffer对象返回出去，DirectByteBuffer是MappedByteBuffer的子类。

观察DirectByteBuffer的构造参数:

```java
protected DirectByteBuffer(int cap, long addr, FileDescriptor fd,Runnable unmapper)
```

![](https://a.a2k6.com/gerald/i/2024/09/01/303v.jpg)

```java
protected DirectByteBuffer(int cap, long addr, FileDescriptor fd,Runnable unmapper)
{
   super(-1, 0, cap, cap, fd);
   // 这个address 就是map系统调用返回的地址
   address = addr;
   cleaner = Cleaner.create(this, unmapper);
   att = null;
}
```

DirectByteBuffer的父类MappedByteBuffer构造函数如下图所示:

```java
MappedByteBuffer(int mark, int pos, int lim, int cap,FileDescriptor fd)
{
    super(mark, pos, lim, cap);
    this.fd = fd;
}
```

MappedByteBuffer的父类ByteBuffer构造函数如下所示:

```java
ByteBuffer(int mark, int pos, int lim, int cap) { // package-private
   // 第四个参数是字节数组,到现在来看这里并没有构造出一个字节数组出来 
   this(mark, pos, lim, cap, null, 0);
}
```

我们现在来看DirectByteBuffer在写的时候发生了什么:

```java
public ByteBuffer put(byte[] src, int offset, int length) {
        if (((long)length << 0) > Bits.JNI_COPY_FROM_ARRAY_THRESHOLD) {
            checkBounds(offset, length, src.length);
            int pos = position();
            int lim = limit();
            assert (pos <= lim);
            int rem = (pos <= lim ? lim - pos : 0);
            if (length > rem)
                throw new BufferOverflowException();
            long srcOffset = ARRAY_BASE_OFFSET + ((long)offset << 0);
            try {
                 // 在目标的地址上,算出偏移量,然后复制过去
                 UNSAFE.copyMemory(src, srcOffset,null,ix(pos),(long)length << 0);
            } finally {
                Reference.reachabilityFence(this);
            }
            position(pos + length);
        } else {
            super.put(src, offset, length);
        }
        return this;
}
private long ix(int i) {
   return address + ((long)i << 0);
}
```

所以mmap是首先向操作系统申请一个页面，这个时候操作系统并不会实际分配一个页面，只有在读取或写入的时候才会触发真正的分配页面，然后这个调用会返回一个地址，然后后面的写入操作在返回的地址上进行追加。

![](https://a.a2k6.com/gerald/i/2024/09/01/2bb.jpg)

我们再会看一下上面的话:

> mmap()函数在调用进程的虚拟地址空间中创建一个新的映射。新映射的起始地址在addr参数中指定。length参数指定映射的长度(必须大于0)

所以这份虚拟地址空间看起来并不占用应用进程的内存(后面的文章会分析，其实还是占用的)，只是请求操作系统分配一个虚拟页，向你返回地址。 写到这里我想起《我们来聊聊JVM的GC吧》里面提到:

> 在最底层，JVM 通过 mmap 接口向操作系统申请内存映射，每次申请 2MB 空间，这里是虚拟内存映射，不是真的就消耗了主存的 2MB，只有之后在使用的时候才会真的消耗内存。申请的这些内存放到一个链表中 VirtualSpaceList，作为其中的一个 Node。

在《用Java的BIO和NIO、Netty实现HTTP服务器(六) 从问题中来学习Netty》我们提到了可以通过Unsafe申请内存, 一般会像下面这样使用:

```java
Field field = Unsafe.class.getDeclaredField("theUnsafe");
// 正常使用不给我们用 
field.setAccessible(true);
Unsafe unsafe = (Unsafe)field.get(null);
// 申请1024个字节,在堆外分配
// 返回申请内存的首地址
long l = unsafe.allocateMemory(1024);
// 释放内存这段内存在堆外
unsafe.freeMemory(l);
```

让我们来重新审视内存，大多数现代计算机都将内存分割为字节, 每个字节可以存储8位的信息:

![](https://a.a2k6.com/gerald/i/2024/09/07/tm6o.jpg)

每个字节有唯一地址(address), 用来和内存中的其他字节相区别。如果内存中有n个字节，那么可以把地址看作0~n-1的数。

![](https://a.a2k6.com/gerald/i/2024/09/07/ug31.jpg)

可直接执行的程序由代码(C语言中与语句对应的机器指令)和数据(原始程序中的变量)两部分构成。程序中的每个变量占有一个或多个字节内存，把第一个字节的地址成为变量的地址。若某变量i占有地址为2000和2001的两个字节，所以变量i的地址是2000:

![](https://a.a2k6.com/gerald/i/2024/09/07/4eqo.jpg)

所谓指针就是内存地址的别名，我们可以通过内存地址来访问地址中的内容：

```c
int main(void){
    int i = 4;
    int *p = &i;
    printf("ptr指向的地址: %p\n", (void*)p);
    printf("%d\n",*p);
    return 0;
}
```

在《用Java的BIO和NIO、Netty实现HTTP服务器(六) 从问题中来学习Netty》里面我们也指出allocateMemory其实是调用了os的malloc函数，上面的操作在C语言里面对应的为:

```C
// void 表示暂时没有类型,可以将其转换成任意类型的指针。
void *address = malloc(1024);
```

### malloc vs mmap

在Linux上mmap调用的函数声明如下图所示:

```c
void *mmap(void *addr, size_t length, int prot, int flags, int fd, off_t > offset);  
```

让我产生好奇的是mmap调用和malloc都返回一个地址，我翻看了Linux上mmap调用的注释:

> mmap() creates a new mapping in the virtual address space of the   calling process.  The starting address for the new mapping is   specified in addr.  The length argument specifies the length of  the mapping (which must be greater than 0)。
>
> mmap()在调用进程的虚拟地址空间中创建一个新的映射。新映射的起始地址由addr参数指定。length参数指定了映射的长度(必须大于0)。
>
> After the mmap() call has returned, the file descriptor, fd,   can  be closed immediately without invalidating the mapping.
>
> 在mmap调用返回，文件描述符`fd`可以立即关闭,而不会使映射失效。
>
> If addr is NULL, then the kernel chooses the (page-aligned)  address at which to create the mapping; this is the most portable method of creating a new mapping. If addr is not NULL, then the kernel takes it as a hint about where to place the mapping; 
>
> 如果地址为空，则内核选择创建映射的(页面对齐的)地址;这是创建新映射的最可移植的方法。如果`addr`不为`NULL`,则内核将其作为放置映射位置的提示;
>
> onLinux, the kernel will pick a nearby page boundary (but always above or equal to the value specified by /proc/sys/vm/mmap_min_addr) and attempt to create the mapping there. 
>
> 在Linux上,内核会选择附近的页面边界(但总是大于或等于`/proc/sys/vm/mmap_min_addr`指定的值),并尝试在那里创建映射。
>
> If another mapping already exists there, the kernel picks a new address that may or may not depend on the hint. Theaddress of the new mapping is returned as the result of the call.
>
> 如果那里已经存在另一个映射,内核会选择一个新的地址,这个地址可能依赖于提示,也可能不依赖。新映射的地址作为调用的结果返回

malloc的注释是:

>   The malloc() function allocates size bytes and returns a pointer to the allocated memory.  The memory is not initialized.  If size
>   is 0, then malloc() returns a unique pointer value that can later  be successfully passed to free().  (See "Nonportable behavior"  for portability issues.)
>
>   malloc函数分配指定字节返回分配内存的指针。内存没有被初始化，如果传入的是0，返回一个唯一的指针值，这个值可以传递给后面的函数。返回的也是一个地址。

#### Windows下的mmap

但我在windows上写代码的，于是向看下windows的实现：

```java
JNIEXPORT jlong JNICALL
Java_sun_nio_ch_FileChannelImpl_map0(JNIEnv *env, jobject this,
                               jint prot, jlong off, jlong len)
{
    void *mapAddress = 0;
    jint lowOffset = (jint)off;
    jint highOffset = (jint)(off >> 32);
    jlong maxSize = off + len;
    jint lowLen = (jint)(maxSize);
    jint highLen = (jint)(maxSize >> 32);
    jobject fdo = (*env)->GetObjectField(env, this, chan_fd);
    HANDLE fileHandle = (HANDLE)(handleval(env, fdo));
    HANDLE mapping;
    DWORD mapAccess = FILE_MAP_READ;
    DWORD fileProtect = PAGE_READONLY;
    DWORD mapError;
    BOOL result;

    if (prot == sun_nio_ch_FileChannelImpl_MAP_RO) {
        fileProtect = PAGE_READONLY;
        mapAccess = FILE_MAP_READ;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_RW) {
        fileProtect = PAGE_READWRITE;
        mapAccess = FILE_MAP_WRITE;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_PV) {
        fileProtect = PAGE_WRITECOPY;
        mapAccess = FILE_MAP_COPY;
    }
    // 主要关注CreateFileMapping调用
    mapping = CreateFileMapping(
        fileHandle,      /* Handle of file */
        NULL,            /* Not inheritable */
        fileProtect,     /* Read and write */
        highLen,         /* High word of max size */
        lowLen,          /* Low word of max size */
        NULL);           /* No name for object */

    if (mapping == NULL) {
        JNU_ThrowIOExceptionWithLastError(env, "Map failed");
        return IOS_THROWN;
    }
	// MapViewOfFile调用
    mapAddress = MapViewOfFile(
        mapping,             /* Handle of file mapping object */
        mapAccess,           /* Read and write access */
        highOffset,          /* High word of offset */
        lowOffset,           /* Low word of offset */
        (DWORD)len);         /* Number of bytes to map */
    mapError = GetLastError();
    result = CloseHandle(mapping);
    if (result == 0) {
        JNU_ThrowIOExceptionWithLastError(env, "Map failed");
        return IOS_THROWN;
    }
    if (mapAddress == NULL) {
        if (mapError == ERROR_NOT_ENOUGH_MEMORY)
            JNU_ThrowOutOfMemoryError(env, "Map failed");
        else
            JNU_ThrowIOExceptionWithLastError(env, "Map failed");
        return IOS_THROWN;
    }
    return ptr_to_jlong(mapAddress);
}

```

这里我们可以看到主要调用为:

![](https://a.a2k6.com/gerald/i/2024/09/07/7w2oy.jpg)



CreateFileMapping的函数说明为: 为指定文件创建或打开命名或未命名的文件映射对象。如果函数成功，则返回值是新创建的文件映射对象的句柄。

MapViewOfFile 的函数说明为: 将文件映射的视图映射到调用进程的地址空间中。如果函数成功，则返回值是映射视图的起始地址。

#### Linux下的mmap实现

于是又看了下Linux的实现:

```c
JNIEXPORT jlong JNICALL
Java_sun_nio_ch_FileChannelImpl_map0(JNIEnv *env, jobject this,
                                     jint prot, jlong off, jlong len)
{
    void *mapAddress = 0;
    jobject fdo = (*env)->GetObjectField(env, this, chan_fd);
    jint fd = fdval(env, fdo);
    int protections = 0;
    int flags = 0;

    if (prot == sun_nio_ch_FileChannelImpl_MAP_RO) {
        protections = PROT_READ;
        flags = MAP_SHARED;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_RW) {
        protections = PROT_WRITE | PROT_READ;
        flags = MAP_SHARED;
    } else if (prot == sun_nio_ch_FileChannelImpl_MAP_PV) {
        protections =  PROT_WRITE | PROT_READ;
        flags = MAP_PRIVATE;
    }
    mapAddress = mmap64(
        0,                    /* Let OS decide location */
        len,                  /* Number of bytes to map */
        protections,          /* File permissions */
        flags,                /* Changes are shared */
        fd,                   /* File descriptor of mapped file */
        off);                 /* Offset into file */

    if (mapAddress == MAP_FAILED) {
        if (errno == ENOMEM) {
            JNU_ThrowOutOfMemoryError(env, "Map failed");
            return IOS_THROWN;
        }
        return handle(env, -1, "Map failed");
    }

    return ((jlong) (unsigned long) mapAddress);
}
```

调用的是mmap64，在64位机器上mmap64和mmap没有什么不同，引入mmap64是为了在32位系统上启用大文件支持, mmap和mmap在64位系统上没有区别。

#### 重新理解内存划分和内存分配问题

我们知道Hotspot 在启动的时候将使用的内存按照不同的用途分为堆、栈、本地方法栈、虚拟机栈，元空间。在上面我们看到Linux上mmap的注释:

> mmap()在调用进程的虚拟地址空间中创建一个新的映射。新映射的起始地址由addr参数指定。length参数指定了映射的长度(必须大于0)。

这个进程的虚拟地址空间让我感到对有些陌生，我对这个概念有些似懂非懂，结合JVM运行时区域划分，这个虚拟的地址空间该怎么理解呢，它在堆外吗?

然后我们再回忆一下操作系统对内存的管理:

> 操作系统以页面为单位管理内存，页面的大小通常为4096字节(但某些体系结构或操作系统使用更大的页面)，操作系统无法将小于一个页大小的内存空间分配给进程。假设你需要10个字节来存储一个字符串。分配4096个字节而只使用前10个字节是非常浪费的。内存分配器可以向操作系统请求一个页面，并将该页面分割成更小的分配

> 虚拟内存（英语：Virtual memory）是计算机系统内存管理的一种技术。它使得应用程序认为它拥有连续可用的内存（一个连续完整的地址空间），而实际上物理内存通常被分隔成多个内存碎片，还有部分暂时存储在外部磁盘存储器上，在需要时进行数据交换。与没有使用虚拟内存技术的系统相比，使用这种技术使得大型程序的编写变得更容易，对真正的物理内存（例如RAM）的使用也更有效率  《维基百科》

所以mmap也可以像malloc一样分配内存，这对应mmap的匿名映射:

```c
p = mmap(0, size, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0);
```

在《我们来聊聊JVM的GC吧》里面提到元空间就是通过mmap来向操作系统申请内存的。 在调用mmap进行文件映射的时候，是将进程虚拟内存空间中的某一段虚拟内存区域与磁盘中某个文件的某段区域进行映射。 当我们启动JVM的时候，JVM根据设定来使用glibc来初始化堆内存，之后由JVM的gc来管理这块内存。在参考文档[11]中我们可以看到现代进程虚拟内存空间的主要区域，有一块区域专门用来存储： 用于存放动态链接库以及内存映射区域的文件映射与匿名映射区。

![](https://a.a2k6.com/gerald/i/2024/09/08/6gys5.jpg)

注意上门的堆并不同于JVM中的堆，这里的堆泛指的是存储动态申请内存的地方。到现在我们已经心满意足的得到了我们想要的答案，我们创建的文件映射消耗的虚拟内存在进程里面会专门有一块地址空间存储。

### 回忆RandomAccessFile

这里我们简单回忆一下RandomAccessFile。RandomAccessFile, 从字面意思上意为随机读写，所谓随机读写就是将文件文件当作一个数组来看待，就像是一个大型的字节数组一样，在RandomAccessFile里面有文件指针的游标或索引，用于指向数组中的特定位置，读取操作从文件指针开始读取字节，并将文件指针推进到已读字节之后，如果RandomAccessFile 是在读写模式下创建的(也就是RandomAccessFile的第二个参数，read和write)，那么写操作也是可用的。写操作从文件指针开始写入字节，并将文件指针推进到已写入字节之后。 如果写操作写入的位置，超过了当前数组的末尾,则数组会被扩展。可以通过`getFilePointer`方法读取文件指针的当前位置,并通过`seek`方法设置文件指针的位置

### 对比普通的IO流读取写入

在web程序中一个比较普遍的场景是，获取磁盘的数据，然后将磁盘的数据通过Socket关联的流将数据传送出去:

![](https://a.a2k6.com/gerald/i/2024/09/01/r12q.jpg)



在上面的讨论中我们已经直到，数据的流动，应用程序首先从磁盘到达缓存管理器，然后从缓存到达进程里面，read调用会经历两次上下文切换, read发起用户态转内核态，read返回，内核态转用户态。这一动作通常由DMA完成，从磁盘读取内容到达系统缓冲区，然后再到达应用进程。这期间发生了两次复制，一次由磁盘到操作系统的缓存，然后再由操作系统的缓存复制到应用进程的内存。

然后通过Socket发起send调用，然后由用户态切换到内核态，这个时候send调用将数据同样放入操作系统的内核缓冲区，这个缓冲区与Socket相关。再次发生数据复制。接着send调用返回，内核态转用户态。然后由DMA将数据从内核缓冲区复制到协议引擎，产生第四次数据复制。在参考文档中[16] 中我们可以看到在X86上切换的成本为100ns，这意味着在上下文切换次数并不多的情况下减少上下文切换次数带来的性能提升比较小。

![](https://a.a2k6.com/gerald/i/2024/09/08/31lb.jpg)

对应的代码如下所示:

```java
public class ServerSocketChannelTest {
    private static final ExecutorService POOL = Executors.newFixedThreadPool(4);

    public static void main(String[] args) throws Exception {
        try (ServerSocketChannel serverSocketChannel = ServerSocketChannel.open()) {
            serverSocketChannel.bind(new InetSocketAddress(8080));
            // 不断的接收连接
            while (true) {
                SocketChannel socketChannel = serverSocketChannel.accept();
                POOL.execute(() -> processSocket(socketChannel));
            }
        }
    }
    private static void processSocket(SocketChannel socketChannel) {

        try(FileInputStream fileInputStream =  new FileInputStream("D:\\学习资料\\Flexcil_v1.1.6.14_.apk"); ) {
            int bytesRead = 0;
            ByteBuffer byteBuffer = ByteBuffer.allocate(4096);
            while (((bytesRead = fileInputStream.read(byteBuffer.array())) != -1)) {
                byteBuffer.limit(bytesRead);
                socketChannel.write(byteBuffer);
                byteBuffer.clear();
            }
            // 关闭资源
            fileInputStream.close();
            socketChannel.close();
        } catch (Exception e) {
            throw new RuntimeException(e);
        }
    }
}
```

```java
try (Socket socket = new Socket()) {
    socket.connect(new InetSocketAddress(8080));
    try (InputStream inputStream = socket.getInputStream();
         FileOutputStream fileOutputStream = new FileOutputStream("D:\\tmp\\Flexcil_v1.1.6.14_.apk");
    ) {
        byte[] array = new byte[4096];
        int readBytes = 0;
        while ((readBytes = inputStream.read(array)) != -1) {
            fileOutputStream.write(array, 0, readBytes);
        }
    }
}
```

### 用mmap方式来改写

现在让我们用mmap方式来改写一下, 我们只改processSocket中的方法:

```java
private static void processSocket(SocketChannel socketChannel) {
    try(FileInputStream fileInputStream =  new FileInputStream("D:\\学习资料\\Flexcil_v1.1.6.14_.apk"); ) {
        MappedByteBuffer mappedByteBuffer = fileInputStream.getChannel().map(FileChannel.MapMode.READ_ONLY, 0, 				fileInputStream.getChannel().size());
        socketChannel.write(mappedByteBuffer);
        socketChannel.close();
    } catch (Exception e) {
        throw new RuntimeException(e);
    }
}
```

我们在上面的文章讨论了，MappedByteBuffer最终是DirectByteBuffer，然后我们接着分析这个写入的流程:

```java
public int write(ByteBuffer buf) throws IOException {
    Objects.requireNonNull(buf);
    writeLock.lock();
    try {
        ensureOpenAndConnected();
        boolean blocking = isBlocking();
        int n = 0;
        try {
            beginWrite(blocking);
            if (blocking) {
                do {
                    // 核心代码也就是这里
                    n = IOUtil.write(fd, buf, -1, nd);
                } while (n == IOStatus.INTERRUPTED && isOpen());
            } else {
                n = IOUtil.write(fd, buf, -1, nd);
            }
        } finally {
            endWrite(blocking, n > 0);
            if (n <= 0 && isOutputClosed)
                throw new AsynchronousCloseException();
        }
        return IOStatus.normalize(n);
    } finally {
        writeLock.unlock();
    }
}
```

然后走到的IOUtil的write方法上:

```java
static int write(FileDescriptor fd, ByteBuffer src, long position,
                 boolean directIO, int alignment, NativeDispatcher nd)
    throws IOException
{	
	// 省略无关代码
    if (src instanceof DirectBuffer) {
        return writeFromNativeBuffer(fd, src, position, directIO, alignment, nd);
    }
}
```

```java
private static int writeFromNativeBuffer(FileDescriptor fd, ByteBuffer bb,
                                         long position, boolean directIO,
                                         int alignment, NativeDispatcher nd)
    throws IOException
{
    int pos = bb.position();
    int lim = bb.limit();
    assert (pos <= lim);
    int rem = (pos <= lim ? lim - pos : 0);

    if (directIO) {
        Util.checkBufferPositionAligned(bb, pos, alignment);
        Util.checkRemainingBufferSizeAligned(rem, alignment);
    }

    int written = 0;
    if (rem == 0)
        return 0;
    if (position != -1) {
        written = nd.pwrite(fd,
                            ((DirectBuffer)bb).address() + pos,
                            rem, position);
    } else {
        //  最后走到这里
        written = nd.write(fd, ((DirectBuffer)bb).address() + pos, rem);
    }
    if (written > 0)
        bb.position(pos + written);
}
```

###  其他创建mmap的方式

上面我们讲了从RandomAccessFile 获取MappedByteBuffer、从FileInputStream 获取MappedByteBuffer，其实都从FileChannel里面获取，我们也可以直接创建一个Channel：

```java
FileChannel fc =
        FileChannel.open( FileSystems.getDefault().getPath("目标paths"),
                StandardOpenOption.WRITE,
                StandardOpenOption.READ);
MappedByteBuffer mbb =
        fc.map(FileChannel.MapMode.READ_WRITE,
                0,          // position
                fc.size()); // size
```

### transferto

 

###  再来看RocketMQ的刷盘



## 总结一下







## 参考资料

[1] What they don’t tell you about demand paging in school  https://offlinemark.com/demand-paging/

[2]   malloc vs mmap in C https://stackoverflow.com/questions/1739296/malloc-vs-mmap-in-c

[3] What are void pointers for in C++? https://stackoverflow.com/questions/2860626/what-are-void-pointers-for-in-c
[4] Is there any difference between mmap vs mmap64? https://stackoverflow.com/questions/59453555/is-there-any-difference-between-mmap-vs-mmap64

[5] CreateFileMappingW https://learn.microsoft.com/zh-cn/windows/win32/api/memoryapi/nf-memoryapi-createfilemappingw?redirectedfrom=MSDN

[6]  Why is malloc() considered a library call and not a system call? https://stackoverflow.com/questions/71413587/why-is-malloc-considered-a-library-call-and-not-a-system-call

[7] Is there any difference between mmap vs mmap64?  https://stackoverflow.com/questions/59453555/is-there-any-difference-between-mmap-vs-mmap64 

[8] Why is malloc() considered a library call and not a system call? https://stackoverflow.com/questions/71413587/why-is-malloc-considered-a-library-call-and-not-a-system-call

[9] How to use mmap to allocate a memory in heap?  https://stackoverflow.com/questions/4779188/how-to-use-mmap-to-allocate-a-memory-in-heap

[10]  mmap 内存映射，是越过了操作系统，直接通过内存访问文件吗？  https://www.zhihu.com/question/522132580/answer/3241695059

[11] 一步一图带你深入理解 Linux 虚拟内存管理  https://mp.weixin.qq.com/s?__biz=Mzg2MzU3Mjc3Ng==&mid=2247486732&idx=1&sn=435d5e834e9751036c96384f6965b328&chksm=ce77cb4bf900425d33d2adfa632a4684cf7a63beece166c1ffedc4fdacb807c9413e8c73f298&scene=178&cur_album_id=2559805446807928833#rd

[12] 大量类加载器创建导致诡异FullGC  https://heapdump.cn/article/1924890

[13]  JDK-8268893  https://bugs.openjdk.org/browse/JDK-8268893

[14] Temurin™ Supported Platforms  https://adoptium.net/zh-CN/supported-platforms/

[15] glibc  https://sourceware.org/glibc/wiki/MallocInternals

[16] Why is malloc() considered a library call and not a system call? https://stackoverflow.com/questions/71413587/why-is-malloc-considered-a-library-call-and-not-a-system-call

[17] Chapter 16: The Page Cache and Page Writeback https://github.com/firmianay/Life-long-Learner/blob/master/linux-kernel-development/chapter-16.md

[18] Why does not Redis use linux zero-copy syscall api ? https://github.com/redis/redis/issues/12682

[19]   一步一图带你深入理解 Linux 物理内存管理    https://www.cnblogs.com/binlovetech/p/16914715.html

[20] 文件IO原理及Kafka高效读写原因分析 https://www.eula.club/blogs/%E6%96%87%E4%BB%B6IO%E5%8E%9F%E7%90%86%E5%8F%8AKafka%E9%AB%98%E6%95%88%E8%AF%BB%E5%86%99%E5%8E%9F%E5%9B%A0%E5%88%86%E6%9E%90.html#_1-%E5%89%8D%E8%A8%80
