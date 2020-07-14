# Process Management

### fork
![Linux fork](../Images/linux-fork.jpg)

### exec
```C++
do_execve(getname(filename), argv, envp);
    do_execveat_common
        exec_binprm
            search_binary_handler // fs/exec.c
                load_elf_binary // fs/binfmt_elf.c
                    setup_new_exec(); // set mmap_base
                    setup_arg_pages();
                    elf_map(); // map the code in elf file to memory
                    set_brk(); // setup heap area
                    load_elf_interp(); // load dependent *.so
                    start_thread(regs, elf_entry, bprm->p); // arch/x86/kernel/process_32.c
```

### pthread
```C++
// Linux pthread implementation
__pthread_create_2_1 // binfmt_elf.c
    ALLOCATE_STACK (iattr, &pd); // allocatestack.c
        /* Adjust the stack size for alignment. */
        /* call mmap to alloc thread statck in process heap*/
        /* set protection of this thread stack memory */
        /* populate member: stackblock、stackblock_size、guardsize、specific */
        /* And add to the list of stacks in use. */
    // start thread
    create_thread (pd, iattr, stackaddr, stacksize); // sysdeps/pthread/createthread.c
    do_clone (pd, attr, clone_flags, start_thread/*pthread func*/, stackaddr, stacksize, 1);
        ARCH_CLONE (start_thread, stackaddr, stacksize,, clone_flags, pd, &pd->tid, TLS_VALUE, &pd->tid);
            _do_fork();
    start_thread // pthread_creat.c
        THREAD_SETMEM (pd, result, pd->start_routine (pd->arg));
        // pd->result = pd->start_routine(pd->arg);
    __free_tcb
        __deallocate_stack
            queue_stack
```

### schedule
##### Voluntary Schedule
```c++
schedule(void)
    __schedule(false), // kernel/sched/core.c
        pick_next_task(rq, prev, &rf);
        context_switch(rq, prev, next, &rf);
            switch_mm_irqs_off(prev->active_mm, next->mm, next);
            switch_to(prev, next, prev);
                __switch_to_asm(); // switch registers, but not EIP [arch/x86/entry/entry_64.S]
                    __switch_to(); // switch stack [arch/x86/kernel/process_32.c]
                        this_cpu_write(current_task, next_p); //
            barrier();
            return finish_task_switch(prev);
```


##### Involuntary Shcedule(preempty)
```C++
// 1. mark TIF_NEED_RESCHED
// no time slice
scheduler_tick(); // kernel/sched/core.c
    task_tick_fair(rq, curr, 0);
        entity_tick(cfs_rq, se, queued);
            check_preempt_tick(cfs_rq, curr);
                resched_curr(rq_of(cfs_rq));
                    set_tsk_need_resched();

// wake up
try_to_wake_up(); // kernel/sched/core.c
    ttwu_queue
        ttwu_do_activate
            ttwu_do_wakeup
                check_preempt_curr
                    resched_curr
                        --->

// 2. real time shceudle
// real user space preempty time: 1. return from system call
do_syscall_64();
    syscall_return_slowpath();
        prepare_exit_to_usermode();
            exit_to_usermode_loop(struct pt_regs *regs, u32 cached_flags) {
                while (true) {
                    if (cached_flags & _TIF_NEED_RESCHED)
                        schedule();

                    if (cached_flags & _TIF_SIGPENDING) {
                        do_signal(regs) {
                            struct ksignal ksig;
                            if (get_signal(&ksig)) {
                                handle_signal(&ksig, regs) {
                                    bool stepping, failed;
                                    if (syscall_get_nr(current, regs) >= 0) {
                                        switch (syscall_get_error(current, regs)) {
                                        case -ERESTART_RESTARTBLOCK:
                                        case -ERESTARTNOHAND:
                                            regs->ax = -EINTR;
                                            break;
                                        case -ERESTARTSYS:
                                            if (!(ksig->ka.sa.sa_flags & SA_RESTART)) {
                                                regs->ax = -EINTR;
                                                break;
                                            }
                                        case -ERESTARTNOINTR:
                                            regs->ax = regs->orig_ax;
                                            regs->ip -= 2;
                                            break;
                                        }
                                    }
                                    failed = (setup_rt_frame(ksig, regs) < 0) {
                                        static int __setup_rt_frame(int sig, struct ksignal *ksig,
                                                sigset_t *set, struct pt_regs *regs)
                                        {
                                            struct rt_sigframe __user *frame;
                                            void __user *fp = NULL;
                                            int err = 0;

                                            frame = get_sigframe(&ksig->ka, regs, sizeof(struct rt_sigframe), &fp);
                                            put_user_try {
                                                if (ksig->ka.sa.sa_flags & SA_RESTORER) {
                                                    put_user_ex(ksig->ka.sa.sa_restorer, &frame->pretcode);
                                                }
                                            } put_user_catch(err);

                                            err |= setup_sigcontext(&frame->uc.uc_mcontext, fp, regs, set->sig[0]);
                                            err |= __copy_to_user(&frame->uc.uc_sigmask, set, sizeof(*set));

                                            regs->di = sig;
                                            regs->ax = 0;

                                            regs->si = (unsigned long)&frame->info;
                                            regs->dx = (unsigned long)&frame->uc;
                                            regs->ip = (unsigned long) ksig->ka.sa.sa_handler;

                                            regs->sp = (unsigned long)frame;
                                            regs->cs = __USER_CS;
                                            return 0;
                                        }
                                    }
                                    signal_setup_done(failed, ksig, stepping);
                                }
                                return;
                            }

                            if (syscall_get_nr(current, regs) >= 0) {
                                switch (syscall_get_error(current, regs)) {
                                case -ERESTARTNOHAND:
                                case -ERESTARTSYS:
                                case -ERESTARTNOINTR:
                                    regs->ax = regs->orig_ax;
                                    regs->ip -= 2;
                                    break;
                                case -ERESTART_RESTARTBLOCK:
                                    regs->ax = get_nr_restart_syscall(regs);
                                    regs->ip -= 2;
                                    break;
                                }
                            }
                            restore_saved_sigmask();
                        }
                    }

                    if (!(cached_flags & EXIT_TO_USERMODE_LOOP_FLAGS))
                        break;
                }
            }


    // restore user space stack upon return from sighandler
    asmlinkage long sys_rt_sigreturn(void)
    {
        struct pt_regs *regs = current_pt_regs();
        struct rt_sigframe __user *frame;
        sigset_t set;
        unsigned long uc_flags;

        frame = (struct rt_sigframe __user *)(regs->sp - sizeof(long));
        if (__copy_from_user(&set, &frame->uc.uc_sigmask, sizeof(set)))
            goto badframe;
        if (__get_user(uc_flags, &frame->uc.uc_flags))
            goto badframe;

        set_current_blocked(&set);

        if (restore_sigcontext(regs, &frame->uc.uc_mcontext, uc_flags))
            goto badframe;
        return regs->ax;
    }

// real user space preempty time: 2. return from interrupt
do_IRQ
    retint_user //arch/x86/entry/entry_64.S
        prepare_exit_to_usermode
            exit_to_usermode_loop
                --->


// real kernel preempty time: 1. preempty_enble
preempt_enable
    preempt_count_dec_and_test
        preempt_schedule
            preempt_schedule_common
                __schedule
                    --->

// real kernel preempty time: 2. return from interrupt
do_IRQ
    retint_kernel
        prepare_exit_to_usermode
            --->

```

# Signal
```C++
struct task_struct {
    /* Signal handlers: */
    struct signal_struct    *signal;
    struct sighand_struct __rcu    *sighand;
    sigset_t      blocked;
    sigset_t      real_blocked;
    /* Restored if set_restore_sigmask() was used: */
    sigset_t      saved_sigmask;
    struct sigpending    pending;
    unsigned long      sas_ss_sp;
    size_t        sas_ss_size;
    unsigned int      sas_ss_flags;
};
/*
kill->kill_something_info->kill_pid_info->group_send_sig_info->do_send_sig_info
tkill->do_tkill->do_send_specific->do_send_sig_info
tgkill->do_tkill->do_send_specific->do_send_sig_info
rt_sigqueueinfo->do_rt_sigqueueinfo->kill_proc_info->kill_pid_info->group_send_sig_info->do_send_sig_info
*/


do_send_sig_info() {
    send_signal() {
        __send_signal(int sig, struct siginfo *info, struct task_struct *t, int group, int from_ancestor_ns) {
            struct sigpending *pending;
            struct sigqueue *q;
            int override_rlimit;
            int ret = 0, result;
            pending = group ? &t->signal->shared_pending : &t->pending;
            if (legacy_queue(pending, sig) {
                return (sig < SIGRTMIN) && sigismember(&pending->signal, sig);
            })
                goto ret;

            if (sig < SIGRTMIN)
                override_rlimit = (is_si_special(info) || info->si_code >= 0);
            else
                override_rlimit = 0;

            q = __sigqueue_alloc(sig, t, GFP_ATOMIC | __GFP_NOTRACK_FALSE_POSITIVE,
                override_rlimit);
            if (q) {
                list_add_tail(&q->list, &pending->list);
                switch ((unsigned long) info) {
                case (unsigned long) SEND_SIG_NOINFO:
                    q->info.si_signo = sig;
                    q->info.si_errno = 0;
                    q->info.si_code = SI_USER;
                    q->info.si_pid = task_tgid_nr_ns(current,
                            task_active_pid_ns(t));
                    q->info.si_uid = from_kuid_munged(current_user_ns(), current_uid());
                    break;
                case (unsigned long) SEND_SIG_PRIV:
                    q->info.si_signo = sig;
                    q->info.si_errno = 0;
                    q->info.si_code = SI_KERNEL;
                    q->info.si_pid = 0;
                    q->info.si_uid = 0;
                    break;
                default:
                    copy_siginfo(&q->info, info);
                    if (from_ancestor_ns)
                        q->info.si_pid = 0;
                    break;
                }
                userns_fixup_signal_uid(&q->info, t);

            }
            out_set:
            signalfd_notify(t, sig);
            sigaddset(&pending->signal, sig);
            complete_signal(sig, t, group) {
                struct signal_struct *signal = p->signal;
                struct task_struct *t;
                if (wants_signal(sig, p))
                    t = p;
                else if (!group || thread_group_empty(p))
                    return;
                else {
                    t = signal->curr_target;
                    while (!wants_signal(sig, t)) {
                        t = next_thread(t);
                        if (t == signal->curr_target)
                            return;
                    }
                    signal->curr_target = t;
                }
                signal_wake_up(t, sig == SIGKILL) {
                    set_tsk_thread_flag(t, TIF_SIGPENDING);
                    if (!wake_up_state(t, state | TASK_INTERRUPTIBLE))
                        kick_process(t);
                }
            }
            ret:
            return ret;
        }
    }
}

// signal real handing see <Modern Operating System.md> exit_to_usermode_loop
```
![Linux signal handling](../Images/linux-signal-handling.png)

# Memory Management
### malloc
```C++
// mm/mmap.c
SYSCALL_DEFINE1(brk, unsigned long, brk)
{
    unsigned long retval;
    unsigned long newbrk, oldbrk;
    struct mm_struct *mm = current->mm;
    struct vm_area_struct *next;
    //
    newbrk = PAGE_ALIGN(brk);
    oldbrk = PAGE_ALIGN(mm->brk);
    if (oldbrk == newbrk)
        goto set_brk;

    /* Always allow shrinking brk. */
    if (brk <= mm->brk) {
        if (!do_munmap(mm, newbrk, oldbrk-newbrk, &uf))
            goto set_brk;
        goto out;
    }

    /* Check against existing mmap mappings. */
    next = find_vma(mm, oldbrk);
    if (next && newbrk + PAGE_SIZE > vm_start_gap(next))
        goto out;

    /* Ok, looks good - let it rip. */
    if (do_brk(oldbrk, newbrk-oldbrk, &uf) < 0)
        do_brk_flags();
            find_vma_links();
            vma_merge();
            kmem_cache_zalloc();
            INIT_LIST_HEAD(&vma->anon_vma_chain);
            vma_link(mm, vma, prev, rb_link, rb_parent); // insert mm_struct rb_mm;
        goto out;

set_brk:
  mm->brk = brk;
//
  return brk;
out:
  retval = mm->brk;
  return retval
```

### mmap
```C++
// Linux mmap implementation
ksys_mmap_pgoff(); // mm/mmap.c
    vm_mmap_pgoff();
        do_mmap_pgoff();
            do_mmap();
                get_unmapped_area() {
                    unsigned long (*get_area)(struct file *, unsigned long, unsigned long,
                        unsigned long, unsigned long);
                    get_area = current->mm->get_unmapped_area;   // 1. anonymous mapping
                    if (file) {
                        get_area = file->f_op->get_unmapped_area; // 2. file mapping
                    } else if (flags & MAP_SHARED) {
                        get_area = shmem_get_unmapped_area;      // 3. shmem mapping
                    }
                    addr = get_area(file, addr, len, pgoff, flags);
                }
                mmap_region() {
                    vma = vma_merge(mm, prev, addr, addr + len, vm_flags,
                            NULL, file, pgoff, NULL, NULL_VM_UFFD_CTX);
                    if (vma)
                        goto out;

                    vma = vm_area_alloc(mm);
                    if (!vma) {
                        error = -ENOMEM;
                        goto unacct_error;
                    }

                    vma->vm_start = addr;
                    vma->vm_end = addr + len;
                    vma->vm_flags = vm_flags;
                    vma->vm_page_prot = vm_get_page_prot(vm_flags);
                    vma->vm_pgoff = pgoff;

                    if (file) {
                        if (vm_flags & VM_DENYWRITE) {
                            error = deny_write_access(file);
                        }
                        if (vm_flags & VM_SHARED) {
                            error = mapping_map_writable(file->f_mapping);
                        }
                        vma->vm_file = get_file(file); // map file to vma
                        error = call_mmap(file, vma) {
                            file->f_op->mmap(file, vma) {
                                ext4_file_mmap() {
                                    vma->vm_ops = &ext4_file_vm_ops; // map file ops to vma ops
                                }
                            }
                        }
                    } else if (vm_flags & VM_SHARED) {
                        error = shmem_zero_setup(vma);
                    } else {
                        vma_set_anonymous(vma);
                    }

                    vma_link(mm, vma, prev, rb_link, rb_parent) {
                        __vma_link(mm, vma, prev, rb_link, rb_parent);
                        __vma_link_file(vma) {
                            vma_interval_tree_insert(vma, &mapping->i_mmap); // map file to vma
                        }
                    }
                }
```

