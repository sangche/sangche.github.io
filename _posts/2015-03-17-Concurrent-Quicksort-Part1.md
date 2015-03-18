---
title: Concurrent Quicksort in Scala Futures, Part1
layout: post
comments: false
---

Scala provides `Futures` and `Promises` to help develop concurrent, asynchronous systems which outperform
synchronous systems in most cases.
Future is, with monadic features embeded, simply a block of code which runs in an implicit execution context from fork-join-pool executor service
by default, if you import ExcutionContext.Implicit.global in scala.concurrent package.

Let's assume that the big problem here is, for example, to sort a huge list of integers. By applying concurrent divide-and-conquer with Futures, I would like to see how much it can improve performance.

I will implement and compare two versions of quick sorting a big integer list. One is none concurrent recursive quicksort running in a single task. The other impelmentation is to partition the list into sub-lists until some threshold and sort them concurrently to improve performance, using scala Futures.

Here is a non-concurrent `quicksort` in Scala
{% highlight scala %}
def quicksort(data: List[Int]): List[Int] = {
    if (data.isEmpty) {
      data
    } else {
      val pivot = data.head
      val (lP, rP) = data.tail partition (_ < pivot)
      quicksort(lP) ++ (pivot :: quicksort(rP))
    }
  }
{% endhighlight %}
This is recursive function running from start to end in one thread.

Now, concurrent version of quicksort with Futures.
Here is `concurrentQuicksort` implementation.
{% highlight scala %}
def concurrentQuicksort(data: List[Int]): List[Int] = {
    if (data.size < THRESHOLD) quicksort(data)
    else {
      val pivot = data.head
      val (lP, rP) = data.tail partition (_ < pivot)
      val f_value = for {
        left <- Future(concurrentQuicksort(lP))
        right <- Future(concurrentQuicksort(rP))
      } yield left ++ (pivot :: right)

      Await.result(f_value, 10 seconds)
    }
  }
{% endhighlight %}

For better understanding, the execution model of this concurrentQuicksort is depicted as execution tree structure as in the picture below.

![execution tree](https://raw.githubusercontent.com/sangche/sangche.github.io/master/pics/0503010/tree1.PNG)

Ok, now to measure performance of the two implementations, here is helper function `time`.

{% highlight scala %}
def time[R](block: => R)(name: String = "NoName"): R = {
    val start = System.nanoTime()
    val result = block
    val end = System.nanoTime()
    println(s"time taken: ${(end - start) / 1000} by $name")
    result
  }
{% endhighlight %}

`main` function has 800000 random integers as a test data to sort out.
{% highlight scala %}
val THRESHOLD = 20000

def main(args: Array[String]) = {
  val testData = List.fill(800000)(Random.nextInt)
  val result1 = time {
    quicksort(testData)
  }("quicksort")
  val result2 = time {
    concurrentQuicksort(testData)
  }("concurrentQuicksort")

  if (result1 == result2) println("PASS") else println("FAIL")
}
{% endhighlight %}
Let's run main 5 times to compare performance of the two implementations. Here is my test result(my CPU has 4 cores)

{% highlight %}
time taken: 3828665 by quicksort
time taken: 2689388 by concurrentQuicksort
PASS

time taken: 3503656 by quicksort
time taken: 2417930 by concurrentQuicksort
PASS

time taken: 4020424 by quicksort
time taken: 2794825 by concurrentQuicksort
PASS

time taken: 4001226 by quicksort
time taken: 3821625 by concurrentQuicksort
PASS

time taken: 3681652 by quicksort
time taken: 2379807 by concurrentQuicksort
PASS
{% endhighlight %}
The result clearly shows that there is some performance gain with Futures.

But, note that the performance of concurrent implementation is more variant than non concurrent implementation.
That's because the degree of balance in the execution tree affects more in concurrent implementation, due to threads management cost.

In extremely unbalanced case, the performance of concurrent quicksort would be worse than non concurrent quicksort.

Also with await block in all nodes of the execution tree, except leaf nodes, they wait for their child executions getting done. So, there is a good chance of await timeout, if you don't have enough threads in the thread pool of your execution context. Try test with your own `ExecutionContext` from fork-join-executor with threads 4.

In conclusion, concurrent quicksort implementation improves performance. However, this version of concurrent quicksort implementation is not acceptable for production use case.

In part 2, I will discuss about better solutions for the concurrent quicksort with Scala Futures.

Complete code is [here](https://gist.github.com/sangche/8d1780ced558725bcd1a)