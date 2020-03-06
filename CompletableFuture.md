 **CompletableFuture**



```java
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;
import java.util.function.BiConsumer;


private void method() throws ExecutionException, InterruptedException {
    	// 执行函数内的代码(休眠10秒)
        CompletableFuture<String> f1 = CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(10);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            return "f1";
        });
		// 当f1执行完成后,调用本函数,f1的执行返回值"f1",作为本函数的输入.
        f1.whenCompleteAsync(new BiConsumer<String, Throwable>() {
            @Override
            public void accept(String s, Throwable throwable) {
                System.out.println(System.currentTimeMillis() + ":" + s);
            }
        });

        // 执行函数内的代码(休眠5秒)
        CompletableFuture<String> f2 = CompletableFuture.supplyAsync(() -> {
            try {
                TimeUnit.SECONDS.sleep(5);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            return "f2";
        });
		// 当f2执行完成后,调用本函数,f2的执行返回值"f2",作为本函数的输入.
        f2.whenCompleteAsync(new BiConsumer<String, Throwable>() {
            @Override
            public void accept(String s, Throwable throwable) {
                System.out.println(System.currentTimeMillis() + ":" + s);
            }
        });
    
		// 定义为所有函数(allOf)
        CompletableFuture<Void> all = CompletableFuture.allOf(f1, f2);
        // 定义为任一函数(anyOf)
        //  CompletableFuture<Object> any = CompletableFuture.anyOf(f1,f2);
        // 超时,注意：即使超时了也只是get所在程抛出了异常(主线程终止),子线程还在运行
        //  CompletableFuture.allOf(f1, f2).get(11,TimeUnit.SECONDS);
    

        System.out.println(System.currentTimeMillis() + ":阻塞");
        // 阻塞，直到所有任务结束(allOf模式)。
        // 阻塞，任何一个任务结束(anyOf模式)。
        all.join();
        System.out.println(System.currentTimeMillis() + ":阻塞结束");

        //
        System.out.println("任务均已完成。");
    }
```



上面的例子，注意几点：

1.  all.join();运行时，如果抛出异常，可能有的线程还在运行(没有停止)。
2.  all.get(11,TimeUnit.SECONDS);，如果get超时，可能有的线程还在运行(没有停止)。
3.  future.cancel(true)，发送中断信号到线程，但线程不见得马上就能停止，但不管线程有没真的停止，future.isDone()马上就能返回true。