---
layout: post
title: Binder驱动 学习
abbrlink: 1abfc4dfba354c8996d7a1d19fa8fbbb
tags: []
categories:
  - framework
  - binder
date: 1755140447895
updated: 1756263112501
---

![ac9e5f7b3ed0c06c14d3f9ce50d3e726.png](/resources/a893b84c9cd44d84be231e0b41c0999e.png)

//android-msm-crosshatch-4.9-android11\
// msm-google/drivers/android/binder.c

### 前置

- BC\_TRANSACTION 进程向Binder驱动发送数据 BC开头，B代表Binder，而C代表Command
  - Binder驱动在收到BC\_TRANSACTION之后，会将分配内存，将请求数据保存到所分配的内存中。
- BR\_xxxx: Binder驱动回复:R表示Reply
  ![267dd03fb4e5c9959197201b1a99db7f.png](/resources/8eb9b42edbd04d1cb81d213df60f63fc.png)

```cpp
//内核中 描述Binder上下文信息,每有一个程序打开该文件节点时；Binder驱动中都会新建一个binder_proc对象来保存该进程的上下文信息。
struct binder_proc {
    struct hlist_node proc_node;
    struct rb_root threads; //binder线程 池
    struct rb_root nodes;// binder_node组成的红黑树,的实现(服务端 BbBinder)
    struct rb_root refs_by_desc;// handl 遍历 binder_ref(客户端 BpBinder)
    struct rb_root refs_by_node;//
    struct list_head waiting_threads;
    int pid;
    struct task_struct *tsk;
    struct files_struct *files;
    struct mutex files_lock;
    struct hlist_node deferred_work_node;
    int deferred_work;
    bool is_dead;

    struct list_head todo; // todo对列
    struct binder_stats stats;
    struct list_head delivered_death;
    int max_threads;
    int requested_threads;
    int requested_threads_started;
    atomic_t tmp_ref;
    struct binder_priority default_priority;
    struct dentry *debugfs_entry;
    struct binder_alloc alloc;
    struct binder_context *context;
    spinlock_t inner_lock;
    spinlock_t outer_lock;
};
//服务端实现
struct binder_node {
	int debug_id;
	struct binder_work work;
	union {
		struct rb_node rb_node;			 // 如果这个Binder实体还在使用，则将该节点链接到proc->nodes中。
		struct hlist_node dead_node;	 // 如果这个Binder实体所属的进程已经销毁，而这个Binder实体又被其它进程所引用，则这个Binder实体通过dead_node进入到一个哈希表中去存放
	};
	struct binder_proc *proc;			//该Binder实体所属的Binder进程
	struct hlist_head refs;				//该Binder实体的所有Binder引用组成的链表
	int internal_strong_refs;
	int local_weak_refs;
	int local_strong_refs;
	binder_uintptr_t ptr;				// Binder实体在用户空间的地址(为Binder实体对应的Server在用户空间的本地Binder的引用)
	binder_uintptr_t cookie;			// Binder实体在用户空间的其他数据(为Binder实体对应的Server在用户空间的本地Binder自身)
	unsigned has_strong_ref:1;
	unsigned pending_strong_ref:1;
	unsigned has_weak_ref:1;
	unsigned pending_weak_ref:1;
	unsigned has_async_transaction:1;
	unsigned accept_fds:1;
	unsigned min_priority:8;
	struct list_head async_todo;
};

// 客户端代理
struct binder_ref {
	/* Lookups needed: */
	/*   node + proc => ref (transaction) */
	/*   desc + proc => ref (transaction, inc/dec ref) */
	/*   node => refs + procs (proc exit) */
	int debug_id;
	struct rb_node rb_node_desc;	//关联到所属进程binder_proc->refs_by_desc红黑树中
	struct rb_node rb_node_node;	// 关联到所属进程binder_proc->refs_by_node红黑树中
	struct hlist_node node_entry;	// 关联到binder_node->refs哈希表中
	struct binder_proc *proc;		//该Binder引用所属的Binder进程
	struct binder_node *node;		//该Binder引用对应的Binder实体
	uint32_t desc;					//描述
	int strong;
	int weak;
	struct binder_ref_death *death;
};


static int binder_open(struct inode *nodp, struct file *filp)
{
    struct binder_proc *proc;
    struct binder_device *binder_dev;

    binder_debug(BINDER_DEBUG_OPEN_CLOSE, "%s: %d:%d\n", __func__,
             current->group_leader->pid, current->pid);

    proc = kzalloc(sizeof(*proc), GFP_KERNEL);
    if (proc == NULL)
        return -ENOMEM;
    spin_lock_init(&proc->inner_lock);
    spin_lock_init(&proc->outer_lock);
    atomic_set(&proc->tmp_ref, 0);
    get_task_struct(current->group_leader);
    proc->tsk = current->group_leader;
    mutex_init(&proc->files_lock);
    //初始化 todo 集合
    INIT_LIST_HEAD(&proc->todo);
    if (binder_supported_policy(current->policy)) {
        proc->default_priority.sched_policy = current->policy;
        proc->default_priority.prio = current->normal_prio;
    } else {
        proc->default_priority.sched_policy = SCHED_NORMAL;
        proc->default_priority.prio = NICE_TO_PRIO(0);
    }

    binder_dev = container_of(filp->private_data, struct binder_device,
                  miscdev);
    proc->context = &binder_dev->context;
    //初始化
    binder_alloc_init(&proc->alloc);

    binder_stats_created(BINDER_STAT_PROC);
    proc->pid = current->group_leader->pid;
    INIT_LIST_HEAD(&proc->delivered_death);
    INIT_LIST_HEAD(&proc->waiting_threads);
    //赋值给 filp->private_data
    filp->private_data = proc;

    mutex_lock(&binder_procs_lock);
    hlist_add_head(&proc->proc_node, &binder_procs);
    mutex_unlock(&binder_procs_lock);

    if (binder_debugfs_dir_entry_proc) {
        char strbuf[11];

        snprintf(strbuf, sizeof(strbuf), "%u", proc->pid);
        /*
         * proc debug entries are shared between contexts, so
         * this will fail if the process tries to open the driver
         * again with a different context. The priting code will
         * anyway print all contexts that a given PID has, so this
         * is not a problem.
         */
        proc->debugfs_entry = debugfs_create_file(strbuf, 0444,
            binder_debugfs_dir_entry_proc,
            (void *)(unsigned long)proc->pid,
            &binder_proc_fops);
    }

    return 0;
}
//vm_area_struct: 用户空间虚拟地址 vm_struct: 内核空间虚拟地址, 映射一块数据接收缓存区
static int binder_mmap(struct file *filp, struct vm_area_struct *vma)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    const char *failure_string;

    if (proc->tsk != current->group_leader)
        return -EINVAL;
    // 限制虚拟 空间不能超过 4M
    if ((vma->vm_end - vma->vm_start) > SZ_4M)
        vma->vm_end = vma->vm_start + SZ_4M;

    binder_debug(BINDER_DEBUG_OPEN_CLOSE,
             "%s: %d %lx-%lx (%ld K) vma %lx pagep %lx\n",
             __func__, proc->pid, vma->vm_start, vma->vm_end,
             (vma->vm_end - vma->vm_start) / SZ_1K, vma->vm_flags,
             (unsigned long)pgprot_val(vma->vm_page_prot));

    if (vma->vm_flags & FORBIDDEN_MMAP_FLAGS) {
        ret = -EPERM;
        failure_string = "bad vm_flags";
        goto err_bad_arg;
    }
    vma->vm_flags |= VM_DONTCOPY | VM_MIXEDMAP;
    vma->vm_flags &= ~VM_MAYWRITE;

    vma->vm_ops = &binder_vm_ops;
    vma->vm_private_data = proc;
    //核心方法 
    ret = binder_alloc_mmap_handler(&proc->alloc, vma);
    if (ret)
        return ret;
    mutex_lock(&proc->files_lock);
    proc->files = get_files_struct(current);
    mutex_unlock(&proc->files_lock);
    return 0;

err_bad_arg:
    pr_err("%s: %d %lx-%lx %s failed %d\n", __func__,
           proc->pid, vma->vm_start, vma->vm_end, failure_string, ret);
    return ret;
}
```

