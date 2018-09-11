title: Hystrix之信号量隔离熔断
author: Mr.G
tags:
  - 微服务
  - Hystrix
categories:
  - 微服务
date: 2018-09-11 10:38:00
---
> Hystrix 是spring cloud默认集成的熔断组件，用于保护系统的流量过载。Hystrix支持两种方式的隔离熔断操作。线程池、信号量；下面主要通过源码学习Hystrix是怎么实现信号量的熔断隔离。
<!--more -->
##### Semaphore

Hystrix 信号量模型以及实现主要在com.netflix.hystrix.AbstractCommand以及其内部类中，下面看下Semaphore的模型接口以及实现

- TryableSemaphore

```java
 /**
     * 信号量接口
     */
    /* package */static interface TryableSemaphore {

        /**
         * 获取信号量，当获取到时返回true，否则返回false
         * Use like this:
         * <p>
         * 
         * <pre>
         * if (s.tryAcquire()) {
         * try {
         * // do work that is protected by 's'
         * } finally {
         * s.release();
         * }
         * }
         * </pre>
         * 
         * @return boolean
         */
        public abstract boolean tryAcquire();

        /**
         * 当tryAcquire获取到信号量时，使用完成后调用此方法释放信号量，使信号量计数器复原
         * ONLY call release if tryAcquire returned true.
         * <p>
         * 
         * <pre>
         * if (s.tryAcquire()) {
         * try {
         * // do work that is protected by 's'
         * } finally {
         * s.release();
         * }
         * }
         * </pre>
         */
        public abstract void release();

        /**
         * 获取当前信号的计数器的大小
         * @return
         */
        public abstract int getNumberOfPermitsUsed();

    }
```
> TryableSemaphore接口提供信号量的获取以及释放操作。
> - tryAcquire: 获取信号量，获取到时返回true，获取失败返回false
> - release ：tryAcquire返回true时，通过此方法释放持有的信号量，复原信号量计数器
> - getNumberOfPermitsUsed:获取已使用的信号量数

- TryableSemaphoreNoOp

```java
    /**
     * 空操作信号量实现，此类调用tryAcquire永远返回true，始终可以获取信号量，不做最大信号量限制
     */
    /* package */static class TryableSemaphoreNoOp implements TryableSemaphore {

        public static final TryableSemaphore DEFAULT = new TryableSemaphoreNoOp();

        @Override
        public boolean tryAcquire() {
            return true;
        }

        @Override
        public void release() {

        }

        @Override
        public int getNumberOfPermitsUsed() {
            return 0;
        }

    }
```
> TryableSemaphoreNoOp类为不限制信号量的实现，当hystrix.command.default.execution.isolation.strategy!=SEMAPHORE是默认使用TryableSemaphoreNoOp标识，任何时候都可以通过信号量的检测。

- TryableSemaphoreActual

```java
  /**
     *  信号量，仅支持tryAcquire，无阻塞功能，信号量大小可动态配置
     * Semaphore that only supports tryAcquire and never blocks and that supports a dynamic permit count.
     * <p>
     * Using AtomicInteger increment/decrement instead of java.util.concurrent.Semaphore since we don't need blocking and need a custom implementation to get the dynamic permit count and since
     * AtomicInteger achieves the same behavior and performance without the more complex implementation of the actual Semaphore class using AbstractQueueSynchronizer.
     */
    /* package */static class TryableSemaphoreActual implements TryableSemaphore {
        /**
         * 许可通过的最大信号量数值，可在配置中配置
         */
        protected final HystrixProperty<Integer> numberOfPermits;
        /**
         * 信号量计数器，线程安全
         */
        private final AtomicInteger count = new AtomicInteger(0);

        public TryableSemaphoreActual(HystrixProperty<Integer> numberOfPermits) {
            this.numberOfPermits = numberOfPermits;
        }

        @Override
        public boolean tryAcquire() {
            //信号量加一并返回新的总数
            int currentCount = count.incrementAndGet();
            //对比计数总数与最大许可的数值
            if (currentCount > numberOfPermits.get()) {
                //计数总数大于最大许可，返回false，标识无信号量可用
                //信号量减一，复原计数器
                count.decrementAndGet();
                return false;
            } else {
                return true;
            }
        }

        /**
         * 释放持有的信号量
         */
        @Override
        public void release() {
            //信号量减一
            count.decrementAndGet();
        }

        /**
         * 获取信号量已使用的值
         * @return
         */
        @Override
        public int getNumberOfPermitsUsed() {
            return count.get();
        }

    }

```
> TryableSemaphoreActual类最大信号量可配置，通过hystrix.command.default.fallback.isolation.semaphore.maxConcurrentRequests配置。
>  - tryAcquire： 对已使用的信号量+1；比较计数器与最大许可，判断是否有可用的信号量，有返回true，无返回false；