### page fault
```C++
struct address_space {
  struct inode    *host;
  struct xarray    i_pages;
  struct rb_root_cached  i_mmap;
  atomic_t    i_mmap_writable;
  const struct address_space_operations *a_ops;

  gfp_t      gfp_mask;
  struct rw_semaphore  i_mmap_rwsem;
  unsigned long    nrpages;
  unsigned long    nrexceptional;
  pgoff_t      writeback_index;
  unsigned long    flags;
  errseq_t    wb_err;
  spinlock_t    private_lock;
  struct list_head  private_list;
  void      *private_data;
};

do_page_fault() {
    __do_page_fault();
        vmalloc_fault(address) { // fault in kernel

        }

        handle_mm_fault(vma, address, flags); // fault in usr space
            handle_pte_fault(&vmf);
                do_anonymous_page(vmf) { // 1. map physical memory
                    pte_alloc(); // alloc a page table item
                    alloc_zeroed_user_highpage_movable() {
                        alloc_pages_vma();
                        __alloc_pages_nodemask(); // alloc psyhic memory
                            --->
                    }
                    mk_pte(page, vma->vm_page_prot);
                    set_pte_at(vma->vm_mm, vmf->address, vmf->pte, entry);
                }

                do_fault(vmf) {          // 2. map to a file
                    __do_fault() {
                        vma->vm_ops->fault(vmf) {
                            ext4_filemap_fault() {
                                struct inode *inode = file_inode(vmf->vma->vm_file);
                                filemap_fault(vmf) {
                                    struct file *file = vmf->vma->vm_file;
                                    struct address_space *mapping = file->f_mapping;
                                    struct inode *inode = mapping->host;
                                    pgoff_t offset = vmf->pgoff;
                                    struct page *page; // page cache in physic of the file
                                    int ret = 0;

                                    page = find_get_page(mapping, offset);
                                    if (likely(page) && !(vmf->flags & FAULT_FLAG_TRIED)) {
                                        do_async_mmap_readahead(vmf->vma, ra, file, page, offset);
                                    } else if (!page) {
                                        goto no_cached_page;
                                    }

                                    vmf->page = page;
                                    return ret | VM_FAULT_LOCKED;

                                    no_cached_page:
                                    page_cache_read(file, offset, vmf->gfp_mask) {
                                        struct address_space *mapping = file->f_mapping;
                                        struct page *page;
                                        page = __page_cache_alloc(gfp_mask|__GFP_COLD);
                                        ret = add_to_page_cache_lru(page, mapping, offset, gfp_mask & GFP_KERNEL);
                                        ret = mapping->a_ops->readpage(file, page) {
                                            struct address_space *mapping = file->f_mapping;
                                            struct page *page;
                                            page = __page_cache_alloc(gfp_mask|__GFP_COLD);
                                            ret = add_to_page_cache_lru(page, mapping, offset, gfp_mask & GFP_KERNEL);
                                            ret = mapping->a_ops->readpage(file, page) {
                                                ext4_read_inline_page(inode, page) {
                                                    void *kaddr = kmap_atomic(page);
                                                    ret = ext4_read_inline_data(inode, kaddr, len, &iloc);
                                                    flush_dcache_page(page);
                                                    kunmap_atomic(kaddr);
                                                }
                                            }
                                        }
                                    }
                                }
                            }
                        }
                    }
                }
                do_swap_page(vmf) {      // 3. map to swap
                    struct vm_area_struct *vma = vmf->vma;
                    struct page *page, *swapcache;
                    struct mem_cgroup *memcg;
                    swp_entry_t entry;
                    pte_t pte;
                    entry = pte_to_swp_entry(vmf->orig_pte);
                    page = lookup_swap_cache(entry);
                    if (!page) {
                        page = swapin_readahead(entry, GFP_HIGHUSER_MOVABLE, vma, vmf->address) {
                            int swap_readpage(struct page *page, bool do_poll) {
                                struct bio *bio;
                                int ret = 0;
                                struct swap_info_struct *sis = page_swap_info(page);
                                blk_qc_t qc;
                                struct block_device *bdev;
                                if (sis->flags & SWP_FILE) {
                                    struct file *swap_file = sis->swap_file;
                                    struct address_space *mapping = swap_file->f_mapping;
                                    ret = mapping->a_ops->readpage(swap_file, page);
                                    return ret;
                                }
                            }
                        }
                    }
                    swapcache = page;
                    pte = mk_pte(page, vma->vm_page_prot);
                    set_pte_at(vma->vm_mm, vmf->address, vmf->pte, pte);
                    vmf->orig_pte = pte;
                    swap_free(entry);
                }
}


static struct mm_struct *dup_mm(struct task_struct *tsk)
{
  struct mm_struct *mm, *oldmm = current->mm;
  mm = allocate_mm() {
      kmem_cache_alloc(mm_cachep, GFP_KERNEL);
  }
  memcpy(mm, oldmm, sizeof(*mm));
  mm_init(mm, tsk, mm->usr_ns) {
        mm_alloc_pgd(mm) {
            mm->pdg = pgd_alloc() { // arch/x86/mm/pgtable.c
                pgd_t *pgd = _pgd_alloc();
                pgd_ctor(mm, pgd) {
                    if (CONFIG_PGTABLE_LEVELS == 2 ||
                        (CONFIG_PGTABLE_LEVELS == 3 && SHARED_KERNEL_PMD) ||
                        CONFIG_PGTABLE_LEVELS >= 4) {
                        clone_pgd_range(pgd + KERNEL_PGD_BOUNDARY,
                            swapper_pg_dir + KERNEL_PGD_BOUNDARY,
                            KERNEL_PGD_PTRS);
                    }
                }
            }
        }
  }
  err = dup_mmap(mm, oldmm);
  return mm;
}
```

### Buddy System
```C++
typedef struct pglist_data {
  struct zone node_zones[MAX_NR_ZONES];
  struct zonelist node_zonelists[MAX_ZONELISTS];
  int nr_zones;
  struct page *node_mem_map;
  unsigned long node_start_pfn;
  unsigned long node_present_pages; /* total number of physical pages */
  unsigned long node_spanned_pages; /* total size of physical page range, including holes */
  int node_id;
} pg_data_t;


struct zone {
  struct pglist_data  *zone_pgdat;
  struct per_cpu_pageset __percpu *pageset;

  unsigned long    zone_start_pfn;

  unsigned long    managed_pages;
  unsigned long    spanned_pages;
  unsigned long    present_pages;

  const char    *name;
  struct free_area  free_area[MAX_ORDER];
  unsigned long    flags;
  spinlock_t    lock;
} ____cacheline_internodealigned_in_

// Linux buddy system
alloc_pages();
    alloc_pages_current(gfp_mask, order);
        struct mempolicy *pol = &default_policy;
        struct page *page;
        //
        page = __alloc_pages_nodemask(gfp, order, policy_node(gfp, pol, numa_node_id()), policy_nodemask(gfp, pol));
            get_page_from_freelist();
                //
                for_next_zone_zonelist_nodemask(zone, z, ac->zonelist, ac->high_zoneidx, ac->nodemask) {
                    struct page *page;
                    page = rmqueue(ac->preferred_zoneref->zone, zone, order, gfp_mask, alloc_flags, ac->migratetype);
                        __rmqueue();
                            rmqueue_smallest() {
                                /* Find a page of the appropriate size in the preferred list */
                                for (current_order = order; current_order < MAX_ORDER; ++current_order) {
                                    area = &(zone->free_area[current_order]);
                                    page = list_first_entry_or_null(&area->free_list[migratetype], struct page, lru);
                                    if (!page)
                                        continue;
                                    list_del(&page->lru);
                                    rmv_page_order(page);
                                    area->nr_free--;
                                    expand(zone, page, order, current_order, area, migratetype) {
                                        unsigned long size = 1 << high;
                                        while (high > low) {
                                            area--;
                                            high--;
                                            size >>= 1;
                                            list_add(&page[size].lru, &area->free_list[migratetype]);
                                            area->nr_free++;
                                            set_page_order(&page[size], high);
                                        }
                                    }

                                    set_pcppage_migratetype(page, migratetype); return page;
                                }
                                return NULL;
                            }
                }
        return page;
```
```C++
struct kmem_cache {
  struct kmem_cache_cpu __percpu *cpu_slab;
  /* Used for retriving partial slabs etc */
  unsigned long flags;
  unsigned long min_partial;
  int size;    /* The size of an object including meta data */
  int object_size;  /* The size of an object without meta data */
  int offset;    /* Free pointer offset. */
#ifdef CONFIG_SLUB_CPU_PARTIAL
  int cpu_partial;  /* Number of per cpu partial objects to keep around */
#endif
  struct kmem_cache_order_objects oo;
  /* Allocation and freeing of slabs */
  struct kmem_cache_order_objects max;
  struct kmem_cache_order_objects min;
  gfp_t allocflags;  /* gfp flags to use on each alloc */
  int refcount;    /* Refcount for slab cache destroy */
  void (*ctor)(void *);
  const char *name;  /* Name (only for display!) */
  struct list_head list;  /* List of slab caches */
  struct kmem_cache_node *node[MAX_NUMNODES];
};

// all chches will put into LIST_HEAD(slab_caches)

struct kmem_cache_cpu {
  void **freelist;  /* Pointer to next available object */
  unsigned long tid;  /* Globally unique transaction id */
  struct page *page;  /* The slab from which we are allocating */
#ifdef CONFIG_SLUB_CPU_PARTIAL
  struct page *partial;  /* Partially allocated frozen slabs */
#endif
};


struct kmem_cache_node {
  spinlock_t list_lock;
#ifdef CONFIG_SLUB
  unsigned long nr_partial;
  struct list_head partial;
#endif
};
```
![kmem_cache_cpu_node](../Images/kmem-cache-cpu-node.jpg)

### Slab/slub/slob System
```C++
// Linux slab/slub/slob system
static struct kmem_cache *task_struct_cachep;

static inline struct task_struct *alloc_task_struct_node(int node)
{
    return kmem_cache_alloc_node(task_struct_cachep, GFP_KERNEL, node);
}

static inline void free_task_struct(struct task_struct *tsk)
{
    kmem_cache_free(task_struct_cachep, tsk);
}


alloc_task_struct_node();
    kmem_cache_alloc_node(task_struct_cachep, GFP_KERNEL, node);
        slab_alloc_node();
            __slab_alloc() {
                // 1. try freelist again in case of cpu migration or IRQ
                freelist = get_freelist(s, page);

                // 2. replace cpu page with partial
                if (slub_percpu_partial(c)) {
                    page = c->page = slub_percpu_partial(c); // c->partial
                    slub_set_percpu_partial(c, page); // c->partical = page->next
                    stat(s, CPU_PARTIAL_ALLOC);
                    goto redo;
                }

                // 3. need alloc new slak objects
               freelist = new_slab_objects(struct kmem_cache *s, gfp_t flags,
                    int node, struct kmem_cache_cpu **pc)
                {
                    // 3.1. get partial from kmem_cache_node indexed by node
                    freelist = get_partial_node(struct kmem_cache *s, struct kmem_cache_node *n,
                            struct kmem_cache_cpu *c, gfp_t flags)
                    {
                        struct page *page, *page2;
                        void *object = NULL;
                        int available = 0;
                        int objects;
                        list_for_each_entry_safe(page, page2, &n->partial, lru) {
                            // get a page memory from kmem_cache_node, return its freelist
                            void *t = acquire_slab(s, n, page, object == NULL, &objects);
                            if (!t)
                            break;

                            available += objects;
                            if (!object) {
                                c->page = page;
                                stat(s, ALLOC_FROM_PARTIAL);
                                object = t;
                            } else {
                                put_cpu_partial(s, page, 0);
                                stat(s, CPU_PARTIAL_NODE);
                            }
                            if (!kmem_cache_has_cpu_partial(s)
                                || available > slub_cpu_partial(s) / 2)
                                break;
                        }
                        return object;
                    }
                    if (freelist)
                        return freelist;

                    // 3.2. no memory in kmem_cache_node, alloc new
                    page = new_slab(s, flags, node) {
                        page *allocate_slab(struct kmem_cache *s, gfp_t flags, int node)
                        {
                            struct page *page;
                            struct kmem_cache_order_objects oo = s->oo;
                            gfp_t alloc_gfp;
                            void *start, *p;
                            int idx, order;
                            bool shuffle;
                            flags &= gfp_allowed_mask;
                            page = alloc_slab_page(s, alloc_gfp, node, oo);
                            if (unlikely(!page)) {
                                oo = s->min;
                                alloc_gfp = flags;
                                page = alloc_slab_page(s, alloc_gfp, node, oo);
                                    __alloc_pages_node();
                                        __alloc_pages(gfp_mask, order, nid);
                                            __alloc_pages_nodemask();
                                                --->

                                if (unlikely(!page))
                                    goto out;
                                stat(s, ORDER_FALLBACK);
                            }
                            return page;
                        }
                    }
                    if (page) {
                        c = raw_cpu_ptr(s->cpu_slab);
                        if (c->page)
                            flush_slab(s, c);

                        freelist = page->freelist;
                        page->freelist = NULL;

                        stat(s, ALLOC_SLAB);
                        c->page = page;
                        *pc = c;
                    } else
                        freelist = NULL;
                    return freelis
                }
```

![linux-mem-buddy-slab-system.jpg](../Images/linux-mem-buddy-slab-system.jpg)

### kswapd

```C++
//1. actice page out when alloc
get_page_from_freelist();
    node_reclaim();
        __node_reclaim();
            shrink_node();

// 2. positive page out by kswapd
static int kswapd(void *p)
{
  unsigned int alloc_order, reclaim_order;
  unsigned int classzone_idx = MAX_NR_ZONES - 1;
  pg_data_t *pgdat = (pg_data_t*)p;
  struct task_struct *tsk = current;

    for ( ; ; ) {

        kswapd_try_to_sleep(pgdat, alloc_order, reclaim_order,
            classzone_idx);

        reclaim_order = balance_pgdat(pgdat, alloc_order, classzone_idx);

    }
}

balance_pgdat();
    kswapd_shrink_node();
        shrink_node();
            shrink_node_memcg();

```
![Linux Page State](../Images/linux-page-state.png)

```C++
// arch/x86/kernel/cpu/common.c
DEFINE_PER_CPU_PAGE_ALIGNED(struct gdt_page, gdt_page) = { .gdt = {
    [GDT_ENTRY_KERNEL32_CS]    = GDT_ENTRY_INIT(0xc09b, 0, 0xfffff),
    [GDT_ENTRY_KERNEL_CS]    = GDT_ENTRY_INIT(0xa09b, 0, 0xfffff),
    [GDT_ENTRY_KERNEL_DS]    = GDT_ENTRY_INIT(0xc093, 0, 0xfffff),
    [GDT_ENTRY_DEFAULT_USER32_CS]  = GDT_ENTRY_INIT(0xc0fb, 0, 0xfffff),
    [GDT_ENTRY_DEFAULT_USER_DS]  = GDT_ENTRY_INIT(0xc0f3, 0, 0xfffff),
    [GDT_ENTRY_DEFAULT_USER_CS]  = GDT_ENTRY_INIT(0xa0fb, 0, 0xfffff),
}};

#define __KERNEL_CS (GDT_ENTRY_KERNEL_CS*8)
#define __KERNEL_DS (GDT_ENTRY_KERNEL_DS*8)
#define __USER_DS (GDT_ENTRY_DEFAULT_USER_DS*8 + 3)
#define __USER_CS (GDT_ENTRY_DEFAULT_USER_CS*8 + 3)

```

