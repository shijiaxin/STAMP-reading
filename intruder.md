#Intruder

##简介

##实现
###<center>数据结构
global_defaultSignatures是一个全局的数组，存储了一系列关键词，如果某个package里面有这里的单词就可以认为是恶意攻击。



###<center>初始化
MAIN函数在intruder.c里面。
dictionary\_alloc，返回一个vector，里面存的就是上述global\_defaultSignatures里面的单词。

stream\_generate，产生一系列的streams，其中包含一定比例的attack。对于每一个streams，程序都会调用splitIntoPackets，把它随机分成若干个package。每个package都有一个header，表示它属于哪个streams，以及它是第一个package等信息。所有的package会被存在同一个queue里面，最后调用queue\_shuffle，把所有的package打乱。

###<center>并行执行

主体在processPackets函数里面，每个线程进入一个大循环。
循环的第一个Transaction是

		bytes = TMSTREAM_GETPACKET(streamPtr);
		
从queue里面取出一个package进行处理。如果bytes为空则退出循环。

-
第二个Transaction是对package的具体处理逻辑，代码在TMDECODER_PROCESS函数里面，

为了处理package，程序定义了个全局的map，fragmentedMapPtr。这个map的每一项是一个list，每个list存储属于同一个stream的所有package。

根据取出的package，主要分如下几种情况讨论:
- package所在的stream只有一个package。
	- 直接处理这个stream。
- package所在的stream还没有被插入到全局的MAP里面。
	- 构建一个新的list插入到全局的MAP里面，再把package插入到list里面。
- package所在的stream已经有其他package被处理了，但是这一个并不是最后一个。
	- 简单把它插入即可。
- 自己这个package是最后一个。
	- 把list从map里取出，然后处理这个stream。

所谓的处理stream，其实只是把原始的各个package的信息按照顺序组装起来，变成一个完整的stream，然后插入到decodedQueuePtr这个queue里面。

-
第三个Transaction是取出一个完整的stream。
取出来之后在Transaction外检查一下这个stream是不是包含什么不和谐的字符串。

##Transaction介绍
- 取出package
- 处理package
- 取出stream