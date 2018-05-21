---
layout: post
title: 【SPDK】四、reactor线程
date: 2018-05-21 
tags: SPDK
---

&emsp;&emsp;reactor线程是SPDK中负责实际业务处理逻辑的单元，它们在vhsot服务启动时创建，直到服务停止。目前还不支持reactor线程的动态增减。

### reactor线程总流程

&emsp;&emsp;我们顺着vhost进程的代码执行顺序来看看总体流程：

```
spdk/app/vhost/vhost.c:

int
main(int argc, char *argv[])
{
    struct spdk_app_opts opts = {};
    int rc;

    /* 首先进行参数解析，解析后的结果保存于opts中 */

    vhost_app_opts_init(&opts);

    if ((rc = spdk_app_parse_args(argc, argv, &opts, "f:S:",
        vhost_parse_arg, vhost_usage)) !=
        SPDK_APP_PARSE_ARGS_SUCCESS) {
        exit(rc);
    }

    ...

    /* 接着根据配置文件指明的物理核启动reactors线程(主线程最终也成为一个reactor)。
        这些reactors线程会执行轮循函数，直到外部将服务状态置为退出 */

    /* Blocks until the application is exiting */
    rc = spdk_app_start(&opts, vhost_started, NULL, NULL);

    /* 所有reactor线程退出后，进行资源清理 */
    spdk_app_fini();

    return rc;
}
```

&emsp;&emsp;上述整体流程中最为重要的便是spdk_app_start函数，该函数内部调用了DPDK关于系统CPU、内存、PCI设备管理等通用性服务代码，这里我们尽可能以理解其功能为主而不做深入的代码分析：

```
spdk/lib/event/app.c:

int
spdk_app_start(struct spdk_app_opts *opts, spdk_event_fn start_fn,
void *arg1, void *arg2)
{
    struct spdk_conf	*config = NULL;
    int			rc;
    struct spdk_event	*app_start_event;

    ...

    /* 将配置文件中的内容导入到config对象中 */
    config = spdk_app_setup_conf(opts->config_file);
    ...
    spdk_app_read_config_file_global_params(opts);

    ...

    /* 调用DPDK系统服务：
        (1)通过内核sysfs获取物理CPU信息，并通过配置文件指定的运行核，在各个核上启动服务线程；
        各服务线程启动后因为在等待主线程给它们发送需要执行的任务而处于睡眠状态；
        (2)基于大页内存创建内存池以供其它模块使用；
        (3)初始化PCI设备枚举服务，可以实现类似内核的设备发现及驱动初始化流程。SPDK基于此并借
        助内核uio或vfio驱动实现全用户态的PCI驱动 */
     /* 完成DPDK的初始化后，SPDK会建立一张由vva(vhost virtual address)到pa(physical address)
        的内存映射表g_vtophys_map。每当有新的内存映射到vhost中时，都需要调用spdk_mem_register在该
        表中注册新的映射关系。设计该表的原因是当SPDK向物理设备发送DMA请求时，需要向设备提供pa而非vva */
    if (spdk_app_setup_env(opts) < 0) {
        ...
    }

    /* 这里为reactors分配相应的内存 */
    /*
     * If mask not specified on command line or in configuration file,
     *  reactor_mask will be 0x1 which will enable core 0 to run one
     *  reactor.
     */
    if ((rc = spdk_reactors_init(opts->max_delay_us)) != 0) {
        ...
    }

    ...

    /* 设置一些全局变量 */
    memset(&g_spdk_app, 0, sizeof(g_spdk_app));
    g_spdk_app.config = config;
    g_spdk_app.shm_id = opts->shm_id;
    g_spdk_app.shutdown_cb = opts->shutdown_cb;
    g_spdk_app.rc = 0;
    g_init_lcore = spdk_env_get_current_core();
    g_app_start_fn = start_fn;
    g_app_start_arg1 = arg1;
    g_app_start_arg2 = arg2;
    app_start_event = spdk_event_allocate(g_init_lcore, start_rpc, (void *)opts->rpc_addr, NULL);

    /* 初始化SPDK的各个子系统，如bdev、vhost均为子系统。但这里需注意一点，此处仅是产生了一个初始化事件，事件的处理要在
        reactor线程正式进入轮循函数后才开始 */
    spdk_subsystem_init(app_start_event);

    /* 从此处开始，各个线程(包括主线程)开始执行_spdk_reactor_run，线程名也正式变更为reactor_X；
        直到所有线程均退出_spdk_reactor_run后，主线程才会返回 */
    /* This blocks until spdk_app_stop is called */
    spdk_reactors_start();

    return g_spdk_app.rc;
    ...    
}
```