- binder\_alloc.c

```cpp
//alloc: binder_proc
int binder_alloc_mmap_handler(struct binder_alloc *alloc,
                  struct vm_area_struct *vma)
{
    int ret;
    const char *failure_string;
    struct binder_buffer *buffer;

    mutex_lock(&binder_alloc_mmap_lock);
    if (alloc->buffer) {
        ret = -EBUSY;
        failure_string = "already mapped";
        goto err_already_mapped;
    }

    alloc->buffer = (void __user *)vma->vm_start;
    mutex_unlock(&binder_alloc_mmap_lock);
    // 申请内存,按页申请
    alloc->pages = kzalloc(sizeof(alloc->pages[0]) *
                   ((vma->vm_end - vma->vm_start) / PAGE_SIZE),
                   GFP_KERNEL);
    if (alloc->pages == NULL) {
        ret = -ENOMEM;
        failure_string = "alloc page array";
        goto err_alloc_pages_failed;
    }
    alloc->buffer_size = vma->vm_end - vma->vm_start;

    buffer = kzalloc(sizeof(*buffer), GFP_KERNEL);
    if (!buffer) {
        ret = -ENOMEM;
        failure_string = "alloc buffer struct";
        goto err_alloc_buf_struct_failed;
    }

    buffer->user_data = alloc->buffer;
    list_add(&buffer->entry, &alloc->buffers);
    buffer->free = 1;
    binder_insert_free_buffer(alloc, buffer);
    //异步空间: buffer_size/2
    alloc->free_async_space = alloc->buffer_size / 2;
    barrier();
    alloc->vma = vma;
    alloc->vma_vm_mm = vma->vm_mm;
    /* Same as mmgrab() in later kernel versions */
    atomic_inc(&alloc->vma_vm_mm->mm_count);

    return 0;

err_alloc_buf_struct_failed:
    kfree(alloc->pages);
    alloc->pages = NULL;
err_alloc_pages_failed:
    mutex_lock(&binder_alloc_mmap_lock);
    alloc->buffer = NULL;
err_already_mapped:
    mutex_unlock(&binder_alloc_mmap_lock);
    pr_err("%s: %d %lx-%lx %s failed %d\n", __func__,
           alloc->pid, vma->vm_start, vma->vm_end, failure_string, ret);
    return ret;
}

static int binder_update_page_range(struct binder_alloc *alloc, int allocate,
                    void __user *start, void __user *end)
{
    ...
    bool need_mm = false;
    //申请地址
    for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
        page = &alloc->pages[(page_addr - alloc->buffer) / PAGE_SIZE];
        //找到没有被使用的
        if (!page->page_ptr) {
            //需要重新申请
            need_mm = true;
            break;
        }
    }
    ...
    for (page_addr = start; page_addr < end; page_addr += PAGE_SIZE) {
        int ret;
        bool on_lru;
        size_t index;

        index = (page_addr - alloc->buffer) / PAGE_SIZE;
        page = &alloc->pages[index];

        if (page->page_ptr) {
            trace_binder_alloc_lru_start(alloc, index);

            on_lru = list_lru_del(&binder_alloc_lru, &page->lru);
            WARN_ON(!on_lru);

            trace_binder_alloc_lru_end(alloc, index);
            continue;
        }

        if (WARN_ON(!vma))
            goto err_page_ptr_cleared;

        trace_binder_alloc_page_start(alloc, index);
        //申请物理内存
        page->page_ptr = alloc_page(GFP_KERNEL |
                        __GFP_HIGHMEM |
                        __GFP_ZERO);
        if (!page->page_ptr) {
            pr_err("%d: binder_alloc_buf failed for page at %pK\n",
                alloc->pid, page_addr);
            goto err_alloc_page_failed;
        }
        page->alloc = alloc;
        INIT_LIST_HEAD(&page->lru);

        user_page_addr = (uintptr_t)page_addr;
        //把用户虚拟地址与真实地址进行映射
        ret = vm_insert_page(vma, user_page_addr, page[0].page_ptr);
        if (ret) {
            pr_err("%d: binder_alloc_buf failed to map page at %lx in userspace\n",
                   alloc->pid, user_page_addr);
            goto err_vm_insert_page_failed;
        }

        if (index + 1 > alloc->pages_high)
            alloc->pages_high = index + 1;

        trace_binder_alloc_page_end(alloc, index);
        /* vm_insert_page does not seem to increment the refcount */
    }
    
}
```

