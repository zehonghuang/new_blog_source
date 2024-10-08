+++
title = '【JRaft源码分析03】成员变化'
date = 2021-03-17T10:14:50+08:00
draft = false
tags = [
    "CAP理论",
    "RAFT算法",
    "分布式一致性",
    "java"
]
categories = [
    "分布式框架"
]
+++

第三篇说成员变化，有了对选举和日志复制的认识，这个模块就很轻松简单了。

成员变化就两种情况，增加删除更换节点，和转移领导人。

## 1、更改一般节点

![一般成员节点变化](https://raw.githubusercontent.com/zehonghuang/github_blog_bak/master/source/image/%E4%B8%80%E8%88%AC%E8%8A%82%E7%82%B9%E6%88%90%E5%91%98%E5%8F%98%E5%8C%96.png)
<!--more-->

这张图看起来很复杂，但整体流程其实很简单，非常清晰明了的。下面的方法就是修改配置的统一入口。

``` java
private void unsafeRegisterConfChange(final Configuration oldConf, final Configuration newConf, final Closure done) {

    Requires.requireTrue(newConf.isValid(), "Invalid new conf: %s", newConf);
    // The new conf entry(will be stored in log manager) should be valid
    Requires.requireTrue(new ConfigurationEntry(null, newConf, oldConf).isValid(), "Invalid conf entry: %s",
        newConf);

    if (this.state != State.STATE_LEADER) {
        // error
        return;
    }
    // 配置正在更改中
    if (this.confCtx.isBusy()) {
        if (done != null) {
            Utils.runClosureInThread(done, new Status(RaftError.EBUSY, "Doing another configuration change."));
        }
        return;
    }
    if (this.conf.getConf().equals(newConf)) {
        Utils.runClosureInThread(done);
        return;
    }
    this.confCtx.start(oldConf, newConf, done);//ConfigurationCtx，启动更新配置流程。
}
```
JRaft把更新配置拆解为四个步骤：
- 1、STAGE_CATCHING_UP
    - 如果有追加或更换新节点，需要使新节点日志跟集群同步，复制完成日志后，调用catchUpClosure，下一步
- 2、STAGE_JOINT
    - 将新旧配置复制到Follower，收到大部分回应后，下一步
- 3、STAGE_STABLE
    - 通知Follower删除旧配置，收到大部分回应后，下一步
- 4、STAGE_NONE
    - ✅

### 1.1、日志追赶(STAGE_CATCHING_UP)

``` java
void start(final Configuration oldConf, final Configuration newConf, final Closure done) {
    this.done = done;
    this.stage = Stage.STAGE_CATCHING_UP;
    this.oldPeers = oldConf.listPeers();
    this.newPeers = newConf.listPeers();
    this.oldLearners = oldConf.listLearners();
    this.newLearners = newConf.listLearners();
    final Configuration adding = new Configuration();
    final Configuration removing = new Configuration();
    newConf.diff(oldConf, adding, removing);
    this.nchanges = adding.size() + removing.size();

    addNewLearners();
    if (adding.isEmpty()) {//只删不增
        nextStage();
        return;
    }
    addNewPeers(adding);
}

private void addNewPeers(final Configuration adding) {
    this.addingPeers = adding.listPeers();
    for (final PeerId newPeer : this.addingPeers) {
        if (!this.node.replicatorGroup.addReplicator(newPeer)) {
            onCaughtUp(this.version, newPeer, false);//复制器启动异常，立即放弃追赶
            return;
        }
        final OnCaughtUp caughtUp = new OnCaughtUp(this.node, this.node.currTerm, newPeer, this.version);
        final long dueTime = Utils.nowMs() + this.node.options.getElectionTimeoutMs();
        //设置caughtUp回调，等待replicator完成复制工作
        if (!this.node.replicatorGroup.waitCaughtUp(newPeer, this.node.options.getCatchupMargin(), dueTime, caughtUp)) {
            onCaughtUp(this.version, newPeer, false);
            return;
        }
    }
}
```

`Replicator`启动后依旧做日常工作，`onAppendEntriesReturned`每次复制成功都会执行`r.notifyOnCaughtUp(RaftError.SUCCESS.getNumber(), false)`。

``` java
private void notifyOnCaughtUp(final int code, final boolean beforeDestroy) {
    if (this.catchUpClosure == null) {
        return;
    }
    //这里有一大段关于waitCaughtUp定时器的，不说了

    final CatchUpClosure savedClosure = this.catchUpClosure;
    this.catchUpClosure = null;
    Utils.runClosureInThread(savedClosure, savedClosure.getStatus());//执行上面说到的OnCaughtUp
}

private void onCaughtUp(final PeerId peer, final long term, final long version, final Status st) {
    this.writeLock.lock();
    try {
        // 被选下去
        if (term != this.currTerm && this.state != State.STATE_LEADER) {
            return;
        }
        if (st.isOk()) {
            // Caught up successfully
            this.confCtx.onCaughtUp(version, peer, true);
            return;
        }
        // 超时继续等待
        if (st.getCode() == RaftError.ETIMEDOUT.getNumber()
            && Utils.monotonicMs() - this.replicatorGroup.getLastRpcSendTimestamp(peer) <= this.options
                .getElectionTimeoutMs()) {
            // ...return
        }
        this.confCtx.onCaughtUp(version, peer, false);
    } finally {
        this.writeLock.unlock();
    }
}

void com.alipay.sofa.jraft.core.NodeImpl.ConfigurationCtx#onCaughtUp
        (final long version, final PeerId peer, final boolean success) {
    
    Requires.requireTrue(this.stage == Stage.STAGE_CATCHING_UP, "Stage is not in STAGE_CATCHING_UP");
    if (success) {
        this.addingPeers.remove(peer);
        if (this.addingPeers.isEmpty()) {
            nextStage();//下一步，STAGE_JOINT
            return;
        }
        return;
    }
    reset(new Status(RaftError.ECATCHUP, "Peer %s failed to catch up.", peer));
}
```

### 1.2、同步配置信息(STAGE_JOINT & STAGE_STABLE)

这个过程有两个步骤，第一步同步新旧配置，所有写入操作兼容两种配置，第二步则是通知Follwer删除旧配置。

在过渡到第二步之前，Leader需要new & old quonum <=0，才能提交数据。
``` java
//com.alipay.sofa.jraft.core.NodeImpl.ConfigurationCtx
void nextStage() {
    Requires.requireTrue(isBusy(), "Not in busy stage");
    switch (this.stage) {
        case STAGE_CATCHING_UP:
            if (this.nchanges > 1) {
                this.stage = Stage.STAGE_JOINT;
                this.node.unsafeApplyConfiguration(new Configuration(this.newPeers, this.newLearners),
                    new Configuration(this.oldPeers), false);// new and old
                return;
            }
            // 如果只更改一个节点，网络出现故障的情况下，新增节点要么被孤立要么融入
            // 删除节点要么成功要么被孤立，不影响系统可用性
            // 可以跳过同时保留新旧配置
        case STAGE_JOINT:
            this.stage = Stage.STAGE_STABLE;
            this.node.unsafeApplyConfiguration(new Configuration(this.newPeers, this.newLearners), null, false); // only new
            break;
        case STAGE_STABLE:
            final boolean shouldStepDown = !this.newPeers.contains(this.node.serverId);
            reset(new Status());
            if (shouldStepDown) {
                this.node.stepDown(this.node.currTerm, true, new Status(RaftError.ELEADERREMOVED,
                    "This node was removed."));
            }
            break;
        case STAGE_NONE:
            // noinspection ConstantConditions
            Requires.requireTrue(false, "Can't reach here");
            break;
    }
}
//写日志，等复制给Follower后，调用configurationChangeDone，进入下一步STAGE_STABLE
private void unsafeApplyConfiguration(final Configuration newConf, final Configuration oldConf,
                                          final boolean leaderStart) {
    Requires.requireTrue(this.confCtx.isBusy(), "ConfigurationContext is not busy");
    final LogEntry entry = new LogEntry(EnumOutter.EntryType.ENTRY_TYPE_CONFIGURATION);
    entry.setId(new LogId(0, this.currTerm));
    entry.setPeers(newConf.listPeers());
    entry.setLearners(newConf.listLearners());
    if (oldConf != null) {
        entry.setOldPeers(oldConf.listPeers());
        entry.setOldLearners(oldConf.listLearners());
    }
    final ConfigurationChangeDone configurationChangeDone = new ConfigurationChangeDone(this.currTerm, leaderStart);
    // Use the new_conf to deal the quorum of this very log
    if (!this.ballotBox.appendPendingTask(newConf, oldConf, configurationChangeDone)) {
        Utils.runClosureInThread(configurationChangeDone, new Status(RaftError.EINTERNAL, "Fail to append task."));
        return;
    }
    final List<LogEntry> entries = new ArrayList<>();
    entries.add(entry);
    this.logManager.appendEntries(entries, new LeaderStableClosure(entries));
    checkAndSetConfiguration(false);
}
```

<font color=#f28080>

但是STAGE_JOINT存在一个隐患，如果Leader没来得及把`new&old`配置复制到所有Follwer就崩溃了，那么会出现一批只有`old`和一批`new&old`的节点，极有可能分别选举出term相同的Leader，一个同时存在于`new&old`配置的Follwer会收到来自不同Leader的消息。

之前看到handleAppendEntriesRequest的处理，就是直接`currTerm+1`强制让两个Leader下台并发起选举，成为新任领导。

[1] 那么新Leader可以通过日志，向只有`old`的节点追加`new`，这样集群配置会长期处在一个unstable状态。发生下次成员变化时，会直接覆盖原来的`old`，直到步骤走到`STAGE_NONE`才会恢复到stable状态。

STAGE_STABLE阶段依然有风险，Leader崩溃后会出现一批只有`new`和一批`new&old`的节点，同样会分别出现两个Leader，`new`和`new&old`配置中的Follower都有可能收到来自不同Leader的消息。

同样在handleAppendEntriesRequest做处理，如果`new`受到，`currTerm+1`可以使new且删除old的配置复制到其他节点。反之，回带上面[1]的状态。

</font>

## 2、Leader转移

这部分的逻辑更加容易，就不多解释了，可自行阅读源码。
``` java
public Status transferLeadershipTo(final PeerId peer) {
    Requires.requireNonNull(peer, "Null peer");
    this.writeLock.lock();
    try {
        if (this.state != State.STATE_LEADER) {
            return new Status(this.state == State.STATE_TRANSFERRING ? RaftError.EBUSY : RaftError.EPERM,
                    "Not a leader");
        }
        if (this.confCtx.isBusy()) {
            //nope！更改配置过程非常混乱，拒绝转移Leader
            return new Status(RaftError.EBUSY, "Changing the configuration");
        }

        PeerId peerId = peer.copy();
        // if peer_id is ANY_PEER(0.0.0.0:0:0), the peer with the largest
        // last_log_id will be selected.
        if (peerId.equals(PeerId.ANY_PEER)) {
            if ((peerId = this.replicatorGroup.findTheNextCandidate(this.conf)) == null) {
                return new Status(-1, "Candidate not found for any peer");
            }
        }
        // ...

        final long lastLogIndex = this.logManager.getLastLogIndex();
        if (!this.replicatorGroup.transferLeadershipTo(peerId, lastLogIndex)) {
            return new Status(RaftError.EINVAL, "No such peer %s", peer);
        }
        this.state = State.STATE_TRANSFERRING;
        final Status status = new Status(RaftError.ETRANSFERLEADERSHIP,
            "Raft leader is transferring leadership to %s", peerId);
        onLeaderStop(status);
        final StopTransferArg stopArg = new StopTransferArg(this, this.currTerm, peerId);
        this.stopTransferArg = stopArg;
        this.transferTimer = this.timerManager.schedule(() -> onTransferTimeout(stopArg),
            this.options.getElectionTimeoutMs(), TimeUnit.MILLISECONDS);//转移Leader有超时时间

    } finally {
        this.writeLock.unlock();
    }
    return Status.OK();
}

// 1 Leader: replicatorGroup.transferLeadershipTo -> Replicator.transferLeadership
// 2 Follower: com.alipay.sofa.jraft.core.NodeImpl#handleTimeoutNowRequest
// 3 Leader: com.alipay.sofa.jraft.core.Replicator#onTimeoutNowReturned
```