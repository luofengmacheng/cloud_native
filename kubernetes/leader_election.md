## leader-election选主机制

### 1 为什么需要leader-election？

在集群中存在某种业务场景，一批相同功能的进程同时运行，但是同一时刻，只能有一个工作，只有当正在工作的进程异常时，才会由另一个进程进行接管。这种业务逻辑通常用于实现一主多从。

如果有人认为，传统应用需要部署多个通常是为了容灾，而在k8s上运行的Pod受控制器管理，如果Pod异常或者Pod所在宿主机宕机，Pod是可以漂移到其他节点的，所以，不需要部署多个Pod，只需要部署一个Pod就行。k8s上的Pod确实可以漂移，但是，如果宿主机宕机，k8s认为Pod异常，并在其他节点重建Pod是有周期的，不能在查询不到Pod状态时立刻就将Pod驱逐掉，也许节点只是临时不可用呢？例如，负载很高，因此，判断宿主机宕机需要有个时间短。![k8s节点故障时，工作负载的调度周期](https://blog.csdn.net/tonadowang/article/details/117421411) 因此，在k8s中运行一主多从是为了能够实现主的快速切换。

### 2 kubernetes中的leader-election

k8s中也有这种业务场景，在多master场景下，只能有一个master上的进程工作，例如，scheduler和controller-manager。以scheduler来说，它的工作是给Pod分配合适的宿主机，如果有多个scheduler同时运行，就会出现竞争，因此，如果允许这种场景存在的话，就又需要实现一种调度逻辑：某个Pod由哪个scheduler进行调度，这相当于又要实现一层调度。但是，实际上调度工作是相对比较简单的，不需要多个scheduler进行负载，只需要一个scheduler进行调度就行。因此，k8s提供了leader-election的能力。

leader-election的具体工作方式是：各候选者将自身的信息写入某一个资源，如果写成功，某个后选择就称为了主，其他就是备，同时，在之后主会定期更新资源的时间，如果超过一段时间未更新时间，其他候选者发现资源的最后更新时间超过一定值，就会认为主挂掉，然后会向资源写入自身信息，从而成为新的主。

基于该原理，有一个现成的镜像可以使用：instana/leader-elector。

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: leader
  name: leader
spec:
  replicas: 3
  selector:
    matchLabels:
      app: leader
  template:
    metadata:
      labels:
        app: leader
    spec:
      containers:
      - image: instana/leader-elector:0.5.13
        name: leader
        command: ["/app/server","--id=$(POD_NAME)","--election=leader-election","--http=0.0.0.0:4040"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
```

上面的yaml有两个需要注意的地方：

* /app/server是二进制程序，id参数是候选者的唯一标识，election是资源名称，http是应用监听的IP和端口号
* 将pod的名称作为id参数，也就是候选者的唯一标识

创建deploy后，会启动三个Pod，通过kubectl logs可以看到只有一个Pod成为主，也就是向资源名称为leader-election的Endpoint写入了自身的Pod名称。然后通过代理(kubectl proxy)访问：http://localhost:8001/api/v1/namespaces/default/pods/<leader-pod-name>:4040/proxy/，就会看到主的Pod名称。

知道了leader-election的大概原理，也知道了上面的镜像可以直接实现主的选举，那么如何使用呢？

#### 2.1 Sidecar

直接将上面的leader-elector镜像作为Sidecar，将Pod名称作为候选者的唯一标识，然后将Pod名称也注入到环境变量，在业务进程起来后，定时调用`http://localhost:4040`就可以获取主，如果发现主的名称与自身的Pod的名称一致，就执行业务逻辑，否则一直等待。

#### 2.2 SDK

使用Sidecar的好处是比较方便，开发成本低，不便的地方就是，适用场景有限，只能写入Endpoint资源。因此，在某些场景下，可以使用SDK，直接基于leader-election库开发。

![k8s-leader-election](https://raw.githubusercontent.com/mayankshah1607/k8s-leader-election/master/main.go)

创建一个Lease类型的锁(当然，也可以是其他类型，但是lease更加轻量)，创建资源时需要指定资源的命名空间、名称、标识(这一批Pod都会该命名空间的资源写入自身的唯一标识)。然后调用leaderelection库中的RunOrDie()函数，此时会指定：

* Lock：资源锁，将前面创建的Lease类型锁填入
* ReleaseOnCancel：
* LeaseDuration：租约时间
* RenewDeadline：leader刷新超时
* RetryPeriod：刷新租约的时间间隔
* Callbacks：指定成为leader时要执行的业务逻辑(OnStartedLeading)，从leader变成非leader时要执行的逻辑(OnStoppedLeading)，leader变更时要执行的逻辑(OnNewLeader)。

### 3 具体实现机制

``` golang
// leaderelection/leaderelection.go
func (le *LeaderElector) Run(ctx context.Context) {
	defer runtime.HandleCrash()
	defer func() {
		le.config.Callbacks.OnStoppedLeading()
	}()

	// 申请资源锁，有三种情形：
	// 1 出错，则返回false，Run()直接退出
	// 2 获取到锁了，则返回true，执行回调函数
	// 3 未获取到锁，acquire()函数不会返回
	if !le.acquire(ctx) {
		return // ctx signalled done
	}
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	// 申请成功后，执行回调函数
	go le.config.Callbacks.OnStartedLeading(ctx)

	// 定时刷新租约
	le.renew(ctx)
}

// 申请资源锁
func (le *LeaderElector) acquire(ctx context.Context) bool {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()
	succeeded := false
	desc := le.config.Lock.Describe()
	klog.Infof("attempting to acquire leader lease %v...", desc)

	// 每隔RetryPeriod去申请资源锁，或者更新
	wait.JitterUntil(func() {
		succeeded = le.tryAcquireOrRenew(ctx)
		le.maybeReportTransition()
		if !succeeded {
			// 没有获取到锁，下一次再尝试
			klog.V(4).Infof("failed to acquire lease %v", desc)
			return
		}

		// 成功获取到锁，则退出
		le.config.Lock.RecordEvent("became leader")
		le.metrics.leaderOn(le.config.Name)
		klog.Infof("successfully acquired lease %v", desc)
		cancel()
	}, le.config.RetryPeriod, JitterFactor, true, ctx.Done())
	return succeeded
}

func (le *LeaderElector) renew(ctx context.Context) {
	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	// 每隔RetryPeriod尝试更新租约
	wait.Until(func() {
		timeoutCtx, timeoutCancel := context.WithTimeout(ctx, le.config.RenewDeadline)
		defer timeoutCancel()
		err := wait.PollImmediateUntil(le.config.RetryPeriod, func() (bool, error) {
			return le.tryAcquireOrRenew(timeoutCtx), nil
		}, timeoutCtx.Done())

		le.maybeReportTransition()
		desc := le.config.Lock.Describe()
		if err == nil {
			klog.V(5).Infof("successfully renewed lease %v", desc)
			return
		}
		le.config.Lock.RecordEvent("stopped leading")
		le.metrics.leaderOff(le.config.Name)
		klog.Infof("failed to renew lease %v: %v", desc, err)
		cancel()
	}, le.config.RetryPeriod, ctx.Done())

	// if we hold the lease, give it up
	if le.config.ReleaseOnCancel {
		le.release()
	}
}

// 尝试获取或者更新资源锁
func (le *LeaderElector) tryAcquireOrRenew(ctx context.Context) bool {
	now := metav1.Now()
	leaderElectionRecord := rl.LeaderElectionRecord{
		HolderIdentity:       le.config.Lock.Identity(),
		LeaseDurationSeconds: int(le.config.LeaseDuration / time.Second),
		RenewTime:            now,
		AcquireTime:          now,
	}

	// 1 获取资源锁记录
	oldLeaderElectionRecord, oldLeaderElectionRawRecord, err := le.config.Lock.Get(ctx)
	if err != nil {
		if !errors.IsNotFound(err) {
			klog.Errorf("error retrieving resource lock %v: %v", le.config.Lock.Describe(), err)
			return false
		}

		// 创建资源锁
		if err = le.config.Lock.Create(ctx, leaderElectionRecord); err != nil {
			klog.Errorf("error initially creating leader election record: %v", err)
			return false
		}

		le.setObservedRecord(&leaderElectionRecord)

		return true
	}

	// 2 将资源锁记录与缓存的上一次的值进行对比
	// 如果当前不是leader，并且资源锁没有过期，则退出
	if !bytes.Equal(le.observedRawRecord, oldLeaderElectionRawRecord) {
		le.setObservedRecord(oldLeaderElectionRecord)

		le.observedRawRecord = oldLeaderElectionRawRecord
	}
	if len(oldLeaderElectionRecord.HolderIdentity) > 0 &&
		le.observedTime.Add(le.config.LeaseDuration).After(now.Time) &&
		!le.IsLeader() {
		klog.V(4).Infof("lock is held by %v and has not yet expired", oldLeaderElectionRecord.HolderIdentity)
		return false
	}

	// 3. We're going to try to update. The leaderElectionRecord is set to it's default
	// here. Let's correct it before updating.
	if le.IsLeader() {
		// 当前是leader，锁资源未过期，将之前的资源锁的数据填充到新的资源锁中(申请锁时间，切换次数)
		leaderElectionRecord.AcquireTime = oldLeaderElectionRecord.AcquireTime
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions
	} else {
		// 当前不是leader
		leaderElectionRecord.LeaderTransitions = oldLeaderElectionRecord.LeaderTransitions + 1
	}

	// 更新资源锁
	if err = le.config.Lock.Update(ctx, leaderElectionRecord); err != nil {
		klog.Errorf("Failed to update lock: %v", err)
		return false
	}

	le.setObservedRecord(&leaderElectionRecord)
	return true
}
```