# IPC
### PIPE
```C++
SYSCALL_DEFINE1(pipe, int __user *, fildes)
{
  return sys_pipe2(fildes, 0);
}

SYSCALL_DEFINE2(pipe2, int __user *, fildes, int, flags)
{
  struct file *files[2];
  int fd[2];
  int error;

  error = __do_pipe_flags(fd, files, flags);
  if (!error) {
    if (unlikely(copy_to_user(fildes, fd, sizeof(fd)))) {
      error = -EFAULT;
    } else {
      fd_install(fd[0], files[0]);
      fd_install(fd[1], files[1]);
    }
  }
  return error;
}


static int __do_pipe_flags(int *fd, struct file **files, int flags)
{
    int error;
    int fdw, fdr;
    error = create_pipe_files(files, flags);
    error = get_unused_fd_flags(flags);
    fdr = error;

    error = get_unused_fd_flags(flags);
    fdw = error;

    fd[0] = fdr;
    fd[1] = fdw;
    return 0;
}


int create_pipe_files(struct file **res, int flags)
{
    int err;
    struct inode *inode = get_pipe_inode();
    struct file *f;
    struct path path;
    path.dentry = d_alloc_pseudo(pipe_mnt->mnt_sb, &empty_name);
    path.mnt = mntget(pipe_mnt);

    d_instantiate(path.dentry, inode);

    f = alloc_file(&path, FMODE_WRITE, &pipefifo_fops);
    f->f_flags = O_WRONLY | (flags & (O_NONBLOCK | O_DIRECT));
    f->private_data = inode->i_pipe;

    res[0] = alloc_file(&path, FMODE_READ, &pipefifo_fops);
    path_get(&path);
    res[0]->private_data = inode->i_pipe;
    res[0]->f_flags = O_RDONLY | (flags & O_NONBLOCK);
    res[1] = f;
    return 0;
}

static struct file_system_type pipe_fs_type = {
  .name    = "pipefs",
  .mount    = pipefs_mount,
  .kill_sb  = kill_anon_super,
};

static int __init init_pipe_fs(void)
{
  int err = register_filesystem(&pipe_fs_type);

  if (!err) {
    pipe_mnt = kern_mount(&pipe_fs_type);
  }
}

static struct inode * get_pipe_inode(void)
{
  struct inode *inode = new_inode_pseudo(pipe_mnt->mnt_sb);
  struct pipe_inode_info *pipe;
  inode->i_ino = get_next_ino();

  pipe = alloc_pipe_info();
  inode->i_pipe = pipe;
  pipe->files = 2;
  pipe->readers = pipe->writers = 1;
  inode->i_fop = &pipefifo_fops;
  inode->i_state = I_DIRTY;
  inode->i_mode = S_IFIFO | S_IRUSR | S_IWUSR;
  inode->i_uid = current_fsuid();
  inode->i_gid = current_fsgid();
  inode->i_atime = inode->i_mtime = inode->i_ctime = current_time(inode);

  return inode;
}
```

### sem, shm, msg
```C++
struct ipc_namespace {
  struct ipc_ids  ids[3];
}

#define IPC_SEM_IDS  0
#define IPC_MSG_IDS  1
#define IPC_SHM_IDS  2

#define sem_ids(ns)  ((ns)->ids[IPC_SEM_IDS])
#define msg_ids(ns)  ((ns)->ids[IPC_MSG_IDS])
#define shm_ids(ns)  ((ns)->ids[IPC_SHM_IDS])


struct ipc_ids {
  int in_use;
  unsigned short seq;
  struct rw_semaphore rwsem;
  struct idr ipcs_idr;
  int next_id;
};

struct idr {
  struct radix_tree_root  idr_rt;
  unsigned int    idr_next;
};
```
![ipc_ids](../Images/linux-ipc-ipc_ids.png)
```C++
struct kern_ipc_perm *ipc_obtain_object_idr(struct ipc_ids *ids, int id)
{
  struct kern_ipc_perm *out;
  int lid = ipcid_to_idx(id);
  out = idr_find(&ids->ipcs_idr, lid);
  return out;
}

static inline struct sem_array *sem_obtain_object(struct ipc_namespace *ns, int id)
{
  struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&sem_ids(ns), id);
  return container_of(ipcp, struct sem_array, sem_perm);
}

static inline struct msg_queue *msq_obtain_object(struct ipc_namespace *ns, int id)
{
  struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&msg_ids(ns), id);
  return container_of(ipcp, struct msg_queue, q_perm);
}

static inline struct shmid_kernel *shm_obtain_object(struct ipc_namespace *ns, int id)
{
  struct kern_ipc_perm *ipcp = ipc_obtain_object_idr(&shm_ids(ns), id);
  return container_of(ipcp, struct shmid_kernel, shm_perm);
}

SYSCALL_DEFINE3(shmget, key_t, key, size_t, size, int, shmflg)
{
  struct ipc_namespace *ns;
  static const struct ipc_ops shm_ops = {
    .getnew = newseg,
    .associate = shm_security,
    .more_checks = shm_more_checks,
  };
  struct ipc_params shm_params;
  ns = current->nsproxy->ipc_ns;
  shm_params.key = key;
  shm_params.flg = shmflg;
  shm_params.u.size = size;
  return ipcget(ns, &shm_ids(ns), &shm_ops, &shm_params);
}


int ipcget(struct ipc_namespace *ns, struct ipc_ids *ids,
    const struct ipc_ops *ops, struct ipc_params *params)
{
  if (params->key == IPC_PRIVATE)
    return ipcget_new(ns, ids, ops, params);
  else
    return ipcget_public(ns, ids, ops, params);
}

static int ipcget_public(struct ipc_namespace *ns, struct ipc_ids *ids,
    const struct ipc_ops *ops, struct ipc_params *params)
{
  struct kern_ipc_perm *ipcp;
  int flg = params->flg;
  int err;
  ipcp = ipc_findkey(ids, params->key);
  if (ipcp == NULL) {
    if (!(flg & IPC_CREAT))
      err = -ENOENT;
    else
      err = ops->getnew(ns, params);
  } else {
    if (flg & IPC_CREAT && flg & IPC_EXCL)
      err = -EEXIST;
    else {
      err = 0;
      if (ops->more_checks)
        err = ops->more_checks(ipcp, params);
    }
  }
  return err;
}


static int newseg(struct ipc_namespace *ns, struct ipc_params *params)
{
  key_t key = params->key;
  int shmflg = params->flg;
  size_t size = params->u.size;
  int error;
  struct shmid_kernel *shp;
  size_t numpages = (size + PAGE_SIZE - 1) >> PAGE_SHIFT;
  struct file *file;
  char name[13];
  vm_flags_t acctflag = 0;

  shp = kvmalloc(sizeof(*shp), GFP_KERNEL);
  shp->shm_perm.key = key;
  shp->shm_perm.mode = (shmflg & S_IRWXUGO);
  shp->mlock_user = NULL;

  shp->shm_perm.security = NULL;
  file = shmem_kernel_file_setup(name, size, acctflag);
  shp->shm_cprid = task_tgid_vnr(current);
  shp->shm_lprid = 0;
  shp->shm_atim = shp->shm_dtim = 0;
  shp->shm_ctim = get_seconds();
  shp->shm_segsz = size;
  shp->shm_nattch = 0;
  shp->shm_file = file;
  shp->shm_creator = current;

  error = ipc_addid(&shm_ids(ns), &shp->shm_perm, ns->shm_ctlmni);
  list_add(&shp->shm_clist, &current->sysvshm.shm_clist);
  file_inode(file)->i_ino = shp->shm_perm.id;

  ns->shm_tot += numpages;
  error = shp->shm_perm.id;
  return error;
}

int __init shmem_init(void)
{
  int error;
  error = shmem_init_inodecache();
  error = register_filesystem(&shmem_fs_type);
  shm_mnt = kern_mount(&shmem_fs_type);
  return 0;
}

static struct file_system_type shmem_fs_type = {
  .owner    = THIS_MODULE,
  .name    = "tmpfs",
  .mount    = shmem_mount,
  .kill_sb  = kill_litter_super,
  .fs_flags  = FS_USERNS_MOUNT,
};

struct file *shmem_kernel_file_setup(const char *name, loff_t size, unsigned long flags)
{
  return __shmem_file_setup(name, size, flags, S_PRIVATE);
}

static struct file *__shmem_file_setup(const char *name, loff_t size,
               unsigned long flags, unsigned int i_flags)
{
  struct file *res;
  struct inode *inode;
  struct path path;
  struct super_block *sb;
  struct qstr this;

  this.name = name;
  this.len = strlen(name);
  this.hash = 0; /* will go */
  sb = shm_mnt->mnt_sb;
  path.mnt = mntget(shm_mnt);
  path.dentry = d_alloc_pseudo(sb, &this);
  d_set_d_op(path.dentry, &anon_ops);

  inode = shmem_get_inode(sb, NULL, S_IFREG | S_IRWXUGO, 0, flags);
  inode->i_flags |= i_flags;
  d_instantiate(path.dentry, inode);
  inode->i_size = size;

  res = alloc_file(&path, FMODE_WRITE | FMODE_READ,
      &shmem_file_operations);
  return res;
}

static const struct file_operations shmem_file_operations = {
  .mmap    = shmem_mmap,
  .get_unmapped_area = shmem_get_unmapped_area,
#ifdef CONFIG_TMPFS
  .llseek    = shmem_file_llseek,
  .read_iter  = shmem_file_read_iter,
  .write_iter  = generic_file_write_iter,
  .fsync    = noop_fsync,
  .splice_read  = generic_file_splice_read,
  .splice_write  = iter_file_splice_write,
  .fallocate  = shmem_fallocate,
#endif
};


SYSCALL_DEFINE3(shmat, int, shmid, char __user *, shmaddr, int, shmflg)
{
    unsigned long ret;
    long err;
    err = do_shmat(shmid, shmaddr, shmflg, &ret, SHMLBA);
    force_successful_syscall_return();
    return (long)ret;
}

long do_shmat(int shmid, char __user *shmaddr, int shmflg,
        ulong *raddr, unsigned long shmlba)
{
  struct shmid_kernel *shp;
  unsigned long addr = (unsigned long)shmaddr;
  unsigned long size;
  struct file *file;
  int    err;
  unsigned long flags = MAP_SHARED;
  unsigned long prot;
  int acc_mode;
  struct ipc_namespace *ns;
  struct shm_file_data *sfd;
  struct path path;
  fmode_t f_mode;
  unsigned long populate = 0;

  prot = PROT_READ | PROT_WRITE;
  acc_mode = S_IRUGO | S_IWUGO;
  f_mode = FMODE_READ | FMODE_WRITE;

  ns = current->nsproxy->ipc_ns;
  shp = shm_obtain_object_check(ns, shmid);

  path = shp->shm_file->f_path;
  path_get(&path);
  shp->shm_nattch++;
  size = i_size_read(d_inode(path.dentry));

  sfd = kzalloc(sizeof(*sfd), GFP_KERNEL);

  file = alloc_file(&path, f_mode,
        is_file_hugepages(shp->shm_file) ?
        &shm_file_operations_huge :
        &shm_file_operations);

  file->private_data = sfd;
  file->f_mapping = shp->shm_file->f_mapping;
  sfd->id = shp->shm_perm.id;
  sfd->ns = get_ipc_ns(ns);
  sfd->file = shp->shm_file;
  sfd->vm_ops = NULL;

  addr = do_mmap_pgoff(file, addr, size, prot, flags, 0, &populate, NULL);
  *raddr = addr;
  err = 0;

  return err;
}


static int shm_mmap(struct file *file, struct vm_area_struct *vma)
{
  struct shm_file_data *sfd = shm_file_data(file);
  int ret;
  ret = __shm_open(vma);
  ret = call_mmap(sfd->file, vma);
  sfd->vm_ops = vma->vm_ops;
  vma->vm_ops = &shm_vm_ops;
  return 0;
}

static int shmem_mmap(struct file *file, struct vm_area_struct *vma)
{
  file_accessed(file);
  vma->vm_ops = &shmem_vm_ops;
  return 0;
}

static const struct vm_operations_struct shm_vm_ops = {
  .open  = shm_open,  /* callback for a new vm-area open */
  .close  = shm_close,  /* callback for when the vm-area is released */
  .fault  = shm_fault,
};

static const struct vm_operations_struct shmem_vm_ops = {
  .fault    = shmem_fault,
  .map_pages  = filemap_map_pages,
};

static int shm_fault(struct vm_fault *vmf)
{
  struct file *file = vmf->vma->vm_file;
  struct shm_file_data *sfd = shm_file_data(file);
  return sfd->vm_ops->fault(vmf);
}

static int shmem_fault(struct vm_fault *vmf)
{
  struct vm_area_struct *vma = vmf->vma;
  struct inode *inode = file_inode(vma->vm_file);
  gfp_t gfp = mapping_gfp_mask(inode->i_mapping);

  error = shmem_getpage_gfp(inode, vmf->pgoff, &vmf->page, sgp,
          gfp, vma, vmf, &ret);
}

static int shmem_getpage_gfp(struct inode *inode, pgoff_t index,
  struct page **pagep, enum sgp_type sgp, gfp_t gfp,
  struct vm_area_struct *vma, struct vm_fault *vmf, int *fault_type)
{
  page = shmem_alloc_and_acct_page(gfp, info, sbinfo,
        index, false);
}
```
![linux-ipc-sem-shm-msg.png](../Images/linux-ipc-sem-shm-msg.png)


# File Management
![linux-file-vfs-system.png](../Images/linux-file-vfs-system.png)

