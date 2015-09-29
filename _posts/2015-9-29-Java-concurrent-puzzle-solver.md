---
layout: post
title: Java 并发穷举算法实践
excerpt: java 并发穷举算法实践
category: JAVA
---

之前无意中看到推箱子游戏，想了一下这个游戏的求解算法，感觉非常复杂，苦思无果。最近在看《java并发编程实践》时得到启发，可以利用穷举算法来解决这一问题，而且很多类似的问题都可以用穷举算法解决。

这里定义三个类：

~~~
PuzzleState：定义解谜过程中的一个状态或者说一个步骤。
SolvePuzzle：解谜线程管理类
Solving：解谜线程类
~~~

一个接口：

~~~
PuzzleModeling：问题建模，在这里定义问题规则。
~~~

#####PuzleState

~~~java
/**
 * 解谜过程中的一个状态或者一个步骤，与问题建模有关。在推箱子游戏中，是工人的一步移动；<br>
 * 每个状态都维护一个父状态和一个关闭状态列表。父状态用于解决方法的回溯，从目标状态返回起点状态；
 * 关闭列表保存了到达当前状态前已经处理过的状态。<br>
 * 关闭列表在每个状态中作为内部变量维护，保证即使父状态和子状态由不同线程处理，其结果都是一致的。<br>
 * @author nfhy
 * 2015年9月29日
 * <br> 
   myBlog: http://nfhy.wang
 */
public abstract class PuzzleState {
  
  /**每个状态的唯一ID.*/
  protected String uniqueId;
  
  /**指向父状态，计算解决方法价值，回溯解决方法时使用.*/
  protected PuzzleState parent;
  
  /**关闭状态列表，保存状态id，这一列表中的状态不需要再次处理.*/
  private ArrayList<String> closeList;
  
  /**生成状态唯一ID，与问题建模有关.*/
  public abstract void generateUniqueId();
  
  public String getUniqueId() {
    return this.uniqueId;
  }
  
  public void setParent(PuzzleState parent) {
    this.parent = parent;
  }
  
  /**当前状态的后续状态，一般情况下，如果后续状态出现在closeList中，这一状态可以跳过不处理.*/
  public abstract <K extends PuzzleState> List<K> next();
  
  /**从当前状态回溯到起始状态，返回一个状态队列.*/
  public abstract List<PuzzleState> traceBackToStart();
  
  /**计算从当前状态回溯到起始状态的步数，当谜题有多个解时，可以通过这一方法找到最优解.*/
  public abstract int traceBackCost();

  /**在console中打印状态简图的方法.*/
  public void draw(){}
  
}
~~~

####SolvePuzzle

~~~java
/**
 * 解谜线程管理类。维护一个线程池 es；
 * 维护一个开启状态阻塞队列 openList，队列中的状态都是待处理的，所有线程都可以在这里取得任务；<br>
 * 保存一个或多个目标状态 targetStates；维护一个解决方法阻塞队列 findStates，
 * 每找到与目标状态相同的状态，就保存在队列中；<br>
 * 保存一个开始状态 start，所有后续状态的顶层父状态都是开始状态；
 * 保存一个闭锁 latch，当所有线程执行完毕或异常退出时，latch-1<br>
 * @author nfhy
 * 2015年9月29日
 * <br> 
   myBlog: http://nfhy.wang
 */
public class SolvePuzzle {
  
  /**
   * 等待线程执行的最长等待时间.
   */
  private static final int maxWaitLength = 10;
  /**
   * 解密线程的线程池.
   */
  final ExecutorService es = 
      Executors.newFixedThreadPool(Runtime.getRuntime().availableProcessors());
  
  /**开启状态阻塞队列，保存待处理的状态，线程以阻塞方式从队列中取值，阻塞时间超过规定时间，表明没有新的状态产生，线程结束.*/
  final LinkedBlockingQueue<PuzzleState> openList = new LinkedBlockingQueue<PuzzleState>();
  
  /**解决方法阻塞队列，保存找到的解决方法，也就是和目标状态相同的状态.*/
  final LinkedBlockingQueue<PuzzleState> findStates = new LinkedBlockingQueue<PuzzleState>();
  
  /**一个或多个目标状态，有的谜题可能有多个解.*/
  PuzzleState[] targetStates;
  
  /**谜题初始状态.*/
  PuzzleState start;
  
  /**线程执行闭锁，只有所有线程执行完毕后，才开是梳理解决方法.*/
  final CountDownLatch latch = new CountDownLatch(Runtime.getRuntime().availableProcessors());
  
  /**构造方法，提供目标状态和初始状态.
   * @param targetStates 目标状态
   * @param start 初始状态
   */
  public SolvePuzzle(PuzzleState[] targetStates, PuzzleState start) {
    this.targetStates = targetStates;
    this.start = start;
  }
  
  /**解谜开始方法.*/
  public void start() {
    try {
      openList.put(start);
    } catch (InterruptedException e) {
      System.out.println("start interrupted, abort");
      return;
    }
    for (int i = 0; i < Runtime.getRuntime().availableProcessors(); ++i) {
      es.submit(new Solving(targetStates));
    }
    try {
      latch.await();
    } catch (InterruptedException e) {
      System.out.println("latch waiting interrupted");
    }
    shutdown();
    handleResult();
  }
  
