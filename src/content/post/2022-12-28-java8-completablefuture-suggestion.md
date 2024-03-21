---
title: "Java 8 CompletableFuture Suggestion"
description: "`CompletableFuture` is what we use when we want to run tasks in parallel mode. This is just the example from the website,"
publishDate: "28 Dec 2022"
tags: ["java"]
---

## Improving Performance with Java’s CompletableFuture
[Improving Performance with Java’s CompletableFuture](https://reflectoring.io/java-completablefuture/), `CompletableFuture` is what we use when we want to run tasks in parallel mode. This is just the example from the website,

```java
Executor executor = Executors.newFixedThreadPool(10);
var futureCategories = Stream.of(
        new Transaction("1", "description 1"),
        new Transaction("2", "description 2"),
        new Transaction("3", "description 3"),
        new Transaction("4", "description 4"),
        new Transaction("5", "description 5"),
        new Transaction("6", "description 6"),
        new Transaction("7", "description 7"),
        new Transaction("8", "description 8"),
        new Transaction("9", "description 9"),
        new Transaction("10", "description 10")
    )
    .map(transaction -> CompletableFuture.supplyAsync(
            () -> CategorizationService.categorizeTransaction(transaction), executor)
    )
    .collect(toList());
```

## What About Timeout
We wait in a mount of time for all tasks to finish.

- generate `CompletableFuture` from each task
- collect them all
- wrap up and wait in specific timeout

```java
static void play() {
    Random random = new Random();
    Map<Integer, Object> map = new ConcurrentHashMap<>();
    List<CompletableFuture<Void>> all = new ArrayList<>();
    for (int i = 1; i <= 10; i++) {
        int finalI = i;
        CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
            if (random.nextDouble() < 0.5) {
                map.put(finalI, PRESENT);
            } else {
                try {
                    TimeUnit.MILLISECONDS.sleep(1_500 + Math.round(random.nextDouble() * 500));
                } catch (InterruptedException ignore) {
                    //
                }
            }
            return null;
        }, threadPoolExecutor);
        all.add(future);
    }
    try {
        CompletableFuture.allOf(all.toArray(new CompletableFuture<?>[0])).get(1000, TimeUnit.MILLISECONDS);
    } catch (Exception e) {
    }
    Set<Integer> set = map.keySet();
    System.out.println("play result: " + set);
}
```
From a macroscopic perspective, `play` method shall run within 1000ms, while from a microscopic perspective, any task may have not begin to run or have not finished.


## What's the Matter
Every task should begin to run as quickly as possible, and finish up in a short time. If any task is unlucky, it get stuck in the `ThreadPoolExecutore`'s queue, or holds resource of `ThreadPoolExecutore`'s `Thread`. That's the matter.

When the application is at a high traffic, every millisecond counts, which we don't like to waste any. For example, with a thread pool, core size at 4, max size at 8, having a queue, size at 100.

### Issue 1: Queueing
Task of User-8 may have enter the queue earlier than User-5's, as a result, User-8's task get chance to run earlier than User-5's. 

![queueing](https://user-images.githubusercontent.com/2212273/209836738-418628a3-b3f7-44c5-8242-b2bc9f1cbc66.jpg)


### Issue 2: Time is Burning Out
When the `play` method is called, the clock is ticked in at the same moment, but its tasks will be executed by thread pool some time later. In another word, it may happen that `play` method finish work earlier than its tasks, cause tasks remain in the queue. 

![time-burn-out](https://user-images.githubusercontent.com/2212273/209838361-55ed2a11-85e3-4c4f-8df7-18b8d2e9b5c3.jpg)

To sum up, tasks running or to-be-run beyond timeout of its calling method, have to be abondoned, or they will set others to wait, and others to waste.

## We Clean Them
We clean tasks that have not run.

```java
try {
    CompletableFuture.allOf(all.toArray(new CompletableFuture<?>[0])).get(1000, TimeUnit.MILLISECONDS);
} catch (Exception e) {
    all.forEach(f -> {
        if (!f.isDone() && !f.isCancelled() && !f.isCompletedExceptionally()) {
            f.cancel(false);
        }
    });
}
```

We clean tasks that shall not run.
```java
@Override
protected void beforeExecute(Thread t, Runnable r) {
    ScheduledFuture<?> scheduledFuture = scheduledThreadPoolExecutor.schedule(() -> {
        t.interrupt();
    }, 1000, TimeUnit.MILLISECONDS);
    workerSet.put(r, scheduledFuture);
}

@Override
protected void afterExecute(Runnable r, Throwable t) {
    ScheduledFuture scheduledFuture = workerSet.remove(r);
    if (scheduledFuture != null) {
        scheduledFuture.cancel(false);
    }
}
```
Decorate task within a schedule task, while its timeout equals to `play` calling method, if this task finish up and call `afterExecute` schedule task get remove, or it will be interrupted by schedule task.

## In All

code,

```java
public class App {
    static BlockingQueue<Runnable> blockingQueue = new ArrayBlockingQueue<>(100);
    static ThreadPoolExecutor threadPoolExecutor = new InterruptableThreadPoolExecutor(4, 8, 60, TimeUnit.SECONDS, blockingQueue);
    static Object PRESENT = new Object();
    public static void main(String[] args) {
        for (int j = 1; j <= 10; j++) {
            play(j);
        }
        threadPoolExecutor.shutdown();

    }

    /**
     * as like call 10 http request
     */
    static void play(int j) {
        Random random = new Random();
        Map<Integer, Object> map = new ConcurrentHashMap<>();
        List<CompletableFuture<Void>> all = new ArrayList<>();
        for (int i = 1; i <= 10; i++) {
            int finalI = i;
            System.out.printf("%d round %d step enter running\n", j, finalI);
            CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
                if (random.nextDouble() < 0.5) {
                    map.put(finalI, PRESENT);
                } else {
                    try {
                        TimeUnit.MILLISECONDS.sleep(1_500 + Math.round(random.nextDouble() * 500));
                        System.err.printf("%d round %d step, finish sleep\n", j, finalI);
                    } catch (InterruptedException ignore) {
                        //
                    }
                }
                return null;
            }, threadPoolExecutor);
            all.add(future);
        }
        try {
            CompletableFuture.allOf(all.toArray(new CompletableFuture<?>[0])).get(1000, TimeUnit.MILLISECONDS);
        } catch (Exception e) {
            System.out.println(j + " round, before clean: " + threadPoolExecutor.getQueue().size());
            all.forEach(f -> {
                if (!f.isDone() && !f.isCancelled() && !f.isCompletedExceptionally()) {
                    f.cancel(false);
                }
            });
            System.out.println(j + " round, after clean: " + threadPoolExecutor.getQueue().size());
        }
        Set<Integer> set = map.keySet();
        System.out.println(j + " round, play result: " + set);
    }

    static class InterruptableThreadPoolExecutor extends ThreadPoolExecutor {
        static ScheduledExecutorService scheduledThreadPoolExecutor = Executors.newScheduledThreadPool(10);
        static Map<Runnable, ScheduledFuture> workerSet = new ConcurrentHashMap<>();
        public InterruptableThreadPoolExecutor(int corePoolSize, int maximumPoolSize, long keepAliveTime, TimeUnit unit, BlockingQueue<Runnable> workQueue) {
            super(corePoolSize, maximumPoolSize, keepAliveTime, unit, workQueue);
        }

        @Override
        protected void beforeExecute(Thread t, Runnable r) {
            ScheduledFuture<?> scheduledFuture = scheduledThreadPoolExecutor.schedule(() -> {
                t.interrupt();
            }, 1000, TimeUnit.MILLISECONDS);
            workerSet.put(r, scheduledFuture);
        }

        @Override
        protected void afterExecute(Runnable r, Throwable t) {
            ScheduledFuture scheduledFuture = workerSet.remove(r);
            if (scheduledFuture != null) {
                scheduledFuture.cancel(false);
            }
        }

        @Override
        public void shutdown() {
            scheduledThreadPoolExecutor.shutdownNow();
            super.shutdown();
        }
    }
}
```

    