### inode, extents
```C++
struct inode {
  const struct inode_operations  *i_op;
  struct super_block  *i_sb;
  struct address_space  *i_mapping;

  union {
    const struct file_operations  *i_fop;  /* former ->i_op->default_file_ops */
    void (*free_inode)(struct inode *);
  };

  dev_t      i_rdev;
  struct list_head  i_devices;
  union {
    struct pipe_inode_info  *i_pipe;
    struct block_device  *i_bdev;
    struct cdev    *i_cdev;
    char      *i_link;
    unsigned    i_dir_seq;
  };

  umode_t      i_mode;
  unsigned short    i_opflags;
  kuid_t      i_uid;
  kgid_t      i_gid;
  unsigned int    i_flags;

  unsigned long    dirtied_when;  /* jiffies of first dirtying */
  unsigned long    dirtied_time_when;

  struct hlist_node  i_hash;
  struct list_head  i_io_list;  /* backing dev IO list */
  struct bdi_writeback  *i_wb;    /* the associated cgroup wb */

  struct file_lock_context  *i_flctx;
  struct address_space  i_data;
};

struct ext4_inode {
  __le16  i_mode;    /* File mode */
  __le16  i_uid;    /* Low 16 bits of Owner Uid */
  __le32  i_size_lo;  /* Size in bytes */
  __le32  i_atime;  /* Access time */
  __le32  i_ctime;  /* Inode Change time */
  __le32  i_mtime;  /* Modification time */
  __le32  i_dtime;  /* Deletion Time */
  __le16  i_gid;    /* Low 16 bits of Group Id */
  __le16  i_links_count;  /* Links count */
  __le32  i_blocks_lo;  /* Blocks count */
  __le32  i_flags;  /* File flags */

  __le32  i_block[EXT4_N_BLOCKS];/* Pointers to blocks */
  __le32  i_generation;  /* File version (for NFS) */
  __le32  i_file_acl_lo;  /* File ACL */
  __le32  i_size_high;

};

// Each block (leaves and indexes), even inode-stored has header.
struct ext4_extent_header {
  __le16  eh_magic;  /* probably will support different formats */
  __le16  eh_entries;  /* number of valid entries */
  __le16  eh_max;    /* capacity of store in entries */
  __le16  eh_depth;  /* has tree real underlying blocks? */
  __le32  eh_generation;  /* generation of the tree */
};

struct ext4_extent_idx {
  __le32  ei_block;  /* index covers logical blocks from 'block' */
  __le32  ei_leaf_lo;  /* pointer to the physical block of the next *
         * level. leaf or next index could be there */
  __le16  ei_leaf_hi;  /* high 16 bits of physical block */
  __u16  ei_unused;
};

struct ext4_extent {
  __le32  ee_block;  /* first logical block extent covers */
  // the most significant bit used as a flag to identify whether
  // this entent is initialized, 15 bits can present max 128MB data
  __le16  ee_len;    /* number of blocks covered by extent */
  __le16  ee_start_hi;  /* high 16 bits of physical block */
  __le32  ee_start_lo;  /* low 32 bits of physical block */
};
```
![linux-mem-extents.jpg](../Images/linux-mem-extents.jpg)

![linux-ext4-extents.png](../Images/linux-ext4-extents.png)

```C++
const struct inode_operations ext4_dir_inode_operations = {
  .create    = ext4_create,
  .lookup    = ext4_lookup,
  .link    = ext4_link,
  .unlink    = ext4_unlink,
  .symlink  = ext4_symlink,
  .mkdir    = ext4_mkdir,
  .rmdir    = ext4_rmdir,
  .mknod    = ext4_mknod,
  .tmpfile  = ext4_tmpfile,
  .rename    = ext4_rename2,
  .setattr  = ext4_setattr,
  .getattr  = ext4_getattr,
  .listxattr  = ext4_listxattr,
  .get_acl  = ext4_get_acl,
  .set_acl  = ext4_set_acl,
  .fiemap         = ext4_fiemap,
};

// ext4_create->ext4_new_inode_start_handle->__ext4_new_inode
struct inode *__ext4_new_inode(handle_t *handle, struct inode *dir,
             umode_t mode, const struct qstr *qstr,
             __u32 goal, uid_t *owner, __u32 i_flags,
             int handle_type, unsigned int line_no,
             int nblocks)
{
inode_bitmap_bh = ext4_read_inode_bitmap(sb, group);
ino = ext4_find_next_zero_bit((unsigned long *)
                inode_bitmap_bh->b_data,
                EXT4_INODES_PER_GROUP(sb), ino);
}
```

### Meta Block Group
```C++
struct ext4_group_desc
{
  __le32  bg_block_bitmap_lo;  /* Blocks bitmap block */
  __le32  bg_inode_bitmap_lo;  /* Inodes bitmap block */
  __le32  bg_inode_table_lo;  /* Inodes table block */
};
```
![linux-block-group.jpg](../Images/linux-block-group.jpg)

```C++
struct ext4_super_block {
  __le32  s_blocks_count_lo;  /* Blocks count */
  __le32  s_r_blocks_count_lo;  /* Reserved blocks count */
  __le32  s_free_blocks_count_lo;  /* Free blocks count */

  __le32  s_blocks_count_hi;  /* Blocks count */
  __le32  s_r_blocks_count_hi;  /* Reserved blocks count */
  __le32  s_free_blocks_count_hi;  /* Free blocks count */
}
```
![linux-mem-meta-block-group.jpg](../Images/linux-mem-meta-block-group.jpg)

### Directory
```C++
struct ext4_dir_entry {
  __le32  inode;      /* Inode number */
  __le16  rec_len;    /* Directory entry length */
  __le16  name_len;    /* Name length */
  char  name[EXT4_NAME_LEN];  /* File name */
};
struct ext4_dir_entry_2 {
  __le32  inode;      /* Inode number */
  __le16  rec_len;    /* Directory entry length */
  __u8  name_len;    /* Name length */
  __u8  file_type;
  char  name[EXT4_NAME_LEN];  /* File name */
};

struct dx_root
{
  struct fake_dirent dot;
  char dot_name[4];
  struct fake_dirent dotdot;
  char dotdot_name[4];
  struct dx_root_info
  {
    __le32 reserved_zero;
    u8 hash_version;
    u8 info_length; /* 8 */
    u8 indirect_levels;
    u8 unused_flags;
  }
  info;
  struct dx_entry  entries[0];
};

struct dx_entry
{
  __le32 hash;
  __le32 block;
};
```
![linux-directory.jpg](../Images/linux-directory.jpg)

![linux-dir-file-inode.png](../Images/linux-dir-file-inode.png)

### hard/symbolic link
```C++
 ln [args] [dst] [src]
```
![linux-file-link.jpg](../Images/linux-file-link.jpg)

### vfs
![linux-file-arche.jpg](../Images/linux-file-arche.jpg)
```C++
register_filesystem(&ext4_fs_type);

static struct file_system_type ext4_fs_type = {
  .owner    = THIS_MODULE,
  .name    = "ext4",
  .mount    = ext4_mount,
  .kill_sb  = kill_block_super,
  .fs_flags  = FS_REQUIRES_DEV,
};
```

### mount

```C++

SYSCALL_DEFINE5(mount, char __user *, dev_name, char __user *, dir_name, char __user *, type, unsigned long, flags, void __user *, data)
{
  ret = do_mount(kernel_dev, dir_name, kernel_type, flags, options);
}
// do_mount->do_new_mount->vfs_kern_mount

struct vfsmount *
vfs_kern_mount(struct file_system_type *type, int flags, const char *name, void *data)
{
  mnt = alloc_vfsmnt(name);
  root = mount_fs(type, flags, name, data);

  mnt->mnt.mnt_root = root;
  mnt->mnt.mnt_sb = root->d_sb;
  mnt->mnt_mountpoint = mnt->mnt.mnt_root;
  mnt->mnt_parent = mnt;
  list_add_tail(&mnt->mnt_instance, &root->d_sb->s_mounts);
  return &mnt->mnt;
}

struct mount {
  struct hlist_node mnt_hash;
  struct mount *mnt_parent;
  struct dentry *mnt_mountpoint;
  struct vfsmount mnt;

  union {
    struct rcu_head mnt_rcu;
    struct llist_node mnt_llist;
  };
  struct list_head mnt_mounts;  /* list of children, anchored here */
  struct list_head mnt_child;  /* and going through their mnt_child */
  struct list_head mnt_instance;  /* mount instance on sb->s_mounts */
  const char *mnt_devname;  /* Name of device e.g. /dev/dsk/hda1 */
  struct list_head mnt_list;
} __randomize_layout;

struct vfsmount {
  struct dentry *mnt_root;  /* root of the mounted tree */
  struct super_block *mnt_sb;  /* pointer to superblock */
  int mnt_flags;
} __randomize_layout;

struct dentry *
mount_fs(struct file_system_type *type, int flags, const char *name, void *data)
{
  struct dentry *root;
  struct super_block *sb;
  root = type->mount(type, flags, name, data);
  sb = root->d_sb;
}
```
![linux-io-mount-example.jpg](../Images/linux-io-mount-example.jpg)

```C++
struct file {
  struct path    f_path;
  struct inode    *f_inode;  /* cached value */
  const struct file_operations  *f_op;
  struct address_space  *f_mapping;

  spinlock_t                    f_lock;
  enum rw_hint                  f_write_hint;
  atomic_long_t                 f_count;
  unsigned int                  f_flags;
  fmode_t                       f_mode;
  loff_t                         f_pos;
  struct mutex                  f_pos_lock;
  struct fown_struct            f_owner;

  errseq_t    f_wb_err;
  union {
      struct llist_node  fu_llist;
      struct rcu_head   fu_rcuhead;
  } f_u;
};

struct path {
  struct vfsmount *mnt;
  struct dentry *dentry;
};

// memroy chache of dirctories and files
struct dentry {
  unsigned int d_flags;    /* protected by d_lock */
  struct dentry *d_parent;  /* parent directory */
  struct inode *d_inode;    /* Where the name belongs to - NULL is
           * negative */
  const struct dentry_operations *d_op;
  struct super_block *d_sb;  /* The root of the dentry tree */

  struct hlist_bl_node d_hash;  /* lookup hash list */
  union {
    struct list_head d_lru;    /* LRU list */
    wait_queue_head_t *d_wait;  /* in-lookup ones only */
  };
  struct qstr d_name;
  unsigned char d_iname[DNAME_INLINE_LEN];  /* small names */
  struct list_head d_child;  /* child of parent list */
  struct list_head d_subdirs;  /* our children */
} __randomize_layout;
```

### open
```C++
struct task_struct {
  struct fs_struct    *fs;
  struct files_struct *files;
  struct nsproxy  *nsproxy;
}

struct files_struct {
  struct file __rcu * fd_array[NR_OPEN_DEFAULT];
};

SYSCALL_DEFINE3(open, const char __user *, filename, int, flags, umode_t, mode)
{
  return do_sys_open(AT_FDCWD, filename, flags, mode);
}

long do_sys_open(int dfd, const char __user *filename, int flags, umode_t mode)
{
  fd = get_unused_fd_flags(flags);
  if (fd >= 0) {
    struct file *f = do_filp_open(dfd, tmp, &op);
    if (IS_ERR(f)) {
      put_unused_fd(fd);
      fd = PTR_ERR(f);
    } else {
      fsnotify_open(f);
      fd_install(fd, f);
    }
  }
  putname(tmp);
  return fd;
}

struct file *do_filp_open(int dfd, struct filename *pathname,
    const struct open_flags *op)
{
  set_nameidata(&nd, dfd, pathname);
  filp = path_openat(&nd, op, flags | LOOKUP_RCU);
  restore_nameidata();
  return filp;
}


static struct file *path_openat(struct nameidata *nd,
      const struct open_flags *op, unsigned flags)
{
  file = get_empty_filp();
  s = path_init(nd, flags);
  while (!(error = link_path_walk(s, nd)) &&
    (error = do_last(nd, file, op, &opened)) > 0) {
  }
  terminate_walk(nd);
  return file;
}

static int do_last(struct nameidata *nd,
       struct file *file, const struct open_flags *op,
       int *opened)
{
  error = lookup_fast(nd, &path, &inode, &seq); // loopup in dcache
  error = lookup_open(nd, &path, file, op, got_write, opened);
  error = vfs_open(&nd->path, file, current_cred());
}

static int lookup_open(struct nameidata *nd, struct path *path,
  struct file *file,
  const struct open_flags *op,
  bool got_write, int *opened)
{
  // open with O_CREAT flag
  if (!dentry->d_inode && (open_flag & O_CREAT)) {
    error = dir_inode->i_op->create(dir_inode, dentry, mode,
            open_flag & O_EXCL);
  }

  dentry = d_alloc_parallel(dir, &nd->last, &wq);
  struct dentry *res = dir_inode->i_op->lookup(dir_inode, dentry,
                  nd->flags);
  path->dentry = dentry;
  path->mnt = nd->path.mnt;
}

const struct inode_operations ext4_dir_inode_operations = {
  .create    = ext4_create,
  .lookup    = ext4_lookup
}

int vfs_open(const struct path *path, struct file *file,
  const struct cred *cred)
{
  struct dentry *dentry = d_real(path->dentry, NULL, file->f_flags, 0);
  file->f_path = *path;
  return do_dentry_open(file, d_backing_inode(dentry), NULL, cred);
}

static int do_dentry_open(struct file *f,
  struct inode *inode,
  int (*open)(struct inode *, struct file *),
  const struct cred *cred)
{
  f->f_mode = OPEN_FMODE(f->f_flags) | FMODE_LSEEK |
        FMODE_PREAD | FMODE_PWRITE;
  path_get(&f->f_path);
  f->f_inode = inode;
  f->f_mapping = inode->i_mapping;
  f->f_op = fops_get(inode->i_fop);
  open = f->f_op->open;
  error = open(inode, f);
  f->f_flags &= ~(O_CREAT | O_EXCL | O_NOCTTY | O_TRUNC);
  file_ra_state_init(&f->f_ra, f->f_mapping->host->i_mapping);
  return 0;
}


const struct file_operations ext4_file_operations = {
  .open    = ext4_file_open,
};

```
![linux-dcache.jpg](../Images/linux-dcache.jpg)

### read/write
```C++
SYSCALL_DEFINE3(read, unsigned int, fd, char __user *, buf, size_t, count)
{
  struct fd f = fdget_pos(fd);
  loff_t pos = file_pos_read(f.file);
  ret = vfs_read(f.file, buf, count, &pos);

}

SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
    size_t, count)
{
  struct fd f = fdget_pos(fd);
  loff_t pos = file_pos_read(f.file);
    ret = vfs_write(f.file, buf, count, &pos);
}

ssize_t __vfs_read(struct file *file, char __user *buf, size_t count,
       loff_t *pos)
{
  if (file->f_op->read)
    return file->f_op->read(file, buf, count, pos);
  else if (file->f_op->read_iter)
    return new_sync_read(file, buf, count, pos);
  else
    return -EINVAL;
}


ssize_t __vfs_write(struct file *file, const char __user *p, size_t count,
        loff_t *pos)
{
  if (file->f_op->write)
    return file->f_op->write(file, p, count, pos);
  else if (file->f_op->write_iter)
    return new_sync_write(file, p, count, pos);
  else
    return -EINVAL;
}

const struct file_operations ext4_file_operations = {
  .read_iter  = ext4_file_read_iter,
  .write_iter  = ext4_file_write_iter,
  .write_begin = ext4_write_begin,
  .write_end = ext4_write_end
}
// ext4_file_{read, write}_iter->generic_file_{read, write}_iter
ssize_t
generic_file_read_iter(struct kiocb *iocb, struct iov_iter *iter)
{
    if (iocb->ki_flags & IOCB_DIRECT) {
        struct address_space *mapping = file->f_mapping;
        retval = mapping->a_ops->direct_IO(iocb, iter);
    }
    retval = generic_file_buffered_read(iocb, iter, retval);
}

ssize_t __generic_file_write_iter(struct kiocb *iocb, struct iov_iter *from)
{
    if (iocb->ki_flags & IOCB_DIRECT) {
        written = generic_file_direct_write(iocb, from);
    } else {
        written = generic_perform_write(file, from, iocb->ki_pos);
    }
}
```

