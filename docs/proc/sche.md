# LoOS 中的调度器

## 进程状态

```c
enum proc_state {
    TASK_RUNNING = 0,
    TASK_RUNABLE,
    TASK_KILLED,
};
```

### 进程创建

```c
struct proc *proc_alloc() {
    int i;
    for (i=0; i < PROC_MAX; i++) {
        if (proc[i].allocs == UNUSED)
            goto found;
    }
    panic("proc");
    return NULL;

found:
    proc[i].context = NULL;
    proc[i].trapframe = NULL;
    proc[i].fdtable = fdtable_alloc();
    proc[i].pid = last_pid++;
    proc[i].cpid = -1;
    proc[i].allocs = USED;
    proc[i].bss = 0;
    proc[i].pgtbl = pgtbl_create();
    proc[i].mapping_list = mlist_init();
    proc[i].callback = NULL;
    proc[i].timer_call_time = -1;
    // printf("Returning %p for proc[%d]\n", &proc[i], i);
    return &proc[i];
}
```

#### 内核进程创建

```c
void kernel_proc_init() {
    struct proc *kproc = proc_alloc();
    strcpy(kproc->name, "k");
    kproc->pid = 0;
    kproc->state = TASK_RUNNING;
    kproc->context = kmalloc(sizeof(struct context));
    kproc->fdtable = fdtable_alloc();
    printf("allcoed: fdtable %p\n", kproc->fdtable);
}
```

#### 用户进程创建

```
void proc_init() {
    for (int i=0; i < PROC_MAX; i++) {
        memset(&proc[i], 0, sizeof(struct proc));
        proc[i].allocs = UNUSED;
        strcpy(proc[i].cwd, "/");
    }
    kernel_proc_init();
    cur_proc = &proc[0];
    assert(last_pid == 1);
}
```

### 进程特性的改变与设置

#### pid

```c
int get_proc_id() {
    struct proc *p = my_proc();
    assert(p);
    return p->pid;
}
int set_proc_id(int pid) {
    struct proc *p = my_proc();
    assert(p);
    p->pid = pid;
    return 0;
}
int chdir(char *path) {
    struct proc *p = my_proc();
    assert(p);
    // tbd
    // argstr(0, p->cwd, sizeof(p->cwd));
    return 0;
}
```

#### cwd

```c
char* getcwd(char* buf, size_t size) {
    struct proc *p = my_proc();
    assert(p);
    strncpy(buf, p->cwd, size);
    return buf;
}
```

### 进程结构辅助函数

```c

struct proc *proc_find(int pid) {
    for (int i=0; i < PROC_MAX; i++) {
        if ((proc[i].allocs == USED) && (proc[i].pid == pid)) {
            return &proc[i];
        }
    }
    return NULL;
}

struct proc *cur_proc = NULL;
struct proc *my_proc() {
    if(cur_proc == NULL) panic("my proc");
    return cur_proc;
}

void proc_init() {
    for (int i=0; i < PROC_MAX; i++) {
        memset(&proc[i], 0, sizeof(struct proc));
        proc[i].allocs = UNUSED;
        strcpy(proc[i].cwd, "/");
    }
    kernel_proc_init();
    cur_proc = &proc[0];
    assert(last_pid == 1);
}
void proc_print() {
    for (int i=0; i < PROC_MAX; i++) {
        if (proc[i].allocs != UNUSED) {
            char *state = (proc[i].state == TASK_RUNABLE) ? "runnable" :
                (proc[i].state == TASK_RUNNING) ? "running" : "unknown";
            printf("%s: pid %d ppid %d state %s\n", proc[i].name, proc[i].pid, proc[i].ppid, state);
        }
    }
}
```

### 进程自陷

```c
void yield() {
    struct proc *p = my_proc();
    struct proc *kproc = &proc[0];
    if (p->state == TASK_RUNNING) {
        p->state = TASK_RUNABLE;
    }
    kproc->state = TASK_RUNNING;
    // Log("yield: %s\n", p->name);
    swtch(p->context, kproc->context);
}
```

### 进程结束/杀死进程

```c
int kill(struct proc *p) {
    p->state = TASK_KILLED;
    if (p == my_proc()) {
        yield();
    }
    return -1;
}

int exit(int _) {
    struct proc *p = my_proc();
    Log("process %s exit\n", p->name);
    kill(p);
    return -1;
}
```

### fork

