# 基于Rust的无锁队列工程解读

## 问题抽象

现有一个较大的文件(>20G)，通过某种算法可以利用这个文件生成另外一个相同大小的文件，而这个目标文件的生成同时需要用到已经产生的数据部分，即假设两个文件均有n个节点，而新文件中的i号节点需要用到原始文件中的部分节点以及新文件中[0,i-1]中的部分节点，希望通过无锁队列来完成新文件的生成

![问题抽象](/lockfree/1.png "问题抽象")

---

## 解决思路

**总体逻辑**

算法实现的难点在于生成新文件同时需要用到本层的已经生成的数据，不能将程序单独拆分为读取和运算两部分，当读取领先于计算部分时，会存在需要的数据还未生成的情况，需要计算单元单独从已经计算完毕的部分再次载入已经完成的部分。

将整个算法拆分成两个部分，由三个线程实现。两部分分别为纯读取部分，以及读取运算混合部分。由于读取数据的速率慢于计算的速度，所以使用多个线程负责读取以及标记本层缺失的部分。使用一个线程负责执行计算，以及从标记缺失处读取缺失的数据。

为了提升程序的效率，将全部的待运算数据放入内存中连续的地址中，对于复杂的运算，这样会大大提高程序的运行效率。

![总体逻辑](/lockfree/2.png "总体逻辑")

**扩展阅读** 