#### direct IO
```C++
static const struct address_space_operations ext4_aops = {
  .direct_IO    = ext4_direct_IO,
};

static ssize_t ext4_direct_IO(struct kiocb *iocb, struct iov_iter *iter)
{
  struct file *file = iocb->ki_filp;
  struct inode *inode = file->f_mapping->host;
  size_t count = iov_iter_count(iter);
  loff_t offset = iocb->ki_pos;
  ssize_t ret;
  ret = ext4_direct_IO_write(iocb, iter);
}

static ssize_t ext4_direct_IO_write(struct kiocb *iocb, struct iov_iter *iter)
{
  struct file *file = iocb->ki_filp;
  struct inode *inode = file->f_mapping->host;
  struct ext4_inode_info *ei = EXT4_I(inode);
  ssize_t ret;
  loff_t offset = iocb->ki_pos;
  size_t count = iov_iter_count(iter);

  ret = __blockdev_direct_IO(iocb, inode, inode->i_sb->s_bdev, iter,
           get_block_func, ext4_end_io_dio, NULL,
           dio_flags);
}

// __blockdev_direct_IO->do_blockdev_direct_IO
static inline ssize_t
do_blockdev_direct_IO(struct kiocb *iocb, struct inode *inode,
  struct block_device *bdev, struct iov_iter *iter,
  get_block_t get_block, dio_iodone_t end_io,
  dio_submit_t submit_io, int flags)
{
  unsigned i_blkbits = ACCESS_ONCE(inode->i_blkbits);
  unsigned blkbits = i_blkbits;
  unsigned blocksize_mask = (1 << blkbits) - 1;
  ssize_t retval = -EINVAL;
  size_t count = iov_iter_count(iter);
  loff_t offset = iocb->ki_pos;
  loff_t end = offset + count;
  struct dio *dio;
  struct dio_submit sdio = { 0, };
  struct buffer_head map_bh = { 0, };

  dio = kmem_cache_alloc(dio_cache, GFP_KERNEL);
  dio->flags = flags;
  dio->i_size = i_size_read(inode);
  dio->inode = inode;
  if (iov_iter_rw(iter) == WRITE) {
    dio->op = REQ_OP_WRITE;
    dio->op_flags = REQ_SYNC | REQ_IDLE;
    if (iocb->ki_flags & IOCB_NOWAIT)
      dio->op_flags |= REQ_NOWAIT;
  } else {
    dio->op = REQ_OP_READ;
  }
  sdio.blkbits = blkbits;
  sdio.blkfactor = i_blkbits - blkbits;
  sdio.block_in_file = offset >> blkbits;

  sdio.get_block = get_block;
  dio->end_io = end_io;
  sdio.submit_io = submit_io;
  sdio.final_block_in_bio = -1;
  sdio.next_block_for_io = -1;

  dio->iocb = iocb;
  dio->refcount = 1;

  sdio.iter = iter;
  sdio.final_block_in_request =
    (offset + iov_iter_count(iter)) >> blkbits;

  sdio.pages_in_io += iov_iter_npages(iter, INT_MAX);

  retval = do_direct_IO(dio, &sdio, &map_bh);
}

static int do_direct_IO(struct dio *dio, struct dio_submit *sdio,
      struct buffer_head *map_bh)
{
  const unsigned blkbits = sdio->blkbits;
  const unsigned i_blkbits = blkbits + sdio->blkfactor;
  int ret = 0;

  while (sdio->block_in_file < sdio->final_block_in_request) {
    struct page *page;
    size_t from, to;

    page = dio_get_page(dio, sdio);
        from = sdio->head ? 0 : sdio->from;
    to = (sdio->head == sdio->tail - 1) ? sdio->to : PAGE_SIZE;
    sdio->head++;

    while (from < to) {
      unsigned this_chunk_bytes;  /* # of bytes mapped */
      unsigned this_chunk_blocks;  /* # of blocks */
            ret = submit_page_section(dio, sdio, page,
              from,
              this_chunk_bytes,
              sdio->next_block_for_io,
              map_bh);

      sdio->next_block_for_io += this_chunk_blocks;
      sdio->block_in_file += this_chunk_blocks;
      from += this_chunk_bytes;
      dio->result += this_chunk_bytes;
      sdio->blocks_available -= this_chunk_blocks;
      if (sdio->block_in_file == sdio->final_block_in_request)
        break;
      }
  }
}

blk_qc_t submit_bio(struct bio *bio)
{
  return generic_make_request(bio);
}

blk_qc_t generic_make_request(struct bio *bio)
{
  /*
   * bio_list_on_stack[0] contains bios submitted by the current
   * make_request_fn.
   * bio_list_on_stack[1] contains bios that were submitted before
   * the current make_request_fn, but that haven't been processed
   * yet.
   */
  struct bio_list bio_list_on_stack[2];
  blk_qc_t ret = BLK_QC_T_NONE;

  if (current->bio_list) {
    bio_list_add(&current->bio_list[0], bio);
    goto out;
  }

  bio_list_init(&bio_list_on_stack[0]);
  current->bio_list = bio_list_on_stack;
  do {
    struct request_queue *q = bdev_get_queue(bio->bi_bdev);

    if (likely(blk_queue_enter(q, bio->bi_opf & REQ_NOWAIT) == 0)) {
      struct bio_list lower, same;

      /* Create a fresh bio_list for all subordinate requests */
      bio_list_on_stack[1] = bio_list_on_stack[0];
      bio_list_init(&bio_list_on_stack[0]);
      ret = q->make_request_fn(q, bio);

      blk_queue_exit(q);

      /* sort new bios into those for a lower level
       * and those for the same level
       */
      bio_list_init(&lower);
      bio_list_init(&same);
      while ((bio = bio_list_pop(&bio_list_on_stack[0])) != NULL)
        if (q == bdev_get_queue(bio->bi_bdev))
          bio_list_add(&same, bio);
        else
          bio_list_add(&lower, bio);
      /* now assemble so we handle the lowest level first */
      bio_list_merge(&bio_list_on_stack[0], &lower);
      bio_list_merge(&bio_list_on_stack[0], &same);
      bio_list_merge(&bio_list_on_stack[0], &bio_list_on_stack[1]);
    }

    bio = bio_list_pop(&bio_list_on_stack[0]);
  } while (bio);
  current->bio_list = NULL; /* deactivate */
out:
  return ret;
}

struct request_queue {
  // Together with queue_head for cacheline sharing
  struct list_head  queue_head;
  struct request    *last_merge;
  struct elevator_queue  *elevator;
  request_fn_proc    *request_fn;
  make_request_fn    *make_request_fn;
}

struct request {
  struct list_head queuelist;
  struct request_queue *q;
  struct bio *bio;
  struct bio *biotail;
}

struct bio {
  struct bio    *bi_next;  /* request queue link */
  struct block_device  *bi_bdev;
  blk_status_t    bi_status;
  struct bvec_iter  bi_iter;
  unsigned short    bi_vcnt;  /* how many bio_vec's */
  unsigned short    bi_max_vecs;  /* max bvl_vecs we can hold */
  atomic_t    __bi_cnt;  /* pin count */
  struct bio_vec    *bi_io_vec;  /* the actual vec list */

};

struct bio_vec {
  struct page  *bv_page;
  unsigned int  bv_len;
  unsigned int  bv_offset;
}
```
![linux-io-bio.jpg](../Images/linux-io-bio.jpg)


##### init block device
```C++
static struct scsi_device *scsi_alloc_sdev(struct scsi_target *starget,
             u64 lun, void *hostdata)
{
  struct scsi_device *sdev;
  sdev = kzalloc(sizeof(*sdev) + shost->transportt->device_size,
           GFP_ATOMIC);
  sdev->request_queue = scsi_alloc_queue(sdev);
}

struct request_queue *scsi_alloc_queue(struct scsi_device *sdev)
{
  struct Scsi_Host *shost = sdev->host;
  struct request_queue *q;

  q = blk_alloc_queue_node(GFP_KERNEL, NUMA_NO_NODE);
  if (!q)
    return NULL;
  q->cmd_size = sizeof(struct scsi_cmnd) + shost->hostt->cmd_size;
  q->rq_alloc_data = shost;
  q->request_fn = scsi_request_fn;
  q->init_rq_fn = scsi_init_rq;
  q->exit_rq_fn = scsi_exit_rq;
  q->initialize_rq_fn = scsi_initialize_rq;

  if (blk_init_allocated_queue(q) < 0) {
    blk_cleanup_queue(q);
    return NULL;
  }
  __scsi_init_queue(shost, q);
  return q
}

int blk_init_allocated_queue(struct request_queue *q)
{
  q->fq = blk_alloc_flush_queue(q, NUMA_NO_NODE, q->cmd_size);
  blk_queue_make_request(q, blk_queue_bio);
  /* init elevator */
  if (elevator_init(q, NULL)) {
    // struct elevator_type elevator_noop
    // struct elevator_type iosched_deadline
    // struct elevator_type iosched_cfq
  }
}

// make_request_fn -> blk_queue_bio
static blk_qc_t blk_queue_bio(struct request_queue *q, struct bio *bio)
{
  struct request *req, *free;
  unsigned int request_count = 0;

  switch (elv_merge(q, &req, bio)) {
  case ELEVATOR_BACK_MERGE:
    if (!bio_attempt_back_merge(q, req, bio))
      break;
    elv_bio_merged(q, req, bio);
    free = attempt_back_merge(q, req);
    if (free)
      __blk_put_request(q, free);
    else
      elv_merged_request(q, req, ELEVATOR_BACK_MERGE);
    goto out_unlock;
  case ELEVATOR_FRONT_MERGE:
    if (!bio_attempt_front_merge(q, req, bio))
      break;
    elv_bio_merged(q, req, bio);
    free = attempt_front_merge(q, req);
    if (free)
      __blk_put_request(q, free);
    else
      elv_merged_request(q, req, ELEVATOR_FRONT_MERGE);
    goto out_unlock;
  default:
    break;
  }

get_rq:
  req = get_request(q, bio->bi_opf, bio, GFP_NOIO);
  blk_init_request_from_bio(req, bio);
  add_acct_request(q, req, where);
  __blk_run_queue(q);
out_unlock:

  return BLK_QC_T_NONE;
}

enum elv_merge elv_merge(struct request_queue *q, struct request **req,
    struct bio *bio)
{
  struct elevator_queue *e = q->elevator;
  struct request *__rq;

  if (q->last_merge && elv_bio_merge_ok(q->last_merge, bio)) {
    enum elv_merge ret = blk_try_merge(q->last_merge, bio);


    if (ret != ELEVATOR_NO_MERGE) {
      *req = q->last_merge;
      return ret;
    }
  }

  __rq = elv_rqhash_find(q, bio->bi_iter.bi_sector);
  if (__rq && elv_bio_merge_ok(__rq, bio)) {
    *req = __rq;
    return ELEVATOR_BACK_MERGE;
  }

  if (e->uses_mq && e->type->ops.mq.request_merge)
    return e->type->ops.mq.request_merge(q, req, bio);
  else if (!e->uses_mq && e->type->ops.sq.elevator_merge_fn)
    return e->type->ops.sq.elevator_merge_fn(q, req, bio);

  return ELEVATOR_NO_MERGE;
}
```

