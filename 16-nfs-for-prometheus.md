## 背景

监控服务升级时，prometheus容器启动失败，容器日志：

```bash
[root@node-17:/home/ubuntu]$ docker logs 3e62df7edb46
@APP_HOME@ /home/prometheus
level=info ts=2019-03-19T07:57:46.504822118Z caller=main.go:222 msg="Starting Prometheus" version="(version=2.3.2, branch=HEAD, revision=71af5e29e815795e9dd14742ee7725682fa14b7b)"
level=info ts=2019-03-19T07:57:46.505008515Z caller=main.go:223 build_context="(go=go1.10.3, user=root@5258e0bd9cc1, date=20180712-14:02:52)"
level=info ts=2019-03-19T07:57:46.505043708Z caller=main.go:224 host_details="(Linux 3.10.0-693.21.1.el7.x86_64 #1 SMP Thu Feb 14 14:47:28 CST 2019 x86_64 umeops-dbmonitor-umeops-dbmonitoradaptor-ms-1-8cd4q (none))"
level=info ts=2019-03-19T07:57:46.505071027Z caller=main.go:225 fd_limits="(soft=65536, hard=65536)"
level=info ts=2019-03-19T07:57:46.50658559Z caller=main.go:533 msg="Starting TSDB ..."
level=info ts=2019-03-19T07:57:46.50673941Z caller=web.go:415 component=web msg="Start listening for connections" address=0.0.0.0:9090
level=info ts=2019-03-19T07:57:46.513288695Z caller=repair.go:39 component=tsdb msg="found healthy block" mint=1552953600000 maxt=1552960800000 ulid=01D69Y8GKK7JHYM175MPP1TWB9
level=info ts=2019-03-19T07:57:46.51923785Z caller=repair.go:39 component=tsdb msg="found healthy block" mint=1552960800000 maxt=1552968000000 ulid=01D6A547VTX2Q578NB05115078
level=info ts=2019-03-19T07:57:46.525056172Z caller=repair.go:39 component=tsdb msg="found healthy block" mint=1552968000000 maxt=1552975200000 ulid=01D6ABZZ41Z6VDEPJGXGY08AYE
level=info ts=2019-03-19T07:57:46.532293755Z caller=main.go:402 msg="Stopping scrape discovery manager..."
level=info ts=2019-03-19T07:57:46.532338425Z caller=main.go:416 msg="Stopping notify discovery manager..."
level=info ts=2019-03-19T07:57:46.532352122Z caller=main.go:438 msg="Stopping scrape manager..."
level=info ts=2019-03-19T07:57:46.53237764Z caller=main.go:412 msg="Notify discovery manager stopped"
level=info ts=2019-03-19T07:57:46.532412358Z caller=main.go:398 msg="Scrape discovery manager stopped"
level=info ts=2019-03-19T07:57:46.532404557Z caller=manager.go:464 component="rule manager" msg="Stopping rule manager..."
level=info ts=2019-03-19T07:57:46.532446114Z caller=main.go:432 msg="Scrape manager stopped"
level=info ts=2019-03-19T07:57:46.532471713Z caller=manager.go:470 component="rule manager" msg="Rule manager stopped"
level=info ts=2019-03-19T07:57:46.532518295Z caller=notifier.go:512 component=notifier msg="Stopping notification manager..."
level=info ts=2019-03-19T07:57:46.532556543Z caller=main.go:587 msg="Notifier manager stopped"
level=error ts=2019-03-19T07:57:46.532696041Z caller=main.go:596 err="Opening storage failed lock DB directory: resource temporarily unavailable"
```

## 问题分析

