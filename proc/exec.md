# 可执行文件的加载

## 读 ELF 文件

### elf_loader

```c
uintptr_t elf_loader(struct proc *p, char *path) {
    int myfd = openat(0, path, 0);
    if (myfd == -1) return -1;
    read(myfd, elf_buf, sizeof(elf_buf));
    close(myfd);
    // memcpy(elf_buf, demo, sizeof(demo));

    const uint8_t *elf_base = elf_buf;
    assert(elf_base);

    /* parse header */
    Elf_Ehdr *header = (Elf_Ehdr*)elf_base;
    assert(memcmp(header->e_ident, elf_magic, sizeof(elf_magic)) == 0);
    assert(header->e_machine == EM_LOONGARCH);
    assert(header->e_ident[EI_CLASS] == ELFCLASS);
    assert(header->e_type == ET_EXEC || header->e_type == ET_DYN);
    if (memcmp(header->e_ident, elf_magic, sizeof(elf_magic)))
        return -1;

    /* find segment */
    Elf_Phdr *segments = (Elf_Phdr*)(elf_base + header->e_phoff);
    size_t phnum = header->e_phnum, ldnum = 0;
    Log("find %p segment(s) from elf\n", phnum);
    assert(phnum);

    /* load segment to memory */
    for (int i=0; i < phnum; i++) {
        Elf_Phdr *segment = &segments[i];
        // ignore p_type etc.
        uint8_t *dst = (uint8_t*)segment->p_vaddr;
        const uint8_t *src = elf_base + segment->p_offset;
        size_t len = segment->p_filesz;
        
        print_segment_info(segment);
        if (segment->p_type == PT_LOAD) {
            // assert(dst);
            assert(segment->p_filesz <= segment->p_memsz);
            assert(len);

            uint64_t pg_size = (PG_SIZE > segment->p_align) ? PG_SIZE : segment->p_align;
            void *vaddr = (void*)dst;
            uintptr_t start = ALIGN((uintptr_t)dst, pg_size);
            uintptr_t end = ALIGN((uintptr_t)dst + segment->p_memsz -1, pg_size) + pg_size;
            Log("vaddr %p, start %p, end %p\n", vaddr, start, end);

            if ((segment->p_flags & PF_R ) && (segment->p_flags & PF_W)) {
                p->bss = (uint64_t)dst+segment->p_memsz;
                Log("bss: %p\n", p->bss);
                // dirty hack
                end += 4*PG_SIZE;
            }

            alloc_vm(p, start, end, PTE_P|PTE_W|PTE_PLV|PTE_MATL|PTE_D|PTE_V);
            copy_to_va(p->pgtbl, vaddr, src, len);
            // assume all zero
            void *paddr = va_to_pa(p->pgtbl, dst);

            if (paddr == NULL) {
                Log("elf loader fail: vaddr %p, paddr %p\n", dst, paddr);
                return -1;
            }

            ldnum++;
        }
    }
    Log("load %p segment(s)\n", ldnum);
    Log("elf entry %p\n", header->e_entry);
    uintptr_t entry = header->e_entry;
    return entry;
}
```

### 加载新的 ELF 文件

```c
struct proc *exec(char *path, char *argv[], char *envp[]) {
    struct proc *p = proc_alloc();
    p->ppid = proc[0].pid;
    p->state = TASK_RUNABLE;
    if ( cur_proc == &proc[0] ){
        // printf("kproc");
    } else {
        p->fdtable = cur_proc->fdtable;
    }
    
    char *name = (argv[0]) ? argv[0] : path;
    strcpy(p->name, name);

    uintptr_t entry = elf_loader(p, path);
    if (entry == -1) {
        proc_free(p);
        return NULL;
    }

    alloc_vm(p, USER_STACK_BOTTOM, USER_STACK_TOP, PTE_P|PTE_W|PTE_PLV|PTE_MATL|PTE_D|PTE_V);
    uint64_t new_page = va_to_pa(p->pgtbl, USER_STACK_TOP - PG_SIZE) + PG_SIZE;
    new_page = PA2VA(new_page);
    p->context = new_page - sizeof(struct context);

    uint8_t *sp = USER_STACK_TOP;
    uint8_t *bp = USER_STACK_BOTTOM;
    sp -= sizeof(struct context);
    struct context *context = (void*)sp;

    sp -= sizeof(uint64_t) * (ARGS_MAX+2+ENVS_MAX+1);
    uint64_t argc, len, envc;
    uint64_t *ustack = (void*)sp;
    for (argc=0; argc < ARGS_MAX && argv[argc]; argc++) {
        copy_to_va(p->pgtbl, &ustack[argc+1], &bp, sizeof(uintptr_t));
        len = strlen(argv[argc])+1;
        copy_to_va(p->pgtbl, bp, argv[argc], len);
        bp += len;
    }
    if ( argv[argc])
        panic("exec: so many args");

    for (envc=0; envc < ENVS_MAX && envp[envc]; envc++) {
        copy_to_va(p->pgtbl, &ustack[argc+2+envc], &bp, sizeof(uintptr_t));
        len = strlen(envp[envc])+1;
        copy_to_va(p->pgtbl, bp, envp[envc], len);
        bp += len;
    }
    if ( envp[envc])
        panic("exec: so many envs");
    
    sp -= sizeof(struct trapframe);
    struct trapframe *trapframe = (void*)sp;

    uint64_t zero = 0, helper = &exec_helper, user_sp = &ustack[0], prmd = 0x4;
    copy_to_va(p->pgtbl, &ustack[argc+2+envc], &zero, sizeof(uintptr_t));
    copy_to_va(p->pgtbl, &ustack[argc+1], &zero, sizeof(uint64_t));
    copy_to_va(p->pgtbl, &ustack[0], &argc, sizeof(uint64_t));
    copy_to_va(p->pgtbl, &(context->sp), &sp, sizeof(uint64_t));
    copy_to_va(p->pgtbl, &(context->ra), &helper, sizeof(uint64_t));
    copy_to_va(p->pgtbl, &(trapframe->sp), &user_sp, sizeof(uint64_t));
    copy_to_va(p->pgtbl, &(trapframe->prmd), &prmd, sizeof(uint64_t));
    copy_to_va(p->pgtbl, &(trapframe->era), &entry, sizeof(uint64_t));
    return p;
}
```

## 封装器

```c
void exec_helper();
int exec_file(char *path) {
    char *argv[] = {path, NULL};
    char *envp[] = {NULL};
    struct proc *p = exec(path, argv, envp);
    if ( !p)
        return -1;
    return 0;
}
```