#### buffered write
```C++
ssize_t generic_perform_write(struct file *file,
        struct iov_iter *i, loff_t pos)
{
  struct address_space *mapping = file->f_mapping;
  const struct address_space_operations *a_ops = mapping->a_ops;
  do {
    struct page *page;
    unsigned long offset;  /* Offset into pagecache page */
    unsigned long bytes;  /* Bytes to write to page */
    status = a_ops->write_begin(file, mapping, pos, bytes, flags,
            &page, &fsdata) {
                grab_cache_page_write_begin();
                ext4_journal_start();
            }
    copied = iov_iter_copy_from_user_atomic(page, i, offset, bytes);
    flush_dcache_page(page);
    status = a_ops->write_end(file, mapping, pos, bytes, copied,
            page, fsdata) {
                ext4_journal_stop() {
                    // block_write_end->__block_commit_write->mark_buffer_dirty
                }
            }
    pos += copied;
    written += copied;

    balance_dirty_pages_ratelimited(mapping);
  } while (iov_iter_count(i));
}

struct page *grab_cache_page_write_begin(struct address_space *mapping,
          pgoff_t index, unsigned flags)
{
  struct page *page;
  int fgp_flags = FGP_LOCK|FGP_WRITE|FGP_CREAT;
  page = pagecache_get_page(mapping, index, fgp_flags,
      mapping_gfp_mask(mapping));
  if (page)
    wait_for_stable_page(page);
  return page;
}

size_t iov_iter_copy_from_user_atomic(struct page *page,
    struct iov_iter *i, unsigned long offset, size_t bytes)
{
  char *kaddr = kmap_atomic(page), *p = kaddr + offset;
  iterate_all_kinds(i, bytes, v,
    copyin((p += v.iov_len) - v.iov_len, v.iov_base, v.iov_len),
    memcpy_from_page((p += v.bv_len) - v.bv_len, v.bv_page,
         v.bv_offset, v.bv_len),
    memcpy((p += v.iov_len) - v.iov_len, v.iov_base, v.iov_len)
  )
  kunmap_atomic(kaddr);
  return bytes;
}

void balance_dirty_pages_ratelimited(struct address_space *mapping)
{
  struct inode *inode = mapping->host;
  struct backing_dev_info *bdi = inode_to_bdi(inode);
  struct bdi_writeback *wb = NULL;
  int ratelimit;
  if (unlikely(current->nr_dirtied >= ratelimit))
    balance_dirty_pages(mapping, wb, current->nr_dirtied);
}
// balance_dirty_pages->wb_start_background_writebac->wb_wakeup
struct backing_dev_info {
  struct bdi_writeback wb;  /* the root writeback info for this bdi */
}

struct bdi_writeback {
  struct delayed_work dwork;  /* work item used for writeback */
}

struct delayed_work {
  struct work_struct work;
  struct timer_list timer;

  /* target workqueue and CPU ->timer uses to queue ->work */
  struct workqueue_struct *wq;
  int cpu;
};

typedef void (*work_func_t)(struct work_struct *work);
struct work_struct {
  atomic_long_t data;
  struct list_head entry;
  work_func_t func;
};

static void wb_wakeup(struct bdi_writeback *wb)
{
  spin_lock_bh(&wb->work_lock);
  if (test_bit(WB_registered, &wb->state))
    mod_delayed_work(bdi_wq, &wb->dwork, 0);
  spin_unlock_bh(&wb->work_lock);
}

/* global variable, all backing task stored here */
/* bdi_wq serves all asynchronous writeback tasks */
struct workqueue_struct *bdi_wq;

static inline bool mod_delayed_work(struct workqueue_struct *wq,
            struct delayed_work *dwork,
            unsigned long delay)
{ }

static int wb_init(struct bdi_writeback *wb, struct backing_dev_info *bdi,
       int blkcg_id, gfp_t gfp)
{
  wb->bdi = bdi;
  wb->last_old_flush = jiffies;
  INIT_LIST_HEAD(&wb->b_dirty);
  INIT_LIST_HEAD(&wb->b_io);
  INIT_LIST_HEAD(&wb->b_more_io);
  INIT_LIST_HEAD(&wb->b_dirty_time);
  wb->bw_time_stamp = jiffies;
  wb->balanced_dirty_ratelimit = INIT_BW;
  wb->dirty_ratelimit = INIT_BW;
  wb->write_bandwidth = INIT_BW;
  wb->avg_write_bandwidth = INIT_BW;
  spin_lock_init(&wb->work_lock);
  INIT_LIST_HEAD(&wb->work_list);
  INIT_DELAYED_WORK(&wb->dwork, wb_workfn);
  wb->dirty_sleep = jiffies;
}


#define __INIT_DELAYED_WORK(_work, _func, _tflags)      \
  do {                \
    INIT_WORK(&(_work)->work, (_func));      \
    __setup_timer(&(_work)->timer, delayed_work_timer_fn,  \
            (unsigned long)(_work),      \

// wb_workfn->wb_do_writeback->wb_writeback->writeback_sb_inodes
// ->__writeback_single_inode->do_writepages->ext4_writepages
static int ext4_writepages(struct address_space *mapping,
         struct writeback_control *wbc)
{
  struct mpage_da_data mpd;
  struct inode *inode = mapping->host;
  struct ext4_sb_info *sbi = EXT4_SB(mapping->host->i_sb);

  mpd.do_map = 0;
  mpd.io_submit.io_end = ext4_init_io_end(inode, GFP_KERNEL);
  ret = mpage_prepare_extent_to_map(&mpd);
  /* Submit prepared bio */
  ext4_io_submit(&mpd.io_submit);
}

struct mpage_da_data {
  struct inode *inode;
  pgoff_t first_page;  /* The first page to write */
  pgoff_t next_page;  /* Current page to examine */
  pgoff_t last_page;  /* Last page to examine */
  struct ext4_map_blocks map;
  struct ext4_io_submit io_submit;  /* IO submission data */
  unsigned int do_map:1;
};

struct ext4_io_submit {
  struct bio    *io_bio;
  ext4_io_end_t    *io_end;
  sector_t    io_next_block;
};

// mpage_prepare_extent_to_map->mpage_process_page_bufs->
// mpage_submit_page->ext4_bio_write_page->io_submit_add_bh
static int io_submit_init_bio(struct ext4_io_submit *io,
            struct buffer_head *bh)
{
  struct bio *bio;
  bio = bio_alloc(GFP_NOIO, BIO_MAX_PAGES);
  if (!bio)
    return -ENOMEM;
  wbc_init_bio(io->io_wbc, bio);
  bio->bi_iter.bi_sector = bh->b_blocknr * (bh->b_size >> 9);
  bio->bi_bdev = bh->b_bdev;
  bio->bi_end_io = ext4_end_bio;
  bio->bi_private = ext4_get_io_end(io->io_end);
  io->io_bio = bio;
  io->io_next_block = bh->b_blocknr;
  return 0;
}

void ext4_io_submit(struct ext4_io_submit *io)
{
  struct bio *bio = io->io_bio;
  if (bio) {
    int io_op_flags = io->io_wbc->sync_mode == WB_SYNC_ALL ?
          REQ_SYNC : 0;
    io->io_bio->bi_write_hint = io->io_end->inode->i_write_hint;
    bio_set_op_attrs(io->io_bio, REQ_OP_WRITE, io_op_flags);
    submit_bio(io->io_bio);
  }
  io->io_bio = NULL;
}
// --->
```
#### Buffered read
```C++
static ssize_t generic_file_buffered_read(struct kiocb *iocb,
    struct iov_iter *iter, ssize_t written)
{
  struct file *filp = iocb->ki_filp;
  struct address_space *mapping = filp->f_mapping;
  struct inode *inode = mapping->host;
  for (;;) {
    struct page *page;
    pgoff_t end_index;
    loff_t isize;
    page = find_get_page(mapping, index);
    if (!page) {
      if (iocb->ki_flags & IOCB_NOWAIT)
        goto would_block;
      page_cache_sync_readahead(mapping,
          ra, filp,
          index, last_index - index);
      page = find_get_page(mapping, index);
      if (unlikely(page == NULL))
        goto no_cached_page;
    }
    if (PageReadahead(page)) {
      page_cache_async_readahead(mapping,
          ra, filp, page,
          index, last_index - index);
    }
    /*
     * Ok, we have the page, and it's up-to-date, so
     * now we can copy it to user space...
     */
    ret = copy_page_to_iter(page, offset, nr, iter);
    }
}
```
![linux-file-read-write.png](../Images/linux-file-read-write.png)

### Question:
1. How to use inode bit map present all inodes?
2. Does system alloc a block for a dirctory or a file?
3. What does ext4_file_open do?

### TODO
1. xarray

# IO
All devices have the corresponding device file in /dev(is devtmpfs file system), which has inode, but it's not associated with any data in the storage device, it's associated with the device's drive. And this device file belongs to a special file system: devtmpfs.

```c++
lsmod           // list installed modes
insmod *.ko     // install mode
mknod filename type major minor  // create dev file in /dev/

/*
/sys has following directories
/sys/devices represent all kernel devices in layer
/sys/dev has a char directory and block directory, maintain symbolic link files linked to real devices
/sys/block all block devices
/sys/module all modes information
*/
```
![linux-io-sysfs.jpg](../Images/linux-io-sysfs.jpg)

### Char dev
#### kernal module
```C++
module_init(logibm_init);
module_exit(logibm_exit);
```

A kernel module consists:
1. header
  ```C++
  #include <linux/module.h>
  #include <linux/init.h>
  ```
2. define functions, handling kernel module main logic
3. Implement a file_operation interface
  ```C++
  static const struct file_operations lp_fops = {
    .owner    = THIS_MODULE,
    .write    = lp_write,
    .unlocked_ioctl  = lp_ioctl,
    #ifdef CONFIG_COMPAT
    .compat_ioctl  = lp_compat_ioctl,
    #endif
    .open    = lp_open,
    .release  = lp_release,
    #ifdef CONFIG_PARPORT_1284
    .read    = lp_read,
    #endif
    .llseek    = noop_llseek,
  };
  ```
4. define init and exit functions
5. invoke `module_init` and `moudle_exit`
6. declare lisence, invoke MODULE_LICENSE

#### register dev
![linux-io-char-dev-install-open.jpg](../Images/linux-io-char-dev-install-open.jpg)
```C++
static int __init lp_init (void)
{
  if (register_chrdev (LP_MAJOR, "lp", &lp_fops)) {
    printk (KERN_ERR "lp: unable to get major %d\n", LP_MAJOR);
    return -EIO;
  }
}

int __register_chrdev(unsigned int major, unsigned int baseminor,
  unsigned int count, const char *name,
  const struct file_operations *fops)
{
  struct char_device_struct *cd;
  struct cdev *cdev;
  int err = -ENOMEM;

  cd = __register_chrdev_region(major, baseminor, count, name);
  cdev = cdev_alloc();
  cdev->owner = fops->owner;
  cdev->ops = fops;
  kobject_set_name(&cdev->kobj, "%s", name);
  err = cdev_add(cdev, MKDEV(cd->major, baseminor), count);
  cd->cdev = cdev;
  return major ? 0 : cd->major;
}

int cdev_add(struct cdev *p, dev_t dev, unsigned count)
{
  int error;
  p->dev = dev;
  p->count = count;

  error = kobj_map(cdev_map, dev, count, NULL,
       exact_match, exact_lock, p);
  kobject_get(p->kobj.parent);

  return 0;
}
```

#### make node at /dev
```C++
SYSCALL_DEFINE3(mknod, const char __user *, filename, umode_t, mode, unsigned, dev)
{
  return sys_mknodat(AT_FDCWD, filename, mode, dev);
}

SYSCALL_DEFINE4(mknodat, int, dfd, const char __user *, filename, umode_t, mode,
    unsigned, dev)
{
  struct dentry *dentry;
  struct path path;
  dentry = user_path_create(dfd, filename, &path, lookup_flags);
  switch (mode & S_IFMT) {
    case S_IFCHR:
    case S_IFBLK:
      error = vfs_mknod(path.dentry->d_inode,dentry,mode,
          new_decode_dev(dev));
      break;
  }
}

int vfs_mknod(struct inode *dir, struct dentry *dentry, umode_t mode, dev_t dev)
{
  error = dir->i_op->mknod(dir, dentry, mode, dev);
}

// the file system of /dev is devtmpfs
static struct file_system_type dev_fs_type = {
  .name = "devtmpfs",
  .mount = dev_mount,
  .kill_sb = kill_litter_super,
};

static struct dentry *dev_mount(struct file_system_type *fs_type, int flags,
          const char *dev_name, void *data)
{
#ifdef CONFIG_TMPFS
  return mount_single(fs_type, flags, data, shmem_fill_super);
#else
  return mount_single(fs_type, flags, data, ramfs_fill_super);
#endif
}

static const struct inode_operations ramfs_dir_inode_operations = {
  .mknod    = ramfs_mknod,
};

static const struct inode_operations shmem_dir_inode_operations = {
#ifdef CONFIG_TMPFS
  .mknod    = shmem_mknod,
};

void init_special_inode(struct inode *inode, umode_t mode, dev_t rdev)
{
  inode->i_mode = mode;
  if (S_ISCHR(mode)) {
    inode->i_fop = &def_chr_fops;
    inode->i_rdev = rdev;
  } else if (S_ISBLK(mode)) {
    inode->i_fop = &def_blk_fops;
    inode->i_rdev = rdev;
  } else if (S_ISFIFO(mode))
    inode->i_fop = &pipefifo_fops;
  else if (S_ISSOCK(mode))
    ;  /* leave it no_open_fops */
}
```

#### open dev
```C++
const struct file_operations def_chr_fops = {
  .open = chrdev_open,
};

static int chrdev_open(struct inode *inode, struct file *filp)
{
  const struct file_operations *fops;
  struct cdev *p;
  struct cdev *new = NULL;
  int ret = 0;

  p = inode->i_cdev;
  if (!p) {
    struct kobject *kobj;
    int idx;
    kobj = kobj_lookup(cdev_map, inode->i_rdev, &idx);
    new = container_of(kobj, struct cdev, kobj);
    p = inode->i_cdev;
    if (!p) {
      inode->i_cdev = p = new;
      list_add(&inode->i_devices, &p->list);
      new = NULL;
    }
  }
  fops = fops_get(p->ops);

  replace_fops(filp, fops);
  if (filp->f_op->open) {
    ret = filp->f_op->open(inode, filp);
  }
}
```

#### write char dev
```C++
ssize_t __vfs_write(struct file *file, const char __user *p, size_t count, loff_t *pos)
{
  if (file->f_op->write)
    return file->f_op->write(file, p, count, pos);
  else if (file->f_op->write_iter)
    return new_sync_write(file, p, count, pos);
  else
    return -EINVAL;
}

static ssize_t lp_write(struct file * file, const char __user * buf,
            size_t count, loff_t *ppos)
{
  unsigned int minor = iminor(file_inode(file));
  struct parport *port = lp_table[minor].dev->port;
  char *kbuf = lp_table[minor].lp_buffer;
  ssize_t retv = 0;
  ssize_t written;
  size_t copy_size = count;

  /* Need to copy the data from user-space. */
  if (copy_size > LP_BUFFER_SIZE)
    copy_size = LP_BUFFER_SIZE;

  if (copy_from_user (kbuf, buf, copy_size)) {
    retv = -EFAULT;
    goto out_unlock;
  }

  do {
    /* Write the data. */
    written = parport_write (port, kbuf, copy_size);
    if (written > 0) {
      copy_size -= written;
      count -= written;
      buf  += written;
      retv += written;
    }

        if (need_resched())
      schedule ();


    if (count) {
      copy_size = count;
      if (copy_size > LP_BUFFER_SIZE)
        copy_size = LP_BUFFER_SIZE;

      if (copy_from_user(kbuf, buf, copy_size)) {
        if (retv == 0)
          retv = -EFAULT;
        break;
      }
    }
  } while (count > 0);
```
![linux-io-char-dev-write.jpg](../Images/linux-io-char-dev-write.jpg)

