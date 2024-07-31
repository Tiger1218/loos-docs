# 时钟中断

## LoongArch 机制

## 内核函数

### clock_handler

```c
void clock_handler() {
    tick++;
    // should call every proc instead of my_proc()
    struct proc *p = my_proc();
    if (p->callback && (tick >= p->timer_call_time)) {
        p->timer_call_time = -1;
        do_signal(SIGALRM);
    }
    yield();
}
```
