# 从RocketMQ的持久化谈起

[TOC]

##  探秘字节流

在《重学RocketMQ之深化理解与实践思考(一)  架构与消息》中我们讲到了刷盘，也就将数据从内存刷新到磁盘上，也就是持久化，提到持久化这里想到了Redis的持久化，MySQL的持久化。我想到了字节流和字符流、FileInputStream、FileOutputStream、BufferedOutputStream、BufferedInputStream、零拷贝。

程序使用字节进行读取和写入的就是字节流，常见的就有FileInputStream、FileOutputStream。所有的字节流都继承自 InputStream and OutputStream。一般我们向文件里面写内容，使用的是FileOutputStream的write方法:

```java
public void write(byte b[])
```

然后write方法底层是一个native调用:

```java
private native void writeBytes(byte b[], int off, int len, boolean append)
```

这里的native实现，我们看windows下面的调用，对应的实现是FileOutputStream_md.c，文件路径为: jdk/src/windows/native/java/io/FileOutputStream_md.c

 在OpenJDK 8(jdk8-b115,后文不做说明, 统一在这个版本下讨论问题)的FileOutputStream_md.c我们可以看到对应的调用:

```c
JNIEXPORT void JNICALL
Java_java_io_FileOutputStream_writeBytes(JNIEnv *env,
    jobject this, jbyteArray bytes, jint off, jint len, jboolean append) {
    writeBytes(env, this, bytes, off, len, append, fos_fd);
}
```

在io_util.c(文件路径为jdk/src/share/native/java/io/io_util.c)可以看到对应的writeByes调用

```java
void
writeBytes(JNIEnv *env, jobject this, jbyteArray bytes,
           jint off, jint len, jboolean append, jfieldID fid)
{	// 省略无关代码调用
    if (!(*env)->ExceptionOccurred(env)) {
  
            if (append == JNI_TRUE) {
                n = IO_Append(fd, buf+off, len);
            } else {
                n = IO_Write(fd, buf+off, len);
            }       
            off += n;
            len -= n;
        }
    }
    if (buf != stackBuf) {
        free(buf);
    }
}
```

这里的IO_write是一个宏，在io_util_md.h(文件目标路径为:jdk/src/windows/native/java/io/io_util_md.c)里面可以看到对应的宏定义:

```c
#define IO_Write handleWrite
```

在io_util_md.h(jdk/src/windows/native/java/io/io_util_md.h)我们可以看到handleWrite的定义:

```c
JNIEXPORT
jint handleWrite(jlong fd, const void *buf, jint len) {
    return writeInternal(fd, buf, len, JNI_FALSE);
}
static jint writeInternal(jlong fd, const void *buf, jint len, jboolean append)
{
    BOOL result = 0;
    DWORD written = 0;
    HANDLE h = (HANDLE)fd;
    if (h != INVALID_HANDLE_VALUE) {
        OVERLAPPED ov;
        LPOVERLAPPED lpOv;
        if (append == JNI_TRUE) {
            ov.Offset = (DWORD)0xFFFFFFFF;
            ov.OffsetHigh = (DWORD)0xFFFFFFFF;
            ov.hEvent = NULL;
            lpOv = &ov;
        } else {
            lpOv = NULL;
        }
        // 
        result = WriteFile(h,                /* File handle to write */
                           buf,              /* pointers to the buffers */
                           len,              /* number of bytes to write */
                           &written,         /* receives number of bytes written */
                           lpOv);            /* overlapped struct */
    }
    if ((h == INVALID_HANDLE_VALUE) || (result == 0)) {
        return -1;
    }
    return (jint)written;
}
```

在参考文档[6] 里面可以看到，由于缺少了LPOVERLAPPED 参数，这是一个同步调用，但是也不代表会立即刷新到磁盘中, 这是因为内存到磁盘太慢了，默认情况下，Windows 缓存从磁盘读取的和写入到磁盘的文件数据。 这意味着读取操作从系统内存中称为系统文件缓存的区域读取文件数据，而不是从物理磁盘读取文件数据。 相应地，写入操作将文件数据写入系统文件缓存，而不是写入磁盘。这类缓存称为回写缓存。 缓存按文件对象进行管理(见参考文档[7])。

FileOutputStream的构造函数，对应到FileOutputStream_md.c(jdk/src/windows/native/java/io/FileOutputStream_md.c)的

```c
NIEXPORT void JNICALL
Java_java_io_FileOutputStream_open(JNIEnv *env, jobject this,
                                   jstring path, jboolean append) {
    fileOpen(env, this, path, fos_fd,
             O_WRONLY | O_CREAT | (append ? O_APPEND : O_TRUNC));
}
```