这个问题和之前在etcd上遇到的问题类似，联想到etcd不支持nfs存储数据，通过搜索发现，prometheus本地存储似乎不支持nfs，参考prometheus的[issue:3534](https://github.com/prometheus/prometheus/issues/3534)。该issue中，[brian-brazil](https://github.com/brian-brazil)提到[prometheus不支持非POSIX文件系统的本地存储](https://github.com/prometheus/prometheus/issues/3534#issuecomment-348598966)。

此外，在19 Oct 2018用户[Trivikramreddy](https://github.com/Trivikramreddy)遇到了和我们类似的问题，并提交了issue：

[Opening storage failed lock DB directory: resource temporarily unavailable #4761](https://github.com/prometheus/prometheus/issues/4761)

该issue中，[simonpasquier](https://github.com/simonpasquier)也提到了该问题的三个可能的原因：

> Prometheus doesn't permissions to write this directory.
> Or another process is running in parallel and already acquired the lock.
> Or you're using a file system (such as NFS) which isn't fully supported by Prometheus.

这也确实证实了prometheus在NFS的本地存储是有问题的，但他没有给出解决方案，并直接关闭了该issue。

最终在[pr5147](https://github.com/prometheus/prometheus/pull/5147)中，prometheus将本地数据仓储必须要求posix 文件系统的要求写入到文档：

> If your local storage becomes corrupted for whatever reason, your best bet is to shut down Prometheus and remove the entire storage directory. Non POSIX compliant filesystems are not supported by Prometheus's local storage, corruptions may happen, without possibility to recover. However, you can also try removing individual block directories to resolve the problem. This means losing a time window of around two hours worth of data per block directory. Again, Prometheus's local storage is not meant as durable long-term storage.

进一步的，搜索问题中的错误（`lock DB directory`）来源，在创建tsdb数据库连接的方法`db.Open()`，其中报错的代码片段如下：

```go
if !opts.NoLockfile {
    absdir, err := filepath.Abs(dir)
    if err != nil {
        return nil, err
    }
    lockf, _, err := fileutil.Flock(filepath.Join(absdir, "lock"))
    if err != nil {
        return nil, errors.Wrap(err, "lock DB directory")
    }
    db.lockf = lockf
}
```

分析可知这部分代码是检查数据仓储的lock文件的流程：

> check if lock file exists
> if it exists, check for a PID in the local process namespace. if the PID doesn't exist, ignore the lockfile

最终，[fabxc](https://github.com/fabxc)大神指出了问题所在，tsdb使用的文件锁方法 `fileutil.Flock(filepath.Join(absdir, "lock"))`仅支持posix filesystem，内部调用为: `fcntl(F_SET_LK)`，很多nfs的锁实现并没有支持posix规范：

> Finally, while POSIX file locks are supposedly NFS-safe they not always really are as there are still many NFS implementations around where locking is not properly implemented, and NFS tends to be used in heterogenous networks. The biggest problem about this is that there is no way to properly detect whether file locking works on a specific NFS mount (or any mount) or not.

## 解决方法

为此，在[issue2689](https://github.com/prometheus/prometheus/issues/2689)中给出了一个方案，跳过lock文件检查：

> I just stumbled across this issue without a resolution, having seen the same thing in our Mesos/Marathon deployment. I am starting Prometheus with --storage.tsdb.no-lockfile in an attempt to avoid this problem, otherwise will pursue deleting the pidfile on task startup.

因此，对我们的场景来说，没有prometheus多实例的场景，"/home/prometheus/data/"不会有多进程访问冲突问题，可以使用该方法跳过。

此外，还可以考虑的方案是远程数据存储和更换系统的存储卷类型，这里不再详细叙述。

## 补充知识：文件锁

Unix/Linux上有两种文件加锁机制：`fcntl()`和`flock()`。

### `flock()`

flock()源于BSD，所以通过flock()系统调用所施加的文件锁也会被称为BSD文件锁。其系统调用接口如下：

```c
#include <sys/file.h>

int flock(int fd, int operation);
```

其中，fd是待加锁的文件的file describer，这里需要说明的是flock只能对整个文件加锁。operation是锁操作，可以是申请共享锁(LOCK_SH)或者互斥锁(LOCK_EX)，也可以是解锁(LOCK_UN)，还可以是尝试发起非阻塞的锁请求(LOCK_NB)。

最后需要注意的是，`flock()`放置的是劝告式锁，而非强制性的锁。此外由于历史的原因，`flock()`不支持NFS。

### `fcntl`

`fcntl()`是posix标准和SUSv3的规定的文件加锁技术的实现，因此也成为POSIX文件锁。和BSD文件锁相比，POSIX文件锁可以把锁放在文件的任意位置。

通常，使用`fcntl()`系统调用加锁的调用形式如下：

```c
struct flock flockstr;

fcntl(fd, cmd, &flockstr);

struct flock {
  short l_type;    /* lock type: F_RDLCK,F_WRLCK,F_UNLCK */
  short l_whence;  /* SEEK_SET,SEEK_CUR,SEEK_END */
  
  off_t l_start;   
  off_t l_len;
  off_t l_pid;
}
```

其中，cmd参数可取以下值：

- F_SETLK: 用来获取(l_type为F_RDLCK或者F_WRLCK)或者释放(l_type为F_UNLCK)
- F_SETKW: 当区域内有其它进程持有非兼容锁，调用会阻塞
- F_GETLK: 用来检测区域上是否有锁

