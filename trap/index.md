# 中断/异常系统

## 总入口

```c
void trap_handler() {
    uint32_t estat = r_csr_estat();
    uint32_t ecfg = r_csr_ecfg();
    uint64_t era = r_trap_era();
    uint64_t badv = r_csr_badv();

    if (estat & ecfg & HWI_VEC) {
        // this is a hardware interrupt, via IOCR.
        panic("not finish\n");
    } else if (estat & ecfg & TI_VEC) {
        // timer interrupt
        clock_handler();
        w_csr_ticlr(r_csr_ticlr() | CSR_TICLR_CLR);
    } else {
        era += 4;
        w_trap_era(era);
        struct proc *p = my_proc();
        uint8_t ecode = (estat & CSR_ESTAT_ECODE) >> 16;
        uint16_t esubcode = (estat & CSR_ESTAT_ESUBCODE) >> 22;
        if (ecode == E_SYS) {
            // system call
            // intr_on();
            syscall_handler();
        } else if (ecode == E_BRK) {
            // kdb_force = 1;
            break_handler();
            // maybe write trap era
        } else if (ecode == E_FPD) {
            printf("ignore: float point exception\n");
        } else if (ecode == E_PIL || ecode == E_PIS || ecode == E_PIF) {
            printf("%s(%d): page illegal exception\n", p->name, p->pid);
            printf("badv=%p era=%p\n", badv, era);
            printf("ecode = %d\n", ecode);
            printf("pgd=%p\n", r_csr_pgdl());
            printf("pgd(proc) = %p\n", p->pgtbl);
            // dump_mapping(p);
            exit(1);
        } else if (ecode == E_ADE){
            printf("%s: address exception\n", p->name);
            if(esubcode == 1) {
                printf("access memory error\n");
            } else if (esubcode == 0){
                printf("fetch instructions error\n");
            } else {
                panic("esubcode not 1 neither 0");
            }
            printf("badv=%p era=%p\n", badv, era);
            printf("pgd=%p\n", r_csr_pgd());
            exit(1);
        } else if (ecode == E_INE){
            printf("%s: instrcution non-exist exception\n", p->name);
            printf("badv=%p era=%p\n", badv, era);
            printf("badi=%p\n", r_csr_badi());
            exit(1);
        } else {
            printf("%s: trapcause %x era=%p\n", p->name, estat, era);
            printf("ecode = %x badv = %x\n", ecode, badv);
            exit(1);
        }
    }
}
```

## 初始化

```
static void timer_init() {
    uint64_t tcfg = 0x1000000UL | CSR_TCFG_EN | CSR_TCFG_PER;
    w_csr_tcfg(tcfg);
}

void trap_init() {
    uint32_t ecfg = ( 0U << CSR_ECFG_VS_SHIFT ) | HWI_VEC | TI_VEC;
    w_csr_ecfg(ecfg);
    w_csr_eentry((uint64_t)trap_entry);
    w_csr_tlbrentry((uint64_t)handle_tlbr);
    //w_csr_merrentry((uint64)handle_merr);
    timer_init();
    // intr_on();
}
```

