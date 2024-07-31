# 文件描述符管理

## 常用宏



### 进程文件描述符管理

#### fdtable 分配

```
struct fdtable * fdtable_alloc(){
    struct fdtable * result = kmalloc(PGSIZE);
    memset(result, 0, PGSIZE);
    result[STDIN].fds_index = STDIN;
    result[STDIN].busy = 1;
    result[STDOUT].fds_index = STDOUT;
    result[STDOUT].busy = 1;
    result[STDERR].fds_index = STDERR;
    result[STDERR].busy = 1;
    // for ( int i = STDERR + 1 ; i < MAX_FDTABLE ; i ++ ){
        // result[i].free = 1;
    // }
    return result;
}
```

这段代码定义了一个函数 `fdtable_alloc`，用于分配和初始化文件描述表。主要步骤包括：

1. **分配内存**：使用 `kmalloc` 分配一页（通常是4KB）的内存。
2. **清零内存**：使用 `memset` 将分配的内存清零，确保所有条目初始为空。
3. **初始化标准描述符**：设置标准输入（`STDIN`）、标准输出（`STDOUT`）和标准错误（`STDERR`）的文件描述符索引，并标记为忙碌状态。
4. **返回结果**：返回指向分配并初始化的文件描述表的指针。

注释部分代码用于将其他文件描述符标记为空闲，未实际执行。如果需要，可以取消注释并完善逻辑。

#### fdtable 复制

```c
struct fdtable * fdtable_copy(struct fdtable * fdt){
    struct fdtable * result = kmalloc(PGSIZE);
    for ( int i = 0 ; i < MAX_FDTABLE ; i ++ ){
        if ( fdt[i].busy == 1 ) fds[fdt[i].fds_index].refcount += 1;
    }
    memcpy(result, fdt, PGSIZE);
    return result;
}
``` 

`fdtable_copy` 函数用于复制一个文件描述表。它首先分配一页内存用于新的文件描述表，然后遍历原表，如果发现条目处于忙碌状态，则增加对应文件描述符的引用计数，最后将原表的内容复制到新表并返回新表的指针。

## 管道管理

### 管道定义与基础操作

#### 初始化管道

```c
void init_pipes(){
    memset(pipes, 0, sizeof(pipes));
}
```

`init_pipes` 函数用于将管道数组 `pipes` 中的所有元素设置为零，从而初始化管道结构。这样可以确保管道数组在使用前处于已知的初始状态。

#### 新建管道

```c
struct pipe * alloc_pipe(){
    for ( int i = 0 ; i < MAX_PIPES ; i ++ ){
        if ( pipes[i].alloc == 0 ){
            pipes[i].alloc = 1;
            pipes[i].pipe_mem = kmalloc(PGSIZE);
            memset(pipes[i].pipe_mem, 0, PGSIZE);
            pipes[i].readp = 0;
            pipes[i].writep = 0;
            return &pipes[i];
        }
    }
    return -1;
}
```

`alloc_pipe` 函数用于分配并初始化一个管道结构。它遍历管道数组 `pipes`，找到一个未被分配的管道，将其标记为已分配，分配一页内存给管道的缓冲区并清零，然后初始化读写指针（`readp` 和 `writep`）为0，最后返回指向该管道的指针。如果所有管道都已被分配，则返回错误指示符（`-1`）。

### pipe2 系统调用

```c
int pipe(int * fd, uint64_t flags){
    int fd1 = -1;
    int fd2 = -1;
    struct proc * proc = my_proc();
    for (int i = 0; i < MAX_FDTABLE; i++) {
        // Log("proc->fdtable[%d].busy = %d\n", i, proc->fdtable[i].busy);
        if ( proc->fdtable[i].busy == 0 ){
            // Log("fd1 = %d; fd2 = %d\n", fd1, fd2);
            if ( fd2 == -1 ){
                if ( fd1 == -1 ) {
                    fd1 = i;
                } else {
                    fd2 = i;
                }
            }
            if ( fd1 != -1 && fd2 != -1 ) break;
        }
    }
    if ( fd1 == -1 || fd2 == -1 ) panic("pipe fd");

    proc->fdtable[fd1].busy = 1;
    proc->fdtable[fd1].fds_index = alloc_fds();
    ftf(fd1)->refcount += 1;
    proc->fdtable[fd2].busy = 1;
    proc->fdtable[fd2].fds_index = alloc_fds();
    ftf(fd2)->refcount += 1;

    Log("created pipe: %d read and %d write\n", fd1, fd2);
    Log("created pipe: fds %d read and fds %d write\n", proc->fdtable[fd1].fds_index, proc->fdtable[fd2].fds_index);
    struct pipe * pipe = alloc_pipe();
    pipe->readfd = fd1;
    pipe->writefd = fd2;
    ftf(fd1)->pipefs = pipe;
    ftf(fd2)->pipefs = pipe;
    ftf(fd1)->flags |= FILE_PIPE_R_FLAGS;
    ftf(fd2)->flags |= FILE_PIPE_W_FLAGS;
    *fd = fd1;
    *(fd + 1) = fd2;
    return 0;
}
```

