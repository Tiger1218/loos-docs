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

## 管道管理

### 管道定义与基础操作

#### 初始化管道

```c
void init_pipes(){
    memset(pipes, 0, sizeof(pipes));
}
```

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