-[Linux线程与windows线程区别](https://www.cnblogs.com/leisure_chn/p/10393707.html)

-[自旋锁](https://www.cnblogs.com/cxuanBlog/p/11679883.html)

-[CAS算法](https://zhuanlan.zhihu.com/p/137261781)


**共享内存**

三个线程共享两片内存，分别为本层数据内存(layer_label)，读取算法参数内存(ring_buf)，本层内存主线程(consumer)具有读写的权利，子线程(producer)只有读的权力；算法参数内存首先由子线程填充原始文件数据，再填充已经生成的新文件中的数据，并将缺失的数据标记。

子线程之间共享原始文件所对应的内存，二者只负责将所对应的数据放入与主线程共享的算法参数内存中。
![共享内存](/lockfree/3.png "共享内存")

为了实现内存在三各个线程中共享，定义了变量`UnsafeSlice`以及`RingBuf`，`UnsafeCell`中存放的原始数据可以通过`ptr`以及`length` 重新构建，`RingBuf`的重新构建与`UnsafeSlice`相似，只不过指针需要临时获得。

```
#[derive(Debug)]
pub struct UnsafeSlice<'a, T> {
	// holds the data to ensure lifetime correctness
	data: UnsafeCell<&'a mut [T]>,
	/// pointer to the data   
	ptr: *mut T,   
	/// Number of elements, not bytes.   
	len: usize,
}

#[derive(Debug)]
pub struct RingBuf {    
	data: UnsafeCell<Box<[u8]>>,   
	slot_size: usize,   
	num_slots: usize,
}
```
`UnsafeSlice`的重构方法
```
#[inline]
pub unsafe fn as_mut_slice(&self) -> &'a mut [T] {
    slice::from_raw_parts_mut(self.ptr, self.len)
}
```

`RingBuf`的重构方法
```
#[allow(clippy::mut_from_ref)]
unsafe fn slice_mut(&self) -> &mut [u8] {
    slice::from_raw_parts_mut((*self.data.get()).as_mut_ptr(), self.len())
}
```

原始文件所对应的内存(exp_label)，当前层的文件(layer_label)类型为: UnsafeSlice。

参数内存(ring­_buf)的类型为：RingBuf，由于其大小有限，因此为环状结构，即当数据填满所有内存后会从头开始填充，覆盖掉之前的数据，因此consumer的长度不能超过producer的规定长度(lookahead)。

**共享参数**

主线程和子线程之间共享主线程的进度参数(consumer)和子线程总进度(cur_producer)，这两个变量的读取和修改应该为原子操作，即当一个线程在读取数据或修改时，另一个线程不能去访问这个数据。子线程的进度不能超过主线程的进度过多，否则会导致用于参数共享的内不足，并且本层数据的缺失率会增大。而主线程需要等待子线程进度超过主线程进度后才能继续执行（主线程需要子线程读取原始文件中的数据）。

子线程之间需要同步进度，使用两个原子变量实现，所有子线程已经完成的总进度(cur_producer)，而用于告知其余子线程这一段由自己来处理的变量(cur_await)。在整个程序执行中，占位变量要领先于子线程总进度。子线程首先读取占位变量，获取自己可以执行的位置，获取成功后将站为变量进行自增操作，当别的子线程再次访问此变量时，便可以知晓前一段已经被其他线程获取，只需要接着执行即可。数据填充完毕后子线程并不能直接将总进度变量增加，为了防止数据共享内存中空段的出现，需要获取等待上一段的的线程执行完毕（cur_producer增加后），才能将总进度增加，否则会一直等待。这样可以保证每一段数据都被获取到，不会出现空线程的现象。`cur_awaiting`、`cur_producer`、`consumer`的类型为[AtomicU64](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicU64.html)，满足[RMW (read modify write)](https://preshing.com/20120612/an-introduction-to-lock-free-programming/) 无锁程序(Lockfree program) 的规则。
![共享参数](/lockfree/4.png "共享参数")

代码结构
--------

用crossbeam开辟三个线程，首先启动子线程producer，然后执行主线程consumer的循环，直至所有节点产生完毕。

```
Crossbeam::thread::scope(|s| {
    let mut runners = Vec::with_capacity(num_producers);
     
     for i in 0..num_producers {
        
        // sub-thread consumer 
        runners.push(s.spawn(move |_| {
        // read data
        }))
     }
     
     // main-thread producer 
     // calculate
     for runner in runners {
            runner.join().expect("join failed");
     }
     
}).expect("crossbeam scope failure");
```
-**共享参数**

```
Layer_label: UnsafeSlice

Exp_label: UnsafeSlice

Base_parent_missing: UnsafeSlice

Consumer: AtomicU64

Cur_producer: AtomicU64

Cur_awaiting: AtomicU64 

Ring_buf: RingBuf
```

子线程
------

**重要参数**

`ring_buf`（算法参数内存）：由producer所传入的参数节点

`cur_producer`：当前所有producer的进度

`cur_awaiting`：当前所有producer已经分配的进度

`layer_label`s：新文件数据所对应的内存

`exp_labels`：源文件中所对应的内存

`base_parent_missing`：由producer标记的缺失节点

`consumer`：当前consumer的进度

`stride`：每个子线程每次前进的最大步长

`number_nodes`：总结点的数量

`lookahead`：Ring_buf的大小，能存储节点信息的个数

**伪代码**

子线程执行一个循环结构，当所有的任务都被分配完毕时`cur_awaiting >= num_nodes`退出循环。函数大致分为两部分，同步部分和填充数据部分，读取数据通过调用读取函数实现。

```
Loop{

	If Cur_await >=  num_nodes then quit loop
	Cur_await += stride
	For node between origin
	{
		cur_await and added cur_await
		If node –consumer > lookahead then sleep
		Calling fill_buffer function
	}

	If origin cur_await > cur_producer then sleep

}
```

**线程同步**

子线程(producer)之间通过共同维护，子线程总进度变量`cur_producer`，子线程“占位”变量`cur_awaiting`实现同步。对于每个子线程首先从“占位”变量中取出已经被分配的任务节点，向占位变量增加一定的长度告知其他子线程此处任务已经被分配。而在一个子线程获取占位变量后，便可以执行数据填充操作，在向算法参数内存`ring_buf`中填充数据前需要判断待填充数据的序号不能领先主程序超过预留的节点个数`lookahead`，否则算法参数内存中还未使用的数据将会被覆盖掉，在要填充的数据序号不领先于主线程一定长度`lookahead`长度的情况下，将数据填充至共享内存当中，假设子线程当前的进度为`p`，那么应满足`p - consumer <=lookahead`

填充完数据后，子线程则需要修改`cur_producer`变量，在此使用了[CAS(Compare And Swap)的概念](https://zhuanlan.zhihu.com/p/137261781)，子线程不能直接调节子线程总进度变量`cur_producer`，需要判断之前节点的参数是否填充完毕，如果填充完毕则可以修改`cur_producer`变量，告知所有线程(producer)的新完成了部分数据，并且存放在`ring_buf`中，这样不会`ring_buf`中不会出现有些数据还未填充但其他线程(producer)的变量已经显示完毕的情况，与主线程实现同步。

![子线程](/lockfree/5.png "子线程")
**填充函数**

函数首先处理当前层所需要的节点对应序号的数据。由于数据来自于本层需要由主程序处理产生，因此存在概率部分节点还未产生，但基于无锁队列的设计，并且程序也不希望在此处等待这些未产生的节点，因此将这些还未产生的节点标记至`base_parent_missing`中，由主程序执行到此自行获取。

在标记完毕本层的数据后，取出放入缓存中或者标记为"missing"。程序从上层节点取出数据，写入buffer（上层数据已经产生，因此不存在missing的情况）。

主线程
------

**传入参数**

`ring_buf`：由producer所传入的参数节点

`cur_producer`：当前所有producer的进度

`layer_labels`：新文件数据所对应的内存

`base_parent_missing`(本层缺失标记)：由producer标记的缺失节点

`consumer`：当前consumer的进度

`number_nodes`：总结点的数量

`lookahead`：Ring_buf的大小，能存储节点信息的个数

`i`：现已完成的阶段序号，不断自增直至所有节点计算完毕i>=num_nodes。

**主线程逻辑**

主线程执行一个有限循环结构，直至所有节点都产生完毕，对于每个子线程准备完毕的节点首先根据本层缺失标记填充相应的节点，然后利用算法参数内存中已经准备完毕的所有数据计算得到此节点。

```
While i < num_nodes
{

	If cur_producer < i then sleep
       	
	Get all the done node
	
	For known done nodes
	{
       		
		Use base_parent_missing to fill ring_buf
		Generate node[i] with ring_buf
		i +=1
		consumer +=1
	}
}
```

**主线程执行流程**

首先从共享变量中取出`cur_producer`，并等待主线程中的当前序号数(i)小于producer的总进度时`cur_producer> i`，然后根据现有所有已完成的producer的节点进行运算。

对于已知的consumer完成的所有节点，进行如下操作。首先从`base_parent_missing`中读取缺失的本层节点填充至`ring_buf`中，为接下来的运算做准备。当计算完毕后，将consumer 变量自增（告诉所有子线程已经计算完毕），再把其他必要的变量自增，为下一个节点的计算做准备。当所有已经准备好的节点计算完毕后，需要再次访问判断是否有准备好的节点，如果有则与上相同将所有节点取出并运算。
![主线程](/lockfree/6.png "主线程")

整体模型
--------

"producer"和"consumer"通过`ring_buf`交互，实现同步。
![整体模型](/lockfree/7.png "整体模型")

源代码
------

```
fn create_layer_labels(
    parents_cache: &CacheReader<u32>,
    replica_id: &[u8],
    layer_labels: &mut MmapMut,
    exp_labels: Option<&mut MmapMut>,
    num_nodes: u64,
    cur_layer: u32,
    core_group: Arc<Option<MutexGuard<'_, Vec<CoreIndex>>>>,
) 
```
[code](https://github.com/filecoin-project/rust-fil-proofs/blob/master/storage-proofs-porep/src/stacked/vanilla/create_label/multi.rs)
 at `fn create_layer_labels()`


