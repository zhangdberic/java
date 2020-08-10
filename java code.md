# 并发控制

## 超时等待

```java
	/**
	 * 执行shell命令,带超时限制
	 * @param commandStr
	 * @param outputCharset
	 * @param Timeout(毫秒)
	 * @return
	 * @throws ShellCommandExecuteException 
	 */
	public static String exeCmd(String commandStr, String outputCharset, int timeout)
			throws ShellCommandExecuteException {
		ExecutorService executor = Executors.newSingleThreadExecutor();
		Future<String> result = executor.submit(() -> {
			return exeCmd(commandStr, outputCharset);
		});
		try {
			return result.get(timeout, TimeUnit.MILLISECONDS);
		} catch (InterruptedException ex) {
			throw new ShellCommandExecuteException("execute shell command interrupted", ex);
		} catch (ExecutionException ex) {
			if (ex.getCause() instanceof ShellCommandExecuteException) {
				throw (ShellCommandExecuteException) ex.getCause();
			} else {
				throw new ShellCommandExecuteException("execute shell command error.", ex);
			}
		} catch (TimeoutException ex) {
			throw new ShellCommandExecuteException("execute shell command timeout", ex);
		}finally {
            // 关闭,并给其内的运行线程发送中断信号
			executor.shutdownNow();
		}
	}
```