  /**遍历解决方法，找到最优解.*/
  private void handleResult() {
    PuzzleState result;
    PuzzleState minCostState = null;
    while ((result = findStates.poll()) != null) {
      System.out.println("---- " + result.traceBackCost());
      if (null == minCostState || minCostState.traceBackCost() > result.traceBackCost()) {
        minCostState = result;
      }
    }
    minCostState.draw();
    List<PuzzleState> traces = minCostState.traceBackToStart();
    Collections.reverse(traces);
    for (PuzzleState state : traces) {
      state.draw();
    }  
  }
  
  /**等待线程池关闭，如果100秒仍没有关闭，手动关闭.*/
  public void shutdown() {
    if (!es.isTerminated()) {
      try {
        es.awaitTermination(maxWaitLength, TimeUnit.SECONDS);
      } catch (InterruptedException e) {
        System.out.println("wait for termination interrupted");
      } finally {
        es.shutdown();
      }
    }
  }
~~~

####Solving

~~~java
/**
   * @Title SolvePuzzle.java
   * @author nfhy
   * @date 2015年9月29日
   * @Description 解谜线程类，定义了解谜的过程：<br>
   *     1.从开启队列中取得待处理状态，如果超过等待时间仍没有取到新的状态，跳转步骤5<br>
   *     2.获取该状态的后续状态<br>
   *     3.如果不存在后续状态，返回步骤1；
   *     4.如果存在后续状态，遍历这些状态，如果这些状态与目标状态相同，记录在解决方法队列中，
   *     如果这些状态不在开启列表中，将它们放入开启列表，返回步骤1。<br>
   *     5.结束线程
   * <br> 
     myBlog: http://nfhy.wang
   */
  class Solving implements Callable<Boolean> {
    
    PuzzleState nowState;
    
    PuzzleState[] targetStates;
    
    public Solving(PuzzleState[] targetStates) {
      this.targetStates = targetStates;
    }
    
    private boolean checkTarget(PuzzleState state) {
      for (PuzzleState target : targetStates) {
        if (null != target && target.getUniqueId().equals(state.getUniqueId())) {
          return true;
        }
      }
      return false;
    }
    
    private boolean getState() throws Exception {
      while ((nowState = openList.poll(maxWaitLength, TimeUnit.SECONDS)) != null) {
        List<PuzzleState> nextStates = nowState.next();
        for (PuzzleState state : nextStates) {
          if (checkTarget(state)) {
            findStates.add(state);
          }
          if (!openList.contains(state)) {
            openList.put(state);
          }
        }
      }
      return true;
    }

    @Override
    public Boolean call() {
      try {
        getState();
      } catch (Exception e) {
        e.printStackTrace();
      } finally {
        latch.countDown();
      }
      return true;
    }
    
  }
~~~

为了解决推箱子问题，继承SingleState，实现PuzzleModeling，代码不在这贴了，有兴趣的可以去[我的github下载](https://github.com/nfhy/solveProblem)

测试场景非常简单，5\*4的地图，四周围墙，两个箱子，两个仓库。W表示工人，#表示墙壁，\*表示仓库，B带表箱子。

~~~
初始状态：
 # # # # # #
 # * * B S #
 # W B # * #
 # * * * S #
 # # # # # #
目标状态：

 # # # # # #
 # * * * B #
 # * * # * #
 # W * * B #
 # # # # # #
 
 最优结果回溯（从初始状态到最终状态，最快需要11步）：
 注意，这可能不是唯一的最优解，但肯定是最优解之一。
 # # # # # #
 # * * * B #
 # * * # * #
 # W * * B #
 # # # # # #

 # # # # # #
 # * * B S #
 # W B # * #
 # * * * S #
 # # # # # #

 # # # # # #
 # W * B S #
 # * B # * #
 # * * * S #
 # # # # # #

 # # # # # #
 # * W B S #
 # * B # * #
 # * * * S #
 # # # # # #

 # # # # # #
 # * * W B #
 # * B # * #
 # * * * S #
 # # # # # #

 # # # # # #
 # * W * B #
 # * B # * #
 # * * * S #
 # # # # # #

 # # # # # #
 # * * * B #
 # * W # * #
 # * B * S #
 # # # # # #

 # # # # # #
 # * * * B #
 # W * # * #
 # * B * S #
 # # # # # #

 # # # # # #
 # * * * B #
 # * * # * #
 # W B * S #
 # # # # # #

 # # # # # #
 # * * * B #
 # * * # * #
 # * W B S #
 # # # # # #

 # # # # # #
 # * * * B #
 # * * # * #
 # * * W B #
 # # # # # #

 # # # # # #
 # * * * B #
 # * * # * #
 # * W * B #
 # # # # # #

 # # # # # #
 # * * * B #
 # * * # * #
 # W * * B #
 # # # # # #

~~~