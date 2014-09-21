#kmeans

##应用介绍
kmeans是一个聚类算法，主要目的是给一组给定的点进行分类，分成k类。

给定一系列点，它们各有若干个feature，比如空间上的点就有x，y和z三个feature。kmeans算法假设这些feature的距离最小的点就更应该被划分为同一类。

基于上述思想，kmeans算法大致过程如下：  
- 1.随机生成k个中心点
- 2.对于给定的每个点，分别计算它到这k个中心点的距离，然后记录下其中最近的点的index (0<=index<k)
- 3.对于每个中心点，计算所有属于它的点的平均值，然后把自己的值修改成这个新的平均值，作为下一轮的中心点。
- 4.回到第2步继续执行，或者终止

##具体实现
README文件中推荐非模拟器执行的时候使用如下参数

    low contention:  -m40 -n40 -t0.00001 
    	-i inputs/random-n65536-d32-c16.txt
    high contention: -m15 -n15 -t0.00001 
    	-i inputs/random-n65536-d32-c16.txt  
完成各种初始化工作之后，最核心的代码在normal.c文件中的work函数里。于是我们主要关注work函数。

nclusters表示已经存在的k个中心点，new\_centers表示下一轮的k个中心点，初始全部为0，new\_centers\_len是一个int数组，表示有多少个点属于对应的中心点，也会在计算中被更新。

feature就是要归类的数组，由于计算粒度比较细，所以它被划分为多个CHUNK，每次线程会取出一个CHUNK进行计算，算好之后会获取global\_i 这个值，并进行更新，从而取出一个新的CHUNK。这个取CHUNK的过程会使用TM进行保护。

对于CHUNK中的每个点，线程都会使用common\_findNearestPoint函数去计算距离它最近的中心点的index，这个计算是只读的。点原来属于的中心点如果和计算出来的下一轮的中心点index不一样，就更新一个delta值，这个值用于评估继续计算或者停止计算。仅仅在算好全部的值之后再把delta加到global\_delta中。

每算完一个点的index，线程就会更新new\_centers,以及new\_centers\_len，这个过程也需要用TM进行保护。