##### Semaphore实例创建
 - AbstractCommand 方法中缓存有信号量map如下:
 
```java
/* each circuit has a semaphore to restrict concurrent fallback execution */
    protected static final ConcurrentHashMap<String, TryableSemaphore> fallbackSemaphorePerCircuit = new ConcurrentHashMap<String, TryableSemaphore>();
    /* END FALLBACK Semaphore */

```
- 查询command命令的Semaphore实例根据commandkey

```java
    protected TryableSemaphore getExecutionSemaphore() {
        //校验是否配置以信号量隔离
        if (properties.executionIsolationStrategy().get() == ExecutionIsolationStrategy.SEMAPHORE) {
            if (executionSemaphoreOverride == null) {
                //缓存中获取当前命令的信号量信息
                TryableSemaphore _s = executionSemaphorePerCircuit.get(commandKey.name());
                if (_s == null) {
                    // 创建当前命令的信号量实例，并保存在缓存中
                    // we didn't find one cache so setup
                     executionSemaphorePerCircuit.putIfAbsent(commandKey.name(), new TryableSemaphoreActual(properties.executionIsolationSemaphoreMaxConcurrentRequests()));
                    // assign whatever got set (this or another thread)
                    return executionSemaphorePerCircuit.get(commandKey.name());
                } else {
                    return _s;
                }
            } else {
                return executionSemaphoreOverride;
            }
        } else {
            //返回空实现
            // return NoOp implementation since we're not using SEMAPHORE isolation
            return TryableSemaphoreNoOp.DEFAULT;
        }
    }

```
##### Semaphore在command中使用

```java
 private Observable<R> applyHystrixSemantics(final AbstractCommand<R> _cmd) {
        // mark that we're starting execution on the ExecutionHook
        // if this hook throws an exception, then a fast-fail occurs with no fallback.  No state is left inconsistent
        executionHook.onStart(_cmd);

        /* determine if we're allowed to execute */
        if (circuitBreaker.attemptExecution()) {
            //获取当前命令的信号量信息
            final TryableSemaphore executionSemaphore = getExecutionSemaphore();
            //信号量是否需要释放标识
            final AtomicBoolean semaphoreHasBeenReleased = new AtomicBoolean(false);
            //信号量释放action
            final Action0 singleSemaphoreRelease = new Action0() {
                @Override
                public void call() {
                    if (semaphoreHasBeenReleased.compareAndSet(false, true)) {
                        executionSemaphore.release();
                    }
                }
            };

            final Action1<Throwable> markExceptionThrown = new Action1<Throwable>() {
                @Override
                public void call(Throwable t) {
                    eventNotifier.markEvent(HystrixEventType.EXCEPTION_THROWN, commandKey);
                }
            };

            //信号量获取
            if (executionSemaphore.tryAcquire()) {
                //获取成功，执行命令具体操作，并将信号量释放action添加到流式操作中
                try {
                    /* used to track userThreadExecutionTime */
                    executionResult = executionResult.setInvocationStartTime(System.currentTimeMillis());
                    return executeCommandAndObserve(_cmd)
                            .doOnError(markExceptionThrown)
                            .doOnTerminate(singleSemaphoreRelease)
                            .doOnUnsubscribe(singleSemaphoreRelease);
                } catch (RuntimeException e) {
                    return Observable.error(e);
                }
            } else {
                //获取失败熔断进入Fallback
                return handleSemaphoreRejectionViaFallback();
            }
        } else {
            return handleShortCircuitViaFallback();
        }
    }

```
> applyHystrixSemantics 核心流程：
> 1. 获取TryableSemaphore
> 2. 调用TryableSemaphore.tryAcquire()
> 3. 调用TryableSemaphore.release()

> Hystrix使用了rxJava响应式编码，所以这里的代码组织结构有点难以理解，弄懂rxJava的原理，其实也是很好看懂的。


