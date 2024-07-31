# 内存管理相关异常处理

## TLB 重填

### TLB 初始化

```c
// mm.c
void tlb_init() {
    w_csr_stlbps(0xE);
    invtlb_all();
    w_csr_asid(0x0U);//todo
}
```

```c
// trap.c
void trap_init() {
    ...
    w_csr_tlbrentry((uint64_t)handle_tlbr);
    ...
}
```

### 基于汇编的原理简介

```
#define DIR1BASE   25
#define DIR1WIDTH  11
#define DIR3BASE   36
#define DIR3WIDTH  11
#define PTBASE     14
#define PTWIDTH    11

#define DIR2BASE   0
#define DIR2WIDTH  0
#define DIR4BASE   0
#define DIR4WIDTH  0
#define PTEWIDTH   0
```

### tlb_entry.s

```assembly
.section tlbrentry
.globl handle_tlbr
.align 0x4
handle_tlbr:
    csrwr	$t0, 0x8B
    csrrd	$t0, 0x1B
    lddir   $t0, $t0, 3
    lddir   $t0, $t0, 1
    ldpte	$t0, 0
    ldpte	$t0, 1
    tlbfill
    csrrd	$t0, 0x8B
    ertn
```

```c
void vm_init(void)
{
    w_csr_pwcl((PTEWIDTH << 30)|(DIR2WIDTH << 25)|(DIR2BASE << 20)|(DIR1WIDTH << 15)|(DIR1BASE << 10)|(PTWIDTH << 5)|(PTBASE << 0));
    w_csr_pwch((DIR4WIDTH << 18)|(DIR3WIDTH << 6)|(DIR3BASE << 0));
}
```

## E_PIL/E_PIS/E_PIF

```c
        } else if (ecode == E_PIL || ecode == E_PIS || ecode == E_PIF) {
            printf("%s(%d): page illegal exception\n", p->name, p->pid);
            printf("badv=%p era=%p\n", badv, era);
            printf("ecode = %d\n", ecode);
            printf("pgd=%p\n", r_csr_pgdl());
            printf("pgd(proc) = %p\n", p->pgtbl);
            // dump_mapping(p);
            exit(1);
        } else if (ecode == E_ADE){
```

## E_ADE

E_ADEA / E_ADEF

所有取指操作的访存地址必须4 字节边界对齐，否则将触发取指地址错例外（ADEF）。
除了原子访存指令、整数边界检查访存指令和浮点数边界检查访存指令外，其余的load/store 访存指令
可以实现为允许访存地址不对齐。不过，在一个允许访存地址不对齐的实现中，系统态软件可以通过配置
CSR.MISC 中的ALCL0~ALCL3 控制位，在PLV0~PLV3 特权等级下，对这些load/store 访存指令也进行地
址对齐检查。对于需要进行地址对齐检查的访存指令，如果其访问的地址不是自然对齐1的，将触发地址非
对齐例外（ALE）。

## 总结

### 展望

1. Lazy Page Allocation
2. CoW( Copy on Write )
3. Zero Fill on Demand
4. Demand Paging
5. MMAP

## References

[8.1 Page Fault Basics - MIT6.S081](https://web.archive.org/web/20240115135322/https://mit-public-courses-cn-translatio.gitbook.io/mit6-s081/lec08-page-faults-frans/8.1-page-fault-basics)
