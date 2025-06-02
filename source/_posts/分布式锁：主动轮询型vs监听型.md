---
title: 分布式锁：主动轮询型vs监听回调型
tags: [多线程, 分布式, ]
category:
  - [项目总结]
mathjax: true
toc: true
date: 2025-05-15 18:12:42
---
本文将介绍分布式锁的两种模型，共分三个章节，第一章节讲讲如何通过`redis`实现主动轮询型分布式锁；第二章节简要说下`etcd`中如何实现监听回调型锁，第三章节展示两种模型在不同场景下的性能比对与资源消耗情况。
<!--more-->
并发场景中，“锁”用于对临界资源进行保护，让混乱的并发访问行为在临界区变为秩序的串行访问行为。在本地场景下，由于进程内线程可以共享进程数据，互斥锁的实现较为简单；而分布式场景下，我们需要跨域对多个物理节点执行加锁操作，故而需要依赖像`redis`、`mysql`等状态存储组件，在此基础上实现“分布式锁”。

## 主动轮询型
本节基于[go-redis](https://github.com/redis/go-redis)，实现了单点模式下主动轮询模型的分布式锁，同时额外实现了“红锁”的基本功能，代码已放入笔者[Github仓库](https://github.com/LuoZero-World/distributed-lock)。由于个人水平有限，如有不当之处，敬请谅解，并欢迎大家批评指正😊

### 实现原理
1. **创建分布式锁**
相比于创建锁，该操作更类似于“创建与锁的连接”，为简化操作叫法，下述一律使用“创建分布式锁”。代码中，结构体`RedisLock`中准备了将存储于`Redis`中的KV对，其中key字段用于唯一标识一把分布式锁，而value字段用于标识加锁的主体方身份，具体使用[主机IP+进程ID+ Go协程 ID]实现“身份绑定”
    ```Go
    type RedisLock struct {
        LockOptions
        key    string
        token  string // 表示分布式环境下[主机_进程_协程]想要获得此锁
        client *redis.Client

        //看门狗运行标识 0未运行 1正在运行
        runningWatchDog int32
        //停止看门狗
        stopWatchDog context.CancelFunc
    }

    func NewRedisLock(key string, client *redis.Client, opts ...LockOption) *RedisLock {
        rLock := RedisLock{
        key:    REDIS_LOCK_KEY_PREFIX + key,
        token:  utils.GenerateID(),
        client: client,
        }
        for _, opt := range opts {
        opt(&rLock.LockOptions)
        }
        repairLock(&rLock.LockOptions)
        return &rLock
    }
    ```
2. **加锁**
加锁操作分为阻塞和非阻塞模式，可以在创建分布式锁时进行配置。非阻塞模式下，只会执行一次加锁尝试（**SETNX操作保证原子性，不可重入**），倘若失败，就直接返回错误；阻塞模式下，会通过计时器，每隔50 ms执行一次加锁尝试，如果某次请求加锁成功则直接返回，否则达到等锁超时阈值或者中途发生了预期之外的错误时终止流程。

3. **解锁**
解锁操作基于Lua脚本执行，保证操作的原子性。脚本执行时首先校验当前操作者是否拥有锁的所有权，是则解锁，否则返回错误

### 技术难点
1. **死锁问题**
客户端加锁后因进程崩溃、网络延迟等问题无法正常释放锁，导致锁永久不可用。可以在`Redis`中为每一条锁记录设置超时时间，加锁后超过这个时间锁便会自动释放，但是当业务实际处理时间超过超时时间时，业务处理方的锁会被自动释放，这显然是不合理的。
在`Redisson`中的解决方案是“**看门狗策略**”，在锁的持有方执行业务逻辑处理的过程中时，需要异步启动一个看门狗守护协程，持续为分布式锁的过期阈值进行延期操作。

2. **弱一致性问题**
避免单点故障引起数据丢失问题，`Redis`会基于主从复制的方式实现数据备份增加服务的容错性。为了保证服务的可用性和吞吐量，`Redis`在进行数据的主从同步时，采用的是异步执行机制。这种弱一致性同步就会导致，当使用方A在主节点加锁成功后，主节点突然宕机，数据还未同步至从节点，而后从节点升级为主节点造成锁记录丢失。
在`Redisson`中的解决方案是“**红锁**”，这是一种基于多`Redis`节点的分布式锁增强方案，通过向至少5个独立节点发起加锁请求并需获得半数以上成功响应来确保锁的互斥性，核心思想是**通过多数派投票机制规避单节点故障风险**。该方案网络开销较大且实现复杂，因此在多数情况下已被其他替代方案取代。

## 监听回调型
监听回调型锁在取锁失败时，并不会像主动轮询型锁持续请求锁，而是会监听锁的删除事件：当删除事件发生说明锁被释放了，此时才继续尝试取锁。
本节将重点分析监听回调型分布式锁的一个工程实践案例——`etcd`分布式锁。[etcd](https://github.com/etcd-io/etcd)是一款适合用于共享配置和服务发现的分布式KV存储组件，底层基于分布式共识算法Raft协议保证了存储服务的强一致性和高可用。

### 实现原理
1. **创建分布式锁**
结构体`Session`表示一次访问会话，背后对应的是一笔租约，用户调用`NewSession`方法构造`Session`实例时，首先申请到一个租约ID，然后**异步开启一个守护协程，进行租约续期的相关处理**。
结构体`Mutex`表示分布式锁，其中核心字段包括访问会话s、分布式锁的公共前缀pfx、pfx和租约ID拼接而成的锁使用方key字段myKey，以及锁使用方在pfx下对应的版本字段myRev
    ```Go
    type Session struct {
        client *v3.Client
        opts   *sessionOptions
        id     v3.LeaseID

        cancel context.CancelFunc
        donec  <-chan struct{}
    }

    func NewSession(client *v3.Client, opts ...SessionOption) (*Session, error) {
        lg := client.GetLogger()
        ops := &sessionOptions{ttl: defaultSessionTTL, ctx: client.Ctx()}
        for _, opt := range opts {
            opt(ops, lg)
        }

        id := ops.leaseID
        if id == v3.NoLease {
            resp, err := client.Grant(ops.ctx, int64(ops.ttl))
            if err != nil {
                return nil, err
            }
            id = resp.ID
        }

        ctx, cancel := context.WithCancel(ops.ctx)
        keepAlive, err := client.KeepAlive(ctx, id)
        if err != nil || keepAlive == nil {
            cancel()
            return nil, err
        }

        donec := make(chan struct{})
        s := &Session{client: client, opts: ops, id: id, cancel: cancel, donec: donec}

        // keep the lease alive until client error or cancelled context
        go func() {
            defer close(donec)
            for range keepAlive {
                // eat messages until keep alive channel closes
            }
        }()

        return s,nil
    }
    ```
    ```Go
    type Mutex struct {
        s *Session
        
        pfx   string
        myKey string
        myRev int64
        hdr   *pb.ResponseHeader
    }
    func NewMutex(s *Session, pfx string) *Mutex {
        return &Mutex{s, pfx + "/", "", -1, nil}
    }
    ```

2. **加锁**
调用`Mutex.tryAcquire`方法尝试插入myKey，同时获取当前锁的实际持有者。如果当前锁从未被占用，或者锁持有者的版本号与调用方的版本号一致，则代表加锁成功；否则锁已被他人占用，调用`waitDeletes`方法，监听版本号小于自己且最接近于自己的锁记录数据的删除事件。当接收到解锁事件后，检查自身租约是否过期，没有则加锁成功
    ```Go
    func (m *Mutex) Lock(ctx context.Context) error {
        resp, err := m.tryAcquire(ctx)
        if err != nil {
            return err
        }
        // if no key on prefix / the minimum rev is key, already hold the lock
        ownerKey := resp.Responses[1].GetResponseRange().Kvs
        if len(ownerKey) == 0 || ownerKey[0].CreateRevision == m.myRev {
            m.hdr = resp.Header
            return nil
        }
        client := m.s.Client()
        // wait for deletion revisions prior to myKey
        // TODO: early termination if the session key is deleted before other session keys with smaller revisions.
        _, werr := waitDeletes(ctx, client, m.pfx, m.myRev-1)
        // release lock key if wait failed
        if werr != nil {
            m.Unlock(client.Ctx())
            return werr
        }

        // make sure the session is not expired, and the owner key still exists.
        gresp, werr := client.Get(ctx, m.myKey)
        if werr != nil {
            m.Unlock(client.Ctx())
            return werr
        }

        if len(gresp.Kvs) == 0 { // is the session key lost?
            return ErrSessionExpired
        }
        m.hdr = gresp.Header

        return nil
    }
    ```

3. **监听**
`waitDeletes`方法基于一个for循环自旋，每轮处理中会获取版本号小于自己且最接近于自己的加锁方key，如果key不存在，则说明自己的版本号已经是最小的，退出监听；否则调用`waitDelete`方法阻塞监听这个 key的删除事件。
    ```Go
    func waitDeletes(ctx context.Context, client *v3.Client, pfx string, maxCreateRev int64) (*pb.ResponseHeader, error) {
        getOpts := append(v3.WithLastCreate(), v3.WithMaxCreateRev(maxCreateRev))
        for {
            resp, err := client.Get(ctx, pfx, getOpts...)
            if err != nil {
                return nil, err
            }
            if len(resp.Kvs) == 0 {
                return resp.Header, nil
            }
            lastKey := string(resp.Kvs[0].Key)
            if err = waitDelete(ctx, client, lastKey, resp.Header.Revision); err != nil {
                return nil, err
            }
        }
    }
    ```
    ```Go
    func waitDelete(ctx context.Context, client *v3.Client, key string, rev int64) error {
        cctx, cancel := context.WithCancel(ctx)
        defer cancel()


        var wr v3.WatchResponse
        wch := client.Watch(cctx, key, v3.WithRev(rev))
        for wr = range wch {
            for _, ev := range wr.Events {
                if ev.Type == mvccpb.DELETE {
                    return nil
                }
            }
        }
        if err := wr.Err(); err != nil {
            return err
        }
        if err := ctx.Err(); err != nil {
            return err
        }
        return errors.New("lost watcher waiting for delete")
    }
    ```

4. **解锁**
直接删除调用方的KV对记录即可，如果调用方是持有锁的角色，那么删除KV对记录代表真正意义上的解锁动作；反之调用方并无持有锁，删除KV对就代表退出了抢锁流程，不会对流程产生负面影响。

### 技术难点

1.**死锁问题**
`etcd`提供了租约机制，一旦达到租约上规定的截止时间，租约就会失去效力，这类似于`Redis`中的超时时间。同时，`etcd`中还提供了续约机制，用户可以通过续约操作来延迟租约的过期时间，这类似于`Redisson`中的看门狗策略。

2.**惊群效应**
监听回调型锁的实现过程中，可能出现这样一种情况：如果一把分布式锁的竞争比较激烈，那么**锁的释放事件可能同时被多个的取锁方所监听，一旦锁被释放后所有的取锁方都会一拥而上**，然而一个轮次中只有一个取锁方能够取锁成功，因此这个过程中会存在大量性能损耗，且释放锁时刻瞬间激增的请求流量也可能会对系统稳定性产生负面效应。
`etcd`中基于锁前缀和版本号机制提出的解决方案：对于对于同一把分布式锁，锁记录key拥有共同的前缀，作为锁的标识。加锁方加锁时，会以锁前缀拼接上自身租约ID，生成完整的key，因此**加锁方key都是不同的**，都能插入锁记录数据。每个加锁方插入锁记录数据时，会获得自身key处在锁前缀范围下唯一且递增的版本号。**加锁方插入加锁记录数据不意味着加锁成功**，而是需要在插入数据后查询一次锁前缀下的记录列表，判定自身key对应的版本号是否为其中最小，是则表示加锁成功；否则监听版本号小于自己但最接近自己的那个key的删除事件。
这样所有加锁方会**根据版本号排成一条队列**，每次锁释放只会惊动下一位，惊群问题得以避免。

## 实验对比
评价指标为平均加锁时间，即从发起锁请求到成功获取锁的平均时间、每秒成功加锁次数(LSPS)、P70/P90/P99延迟，以及CPU使用率差异。本节只展示宏观上的实验结论，不展示具体数值。
### 延迟、吞吐量对比
在**并发量较小，但加锁频繁**的场景下，主动轮询型锁的平均加锁时间较低，但是**长尾效应显著**，约有10%的加锁时间为平均时间的10-20倍；而监听回调型锁的平均加锁时间在不断上升，每秒加锁成功数不断下降，说明其**处理能力在逐渐下降**，这可能是因为频繁请求使得etcd中事件堆积所致。
在**并发量较大**的场景下，主动轮询型锁的平均加锁时间随竞争数量线性增长，并且随着竞争程度的激烈，长尾效应愈发明显。同时客户端数量增加10倍，LSPS减少25%；监听回调型锁的平均加锁时间与LSPS并未随着竞争程度增加而明显变化，**较为稳定**。
但是两个场景下监听回调型锁的性能均劣于主动轮询型，这可能与笔者的实验环境仍处于单机状态有关，如不忽略**节点间的通信延迟与实际任务处理时间**，在高并发场景下监听回调型锁应该更具优势。
### 资源消耗对比
监听回调型锁的CPU使用率约比主动轮询型锁约低10%-15%，在锁竞争激烈时CPU使用率的差距更加明显
