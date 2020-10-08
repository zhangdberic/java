# 排查性能问题

## CPU使用率高

### 处理方式1

使用命令：top 发现高负载java进程，例如：进程号12345

使用命令：top -Hp 进程号，例如：top -Hp 12345，查找cpu负载高的线程(轻进程)，例如：轻进程号 7890

使用命令：jstack 进程号|grep -A 10 轻进程号(十六进制，而且涉及到字母的必须小写)，例如：jstack 12345|grep -A 10 1ed2，这里的-A 10为要显示的下行数，可以调整。你可以使用 -C 10来显示上下各10行。

显示结果，重点关注的就是，第1行的，java.lang.Thread.State（线程当前状态）。

### 处理方式2

查询BLOCKED状态的线程，有的时候如果按照**处理方式1**来查找问题，需要一点操作时间，可能问题已经过期了。这里我们使用一种快速的方法，直接定位BLOCKED状态的线程。

使用命令：jstack 进程号 |grep BLOCKED -C 10，例如：jstack 46967 |grep BLOCKED -C 10

### 线程状态

https://www.cnblogs.com/lemon-flm/p/11775893.htmlk

java.lang.Thread.State: TIMED_WAITING (sleeping)     当前正在在执行sleep方法,处在休眠状态。

java.lang.Thread.State: TIMED_WAITING (on object monitor)   当前正在执行object.wait(xxxx)方法，处在挂起状态，等待被notify唤醒。

java.lang.Thread.State: WAITING (parking)   使用concurrent的park方法，处在挂起状态，等待unpark方法唤醒。

java.lang.Thread.State: RUNNABLE 当前线程正在运行。

java.lang.Thread.State: BLOCKED (on object monitor)  线程处于阻塞状态，正在等待一个monitor lock。通常情况下，是因为本线程与其他线程公用了一个锁。其他在线程正在使用这个锁进入某个synchronized同步方法块或者方法，而本线程进入这个同步代码块也需要这个锁，最终导致本线程处于阻塞状态。

例如：我们观察下面的示例，两个线程不一样，但waiting to lock <0x00000006935a7d68>都一样的，说明两个线程都在等待同一把锁。如果多次执行jstack 进程号 |grep BLOCKED -C 10，都发现BLOCKED，而且代码位置都一样，并且都在等待同一把锁，则说明代码写的有性能问题，需要调整。如果间隔5秒再执行jstack 进程号 |grep BLOCKED -C 10发现还是waiting to lock <0xxxxxxxxxxxxx>还是以前的锁，则说明已经死锁了，锁长时间不能释放。

```
"catalina-exec-163" #13923 daemon prio=5 os_prio=0 tid=0x00007fa24c007800 nid=0x811f waiting for monitor entry [0x00007fa3a854f000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at com.fr.web.core.FormSessionIDInfor.getForm2Show(Unknown Source)
	- waiting to lock <0x00000006935a7d68> (a com.fr.web.core.FormSessionIDInfor)
	at com.fr.web.core.A.ED.actionCMD(Unknown Source)
	at com.fr.web.core.WebActionsDispatcher.dealForActionCMD(Unknown Source)
	at com.fr.web.core.WebActionsDispatcher.dealForActionDefaultCmd(Unknown Source)
	at com.fr.web.core.WebActionsDispatcher.dealForActionCMD(Unknown Source)
	at com.fr.web.core.A.WA.process(Unknown Source)
	at com.fr.web.core.ReportDispatcher.dealWithOp(Unknown Source)
	at com.fr.web.core.ReportDispatcher.dealWeblet(Unknown Source)
	at com.fr.web.core.ReportDispatcher.dealWithRequest(Unknown Source)


"pool-15-thread-4" #234 prio=5 os_prio=0 tid=0x00007fa284002800 nid=0xbc1a waiting for monitor entry [0x00007fa3ab2f1000]
   java.lang.Thread.State: BLOCKED (on object monitor)
	at com.fr.web.core.FormSessionIDInfor.clearPageSet(Unknown Source)
	- waiting to lock <0x00000006935a7d68> (a com.fr.web.core.FormSessionIDInfor)
	at com.fr.web.core.FormSessionIDInfor.release(Unknown Source)
	at com.fr.web.core.SessionDealWith.processCloseSession(Unknown Source)
	at com.fr.web.core.SessionDealWith.access$300(Unknown Source)
	at com.fr.web.core.SessionDealWith$4.run(Unknown Source)
	at java.util.concurrent.Executors$RunnableAdapter.call(Executors.java:511)
	at java.util.concurrent.FutureTask.run(FutureTask.java:266)
	at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1149)
	at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:624)
```











