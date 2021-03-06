---
layout: post
title: "high performance priority queues & ruby"
description: ""
date: 2018-04-18
tags: []
comments: false
---

Recently, I explored utilizing a priority queue for a project I am working on in ruby that involves reading in more data that can fit in RAM by multiples, and also writing out that data in realtime.
For a small portion of this external sorting mechanism, I wanted to use a priority queue. However, ruby does not have a priority queue on tap to use. It does utilize [gems](https://www.ruby-lang.org/en/libraries/) 
as sometimes standard integrations, or otherwise popular solutions for install. One of the most [commonly utilized](https://github.com/kanwei/algorithms) 
priority queues in ruby was written in the 2008 [Google Summer of Code](https://summerofcode.withgoogle.com/) project. In fact, it is so commonly
used and heavily integrated with the development community, it is the default repo for: 

`$ gem install algorithms`

Surprisingly, the algorithms priority queue was prohibitvely slow for my needs. I thought I might be inadvertently causing this degradation in my implementation
of this queue with my larger project, but isolated benchmark testing with, real, cpu, and system time with garbage collection purges between each run, as well as [ips](https://github.com/evanphx/benchmark-ips), confirmed the slow performance.
For a sanity check, it turns out [others](https://www.salsify.com/blog/engineering/ruby-scalable-offline-sort) were having issues with the priority queue in this gem as well.

Next, I went to the code to look at the data structures and algorithms used to see if there were any obvious disadvantages for runtimes. Simply looking at the code showed the priority queue container utilized a heap.
Using a heap is a common, and agreed to be a generally optimized implementation. A binary heap is a heap with up to two children. A heap is a specialized data structure that satisfies the heap property. For a max heap, the value of the parent node is
either greater than or equal to the value of the child nodes.

![heap](/assets/images/heap4.png "Center"){: .center-image}

Note that a heap is not a binary search tree (BST). A binary search tree has a more specific ordering property, where all left descendents must be less than or equal to all right
descendents.

We can summarize some of the most commonly known implementations with the following analysis, where **M** is the max element, and **N** is the number of elements in the queue. 

 | implementation        | time         | space   | insert | del max | max
 |:--------              |:-------:     |:--------:| :----: | :----: | :--:
 | naive unordered array | M N          | M    |      1    |    N   |  N 
 | naive ordered array   | M N          | M    |      N    |    1   |  1 
 | binary heap           | N log M      | M    |    log N  |  log N | log N 
 |---- 
 
### queue the code

To understand what could be causing the issue despite the priority queue using a heap, 
I decided to make my own optimized priority queue with a heap in ruby, so I could compare my performance 
to the standard gem.

There are multiple types of data structures that can be used to implement the abstract tree structures of a heap, but
I used an array to implement a binary heap, where the children nodes of parent node at element index `k`, are located at 
child indices `2k` and `2k + 1`. To make the math for referencing parent and child indices from eachother correct across the array, the first
indice 0 is `nil`. The values from the heap example above are located in the array elements according to the indice values, for reference. 

![array](/assets/images/array2.png "Center"){: .center-image}


###### getting the class together

 ```ruby
class PriorityQueue
  attr_reader :elements, :size

  def initialize
    @elements = [nil]
    @size = nil
  end

  def size
    @size = (@elements.size - 1)
  end

  def isEmpty? 
    return @size == 0
  end

  # compares <= 1 + logN
  def <<(element)
    @elements << key
    size
    swim(size)
  end
  ```

###### sink 

While iterating through the elements of the array, we have to find the largest child node, as the heap ordering
does not specify which child node should be larger than the other, just that both `<=` the parent. If the parent
has children, and is less than than a child node, the parent will exchange places, or indices in the array, with the child node, promoting the child node to match is priority 
with a higher ordering in the heap array, while the parent node gets demoted to be a lesser child node. We then
set the indice, now again analyze the parent, newly demoted to a child, as a new parent to its current children, to see if it
should be demoted. 

  ```ruby
  private
  def sink(k)
    # j = child
    while 2*k <= size
      j = 2*k
      # get the largest child
      if j < size && less?(j, j+1)
        j += 1
      end
      #if parent not < largest child
      if !less?(k,j)
        break
      end
      exch(k, j)
      k = j
     end
  end
```

###### swim

If the newly inserted element, now a child, is anything other than the first element, or the root at index 1, and the parent is less, the parent will swap places, 
or indices in the array, with the child. The new indice of the promoted child/inserted element, now parent to its previous parent, will be considered again if it is not the root of the entire heap, to see if it is larger
than its new set of parent. 

  ```ruby

  private
  def swim(k)
    # while not root and k's parent < k
    while k > 1 && less?(k/2, k) 
        exch(k, k/2)
        k = k/2
    end
  end
```

###### pop 
After a highest priority element is popped off the top by being recorded and later returned as max, 
the value exchanges array indexes in the heap array with the lowest priority element. The size of the array is downsized 
to delete the highest priority element which is no longer apart of the heap. The previous lowest priority element, 
is now at the top top of the heap, and needs to be reordered or "sink" to its appropriate priority.


```ruby

  # compares <= 2LogN
  def pop
    max = @elements[1]
    exch(1, size)
    @elements.pop
    size
    sink(1)
    return max
  end

  ```


###### array helpers

```ruby
  private
  def less?(i, j)
    return (@elements[i] < @elements[j])
  end

  private
  def exch(i, j)
    @elements[i], @elements[j] = @elements[j], @elements[i]
  end
```

###### a mini example


```ruby
q = PriorityQueue.new
q << 2
q << 5
q << 7
p q.elements # => [nil, 7, 2, 5]
# q << ...
```

### make no assumptions

Now it was time for some microbenchmarking, utilizing a newly made priority queue. This should lead to some pretty straight forward comparisons also using a heap, in the same language as the 
troublesome gem. 

I used [benchmark.bmbm](https://ruby-doc.org/stdlib-1.9.3/libdoc/benchmark/rdoc/Benchmark.html) to calibrate for memory allocation and garbage collections in the subsequent runs.

```ruby
arr = [1E3, 1E4, 1E5]
arr.each do |n|
   Benchmark.bmbm do |x|
     # google summer of code/ standard gem heap implementation
     fib_heap = Containers::PriorityQueue.new
     my_pq = New_Max_Pq.new

     # push
     x.report("heap: Size: #{n}") { for i in 1..n; fib_heap.push(i,i); end }
     x.report("my pq: Size #{n}") { for i in 1..n; post << i; end }

     # pop
     x.report("heap: Size: #{n}") { for i in 1..n; fib_heap.pop end }
     x.report("my pq: Size #{n}") { for i in 1..n; my_pq.pop ; end }
  end
end
```

While the original benchmark was set up to test up to 400 million inserted objects and pop them off, I didn't need to wait up all night to see the results.

### screenshots or it didn't happen

![results](/assets/images/pictures2.png "Center"){: .center-image}

As it turns out, my [code](https://github.com/GInxh/algorithms) was faster, much faster. 

I should have been excited, but mostly I was confused by the large discrepency. To create a more detailed round of performance testing, 
I decided to start by comparing code for the insertions. Both queues used heaps, but I had not looked into the separate container for the heap instantiated by the priority queue class in the default ruby algorithms gem.


Outside of some obvious increase in object class instantiations for nodes, I realized insertion used a merge function, 
and linked lists. Linked Lists! Merge functions! What is going on here? It turns out the ruby algorithms gem doesn't use a normal heap, it uses a fibonacci heap. My first reaction was, what is a fibonacci heap? 

### reverse engineering amortised runtimes

It seems that a [fibonacci heap](http://www.growingwiththeweb.com/data-structures/fibonacci-heap/overview/) is an exercise for optimizing in [amortised](https://en.wikipedia.org/wiki/Amortized_analysis) analysis.
In fact, both the formal documentation, wikipedia, and all the sources I could find stated that it was extremely 
slow performance wise, and generally accepted to be non ideal for realtime applications, due to large memory overhead, and the realization of worst cases in realtime. I looked at [rubyworks](https://github.com/rubyworks), and it also uses a fibonacci heap. Instead of going into how the concept 
of [potential](https://en.wikipedia.org/wiki/Potential_method) is used to justify the amortised runtime, I wanted to know more about about what is happening in real time.

###### data structure

A fibonacci heap is a doubly linked list that contains the root nodes of a collection of heap ordered trees, where no two heaps have the same order. Each node can
contain up to four pointers. A pointer to the minimum node of the root list is always kept up to date.  

![fib](/assets/images/thefib2.png "Center"){: .center-image}


######  object proliferation

This design indicates alot more objects, utilizing a node class for every element, with up to four pointers. If you are familiar with data structures and runtimes, you should not have to confirm that there should be a high overhead for memory by looking at the code, even without knowing 
about how the methods work and interact with eachother. 

Unlike C, ruby by default has it's own heap, and performs memory allocation and garbage collection ([GC](https://ruby-doc.org/core-2.2.0/GC.html)). This is not the same heap as mentioned in the data structures used for the queue above.
Programs have a stack used for static memory allocation, and a heap that is used for dynamic memory allocation. Variables allocated on the heap have their memory allocated at run time. Accessing this is slower than accessing
stack variables. The heap size is only limited by the size of virtual memory.

The ruby heap is made up of slots. Each slot can hold one object. An object in Ruby is a struct called `RVALUE`. Each `RVALUE` is 40bytes in size, which is also the size of a slot. Ruby initializes a heap with preallocated
memory. By default the heap size is set to 16MB for `Ruby 2.4`. In earlier versions, this limit is 8MB. Additionally, the Ruby heap organizes the heap into pages, where each page can hold 408 slots. 
`GC.stat[:heap__sorted_length]` returns how many pages Ruby has allocated. Because Ruby initially allocates this memory, it does not reflect how many objects have been allocated into memory. `GC.stat[:heap_used]`
returns how many pages are currently in use, versus simply allocated. This number can also contain live objects as well as free slots. `GC.stat[:heap_eden_page_length]` returns live objects. `GC.stat[:heap_tomb_page_length]` 
returns slots with no live objects that will be used when Eden runs out of space.  

Given that most formal documentation on the fibonacci heap evades observation of realtime performance, and ruby garbage collection is rapidly evolving from one release to the next, 
it is probably best to confirm this a significant contributer to the disparity in performance.  

To see how many objects, and thus easily calculate how much memory has been allocated, we can run a test to see how many pages are needed
to scale up the priority queue.  

```ruby
allocated_before = GC.stat(:heap_used)
puts "#{allocated_before}"

arr = [1E3, 1E4, 1E5, 1E6, 1E7, 1E8]
arr.each do |n|
  for i in 1..n
    # test myqueue
    # in a separate run test fibonacci heap
  end
end

allocated_after = GC.stat(:heap_used)
puts "#{allocated_after}"
```

With five runs for each version of the queue, the fibonacci heap consistently requires **2x** as many objects as the array based heap I created. 

###### memory allocation

As mentioned above, Ruby initializes the program with a heap size of 16MB. [This means](https://www.speedshop.co/2017/03/09/a-guide-to-gc-stat.html) 
ruby does not trigger a garbage collection run to see if it can first get rid of stuff it doesn't need anymore, which takes a relatively long time, or go 
ask the kernel for more memory, which also takes a relatively long time, until the ruby programs needs more than 16MB. Ruby GC waits until all the slots in the preallocated heap are full, before it begins to see which
objects it does not need anymore, and frees them. If the slots are still full, and in this case they will be, as all loaded elements will still be in the array, then ruby will allocate more memory.

Ruby GC has internal variables that can be tuned from their default values. `RUBY_GC_HEAP_INIT_SLOTS` allocates the initial number of slots on the Ruby heap. The default
value is 1000. When Ruby does need to allocate more memory, it gets more than it originally had by a factor of the current amount of memory `RUBY_GC_HEAP_GROWTH_FACTOR`, and utilizes this factor every time it scales
up its memory allocation. Currently, this value is set as a default to `1.8x`. If `RUBY_GC_HEAP_GROWTH_FACTOR` is set to allocate more memory than is needed, it will result in unecessary latency, allocating much more 
memory than is needed, slowing down applications that will not use all of the next allocation of memory. 

Amongst the community developers for Ruby GC, [Sam Saffron](https://samsaffron.com/archive/2014/04/08/ruby-2-1-garbage-collection-ready-for-production) 
had originally brought up this issue [here](https://bugs.ruby-lang.org/projects/ruby-trunk) to decrease the default growth factor, but for now it is left as a tunable parameter. Initially, tuning it to `1.3` is a good first edit to compare
performance for applications that do not have exponential object proliferation or otherwise growth expected in the application. 

In this case, the priority queue I made for testing above has insertions loading onto the element queue with 8 byte integers. This means I can load about 2 million integers into my element based priority queue, 
give or take some for the overhead of the class, before ruby triggers more memory allocation. 

This could partially explain why between the second and third run, the fibonacci heap runtime increased **123x**, with a signifcantly higher ratio in system time, as it had to sweep the heap for recycleable memory,
and allocate **1.8x** the current amount of memory multiple times. This should be tuned to fit the number of live objects after a commonly booted process or Rails application is fully booted to avoid GC runs for initialization. 

You can increase `RUBY_GC_HEAP_INIT_SLOTS` to tailor this limit to the size of your program or Rails application. Increasing `RUBY_GC_HEAP_INIT_SLOTS` for both queues in this case. 
would scale down the performance discrepency between the two, but not eliminate it.

###### delayed heap ordering

This is because the fibonacci heap has delayed heap ordering after insertion, which is deferred to the extract minimum function. (The standard fibonacci heap defaults to a min heap, but can be utilized
with a max heap.) The extract minimum function removes the minimum. If the minimum had any children, it adds the children to the root list each as their own heap, with their subheaps, if they have any.

Then, it invokes a merge operation. The merge operation merges all of the heaps of the same order. After this, the minimum is then set to the new minimum in the list. 
The fact that the most expensive methods are not apart of the insertion, further confirms the significance of memory overhead contributing to the repeated insertions with no intermittent extractions, from the `benchmark.bmbm` test above.

###### the worst case is a likely case

While the fibonacci heap advertises **O(log(n))** amortised running times for the extract minimum and delete methods, they have a worst case of **O(n)** due to the deferred heap ordering. 
This is because the heap ordering is not limited to **O(log(n))** convenience from having a balanced tree.

This is not a degradation in performance from an unlikely arrangement of input values. The heap ordering will happen intermittently.
This results in some operations running very fast, and some operations running very slow. More detailed explanations of the standard methods can be found [here](http://www.growingwiththeweb.com/data-structures/fibonacci-heap/overview/)

### lessons learned

###### code 

Read the code [first](https://github.com/kanwei/algorithms/blob/master/lib/containers/heap.rb), documentation [second](https://github.com/kanwei/algorithms/blob/master/lib/containers/priority_queue.rb)

###### know your GC and ObjectSpace

Ruby has had many changes with garbage collection algorithms, and other auxiliary implementations since the 1.9 MRI release. 
`GC.stat` metrics have always been undefined in the formal documentation of all releases. 

Understanding how [GC](https://github.com/ruby/ruby/blob/730d257b5a6bc0808116d91dbbaae73344273781/gc.c) impacts
`GC.stat` outputs leads to understanding the results of the metrics, instead of relying on their
potentially inconsistent and unintuitive output. There are alot of memory profile gems released that were very helpful, 
that are not compatible with `Ruby 2.4` or above. [Looking through](https://github.com/ruby/ruby/blob/730d257b5a6bc0808116d91dbbaae73344273781/gc.c)
the garbage collector will be helpful in creating a memory_profiler gem going forward. 
