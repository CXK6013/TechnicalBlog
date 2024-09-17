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

### Windows下面的实现

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
voidwriteBytes(JNIEnv *env, jobject this, jbyteArray bytes,
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
JNIEXPORT jint handleWrite(jlong fd, const void *buf, jint len) {
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

windows下面的调用链路图如下所示:

![](https://a.a2k6.com/gerald/i/2024/09/16/6kdfy.jpg)

所谓系统调用是一种特殊的函数(一般由操作系统提供)，它允许你跨越保护域。当一个程序在用户态(用户模式下)执行的时候，它不被允许执行在内核态下面运行的代码所允许的操作。例如在用户态下面的程序无法在没有内核帮助的情况下读取文件。当用户程序向操作系统请求服务的时候，系统通过系统调用保护自己不受恶意或有缺陷程序的影响。系统调用执行一条特殊的硬件指令，通常成为“陷阱”，将控制权转移到内核，然后内核决定是否满足该请求。

###  缓存管理器

在对应的函数中我们可以看到应该是开启了缓存(应该表示推测，那段代码适配了windows的多个版本，见在参考文档[9] )，也就是说写入到磁盘的数据首先到操作系统缓冲区，然后再由缓冲管理器刷新到磁盘中:

![](https://a.a2k6.com/gerald/i/2024/08/25/aaag6f.jpg)

读取操作从系统内存中的某个区域（称为系统文件缓存）读取文件数据，而不是从物理磁盘读取。

参考文档[7]这么说道:

> 某些应用程序（如病毒检查软件）要求其写入操作立即刷新到磁盘;Windows 通过写通缓存提供此功能。 进程通过将 FILE_FLAG_WRITE_THROUGH 标志传递到对 CreateFile 的调用中，为特定 I/O 操作启用写通缓存。 启用写通缓存后，数据仍会写入缓存，但缓存管理器会立即将数据写入磁盘，而不会因使用延迟编写器而产生延迟。 进程还可以通过调用 FlushFileBuffers 函数强制刷新已打开的文件。

上面Windows的缓存管理器让我想起来Linux的page cache(见参考文档[9])：

> The Linux kernel implements a disk cache called the `page cache`. The goal of this cache is to minimize disk I/O by storing data in physical memory that would otherwise require disk access.
>
> Linux内核实现了磁盘缓存称之为页面缓存，设计目标是通过将数据存储在物理内存中来减少磁盘I/O。

### Linux下面的实现

在Linux下面对应的实现在jdk/src/solaris/native/java/io/FileOutputStream_md.c中:

```c
JNIEXPORT void JNICALL
Java_java_io_FileOutputStream_writeBytes(JNIEnv *env,
    jobject this, jbyteArray bytes, jint off, jint len, jboolean append) {
    writeBytes(env, this, bytes, off, len, append, fos_fd);
}
```

对应的实现是(jdk/src/share/native/java/io/io_util.c)

```c
#define BUF_SIZE 8192
void writeBytes(JNIEnv *env, jobject this, jbyteArray bytes,
           jint off, jint len, jboolean append, jfieldID fid)
{		
	// 省略无关代码
    if (!(*env)->ExceptionOccurred(env)) {
        off = 0;
        while (len > 0) {
            fd = GET_FD(this, fid);
            if (fd == -1) {
                JNU_ThrowIOException(env, "Stream Closed");
                break;
            }
            if (append == JNI_TRUE) {
                n = IO_Append(fd, buf+off, len);
            } else {
                n = IO_Write(fd, buf+off, len);
            }
            if (n == JVM_IO_ERR) {
                JNU_ThrowIOExceptionWithLastError(env, "Write error");
                break;
            } else if (n == JVM_IO_INTR) {
                JNU_ThrowByName(env, "java/io/InterruptedIOException", NULL);
                break;
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

在jdk/src/solaris/native/java/io/io_util_md.h看到这个IO_Write也是一个宏定义：

```C
#define IO_Write JVM_Write
```

这个JVM_Write在hotspot的JVM.cpp中:

```c
JVM_LEAF(jint, JVM_Write(jint fd, char *buf, jint nbytes))
    JVMWrapper2("JVM_Write (0x%x)", fd);
    //%note jvm_r6
    return (jint)os::write(fd, buf, nbytes);
JVM_END
```

对应Linux 的实现在(hotspot/src/os/linux/vm/os_linux.inline.hpp)中:

```c++
#include <unistd.h>
inline size_t os::write(int fd, const void *buf, unsigned int nBytes) {
  size_t res;
  RESTARTABLE((size_t) ::write(fd, buf, (size_t) nBytes), res);
  return res;
}
```

这里其实调用的是Linux的write函数, Linux中write的调用说明为:

> write() writes up to count bytes from the buffer starting at buf  to the file referred to by the file descriptor fd.
>
> - `write()` 函数尝试将最多 `count` 个字节从 `buf` 指向的缓冲区写入到文件描述符 `fd` 所引用的文件中。

> A successful return from write() does not make any guarantee that data has been committed to disk. On some filesystems, including NFS, it does not even guarantee that space has successfully been reserved for the data. In this case, some errors might be delayed until a future write(), fsync(2), or even close(2). 
>
> 成功调用并不承诺数据到达磁盘，在一些文件系统上，包括NFS，甚至不保证已为数据成功预留了空间。某些错误可能会延迟到未来的 `write()`、`fsync(2)` 或甚至 `close(2)` 调用时才显现。

> The only way to be sure is to call fsync(2) after you are done writing all your data.
>
> 确保数据已写入磁盘的唯一方法是在完成所有数据写入后调用 `fsync(2)`

这里有两个问题，第一个问题在Java中如何将数据确保刷到磁盘上，第二个问题既然没刷到磁盘，这个数据在调用返回之后在哪个位置。

### 如何将数据确保刷新到磁盘上

- 方式一

```java
try(FileOutputStream fileOutputStream = new FileOutputStream("D:\\学习资料\\SDK\\1.txt");){
    fileOutputStream.write("hello world".getBytes());
    fileOutputStream.getFD().sync();
}catch (Exception e) {
    e.printStackTrace();
}
```

​	最终调用的是FileDescriptor的sync方法，这同样也是一个系统调用：

```c
public native void sync() throws SyncFailedException;
```

对应的实现位于jdk/src/solaris/native/java/io/FileDescriptor_md.c下面:

```c
JNIEXPORT void JNICALL
Java_java_io_FileDescriptor_sync(JNIEnv *env, jobject this) {
    int fd = (*env)->GetIntField(env, this, IO_fd_fdID);
    if (JVM_Sync(fd) == -1) {
        JNU_ThrowByName(env, "java/io/SyncFailedException", "sync failed");
    }
}
```

JVM_syn的实现在hotspot/src/share/vm/prims/jvm.cpp:

```c++
JVM_LEAF(jint, JVM_Sync(jint fd))
  JVMWrapper2("JVM_Sync (0x%x)", fd);
  //%note jvm_r6
  return os::fsync(fd);
JVM_END
```

#### Linux上的实现

 然后分配到对应操作系统的调用上，在Linux位于:hotspot/src/os/linux/vm/os_linux.inline.hpp

```
inline int os::fsync(int fd) {
  return ::fsync(fd);
}
```

fsync是一个Linux系统调用将对文件的更改写入内容刷新到磁盘上，由此这让我想起了Redis的AOF持久化机制，那么也要通过Linux的系统调用来将内容刷新到磁盘上，

#### Windows的实现

在windows上的实现为:

```c
// This code is a copy of JDK's sysSync
// from src/windows/hpi/src/sys_api_md.c
// except for the legacy workaround for a bug in Win 98

int os::fsync(int fd) {
  HANDLE handle = (HANDLE)::_get_osfhandle(fd);

  if ( (!::FlushFileBuffers(handle)) &&
         (GetLastError() != ERROR_ACCESS_DENIED) ) {
    /* from winerror.h */
    return -1;
  }
  return 0;
}
```

FlushFileBuffers 是windows的函数作用为将缓冲区的内容刷新到磁盘上。

#### 通过Channel刷



## 探秘缓冲流

在上面的调用中假如我们频繁写入，就会经历多次上下文切换，每次上下文切换都需要成本，参考文档中[16] 中我们可以看到在X86上切换的成本为100ns, 除了这个成本以外，如果每次写入的字节比较小，这就像一辆车没有满载，对系统调用的利用率就不够高。那我们能不能像快递站一样到一个快递就开始派送，而是快递员装满某个小区的快递之后才开始配送。这也就是缓冲流的设计思路。

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

频繁申请内存

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