#### ioctl
```C++
SYSCALL_DEFINE3(ioctl, unsigned int, fd, unsigned int, cmd, unsigned long, arg)
{
  int error;
  struct fd f = fdget(fd);
  error = do_vfs_ioctl(f.file, fd, cmd, arg);
  fdput(f);
  return error;
}

int do_vfs_ioctl(struct file *filp, unsigned int fd, unsigned int cmd,
       unsigned long arg)
{
  int error = 0;
  int __user *argp = (int __user *)arg;
  struct inode *inode = file_inode(filp);

  switch (cmd) {
  case FIONBIO:
    error = ioctl_fionbio(filp, argp);
    break;

  case FIOASYNC:
    error = ioctl_fioasync(fd, filp, argp);
    break;

  case FICLONE:
    return ioctl_file_clone(filp, arg, 0, 0, 0);

  default:
    if (S_ISREG(inode->i_mode))
      error = file_ioctl(filp, cmd, arg);
    else
      error = vfs_ioctl(filp, cmd, arg);
    break;
  }
  return error;
}

long vfs_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
  int error = -ENOTTY;
  if (!filp->f_op->unlocked_ioctl)
    goto out;

  error = filp->f_op->unlocked_ioctl(filp, cmd, arg);
  if (error == -ENOIOCTLCMD)
    error = -ENOTTY;
 out:
  return error;
}

static long lp_ioctl(struct file *file, unsigned int cmd,
      unsigned long arg)
{
  unsigned int minor;
  struct timeval par_timeout;
  int ret;

  minor = iminor(file_inode(file));
  mutex_lock(&lp_mutex);
  switch (cmd) {
  default:
    ret = lp_do_ioctl(minor, cmd, arg, (void __user *)arg);
    break;
  }
  mutex_unlock(&lp_mutex);
  return ret;
}


static int lp_do_ioctl(unsigned int minor, unsigned int cmd,
  unsigned long arg, void __user *argp)
{
  int status;
  int retval = 0;

  switch ( cmd ) {
    case LPTIME:
      if (arg > UINT_MAX / HZ)
        return -EINVAL;
      LP_TIME(minor) = arg * HZ/100;
      break;
    case LPCHAR:
      LP_CHAR(minor) = arg;
      break;
    case LPABORT:
      if (arg)
        LP_F(minor) |= LP_ABORT;
      else
        LP_F(minor) &= ~LP_ABORT;
      break;
    case LPABORTOPEN:
      if (arg)
        LP_F(minor) |= LP_ABORTOPEN;
      else
        LP_F(minor) &= ~LP_ABORTOPEN;
      break;
    case LPCAREFUL:
      if (arg)
        LP_F(minor) |= LP_CAREFUL;
      else
        LP_F(minor) &= ~LP_CAREFUL;
      break;
    case LPWAIT:
      LP_WAIT(minor) = arg;
      break;
    case LPSETIRQ:
      return -EINVAL;
      break;
    case LPGETIRQ:
      if (copy_to_user(argp, &LP_IRQ(minor),
          sizeof(int)))
        return -EFAULT;
      break;
    case LPGETSTATUS:
      if (mutex_lock_interruptible(&lp_table[minor].port_mutex))
        return -EINTR;
      lp_claim_parport_or_block (&lp_table[minor]);
      status = r_str(minor);
      lp_release_parport (&lp_table[minor]);
      mutex_unlock(&lp_table[minor].port_mutex);

      if (copy_to_user(argp, &status, sizeof(int)))
        return -EFAULT;
      break;
    case LPRESET:
      lp_reset(minor);
      break;
     case LPGETFLAGS:
       status = LP_F(minor);
      if (copy_to_user(argp, &status, sizeof(int)))
        return -EFAULT;
      break;
    default:
      retval = -EINVAL;
  }
  return retval
}
```
![linux-io-ioctl.jpg](../Images/linux-io-ioctl.jpg)

### interruption

![linux-io-irq.jpg](../Images/linux-io-irq.jpg)

```C++
static int logibm_open(struct input_dev *dev)
{
  if (request_irq(logibm_irq, logibm_interrupt, 0, "logibm", NULL)) {
    printk(KERN_ERR "logibm.c: Can't allocate irq %d\n", logibm_irq);
    return -EBUSY;
  }
  outb(LOGIBM_ENABLE_IRQ, LOGIBM_CONTROL_PORT);
  return 0;
}

static irqreturn_t logibm_interrupt(int irq, void *dev_id)
{
  char dx, dy;
  unsigned char buttons;

  outb(LOGIBM_READ_X_LOW, LOGIBM_CONTROL_PORT);
  dx = (inb(LOGIBM_DATA_PORT) & 0xf);
  outb(LOGIBM_READ_X_HIGH, LOGIBM_CONTROL_PORT);
  dx |= (inb(LOGIBM_DATA_PORT) & 0xf) << 4;
  outb(LOGIBM_READ_Y_LOW, LOGIBM_CONTROL_PORT);
  dy = (inb(LOGIBM_DATA_PORT) & 0xf);
  outb(LOGIBM_READ_Y_HIGH, LOGIBM_CONTROL_PORT);
  buttons = inb(LOGIBM_DATA_PORT);
  dy |= (buttons & 0xf) << 4;
  buttons = ~buttons >> 5;

  input_report_rel(logibm_dev, REL_X, dx);
  input_report_rel(logibm_dev, REL_Y, dy);
  input_report_key(logibm_dev, BTN_RIGHT,  buttons & 1);
  input_report_key(logibm_dev, BTN_MIDDLE, buttons & 2);
  input_report_key(logibm_dev, BTN_LEFT,   buttons & 4);
  input_sync(logibm_dev);


  outb(LOGIBM_ENABLE_IRQ, LOGIBM_CONTROL_PORT);
  return IRQ_HANDLED
}

irqreturn_t (*irq_handler_t)(int irq, void * dev_id);
enum irqreturn {
  IRQ_NONE    = (0 << 0),
  IRQ_HANDLED    = (1 << 0),
  IRQ_WAKE_THREAD    = (1 << 1),
};
```

#### dev register irq handler
```C++
static inline int __must_check
request_irq(unsigned int irq, irq_handler_t handler, unsigned long flags, const char *name, void *dev)
{
  return request_threaded_irq(irq, handler, NULL, flags, name, dev);
}

int request_threaded_irq(unsigned int irq, irq_handler_t handler,
  irq_handler_t thread_fn, unsigned long irqflags,
  const char *devname, void *dev_id)
{
  struct irqaction *action;
  struct irq_desc *desc;
  int retval;

  desc = irq_to_desc(irq);

  action = kzalloc(sizeof(struct irqaction), GFP_KERNEL);
  action->handler = handler;
  action->thread_fn = thread_fn;
  action->flags = irqflags;
  action->name = devname;
  action->dev_id = dev_id;
  retval = __setup_irq(irq, desc, action);
}

struct irq_data {
  u32      mask;
  unsigned int    irq;
  unsigned long    hwirq;
  struct irq_common_data  *common;
  struct irq_chip    *chip;
  struct irq_domain  *domain;
#ifdef  CONFIG_IRQ_DOMAIN_HIERARCHY
  struct irq_data    *parent_data;
#endif
  void      *chip_data;
};

struct irq_desc {
  struct irqaction  *action;  /* IRQ action list */
  struct module    *owner;
  struct irq_data    irq_data;
  const char    *name;
};

struct irqaction {
  irq_handler_t    handler;
  void      *dev_id;
  void __percpu    *percpu_dev_id;
  struct irqaction  *next;
  irq_handler_t    thread_fn;
  struct task_struct  *thread;
  struct irqaction  *secondary;
  unsigned int    irq;
  unsigned int    flags;
  unsigned long    thread_flags;
  unsigned long    thread_mask;
  const char    *name;
  struct proc_dir_entry  *dir;
};

#ifdef CONFIG_SPARSE_IRQ
static RADIX_TREE(irq_desc_tree, GFP_KERNEL);

struct irq_desc *irq_to_desc(unsigned int irq)
{
  return radix_tree_lookup(&irq_desc_tree, irq);
}
#else /* !CONFIG_SPARSE_IRQ */
struct irq_desc irq_desc[NR_IRQS] __cacheline_aligned_in_smp = {
  [0 ... NR_IRQS-1] = { }
};
struct irq_desc *irq_to_desc(unsigned int irq)
{
  return (irq < NR_IRQS) ? irq_desc + irq : NULL;
}
#endif /* !CONFIG_SPARSE_IRQ */

static int
__setup_irq(unsigned int irq, struct irq_desc *desc, struct irqaction *new)
{
  struct irqaction *old, **old_ptr;
  unsigned long flags, thread_mask = 0;
  int ret, nested, shared = 0;

  new->irq = irq;

  if (new->thread_fn && !nested) {
    ret = setup_irq_thread(new, irq, false);
  }

  old_ptr = &desc->action;
  old = *old_ptr;
  if (old) {
    /* add new interrupt at end of irq queue */
    do {
      thread_mask |= old->thread_mask;
      old_ptr = &old->next;
      old = *old_ptr;
    } while (old);
  }
  *old_ptr = new;
  if (new->thread)
    wake_up_process(new->thread);
}

static int
setup_irq_thread(struct irqaction *new, unsigned int irq, bool secondary)
{
  struct task_struct *t;
  struct sched_param param = {
    .sched_priority = MAX_USER_RT_PRIO/2,
  };

  t = kthread_create(irq_thread, new, "irq/%d-%s", irq, new->name);
  sched_setscheduler_nocheck(t, SCHED_FIFO, &param);
  get_task_struct(t);
  new->thread = t;

  return 0;
}

/* How interrupt happens:
 * 1. extern dev sends physic interrupt to interrupt controller
 * 2. interrupt controller converts interrupt signal to interrupt vector, sends it to each cpu
 * 3. cpu call IRQ hanlder according to interrupt vector
 * 4. IRQ handler converts interrupt vector to abstract interrupt signal, handle it with irq_handler_t */


/* arch/x86/include/asm/irq_vectors.h
 * Linux IRQ vector layout.
 *
 * There are 256 IDT entries (per CPU - each entry is 8 bytes) which can
 * be defined by Linux. They are used as a jump table by the CPU when a
 * given vector is triggered - by a CPU-external, CPU-internal or
 * software-triggered event.
 *
 * Linux sets the kernel code address each entry jumps to early during
 * bootup, and never changes them. This is the general layout of the
 * IDT entries:
 *
 *  Vectors   0 ...  31 : system traps and exceptions - hardcoded events
 *  Vectors  32 ... 127 : device interrupts
 *  Vector  128         : legacy int80 syscall interface
 *  Vectors 129 ... INVALIDATE_TLB_VECTOR_START-1 except 204 : device interrupts
 *  Vectors INVALIDATE_TLB_VECTOR_START ... 255 : special interrupts
 *
 * 64-bit x86 has per CPU IDT tables, 32-bit has one shared IDT table.
 *
 * This file enumerates the exact layout of them: */
#define FIRST_EXTERNAL_VECTOR    0x20 // 32
#define IA32_SYSCALL_VECTOR    0x80   // 128
#define NR_VECTORS       256
#define FIRST_SYSTEM_VECTOR    NR_VECTORS

// arch/x86/kernel/traps.c, per cpu
struct gate_struct {
  u16    offset_low; // system irq hanlder addr
  u16    segment;   // KERNEL_CS
  struct idt_bits  bits;
  u16    offset_middle;  // addr >> 16
#ifdef CONFIG_X86_64
  u32    offset_high;    // adr >> 32
  u32    reserved;      // 0
#endif
};
struct idt_bits {
  u16 ist  : 3,   // DEFAULT_STACK
      zero  : 5,
      type  : 5,  // GATE_{INTERRUPT, TRAP, CALL, TASK}
      dpl  : 2,   // DPL (Descriptor privilege level), DLP0, DLP3
                  // RPL (Requested privilege level)
      p  : 1;
};
```

#### init idt_table
```C++
gate_desc idt_table[NR_VECTORS] __page_aligned_bss;

void __init trap_init(void)
{
  int i;
  set_intr_gate(X86_TRAP_DE, divide_error);
  // ...
  set_intr_gate(X86_TRAP_XF, simd_coprocessor_error);

  /* Reserve all the builtin and the syscall vector: */
  for (i = 0; i < FIRST_EXTERNAL_VECTOR; i++)
    set_bit(i, used_vectors);

#ifdef CONFIG_X86_32
  set_system_intr_gate(IA32_SYSCALL_VECTOR, entry_INT80_32);
  set_bit(IA32_SYSCALL_VECTOR, used_vectors);
#endif
  __set_fixmap(FIX_RO_IDT, __pa_symbol(idt_table), PAGE_KERNEL_RO);
  idt_descr.address = fix_to_virt(FIX_RO_IDT);
}

// set_intr_gate -> _set_gate
static inline void _set_gate(int gate, unsigned type, void *addr,
           unsigned dpl, unsigned ist, unsigned seg)
{
  gate_desc s;
  pack_gate(&s, type, (unsigned long)addr, dpl, ist, seg);
  write_idt_entry(idt_table, gate, &s);
}

// arch/x86/include/asm/traps.h
enum {
  X86_TRAP_DE = 0,  /*  0, Divide-by-zero */
  X86_TRAP_DB,    /*  1, Debug */
  X86_TRAP_NMI,    /*  2, Non-maskable Interrupt */
  X86_TRAP_BP,    /*  3, Breakpoint */
  X86_TRAP_OF,    /*  4, Overflow */
  X86_TRAP_BR,    /*  5, Bound Range Exceeded */
  X86_TRAP_UD,    /*  6, Invalid Opcode */
  X86_TRAP_NM,    /*  7, Device Not Available */
  X86_TRAP_DF,    /*  8, Double Fault */
  X86_TRAP_OLD_MF,  /*  9, Coprocessor Segment Overrun */
  X86_TRAP_TS,    /* 10, Invalid TSS */
  X86_TRAP_NP,    /* 11, Segment Not Present */
  X86_TRAP_SS,    /* 12, Stack Segment Fault */
  X86_TRAP_GP,    /* 13, General Protection Fault */
  X86_TRAP_PF,    /* 14, Page Fault */
  X86_TRAP_SPURIOUS,  /* 15, Spurious Interrupt */
  X86_TRAP_MF,    /* 16, x87 Floating-Point Exception */
  X86_TRAP_AC,    /* 17, Alignment Check */
  X86_TRAP_MC,    /* 18, Machine Check */
  X86_TRAP_XF,    /* 19, SIMD Floating-Point Exception */
  X86_TRAP_IRET = 32,  /* 32, IRET Exception */
};
```
#### init_IRQ
```C++
// after kernel called trap_init(), it invokes init_IRQ() to init other dev interrupt
void __init native_init_IRQ(void)
{
  int i;
  i = FIRST_EXTERNAL_VECTOR;
#ifndef CONFIG_X86_LOCAL_APIC
#define first_system_vector NR_VECTORS
#endif
  for_each_clear_bit_from(i, used_vectors, first_system_vector) {
    /* IA32_SYSCALL_VECTOR could be used in trap_init already. */
    set_intr_gate(i, irq_entries_start +
        8 * (i - FIRST_EXTERNAL_VECTOR));
  }
}

// irq_entries_start defined in arch\x86\entry\entry_{32, 64}
ENTRY(irq_entries_start)
    vector=FIRST_EXTERNAL_VECTOR
    .rept (FIRST_SYSTEM_VECTOR - FIRST_EXTERNAL_VECTOR)
  pushl  $(~vector+0x80)      /* Note: always in signed byte range */
    vector=vector+1
  jmp  common_interrupt /* invoke do_IRQ */
  .align  8
    .endr
END(irq_entries_start)


common_interrupt:
  ASM_CLAC
  addq  $-0x80, (%rsp)      /* Adjust vector to [-256, -1] range */
  interrupt do_IRQ
  /* 0(%rsp): old RSP */
ret_from_intr:

  /* Interrupt came from user space */
GLOBAL(retint_user)

/* Returning to kernel space */
retint_kernel:


/* Note: the interrupt vector interrupt controller sent to
 * each cpu is per cpu local variable, but the abstract
 * layer's virtual signal irq and it's handler is global. */

__visible unsigned int __irq_entry do_IRQ(struct pt_regs *regs)
{
  struct pt_regs *old_regs = set_irq_regs(regs);
  struct irq_desc * desc;
  /* high bit used in ret_from_ code  */
  unsigned vector = ~regs->orig_ax;

  desc = __this_cpu_read(vector_irq[vector]);
  if (!handle_irq(desc, regs)) {

  }

  set_irq_regs(old_regs);
  return 1;
}
```