```c
struct proc *fork(struct proc *p) {
    struct proc *np = proc_alloc();
    copy_pagetable(p, np);

    p->cpid = np->pid;
    np->ppid = p->pid;
    np->state = TASK_RUNABLE;
    np->bss = p->bss;
    strcpy(np->name, p->name);
    // strcat(np->name, "-fork");
    np->fdtable = fdtable_copy(p->fdtable);
    strcpy(np->cwd, p->cwd);
    np->callback = p->callback;
    np->timer_call_time = p->timer_call_time;

    struct trapframe *trapframe = get_trapframe();
    struct context *context = USER_STACK_TOP - sizeof(struct context);
    void *new_page = va_to_pa(np->pgtbl, USER_STACK_TOP - PG_SIZE) + PG_SIZE;
    np->context = PA2VA(new_page) - sizeof(struct context);
    uint8_t *sp = (void*)trapframe;

    uint64_t zero = 0, helper = &exec_helper;
    copy_to_va(np->pgtbl, &(trapframe->a0), &zero, sizeof(uint64_t));
    copy_to_va(np->pgtbl, &(context->sp), &sp, sizeof(uint64_t));
    copy_to_va(np->pgtbl, &(context->ra), &helper, sizeof(uint64_t));
    // Log("forking: %p->%p\n", p, np);
    Log("forking: %d->%d\n", p->pid, np->pid);
    // Log("forking: %s->%s\n", p->name, np->name);
    return np;
}
```

## 调度器源码

```c
void scheduler() {
    Log("scheduler\n");
    struct proc *kproc = &proc[0];
    assert(kproc->pid == 0);
    int run;
    while (1) {
        // proc_print();
        run = 0;
        for (int i=0; i < PROC_MAX; i++) {
            // Log("cur proc proc[%d]\n", i);
            if ((proc[i].allocs == UNUSED) || (proc[i].pid == 0))
                continue;
            if (proc[i].state == TASK_RUNABLE) {
                run = 1;
                kproc->state = TASK_RUNABLE;
                proc[i].state = TASK_RUNNING;
                Log("switch from %s to %s\n", kproc->name, proc[i].name);
                // swtch(kproc->context, proc[i].context);
                switch_to(kproc, &proc[i]);
                Log("ret from %s to %s\n", proc[i].name, kproc->name);
            } else if (proc[i].state == TASK_KILLED) {
                Log("free proc %s\n", proc[i].name);
                if (proc[i].pid == 0)
                    panic("Kernel panic: Attempted to kill init!");
                proc_free(&proc[i]);
            }
        }
        if ( !run) {
            break;
        }
    }
    shutdown();
}
```

## 进程切换

### 汇编层级

```assembly
.globl what
.globl swtch
swtch:
    # save the registers.
    st.d $ra, $a0, 0
    st.d $tp, $a0, 8
    st.d $sp, $a0, 16
    st.d $a0, $a0, 24
    st.d $a1, $a0, 32
    st.d $a2, $a0, 40
    st.d $a3, $a0, 48
    st.d $a4, $a0, 56
    st.d $a5, $a0, 64
    st.d $a6, $a0, 72
    st.d $a7, $a0, 80
    st.d $t0, $a0, 88
    st.d $t1, $a0, 96
    st.d $t2, $a0, 104
    st.d $t3, $a0, 112
    st.d $t4, $a0, 120
    st.d $t5, $a0, 128
    st.d $t6, $a0, 136
    st.d $t7, $a0, 144
    st.d $t8, $a0, 152
    st.d $r21, $a0,160
    st.d $fp, $a0, 168
    st.d $s0, $a0, 176
    st.d $s1, $a0, 184
    st.d $s2, $a0, 192
    st.d $s3, $a0, 200
    st.d $s4, $a0, 208
    st.d $s5, $a0, 216
    st.d $s6, $a0, 224
    st.d $s7, $a0, 232
    st.d $s8, $a0, 240

    # restore register
    ld.d $ra, $a1, 0
    ld.d $tp, $a1, 8
    ld.d $sp, $a1, 16
    ld.d $a0, $a1, 24
    ld.d $a2, $a1, 40
    ld.d $a3, $a1, 48
    ld.d $a4, $a1, 56
    ld.d $a5, $a1, 64
    ld.d $a6, $a1, 72
    ld.d $a7, $a1, 80
    ld.d $t0, $a1, 88
    ld.d $t1, $a1, 96
    ld.d $t2, $a1, 104
    ld.d $t3, $a1, 112
    ld.d $t4, $a1, 120
    ld.d $t5, $a1, 128
    ld.d $t6, $a1, 136
    ld.d $t7, $a1, 144
    ld.d $t8, $a1, 152
    ld.d $r21, $a1,160
    ld.d $fp, $a1, 168
    ld.d $s0, $a1, 176
    ld.d $s1, $a1, 184
    ld.d $s2, $a1, 192
    ld.d $s3, $a1, 200
    ld.d $s4, $a1, 208
    ld.d $s5, $a1, 216
    ld.d $s6, $a1, 224
    ld.d $s7, $a1, 232
    ld.d $s8, $a1, 240
    ld.d $a1, $a1, 32

    jr $ra
	

.globl trap_exit
.globl exec_helper
exec_helper:
    b trap_exit
exec_exit_fail:
    b exec_exit_fail
```

### 进程元信息切换

```c
void switch_to(struct proc * before, struct proc *after) {
    cur_proc = after;
    w_csr_pgdl((uint64_t)after->pgtbl);
    invtlb_all();

    swtch(before->context, after->context);
    invtlb_all();
}
```

## 总结

### 展望

1. 添加僵尸态，方便 wait4, scheduler 等一系列模块。
2. 添加调度种类
3. 重写 struct proc ，增加特性（如 uid, gid ）