fileOpen函数的实现在io_util_md.c(jdk/src/windows/native/java/io/io_util_md.c)中:

```c
void fileOpen(JNIEnv *env, jobject this, jstring path, jfieldID fid, int flags)
{
    jlong h = winFileHandleOpen(env, path, flags);
    if (h >= 0) {
        SET_FD(this, h, fid);
    }
}
```

```c
jlong winFileHandleOpen(JNIEnv *env, jstring path, int flags)
{
    const DWORD access =
        (flags & O_WRONLY) ?  GENERIC_WRITE :
        (flags & O_RDWR)   ? (GENERIC_READ | GENERIC_WRITE) :
        GENERIC_READ;
    const DWORD sharing =
        FILE_SHARE_READ | FILE_SHARE_WRITE;
    const DWORD disposition =
        /* Note: O_TRUNC overrides O_CREAT */
        (flags & O_TRUNC) ? CREATE_ALWAYS :
        (flags & O_CREAT) ? OPEN_ALWAYS   :
        OPEN_EXISTING;
    const DWORD  maybeWriteThrough =
        (flags & (O_SYNC | O_DSYNC)) ?
        FILE_FLAG_WRITE_THROUGH :
        FILE_ATTRIBUTE_NORMAL;
    const DWORD maybeDeleteOnClose =
        (flags & O_TEMPORARY) ?
        FILE_FLAG_DELETE_ON_CLOSE :
        FILE_ATTRIBUTE_NORMAL;
    const DWORD flagsAndAttributes = maybeWriteThrough | maybeDeleteOnClose;
    HANDLE h = NULL;

    if (onNT) {
        WCHAR *pathbuf = pathToNTPath(env, path, JNI_TRUE);
        if (pathbuf == NULL) {
            /* Exception already pending */
            return -1;
        }
        h = CreateFileW(
            pathbuf,            /* Wide char path name */
            access,             /* Read and/or write permission */
            sharing,            /* File sharing flags */
            NULL,               /* Security attributes */
            disposition,        /* creation disposition */
            flagsAndAttributes, /* flags and attributes */
            NULL);
        free(pathbuf);
    } else {
        WITH_PLATFORM_STRING(env, path, _ps) {
            h = CreateFile(_ps, access, sharing, NULL, disposition,
                           flagsAndAttributes, NULL);
        } END_PLATFORM_STRING(env, _ps);
    }
    if (h == INVALID_HANDLE_VALUE) {
        int error = GetLastError();
        if (error == ERROR_TOO_MANY_OPEN_FILES) {
            JNU_ThrowByName(env, JNU_JAVAIOPKG "IOException",
                            "Too many open files");
            return -1;
        }
        throwFileNotFoundException(env, path);
        return -1;
    }
    return (jlong) h;
}
```



### Page Cache VS 缓存管理器

在对应的函数中我们可以看到应该是开启了缓存(应该表示推测，那段代码适配了windows的多个版本，见在参考文档[9] )，也就是说写入到磁盘的数据首先到操作系统缓冲区，然后再由缓冲管理器刷新到磁盘中:

