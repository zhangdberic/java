# apache common-pool2

## 配置属性

### 

| 属性名                         | 默认值  | 建议设置 | 说明                                                         |
| ------------------------------ | ------- | -------- | ------------------------------------------------------------ |
| minIdle                        | 0       | 8        | 最小空闲连接数，缓存保持这些连接不释放                       |
| maxIdle                        | 8       | 8        | 最大空闲连接数，超出的连接对象使用后备释放(关闭)             |
| maxTotal                       | 8       | 100      | 允许最大连接数                                               |
| maxWaitMills                   | -1      | 3000     | 创建连接对象的等待超时(ms)，默认值-1则永远等待               |
|                                |         |          |                                                              |
| lifo                           | true    | false    | false为先进先出，true为后进先出                              |
| testOnBorrow                   | false   | true     | 借出连接检查有效性                                           |
| testOnReturn                   | false   |          | 还回连接检查有效性                                           |
|                                |         |          |                                                              |
| timeBetweenEvictionRunsMillis  | -1      | 30000    | evict(后台清理线程)执行的时间间隔，默认值-1不执行清理线程。只要设置这个值部为-1，则开启清理(evict)线程。 |
| minEvictableIdleTimeMillis     | 1800000 | 600000   | evict线程，循环检查连接空闲时间大于当前属性时，这样的空闲连接将被清理。依据网络质量来设置这个属性。 |
| softMinEvictableIdleTimeMillis | -1      |          | evict线程，循环检查连接空闲时间大于当前属性，并且当前连接池的空闲连接数大于minIdle时，符合条件的空闲连接将被清除。默认值-1这个操作无效。 |
| numTestsPerEvictionRun         | 3       |          | evict线程，每次检查的空闲连接数。设置为-1则检查所有的空闲连接。 |
| evictorShutdownTimeoutMillis   | 10000   |          | evict线程停止退出超时等待时间                                |
| testWhileIdle                  | false   | true     | evict线程检查空闲时候，是否调用activateObject、**validateObject**、passivateObject方法检查空闲连接的有效性，无效的空闲连接将被清理。 |





## 清理线程(EvictionRun)

当前你设置了timeBetweenEvictionRuns属性(不等于默认值-1)，则开启了evict(清理)后台线程，其每间隔timeBetweenEvictionRuns(毫秒数)，执行一次清理工作。

BaseGenericObjectPool

```java
    public final void setTimeBetweenEvictionRunsMillis(
            final long timeBetweenEvictionRunsMillis) {
        this.timeBetweenEvictionRunsMillis = timeBetweenEvictionRunsMillis;
        startEvictor(timeBetweenEvictionRunsMillis);
    }
```

清理工作包括：

1.清理空闲时间超出minEvictableIdleTimeMillis设置(默认1800s)的连接，这有利于防止长时间的连接被空闲无法保证连接有效性，例如长时间空闲底层socket链接已经被防火墙断开等，而且还能减少系统资源消耗。

2.保证最小空闲连接(minIdle)数，如果空闲连接数量不能满足minIdle属性个数，则新创建连接直到满足minIdle设置。

### 计算空闲时间

池内对象(DefaultPooledObject)有一个lastReturnTime属性记录连接对象还回时间，用当前系统时间减去上次还回时间就是空闲时间。如果空闲对象被频繁的借出和还回，则lastReturnTime记录时间越接近系统时间，那么计算出的空闲时间越小。

```java
    @Override
    public long getIdleTimeMillis() {
        final long elapsed = System.currentTimeMillis() - lastReturnTime;
        // elapsed may be negative if:
        // - another thread updates lastReturnTime during the calculation window
        // - System.currentTimeMillis() is not monotonic (e.g. system time is set back)
        return elapsed >= 0 ? elapsed : 0;
    }
```



## maxIdle

maxIdle属性是指pool中最多能保留多少个空闲对象。这个属性主要是在returnObject函数中使用的。在程序使用完连接对象后，会调用returnObject函数将对象返回到pool中，但是如果当前pool中空闲对象的数量大于等于maxIdle设置会调用destroy函数来销毁对象。

GenericObjectPool

```java
        final int maxIdleSave = getMaxIdle();
        if (isClosed() || maxIdleSave > -1 && maxIdleSave <= idleObjects.size()) {
            try {
                destroy(p);
            } catch (final Exception e) {
                swallowException(e);
            }
        } 
```

minIdle和maxIdle两个属性值不等的情况下，多出的空闲连接数，什么时候释放？

maxIdle属性就是应用在returnObject()函数中，保证借出被返回的对象被缓存(不超出maxIdle属性)，多出被销毁。

minIdle属性就是应用在evict()清理线程中，evict线程会保证最小空闲连接数，但evict还是有一个重要的功能就是销毁minEvictableIdleTimeMillis(默认1800s)时间内没有被使用的空闲连接，**这个对解释上面的问题特别重要**，在系统繁忙的时候，空闲连接时间很短，空闲连接对象会被频繁的借出和还回，这个时候不能清除空闲连接。在系统不繁忙(空闲)的时候，evict清理线程就可以把空闲时间长的连接对象清除，但会保证minIdle空闲连接对象数。