`pipe` 函数用于创建一个管道。它首先从当前进程的文件描述表中找到两个空闲的文件描述符，将其标记为忙碌，并分配相应的文件描述符索引和引用计数。然后调用 `alloc_pipe` 分配并初始化管道结构，将读写文件描述符与管道关联，设置相应的读写标志。最后，将这两个文件描述符通过参数 `fd` 返回给调用者。如果没有找到两个空闲的文件描述符，则调用 `panic` 函数终止程序。

### 管道读/写

#### 管道读

```c
    if (ftf(fd)->flags & FILE_PIPE_W_FLAGS){
        memcpy(ftf(fd)->pipefs->pipe_mem + ftf(fd)->pipefs->writep, buf, count);
        ftf(fd)->pipefs->writep += count;
        Log("start pipe write: %x, %x, %x\n", ftf(fd)->pipefs->writep, ftf(fd)->pipefs->readp, count);
        Log("writed char = ");
        for ( int i = ftf(fd)->pipefs->writep - count; i < ftf(fd)->pipefs->writep ; i ++ ){
            Log("%c", *(char*)(ftf(fd)->pipefs->pipe_mem + i));
        }
        if ( ftf(fd)->pipefs->writep - ftf(fd)->pipefs->readp > 0x1000 ) yield();
        return count;
    }
```

这段代码实现了从管道读取数据的操作。它首先检查文件描述符的标志是否包含写入权限（`FILE_PIPE_W_FLAGS`）。如果满足条件，它会从管道的内存缓冲区中读取数据到缓冲区 `buf`，从当前的写入指针位置开始，并更新写入指针的位置。接着，它会记录写入操作的详细信息，包括新的写入指针位置、读取指针位置以及写入的字节数。然后，它通过日志输出写入的字符内容。最后，如果写入指针与读取指针之间的差值超过某个阈值（例如 `0x1000` 字节），则调用 `yield` 函数让出 CPU，以便其他进程能够执行。函数最终返回写入的字节数。

#### 管道写

```c

    if (ftf(fd)->flags & FILE_PIPE_R_FLAGS){
        while ( ftf(fd)->pipefs->readp == ftf(fd)->pipefs->writep && wait_counts ++ <= MAX_WAIT) yield();
        wait_counts = 0;
        // if ( ftf(fd)->pipefs->readp == ftf(fd)->pipefs->writep ) return 0;
        int result = ftf(fd)->pipefs->writep - ftf(fd)->pipefs->readp;
        result = ( result <= count ) ? result : count;
        Log("start pipe read: %x, %x\n", ftf(fd)->pipefs->readp, ftf(fd)->pipefs->readp + result);
        memcpy(buf, ftf(fd)->pipefs->readp + ftf(fd)->pipefs->pipe_mem, result);
        Log("readed char = ");
        for ( int i = ftf(fd)->pipefs->readp; i < ftf(fd)->pipefs->readp + result ; i ++ ){
            Log("%c",*(char*)(ftf(fd)->pipefs->pipe_mem + i));
        }
        ftf(fd)->pipefs->readp += result;
        Log("return result = %d\n", result);
        return result;
        // return ftf(fd)->pipefs->writep - ftf(fd)->pipefs->readp;
    }
```
这段代码实现了管道写入操作。它首先检查文件描述符的标志是否包含读入权限（`FILE_PIPE_R_FLAGS`）。如果有写入权限，它会将缓冲区 `buf` 中的数据复制到管道的内存缓冲区，从当前写入指针位置开始，并更新写入指针的位置。然后，它记录写入操作的详细信息，包括新的写入指针位置、读取指针位置和写入的字节数。如果写入的数据量超过某个阈值（如 `0x1000` 字节），它会调用 `yield` 函数让出 CPU 以便其他进程执行。最后，函数返回写入的字节数。