### 写入数据

- framework 层: IPCThreadState.cpp

```cpp
//数据打包
status_t IPCThreadState::transact(int32_t handle,
                                  uint32_t code, const Parcel& data,
                                  Parcel* reply, uint32_t flags)
{
    ...
    err = writeTransactionData(BC_TRANSACTION, flags, handle, code, data, nullptr);
    ...
    err = waitForResponse(nullptr, nullptr);
    return err;
}
status_t IPCThreadState::writeTransactionData(int32_t cmd, uint32_t binderFlags,
    int32_t handle, uint32_t code, const Parcel& data, status_t* statusBuffer)
{
    // 最终与驱动层通行的代码
    binder_transaction_data tr;
    tr.target.ptr = 0; /* Don't pass uninitialized stack data to a remote process */
    tr.target.handle = handle;
    tr.code = code;
    tr.flags = binderFlags;
    tr.cookie = 0;
    tr.sender_pid = 0;
    tr.sender_euid = 0;

    const status_t err = data.errorCheck();
    if (err == NO_ERROR) {
        tr.data_size = data.ipcDataSize();
        tr.data.ptr.buffer = data.ipcData();
        tr.offsets_size = data.ipcObjectsCount()*sizeof(binder_size_t);
        tr.data.ptr.offsets = data.ipcObjects();
    } else if (statusBuffer) {
        tr.flags |= TF_STATUS_CODE;
        *statusBuffer = err;
        tr.data_size = sizeof(status_t);
        tr.data.ptr.buffer = reinterpret_cast<uintptr_t>(statusBuffer);
        tr.offsets_size = 0;
        tr.data.ptr.offsets = 0;
    } else {
        return (mLastError = err);
    }

    //Parcel              mOut;
    mOut.writeInt32(cmd);
    mOut.write(&tr, sizeof(tr));
}
status_t IPCThreadState::waitForResponse(Parcel *reply, status_t *acquireResult)
{
     uint32_t cmd;
    int32_t err;

    while (1) {
        if ((err=talkWithDriver()) < NO_ERROR) break;
        ...
        switch (cmd) {
        case BR_TRANSACTION_COMPLETE:
            ...
            break;
            ...
    }
}
status_t IPCThreadState::talkWithDriver(bool doReceive)
{
    
    binder_write_read bwr;
    bwr.write_size = outAvail;
    bwr.write_buffer = (uintptr_t)mOut.data();

    if (doReceive && needRead) {
        bwr.read_size = mIn.dataCapacity();
        bwr.read_buffer = (uintptr_t)mIn.data();
    } else {
        bwr.read_size = 0;
        bwr.read_buffer = 0;
    }
    ...
    // Return immediately if there is nothing to do.
    if ((bwr.write_size == 0) && (bwr.read_size == 0)) return NO_ERROR;

    bwr.write_consumed = 0;
    bwr.read_consumed = 0;
    status_t err;
    do {
        ...
        //驱动调用	
        if (ioctl(mProcess->mDriverFD, BINDER_WRITE_READ, &bwr) >= 0)
            err = NO_ERROR;
        else
            err = -errno;
        ...
    }while (err == -EINTR);
    ...
}
```