&emsp;&emsp;再看一下spdk_reactors_start：

```
spdk/lib/event/reactor.c:

void
spdk_reactors_start(void)
{
    struct spdk_reactor *reactor;
    uint32_t i, current_core;
    int rc;

    g_reactor_state = SPDK_REACTOR_STATE_RUNNING;
    g_spdk_app_core_mask = spdk_cpuset_alloc();

    /* 针对主线程之外的其它核上的线程，通过发送通知使它们开始执行_spdk_reactor_run */
    current_core = spdk_env_get_current_core();
    SPDK_ENV_FOREACH_CORE(i) {
        if (i != current_core) {
            reactor = spdk_reactor_get(i);
            rc = spdk_env_thread_launch_pinned(reactor->lcore, _spdk_reactor_run, reactor);
            ...
        }
        spdk_cpuset_set_cpu(g_spdk_app_core_mask, i, true);
    }

    /* 主线程也会执行_spdk_reactor_run */
    /* Start the master reactor */
    reactor = spdk_reactor_get(current_core);
    _spdk_reactor_run(reactor);

    /* 主线程退出后会等待其它核上的线程均退出 */
    spdk_env_thread_wait_all();

    /* 执行到此处，说明vhost服务进程即将退出 */
    g_reactor_state = SPDK_REACTOR_STATE_SHUTDOWN;
    spdk_cpuset_free(g_spdk_app_core_mask);
    g_spdk_app_core_mask = NULL;
}
```

### 轮循函数_spdk_reactor_run

&emsp;&emsp;通过对vhost代码流程的分析，我们看到vhost中所有线程最终都会调用_spdk_reactor_run，该函数是一个死循环，由此实现轮循逻辑：