#### init vector_irq
```C++
typedef struct irq_desc* vector_irq_t[NR_VECTORS];
DECLARE_PER_CPU(vector_irq_t, vector_irq);

// assign virtual irq to a cpu
static int __assign_irq_vector(int irq, struct apic_chip_data *d,
  const struct cpumask *mask,
  struct irq_data *irqdata)
{
  static int current_vector = FIRST_EXTERNAL_VECTOR + VECTOR_OFFSET_START;
  static int current_offset = VECTOR_OFFSET_START % 16;
  int cpu, vector;

  while (cpu < nr_cpu_ids) {
    int new_cpu, offset;

    vector = current_vector;
    offset = current_offset;
next:
    vector += 16;
    if (vector >= first_system_vector) {
      offset = (offset + 1) % 16;
      vector = FIRST_EXTERNAL_VECTOR + offset;
    }
    /* If the search wrapped around, try the next cpu */
    if (unlikely(current_vector == vector))
      goto next_cpu;

    if (test_bit(vector, used_vectors))
      goto next;

    /* Found one! */
    current_vector = vector;
    current_offset = offset;
    /* Schedule the old vector for cleanup on all cpus */
    if (d->cfg.vector)
      cpumask_copy(d->old_domain, d->domain);
    for_each_cpu(new_cpu, vector_searchmask)
      per_cpu(vector_irq, new_cpu)[vector] = irq_to_desc(irq);
    goto update;

next_cpu:
    cpumask_or(searched_cpumask, searched_cpumask, vector_cpumask);
    cpumask_andnot(vector_cpumask, mask, searched_cpumask);
    cpu = cpumask_first_and(vector_cpumask, cpu_online_mask);
    continue;
}

static inline void generic_handle_irq_desc(struct irq_desc *desc)
{
  desc->handle_irq(desc);
}

irqreturn_t __handle_irq_event_percpu(struct irq_desc *desc, unsigned int *flags)
{
  irqreturn_t retval = IRQ_NONE;
  unsigned int irq = desc->irq_data.irq;
  struct irqaction *action;

  record_irq_time(desc);

  for_each_action_of_desc(desc, action) {
    irqreturn_t res;
    res = action->handler(irq, action->dev_id);
    switch (res) {
    case IRQ_WAKE_THREAD:
      __irq_wake_thread(desc, action);
    case IRQ_HANDLED:
      *flags |= action->flags;
      break;
    default:
      break;
    }
    retval |= res;
  }
  return retval;
}
```
 ![linux-io-interrupt-vector.png](../Images/linux-io-interrupt-vector.png)

 ![linux-io-interrupt.png](../Images/linux-io-interrupt.png)

### block dev
```C++
void init_special_inode(struct inode *inode, umode_t mode, dev_t rdev)
{
  inode->i_mode = mode;
  if (S_ISCHR(mode)) {
    inode->i_fop = &def_chr_fops;
    inode->i_rdev = rdev;
  } else if (S_ISBLK(mode)) {
    inode->i_fop = &def_blk_fops;
    inode->i_rdev = rdev;
  } else if (S_ISFIFO(mode))
    inode->i_fop = &pipefifo_fops;
  else if (S_ISSOCK(mode))
    ;  /* leave it no_open_fops */
}

const struct file_operations def_blk_fops = {
  .open           = blkdev_open, // call blkdev_get
  .release        = blkdev_close,
  .llseek         = block_llseek,
  .read_iter      = blkdev_read_iter,
  .write_iter     = blkdev_write_iter,
  .mmap           = generic_file_mmap,
  .fsync          = blkdev_fsync,
  .unlocked_ioctl = block_ioctl,
  .splice_read    = generic_file_splice_read,
  .splice_write   = iter_file_splice_write,
  .fallocate      = blkdev_fallocate,
};

static struct file_system_type ext4_fs_type = {
  .owner    = THIS_MODULE,
  .name    = "ext4",
  .mount    = ext4_mount,
  .kill_sb  = kill_block_super,
  .fs_flags  = FS_REQUIRES_DEV,
};
```

#### mount block device
```C++
static struct dentry *ext4_mount(struct file_system_type *fs_type, int flags, const char *dev_name, void *data)
{
  return mount_bdev(fs_type, flags, dev_name, data, ext4_fill_super);
}

struct dentry *mount_bdev(struct file_system_type *fs_type,
  int flags, const char *dev_name, void *data,
  int (*fill_super)(struct super_block *, void *, int))
{
  struct block_device *bdev;
  struct super_block *s;
  fmode_t mode = FMODE_READ | FMODE_EXCL;
  int error = 0;

  if (!(flags & MS_RDONLY))
    mode |= FMODE_WRITE;

  bdev = blkdev_get_by_path(dev_name, mode, fs_type);
  s = sget(fs_type, test_bdev_super, set_bdev_super, flags | MS_NOSEC, bdev);
  return dget(s->s_root);
}
```
##### blkdev_get_by_path

```C++
struct block_device *blkdev_get_by_path(const char *path, fmode_t mode,
  void *holder)
{
  struct block_device *bdev;
  int err;

  bdev = lookup_bdev(path);             // find blk device
  err = blkdev_get(bdev, mode, holder); // open blk device
  return bdev;
}

// 1. find the block device file under /dev which is devtmpfs file system
struct block_device *lookup_bdev(const char *pathname)
{
  struct block_device *bdev;
  struct inode *inode;
  struct path path;
  int error;

  if (!pathname || !*pathname)
    return ERR_PTR(-EINVAL);

  error = kern_path(pathname, LOOKUP_FOLLOW, &path);
  if (error)
    return ERR_PTR(error);

  inode = d_backing_inode(path.dentry);
  bdev = bd_acquire(inode);
  goto out;
}

// 2. find the block device in bdev file system by its key: dev_t
static struct block_device *bd_acquire(struct inode *inode)
{
  struct block_device *bdev;
  bdev = bdget(inode->i_rdev);
  if (bdev) {
    spin_lock(&bdev_lock);
    if (!inode->i_bdev) {
      bdgrab(bdev);
      inode->i_bdev = bdev;
      inode->i_mapping = bdev->bd_inode->i_mapping;
    }
  }
  return bdev;
}

struct block_device *bdget(dev_t dev)
{
  struct block_device *bdev;
  struct inode *inode;

  inode = iget5_locked(blockdev_superblock, hash(dev),
                  bdev_test, bdev_set, &dev);
  bdev = &BDEV_I(inode)->bdev;

  if (inode->i_state & I_NEW) {
    bdev->bd_contains = NULL;
    bdev->bd_super = NULL;
    bdev->bd_inode = inode;
    bdev->bd_block_size = i_blocksize(inode);
    bdev->bd_part_count = 0;
    bdev->bd_invalidated = 0;
    inode->i_mode = S_IFBLK;
    inode->i_rdev = dev;
    inode->i_bdev = bdev;
    inode->i_data.a_ops = &def_blk_aops;
    mapping_set_gfp_mask(&inode->i_data, GFP_USER);
    spin_lock(&bdev_lock);
    list_add(&bdev->bd_list, &all_bdevs);
    spin_unlock(&bdev_lock);
    unlock_new_inode(inode);
  }
  return bdev;
}

// 3rd file system
struct super_block *blockdev_superblock;
static struct file_system_type bd_type = {
  .name           = "bdev",
  .mount          = bd_mount,
  .kill_sb        = kill_anon_super,
};
/* All block device inodes stored in bdev file system for convenient management.
 * Linux associates block_device with inode of bdev with bdev_inode */
struct bdev_inode {
  struct block_device bdev;
  struct inode vfs_inode;
};

// init blockdev_superblock
void __init bdev_cache_init(void)
{
  static struct vfsmount *bd_mnt;

  bdev_cachep = kmem_cache_create("bdev_cache", sizeof(struct bdev_inode), 0,
    (SLAB_HWCACHE_ALIGN|SLAB_RECLAIM_ACCOUNT|SLAB_MEM_SPREAD|SLAB_ACCOUNT|SLAB_PANIC),
    init_once);
  err = register_filesystem(&bd_type);
  if (err)
    panic("Cannot register bdev pseudo-fs");
  bd_mnt = kern_mount(&bd_type);
  if (IS_ERR(bd_mnt))
    panic("Cannot create bdev pseudo-fs");
  blockdev_superblock = bd_mnt->mnt_sb;   /* For writeback */
}

struct block_device {
  dev_t      bd_dev;  /* not a kdev_t - it's a search key */
  int      bd_openers;
  struct super_block *  bd_super;

  struct block_device *  bd_contains;
  unsigned    bd_block_size;
  struct hd_struct *  bd_part;
  unsigned    bd_part_count;
  int      bd_invalidated;
  struct gendisk *  bd_disk;
  struct request_queue *  bd_queue;
  struct backing_dev_info *bd_bdi;
  struct list_head  bd_list;
};

struct gendisk {
  int major;      /* major number of driver */
  int first_minor;
  int minors; /* maximum number of minors, =1 for disks that can't be partitioned. */
  char disk_name[DISK_NAME_LEN]; /* name of major driver */
  char *(*devnode)(struct gendisk *gd, umode_t *mode);

  struct disk_part_tbl __rcu *part_tbl;
  struct hd_struct part0;

  const struct block_device_operations *fops;
  struct request_queue *queue;
  void *private_data;

  int flags;
  struct kobject *slave_dir;
};

struct hd_struct {
  sector_t start_sect;
  sector_t nr_sects;

  struct device __dev;
  struct kobject *holder_dir;
  int policy, partno;
  struct partition_meta_info *info;

  struct disk_stats dkstats;
  struct percpu_ref ref;
  struct rcu_head rcu_head;
};

// blkdev_get -> __blkdev_get
static int __blkdev_get(struct block_device *bdev, fmode_t mode, int for_part)
{
  struct gendisk *disk;
  struct module *owner;
  int ret;
  int partno;
  int perm = 0;

  if (mode & FMODE_READ)
    perm |= MAY_READ;
  if (mode & FMODE_WRITE)
    perm |= MAY_WRITE;

  disk = get_gendisk(bdev->bd_dev, &partno);

  owner = disk->fops->owner;

  if (!bdev->bd_openers) {
    bdev->bd_disk = disk;
    bdev->bd_queue = disk->queue;
    bdev->bd_contains = bdev;

    if (!partno) { // 1. open whole disk
      ret = -ENXIO;
      bdev->bd_part = disk_get_part(disk, partno);

      if (disk->fops->open) {
        ret = disk->fops->open(bdev, mode);
      }

      if (!ret)
        bd_set_size(bdev,(loff_t)get_capacity(disk)<<9);

      if (bdev->bd_invalidated) {
        if (!ret)
          rescan_partitions(disk, bdev);
      }

    } else {  // 2. open a partition
      struct block_device *whole;
      whole = bdget_disk(disk, 0);
      ret = __blkdev_get(whole, mode, 1);
      bdev->bd_contains = whole;
      bdev->bd_part = disk_get_part(disk, partno);
      bd_set_size(bdev, (loff_t)bdev->bd_part->nr_sects << 9);
    }
  }

  bdev->bd_openers++;
  if (for_part)
    bdev->bd_part_count++;
}

struct gendisk *get_gendisk(dev_t devt, int *partno)
{
  struct gendisk *disk = NULL;

  if (MAJOR(devt) != BLOCK_EXT_MAJOR) { // 1. get the whole device
    struct kobject *kobj;
    kobj = kobj_lookup(bdev_map, devt, partno);
    if (kobj)
      disk = dev_to_disk(kobj_to_dev(kobj));
  } else { // 2. get a partition
    struct hd_struct *part;
    part = idr_find(&ext_devt_idr, blk_mangle_minor(MINOR(devt)));
    if (part && get_disk(part_to_disk(part))) {
      *partno = part->partno;
      disk = part_to_disk(part);
    }
  }
  return disk;
}
```

```C++
static struct kobj_map *bdev_map;
// map a dev_t with a gendisk
static inline void add_disk(struct gendisk *disk)
{
  device_add_disk(NULL, disk);
}

void device_add_disk(struct device *parent, struct gendisk *disk)
{
  blk_register_region(disk_devt(disk), disk->minors, NULL,
          exact_match, exact_lock, disk);
}

void blk_register_region(dev_t devt, unsigned long range, struct module *module,
       struct kobject *(*probe)(dev_t, int *, void *),
       int (*lock)(dev_t, void *), void *data)
{
  kobj_map(bdev_map, devt, range, module, probe, lock, data);
}
```

##### sget
```C++
// drivers/scsi/sd.c
static const struct block_device_operations sd_fops = {
  .owner      = THIS_MODULE,
  .open      = sd_open,
  .release    = sd_release,
  .ioctl      = sd_ioctl,
  .getgeo      = sd_getgeo,
#ifdef CONFIG_COMPAT
  .compat_ioctl    = sd_compat_ioctl,
#endif
  .check_events    = sd_check_events,
  .revalidate_disk  = sd_revalidate_disk,
  .unlock_native_capacity  = sd_unlock_native_capacity,
  .pr_ops      = &sd_pr_ops,
};

static int sd_open(struct block_device *bdev, fmode_t mode)
{ }

static int set_bdev_super(struct super_block *s, void *data)
{
  s->s_bdev = data;
  s->s_dev = s->s_bdev->bd_dev;
  s->s_bdi = bdi_get(s->s_bdev->bd_bdi);
  return 0;
}

struct super_block *sget(struct file_system_type *type,
  int (*test)(struct super_block *,void *),
  int (*set)(struct super_block *,void *),
  int flags,
  void *data)
{
  return sget_userns(type, test, set, flags, user_ns, data);
}

struct super_block *sget_userns(struct file_system_type *type,
  int (*test)(struct super_block *,void *),
  int (*set)(struct super_block *,void *),
  int flags, struct user_namespace *user_ns,
  void *data)
{
  struct super_block *s = NULL;
  struct super_block *old;
  int err;

  if (!s) {
    s = alloc_super(type, (flags & ~MS_SUBMOUNT), user_ns);
  }
  err = set(s, data);

  s->s_type = type;
  strlcpy(s->s_id, type->name, sizeof(s->s_id));
  list_add_tail(&s->s_list, &super_blocks);
  hlist_add_head(&s->s_instances, &type->fs_supers);
  spin_unlock(&sb_lock);
  get_filesystem(type);
  register_shrinker(&s->s_shrink);
  return s;
}
```
![linux-io-gendisk.png](../Images/linux-io-gendisk.png)

![linux-io-bd.png](../Images/linux-io-bd.png)

### Questions:
1. How to implement the IO port of dev register, and mmap of IO dev cache?
2. How to assign virutal irq to a cpu?
3. The interrupt controller sends interrupt to each cpu, which one will handle it?
4. How interrupt controller converts pthysical irq to interrupt vector?
5. Will first 32 interrupt call do_IRQ?