- 内核驱动层

```cpp
static long binder_ioctl(struct file *filp, unsigned int cmd, unsigned long arg)
{
    int ret;
    struct binder_proc *proc = filp->private_data;
    struct binder_thread *thread;
    unsigned int size = _IOC_SIZE(cmd);
    void __user *ubuf = (void __user *)arg;

    /*pr_info("binder_ioctl: %d:%d %x %lx\n",
            proc->pid, current->pid, cmd, arg);*/

    binder_selftest_alloc(&proc->alloc);

    trace_binder_ioctl(cmd, arg);

    ret = wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret)
        goto err_unlocked;

    thread = binder_get_thread(proc);
    if (thread == NULL) {
        ret = -ENOMEM;
        goto err;
    }

    switch (cmd) {
    case BINDER_WRITE_READ:
        ret = binder_ioctl_write_read(filp, cmd, arg, thread);
        if (ret)
            goto err;
        break;
    ...
    }
    default:
        ret = -EINVAL;
        goto err;
    }
    ret = 0;
err:
    if (thread)
        thread->looper_need_return = false;
    wait_event_interruptible(binder_user_error_wait, binder_stop_on_user_error < 2);
    if (ret && ret != -ERESTARTSYS)
        pr_info("%d:%d ioctl %x %lx returned %d\n", proc->pid, current->pid, cmd, arg, ret);
err_unlocked:
    trace_binder_ioctl_done(ret);
    return ret;
}

static int binder_ioctl_write_read(struct file *filp,
                unsigned int cmd, unsigned long arg,
                struct binder_thread *thread)
{
    ...
    if (bwr.write_size > 0) {
        ret = binder_thread_write(proc, thread,
                      bwr.write_buffer,
                      bwr.write_size,
                      &bwr.write_consumed);
        trace_binder_write_done(ret);
        if (ret < 0) {
            bwr.read_consumed = 0;
            if (copy_to_user(ubuf, &bwr, sizeof(bwr)))
                ret = -EFAULT;
            goto out;
        }
    }
    ...
}

static int binder_thread_write(struct binder_proc *proc,
            struct binder_thread *thread,
            binder_uintptr_t binder_buffer, size_t size,
{
    ...
    while (ptr < end && thread->return_error.cmd == BR_OK) {
    if (get_user(cmd, (uint32_t __user *)ptr))
            return -EFAULT;
    //偏移,下一次读到data
    ptr += sizeof(uint32_t);
    switch (cmd) {
        ...
        case BC_TRANSACTION:
        case BC_REPLY: {
            struct binder_transaction_data tr;
            if (copy_from_user(&tr, ptr, sizeof(tr)))
                return -EFAULT;
            ptr += sizeof(tr);
            binder_transaction(proc, thread, &tr,
                       cmd == BC_REPLY, 0);
            break;
        }	
    }
    }
}
static void binder_transaction(struct binder_proc *proc,
                   struct binder_thread *thread,
                   struct binder_transaction_data *tr, int reply,
                   binder_size_t extra_buffers_size)
{
    ...
    if(reply){
    
    }else{
        if (tr->target.handle) {
            ref = binder_get_ref_olocked(proc, tr->target.handle,
                             true);
            if (ref) {
                // 获取 binder进程实现 ---- target_node = ref.node;
                target_node = binder_get_node_refs_for_txn(
                        ref->node, &target_proc,
                        &return_error);
            }
            ...
        }else{
            //servicemanager 进程
            target_node = context->binder_context_mgr_node;
            if (target_node)
                target_node = binder_get_node_refs_for_txn(
                        target_node, &target_proc,
                        &return_error);
            else
                return_error = BR_DEAD_REPLY;
            ...
        }
    
    }
    ...
    // copy parcel data 数据
    if (binder_alloc_copy_user_to_buffer(
                &target_proc->alloc,
                t->buffer, 0,
                (const void __user *)
                    (uintptr_t)tr->data.ptr.buffer,
                tr->data_size)) {
        binder_user_error("%d:%d got transaction with invalid data ptr\n",
                proc->pid, thread->pid);
        return_error = BR_FAILED_REPLY;
        return_error_param = -EFAULT;
        return_error_line = __LINE__;
        goto err_copy_data_failed;
    }
    // 从发起方用户空间数据直接拷贝到接收方内核内存映射中
    //copy parcel里flat_binder_object 、 binder_fd_object ...(具体解析不同 type )的偏移地址的数据
    if (binder_alloc_copy_user_to_buffer(
                &target_proc->alloc,
                t->buffer,
                ALIGN(tr->data_size, sizeof(void *)),
                (const void __user *)
                    (uintptr_t)tr->data.ptr.offsets,
                tr->offsets_size)) {
        binder_user_error("%d:%d got transaction with invalid offsets ptr\n",
                proc->pid, thread->pid);
        return_error = BR_FAILED_REPLY;
        return_error_param = -EFAULT;
        return_error_line = __LINE__;
        goto err_copy_data_failed;
    }
    ...
    //解析 offset 数据
    for (buffer_offset = off_start_offset; buffer_offset < off_end_offset;
         buffer_offset += sizeof(binder_size_t)) {
        struct binder_object_header *hdr;
        size_t object_size;
        struct binder_object object;
        binder_size_t object_offset;
        hdr = &object.hdr;
        switch (hdr->type) {
        case BINDER_TYPE_BINDER:
        case BINDER_TYPE_WEAK_BINDER: {
            struct flat_binder_object *fp;

            fp = to_flat_binder_object(hdr);
            //数据转换 eg: if (fp->hdr.type == BINDER_TYPE_BINDER) fp->hdr.type = BINDER_TYPE_HANDLE;....
            ret = binder_translate_binder(fp, t, thread);
            if (ret < 0) {
                return_error = BR_FAILED_REPLY;
                return_error_param = ret;
                return_error_line = __LINE__;
                goto err_translate_failed;
            }
            binder_alloc_copy_to_buffer(&target_proc->alloc,
                            t->buffer, object_offset,
                            fp, sizeof(*fp));
            
        }break;
        case BINDER_TYPE_HANDLE:
        case BINDER_TYPE_WEAK_HANDLE: {
            struct flat_binder_object *fp;

            fp = to_flat_binder_object(hdr);
            ret = binder_translate_handle(fp, t, thread);
            if (ret < 0) {
                return_error = BR_FAILED_REPLY;
                return_error_param = ret;
                return_error_line = __LINE__;
                goto err_translate_failed;
            }
            binder_alloc_copy_to_buffer(&target_proc->alloc,
                            t->buffer, object_offset,
                            fp, sizeof(*fp));
        } break;
        ...
    }
    ...
    if (!binder_proc_transaction(t, target_proc, target_thread)) {
            binder_inner_proc_lock(proc);
            binder_pop_transaction_ilocked(thread, t);
            binder_inner_proc_unlock(proc);
            goto err_dead_proc_or_thread;
    }	
    ...
        
}
// 遍历红黑树 desc --- handle
static struct binder_ref *binder_get_ref_olocked(struct binder_proc *proc,
                         u32 desc, bool need_strong_ref)
{
    struct rb_node *n = proc->refs_by_desc.rb_node;
    struct binder_ref *ref;

    while (n) {
        ref = rb_entry(n, struct binder_ref, rb_node_desc);

        if (desc < ref->data.desc) {
            n = n->rb_left;
        } else if (desc > ref->data.desc) {
            n = n->rb_right;
        } else if (need_strong_ref && !ref->data.strong) {
            binder_user_error("tried to use weak ref as strong ref\n");
            return NULL;
        } else {
            return ref;
        }
    }
    return NULL;
}


static bool binder_proc_transaction(struct binder_transaction *t,
                    struct binder_proc *proc,
                    struct binder_thread *thread)
{
    ...
    if (!thread && !pending_async)
        //从 waiting_threads 中找一个线程
        thread = binder_select_thread_ilocked(proc);
    if (thread) {
        binder_transaction_priority(thread->task, t, node_prio,
                        node->inherit_rt);
        
        binder_enqueue_thread_work_ilocked(thread, &t->work);
    } else if (!pending_async) {
        binder_enqueue_work_ilocked(&t->work, &proc->todo);
    } else {
        binder_enqueue_work_ilocked(&t->work, &node->async_todo);
    }
    if (!pending_async)
        //唤醒 waiting 线程
        binder_wakeup_thread_ilocked(proc, thread, !oneway /* sync */);

}
static void
binder_enqueue_thread_work_ilocked(struct binder_thread *thread,
                   struct binder_work *work)
{
    //放到 todo队列
    binder_enqueue_work_ilocked(work, &thread->todo);
    thread->process_todo = true;
}
```