![](https://a.a2k6.com/gerald/i/2024/08/25/aaag6f.jpg)

读取操作从系统内存中的某个区域（称为系统文件缓存）读取文件数据，而不是从物理磁盘读取。

参考文档[7]这么说道:

> 某些应用程序（如病毒检查软件）要求其写入操作立即刷新到磁盘;Windows 通过写通缓存提供此功能。 进程通过将 FILE_FLAG_WRITE_THROUGH 标志传递到对 CreateFile 的调用中，为特定 I/O 操作启用写通缓存。 启用写通缓存后，数据仍会写入缓存，但缓存管理器会立即将数据写入磁盘，而不会因使用延迟编写器而产生延迟。 进程还可以通过调用 FlushFileBuffers 函数强制刷新已打开的文件。

上面Windows的缓存管理器让我想起来Linux的page cache(见参考文档[9])：

> The Linux kernel implements a disk cache called the `page cache`. The goal of this cache is to minimize disk I/O by storing data in physical memory that would otherwise require disk access.
>
> Linux内核实现了磁盘缓存称之为页面缓存，设计目标是通过将数据存储在物理内存中来减少磁盘I/O。

操作系统的思路都是相似的，大同小异。

### 探秘缓冲流

在oracle写的教程中《**The Java™ Tutorials** 》(见参考文档8)中对缓冲流是这么说道:

> Most of the examples we've seen so far use *unbuffered* I/O. This means each read or write request is handled directly by the underlying OS. This can make a program much less efficient, since each such request often triggers disk access, network activity, or some other operation that is relatively expensive.
>
> 到目前为止，我们看到的大多数例子都使用了*非缓冲*I/O。这意味着每个读取或写入请求都直接由底层操作系统处理。这可能使程序效率大大降低，因为每个这样的请求通常会触发磁盘访问、网络活动或其他相对昂贵的操作。
>
> To reduce this kind of overhead, the Java platform implements *buffered* I/O streams. Buffered input streams read data from a memory area known as a *buffer*; the native input API is called only when the buffer is empty. Similarly, buffered output streams write data to a buffer, and the native output API is called only when the buffer is full.
>
> 为了减少这种开销，Java平台实现了*缓冲*I/O流。缓冲输入流从称为*缓冲区*的内存区域读取数据；只有当缓冲区为空时，才会调用原生输入API。类似地，缓冲输出流将数据写入缓冲区，只有当缓冲区满时，才会调用原生输出API。

在我刚学IO流的时候，老师在讲这个缓冲流的时候，说道你就将这个理解为用了一辆小推车去运输数据，所以快了很多，这种解释并不接近本质，如果沿用这种思路去解释原理，那么再遇到零拷贝的时候，那解释是不是要上航天飞机，现在我们直接从源码的级别去看缓冲流的设计思路:

```java
protected byte buf[];

public BufferedOutputStream(OutputStream out) {
    this(out, 8192);
}
```

```java
public BufferedOutputStream(OutputStream out, int size) {
    super(out);
    if (size <= 0) {
        throw new IllegalArgumentException("Buffer size <= 0");
    }
    buf = new byte[size];
}
```

```java
public synchronized void write(byte b[], int off, int len) throws IOException {
    if (len >= buf.length) {
        /* If the request length exceeds the size of the output buffer,
           flush the output buffer and then write the data directly.
           In this way buffered streams will cascade harmlessly. */
        flushBuffer();
        out.write(b, off, len);
        return;
    }
    if (len > buf.length - count) {
        flushBuffer();
    }
    System.arraycopy(b, off, buf, count, len);
    count += len;
}
```

从源码中我们可以看到， 使用缓冲流再刷数据的时候，BufferedOutputStream内置了一个字节数组，写入的时候，首先判断传入的字节数组要写入的大小超没超过内置字节数组的大小，如果超过了，先将内置的字节数组通过系统调用刷到系统缓存里面，然后再将传入的字节数组写入。如果内置的字节数组剩余的容量小于要写入的字节大小，则刷缓存，然后将传入的数组复制到内置的缓冲区中。这种设计思路有点像是，单个插入转批量插入。在《译: 通过零拷贝实现高效数据传输》我们提到，一些Web应用提供大量的静态资源，这相当于从磁盘读取大量数据，然后通过Socket将这些数据回写，在Java中也就就是SocketOutputStream的write方法, 

```java
// 将数据从缓冲区刷新到磁盘上
public synchronized void flush() throws IOException 
public void write(byte b[], int off, int len) throws IOException {
    socketWrite(b, off, len);
}
```

```java
private void socketWrite(byte b[], int off, int len) throws IOException {


    if (len <= 0 || off < 0 || len > b.length - off) {
        if (len == 0) {
            return;
        }
        throw new ArrayIndexOutOfBoundsException("len == " + len
                + " off == " + off + " buffer length == " + b.length);
    }

    FileDescriptor fd = impl.acquireFD();
    try {
        socketWrite0(fd, b, off, len);
    } catch (SocketException se) {
        if (se instanceof sun.net.ConnectionResetException) {
            impl.setConnectionResetPending();
            se = new SocketException("Connection reset");
        }
        if (impl.isClosedOrPending()) {
            throw new SocketException("Socket closed");
        } else {
            throw se;
        }
    } finally {
        impl.releaseFD();
    }
}
```

从磁盘中读取数据我们通常用输入流，也就是FileInputStream的read方法：

```java
public int read(byte b[]) throws IOException {
    return readBytes(b, 0, b.length);
}
private native int readBytes(byte b[], int off, int len) throws IOException;
```

在这种情况下数据先从磁盘到应用进程中，然后通过SocketInputStream写入到目的地，应用进程充当了无效的媒介:

![](https://a.a2k6.com/gerald/i/2024/08/31/6664u.jpg)

在发起读取和写入的时候还要经过内核态到用户态之间的切换，在文件比较大的时候，这种切换加上数据移动造成效率低下，那么能否将磁盘的数据直接传输到目的地，数据不再经过应用进程，而只是由应用进程在磁盘和目的地之间建立联系，指明要传输的文件和传输的目的地。

## 零拷贝

事实上在JDK下面还有更高效的复制文件的方式，这两种方式都依赖于操作系统，也就是mmap和transferTo，我们将分别介绍这两种文件复制方式和其高效的原理。

### mmap简介

所谓mmap是Memory-mapped 的缩写也就是内存映射,在Linux中对mmap这个方法的介绍如下:

> mmap()函数在调用进程的虚拟地址空间中创建一个新的映射。新映射的起始地址在addr参数中指定。length参数指定映射的长度(必须大于0)

所谓映射是一种对应关系，定义函数的时候用到了映射:

> 设A,B是两个非空集合，若对A中的任一元素x，依照某种规律或法则f，恒有B中唯一确定的元素y与之对应，则称此对应规律或法则f为一个从A到B的映射。

那么这么来看我们需要进程的虚拟地址空间开辟一段区域和我们需要操作的文件进行映射，那这样我就有一个疑问，我认为开辟区域的这块内存和文件大小是一对一的，也就是说操纵的文件有一个字节，在进程的地址空间也同样需要一个字节，那这样会很消耗内存，如果是操作大文件。实际中，操作系统采取了一种延迟(lazy)分配的方式，当你通过mmap调用一个匿名的页面，内核会狡猾的立即返回一个指针。然后它会等待,直到你通过"触碰"该内存而触发一个缺页异常(page fault),才会进行真正的内存分配工作。这种机制被称为"按需分页"(demand paging)。

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

所以这份虚拟地址空间并不占用应用进程的内存，只是请求操作系统分配一个虚拟页，向你返回地址。 写到这里我想起《我们来聊聊JVM的GC吧》里面提到:

> 在最底层，JVM 通过 mmap 接口向操作系统申请内存映射，每次申请 2MB 空间，这里是虚拟内存映射，不是真的就消耗了主存的 2MB，只有之后在使用的时候才会真的消耗内存。申请的这些内存放到一个链表中 VirtualSpaceList，作为其中的一个 Node。

在《用Java的BIO和NIO、Netty实现HTTP服务器(六) 从问题中来学习Netty》我们提到了可以通过Unsafe申请内存, 一般会像下面这样使用:





### 回忆RandomAccessFile

这里我们简单回忆一下RandomAccessFile。RandomAccessFile, 从字面意思上意为随机读写，所谓随机读写就是将文件文件当作一个数组来看待，就像是一个大型的字节数组一样，在RandomAccessFile里面有文件指针的游标或索引，用于指向数组中的特定位置，读取操作从文件指针开始读取字节，并将文件指针推进到已读字节之后，如果RandomAccessFile 是在读写模式下创建的(也就是RandomAccessFile的第二个参数，read和write)，那么写操作也是可用的。写操作从文件指针开始写入字节，并将文件指针推进到已写入字节之后。 如果写操作写入的位置，超过了当前数组的末尾,则数组会被扩展。可以通过`getFilePointer`方法读取文件指针的当前位置,并通过`seek`方法设置文件指针的位置

### 对比普通的IO流读取写入

在web程序中一个比较普遍的场景是，获取磁盘的数据，然后将磁盘的数据通过Socket关联的流将数据传送出去:

![](https://a.a2k6.com/gerald/i/2024/09/01/r12q.jpg)



在上面的讨论中我们已经直到，数据的流动，应用程序首先从磁盘到达缓存管理器，然后从缓存到达进程里面，read调用会经历两次上下文切换, read发起用户态转内核态，read返回，内核态转用户态。这一动作通常由DMA完成，从磁盘读取内容到达系统缓冲区，然后再到达应用进程。这期间发生了两次复制，一次由磁盘到操作系统的缓存，然后再由操作系统的缓存复制到应用进程的内存。

然后通过Socket发起send调用，然后由用户态切换到内核态，这个时候send调用将数据同样放入操作系统的内核缓冲区，这个缓冲区与Socket相关。再次发生数据复制。接着send调用返回，内核态转用户态。然后由DMA将数据从内核缓冲区复制到协议引擎，产生第四次数据复制。











如果我们用mmap呢，我们首发起系统调用向操作系统申请一块区域和文件进行关联，这会返回一个地址，我们之后的读取或写入都是在这个地址上进行操作。这会产生一次内核态到用户态的转换，一次用户态到内核态的转换，接着我们发起read调用，DMA会将数据从操作系统的缓冲区读取到mmap对应的内存区域。

###  其他创建mmap的方式







### transferto



###  再来看RocketMQ的刷盘







### 到Redis 的刷盘



### 到MySQL 的刷盘





## 总结一下







## 参考资料

[1] What they don’t tell you about demand paging in school  https://offlinemark.com/demand-paging/
