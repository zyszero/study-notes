# JUC系列源码解读 - CyclicBarrier

CyclicBarrier 内部是基于ReentrantLock 和 Condition 来实现功能的

## 使用场景

```java
class Solver {
  final int N;
  final float[][] data;
  final CyclicBarrier barrier;
  class Worker implements Runnable {
    int myRow;
    Worker(int row) { myRow = row; }
    public void run() {
      while (!done()) {
        processRow(myRow);
        try {
          barrier.await();
        } catch (InterruptedException ex) {
          return;
        } catch (BrokenBarrierException ex) {
          return;
        }
      }
    }
  }
  public Solver(float[][] matrix) {
    data = matrix;
    N = matrix.length;
    Runnable barrierAction = () -> mergeRows(...);
    barrier = new CyclicBarrier(N, barrierAction);
    List<Thread> threads = new ArrayList<>(N);
    for (int i = 0; i < N; i++) {
      Thread thread = new Thread(new Worker(i));
      threads.add(thread);
      thread.start();
    }
    // wait until done
    for (Thread thread : threads)
      thread.join();
  }
}
```