![ba252354b91b87a71e8e84e937c358e5.png](/resources/b8857e33f3b0480a93abbd6948af0ee4.png)

![f611e2d84c0464f6248f81fb9ba93618.png](/resources/42e3762b43644c0d801e7e46b5cfee15.png)

```xml
客户端进程                    Binder驱动                    ServiceManager进程
┌─────────────┐             ┌─────────────┐               ┌─────────────┐
│             │ 1.getService│             │               │             │
│             │ handle=0    │特殊处理      │ 2.转发到SM     │             │
│             │────────────►│handle=0     │──────────────►│checkService │
│             │             │直接找到SM    │               │             │
│             │             │的binder_node │               │             │
│             │             │             │ 3.SM返回服务   │在服务列表中  │
│             │             │             │的handle=N     │查找并返回    │
│             │             │             │◄──────────────│handle       │
│             │ 4.驱动处理   │             │               │             │
│             │readStrongBind│为客户端创建  │               │             │
│             │er           │binder_ref   │               │             │
│             │             │handle=N     │               │             │
│             │ 5.用户空间    │             │               │             │
│  BpBinder   │创建BpBinder  │             │               │             │
│ (handle=N)  │◄────────────│             │               │             │
└─────────────┘             └─────────────┘               └─────────────┘


.............
全局 Binder 上下文
┌─────────────────────────────────────┐
│ binder_context                      │
│ ┌─────────────────────────────────┐ │
│ │ binder_context_mgr_node ────────┼─┼──┐
│ └─────────────────────────────────┘ │  │
└─────────────────────────────────────┘  │
                                         │
ServiceManager 进程                       │
┌─────────────────────────────────────┐  │
│ binder_proc (ServiceManager)        │  │
│ ┌─────────────────────────────────┐ │  │
│ │ nodes 红黑树                    │ │  │
│ │ ┌─────────────────────────────┐ │ │  │
│ │ │ binder_node ◄───────────────┼─┼─┼──┘
│ │ │ (ptr=0, cookie=0)           │ │ │
│ │ │ 这就是 SM 的实体对象         │ │ │
│ │ └─────────────────────────────┘ │ │
│ └─────────────────────────────────┘ │
│ refs_by_desc: (空或其他服务的引用)   │
└─────────────────────────────────────┘

客户端进程A
┌─────────────────────────────────────┐
│ binder_proc (客户端A)               │
│ ┌─────────────────────────────────┐ │
│ │ refs_by_desc 红黑树              │ │
│ │ ┌─────────────────────────────┐ │ │
│ │ │ handle=0 -> binder_ref      │ │ │
│ │ │ ┌─────────────────────────┐ │ │ │
│ │ │ │ node ───────────────────┼─┼─┼─┼──┐
│ │ │ │ desc=0                  │ │ │ │  │
│ │ │ └─────────────────────────┘ │ │ │  │
│ │ └─────────────────────────────┘ │ │  │
│ └─────────────────────────────────┘ │  │
└─────────────────────────────────────┘  │
                                         │
        指向同一个 binder_node ◄──────────┘

```

![433508565013c09e38afd31c4e367b6a.png](/resources/cbb05969daef4c85ae96580d040c4631.png)

> 短时间内多次调用,会加到async\_todo队列,可能会导致目标进程Binder内存耗尽(1M/2). 导致后续调用失败(如果目标进程 Binder操作比较耗时), 需要特别注意

参考:<https://blog.csdn.net/tkwxty/article/details/102824924> <https://blog.csdn.net/tkwxty/article/details/102843741>
