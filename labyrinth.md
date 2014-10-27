#Labyrinth

##应用介绍
Labyrinth是一个迷宫寻路算法，主要目的是在给定的迷宫中，为给定的若干组点对找最短路径，这些路径不能交错。

	./labyrinth -i inputs/random-x512-y512-z7-n512.txt

输入文件是一个512\*512\*7的迷宫。

##具体实现

完成初始化后，核心代码在router.c里面的router_solve这个函数里面。

循环的第一个transaction是从workQueuePtr里面取出一对点对。如果没有剩余的点对说明计算结束了。直接break。

第二个transaction做的事情很多。
- grid_copy(myGridPtr, gridPtr);这个是把全局的gridPtr里面的内容复制到局部变量里。
- PdoExpansion这个是修改myGridPtr的内容。本质上它是一个广度优先搜索。最终可能搜到路径也可能没有搜到路径。
- 如果上一步没有搜索到路径，则直接结束这个transaction。搜到的话调用PdoTraceback记录好这个路径。
- TMGRID_ADDPATH把这个路径的值写回全局地图。把地图上的empty设置为full，并把路径记录到一个局部变量里面。

整个transaction伪代码如下

	TM_BEGIN();
    	grid_copy(myGridPtr, gridPtr);
    	if(PdoExpansion(myGridPtr)){
    		path= PdoTraceback(myGridPtr); 
    		TMGRID_ADDPATH(gridPtr);
    	}
    TM_END();
    
需要特别注意的是，TMGRID_ADDPATH的伪代码如下，它会调用一个主动的abort。

	for (i = 1; i < (n-1); i++) {
        value = TM_SHARED_READ(Ptr[i]);
        if (value != GRID_POINT_EMPTY) {
            ABORT();
        }
        TM_SHARED_WRITE(Ptr[i], GRID_POINT_FULL);
    }

第三transaction是在循环结束后，把所有路径添加到一个全局list里面。

##Transaction分析
labyrinth有3个transaction
- 取出点对
- 读地图，计算路径，写回地图。
- 更新路径list。

##bottleneck
grid_copy，PdoExpansion，PdoTraceback这些操作工作量很大，不应该放在transaction里面。

所以我们把它们移出transaction。但是简单地移出之后程序出现正确性bug。
这个bug的主要原因是RTM总是需要有个fallback_handler，一般是拿锁然后重试。所以在走锁的路线的时候，abort就会失去效用。

所以应该修改TMGRID\_ADDPATH的实现，验证它是否成功执行。如果TMGRID\_ADDPATH失败了应该重试。

整个transaction伪代码变成如下

	retry=true;
	while(retry){
		retry=false;
    	grid_copy(myGridPtr, gridPtr);
    	if(PdoExpansion(myGridPtr)){
    		path= PdoTraceback(myGridPtr); 
			TM_BEGIN();
    		retry = TMGRID_ADDPATH(gridPtr);
    		TM_END();
    	}
    }
    
然后需要修改TMGRID_ADDPATH的实现，改为如下：
    
	for (i = 1; i < (n-1); i++) {
        value = TM_SHARED_READ(Ptr[i]);
        if (value != GRID_POINT_EMPTY) {
            return true; //retry
        }
    }
    for (i = 1; i < (n-1); i++) {
    	TM_SHARED_WRITE(Ptr[i], GRID_POINT_FULL);
    }
    return false;
    
修改后性能就很不错了。
    