```
spdk/lib/event/reactor.c:

static int
_spdk_reactor_run(void *arg)
{
    struct spdk_reactor	*reactor = arg;
    struct spdk_poller	*poller;
    uint32_t		event_count;
    uint64_t		idle_started, now;
    uint64_t		spin_cycles, sleep_cycles;
    uint32_t		sleep_us;
    uint32_t		timer_poll_count;
    char			thread_name[32];

    /* 重新命名线程名，reactor_[核号] */
    snprintf(thread_name, sizeof(thread_name), "reactor_%u", reactor->lcore);

    /* 创建SPDK线程对象：
        (1)线程间通过_spdk_reactor_send_msg发送消息，本质是向接收方的event队列中添加事件；
        (2)线程通过_spdk_reactor_start_poller和_spdk_reactor_stop_poller启动和停止poller；
        (3)IO Channel等线程相关对象也会记录到线程对象中 */
    if (spdk_allocate_thread(_spdk_reactor_send_msg,
            _spdk_reactor_start_poller,
            _spdk_reactor_stop_poller,
            reactor, thread_name) == NULL) {
        return -1;
    }
    
    /* spin_cycles代表最短轮循时间 */
    spin_cycles = SPDK_REACTOR_SPIN_TIME_USEC * spdk_get_ticks_hz() / SPDK_SEC_TO_USEC;
    /* sleep_cycles代表最长睡眠时间 */
    sleep_cycles = reactor->max_delay_us * spdk_get_ticks_hz() / SPDK_SEC_TO_USEC;
    idle_started = 0;
    timer_poll_count = 0;

    /* 轮循的死循环正式开始 */
    while (1) {
        bool took_action = false;

        /* 首先，每个reactor线程通过DPDK的无锁队列实现了一个事件队列；这里从事件队列中取出事件并调用事件
            的处理函数。例如，vhost的子系统的初始化即是在spdk_subsystem_init中产生了一个verify事件并
            添加到主线程reactor的事件队列中，该事件处理函数为spdk_subsystem_verify */
        event_count = _spdk_event_queue_run_batch(reactor);
        if (event_count > 0) {
            took_action = true;
        }

        /* 接着，每个reactor线程从active_pollers链表头部取出一个poller并调用其fn函数。poller代表一次
            具体的处理动作，例如处理某个vhost_blk设备的所有IO环中的请求，又或者处理后端NVMe某个queue 
            pair中的所有响应 */
        poller = TAILQ_FIRST(&reactor->active_pollers);
        if (poller) {
            TAILQ_REMOVE(&reactor->active_pollers, poller, tailq);
            poller->state = SPDK_POLLER_STATE_RUNNING;
            poller->fn(poller->arg);
            if (poller->state == SPDK_POLLER_STATE_UNREGISTERED) {
                free(poller);
            } else {
                poller->state = SPDK_POLLER_STATE_WAITING;
                TAILQ_INSERT_TAIL(&reactor->active_pollers, poller, tailq);
            }
            took_action = true;
        }

        /* 最后，reactor线程还实现了定时器逻辑，这里判断是否有定时器到期；如果确有定时器到期则执行其回调并将
            其放到定时器队列尾部 */
        if (timer_poll_count >= SPDK_TIMER_POLL_ITERATIONS) {
            poller = TAILQ_FIRST(&reactor->timer_pollers);
            if (poller) {
                now = spdk_get_ticks();

                if (now >= poller->next_run_tick) {
                    TAILQ_REMOVE(&reactor->timer_pollers, poller, tailq);
                    poller->state = SPDK_POLLER_STATE_RUNNING;
                    poller->fn(poller->arg);
                    if (poller->state == SPDK_POLLER_STATE_UNREGISTERED) {
                        free(poller);
                    } else {
                        poller->state = SPDK_POLLER_STATE_WAITING;
                        _spdk_poller_insert_timer(reactor, poller, now);
                    }
                    took_action = true;
                }
            }
            timer_poll_count = 0;
        } else {
            timer_poll_count++;
        }

        /* 下面的逻辑主要用来决定轮循线程是否可以睡眠一会 */

        if (took_action) {
            /* We were busy this loop iteration. Reset the idle timer. */
            idle_started = 0;
        } else if (idle_started == 0) {
            /* We were previously busy, but this loop we took no actions. */
            idle_started = spdk_get_ticks();
        }

        /* Determine if the thread can sleep */
        if (sleep_cycles && idle_started) {
            now = spdk_get_ticks();
            if (now >= (idle_started + spin_cycles)) { /* 保证轮循线程最少已执行了spin_cycles */
                sleep_us = reactor->max_delay_us;

                poller = TAILQ_FIRST(&reactor->timer_pollers);
                if (poller) {
                    /* There are timers registered, so don't sleep beyond
                     * when the next timer should fire */
                    if (poller->next_run_tick < (now + sleep_cycles)) {
                        if (poller->next_run_tick <= now) {
                            sleep_us = 0;
                        } else {
                            sleep_us = ((poller->next_run_tick - now) *
                                SPDK_SEC_TO_USEC) / spdk_get_ticks_hz();
                        }
                    }
                }

                if (sleep_us > 0) {
                    usleep(sleep_us);
                }

                /* After sleeping, always poll for timers */
                timer_poll_count = SPDK_TIMER_POLL_ITERATIONS;
            }
        }

        if (g_reactor_state != SPDK_REACTOR_STATE_RUNNING) {
            break;
        }
    } /* 死循环结束 */

    ...
    spdk_free_thread();
    return 0;
}
```

&emsp;&emsp;至此，reactor线程整体执行逻辑已分析完成，后续我们将以verify_event为线索开始分析各个子系统的初始化过程。

<br>
转载请注明：[吴斌的博客](https://rootw.github.io) » [【SPDK】四、reactor线程](https://rootw.github.io/2018/05/SPDK-reactors-init/) 
