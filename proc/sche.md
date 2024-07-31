# LoOS 中的调度器

## 进程状态

```c
enum proc_state {
    TASK_RUNNING = 0,
    TASK_RUNABLE,
    TASK_KILLED,
};
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
