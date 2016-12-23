# g3ar 渗透编程工具包： 极速打造渗透测试工具


## 功能概述

1. 多线程与多进程管理：
	* 线程池与多线程调用极简化
	* 异步
	* Taskbulter 对进程内的线程监控与管理
* 爆破用字典管理：
	* 支持任意大小的字典读取（大字典不再发愁）
	* 进度保存
	* 获取当前进行进度
* 日志记录器：
	* 日志管理极简化
	* 装饰器接口
	* 自动追踪函数，日志记录函数的调用，Trace，和异常情况


## 多线程管理：

### ThreadPool：更优雅的线程池
线程池模块可以有效减少程序在开启和结束线程上的开销，达到批量执行任务并且，同时，该模块以异步方式工作，通过一个结果队列来输出结果，方便你在运行的过程中完成其他的工作，也不必烦恼找不到办法拿到多线程执行的结果。  

当然为了更舒服的编程体验我们自然不能和 `Stackless gevent eventlet` 之类的相比了。

基本使用方法如下：

	from g3ar import ThreadPool

	#
	# 定义函数（任何参数形式都可以）
	#
    def testthreadpool_function(arg1, arg2='adsf'):
        sleep(1)
        return arg1, arg2

	# 创建一个线程池
    pool = ThreadPool(thread_max=30)

	# 启动这个线程池并且初始化线程池
    pool.start()

	# 为线程池添加任务
    for i in range(40):
        pool.feed(testthreadpool_function, arg1=i)
	
	#
	# 结果获取方式之一：
	# 通过一个 Generator 来获取
	#
    result_gen = pool.get_result_generator()

	# Generator 的使用方法
    for i in result_gen:
        print i


	#
	# 再次添加任务
	#
    for i in range(21,41):
        pool.feed(testthreadpool_function, arg1=i)

	# 通过获取结果队列的方式来获得任务结果
    result_queue = pool.get_result_queue()

	#
	# 使用结果队列
	#
    while True:
        try:
            print result_queue.get(timeout=2)
        except Empty:
            print 'End!'
            break

除此之外还有几个常用属性和方法：

#### 属性
`pool.task_count` 返回值是一个整数表示当前任务数量  
`pool.executed_task_count` 返回值是一个整数表示已经执行的任务数量（包括出错的任务）  
`pool.percent` 一个 0-1 的浮点型数，表示当前执行百分比  

#### 方法
pool.start() 启动线程池  
pool.stop() 停止线程池
pool.get_result_queue() 获取结果队列  
pool.get_task_queue() 获取任务队列  
pool.result_generator() 获取结果生成器  
pool.feed(target_func, *vargs, **kwargs) 传入想要执行的函数 `vargs` 和 `kwargs` 保证你可以传入原来参数的任意参数（只要不使用 `target_func` 和 `function` 作为形式参数的参数名就 OK）  

### Contractor：短时间大开销执行已知任务

Contractor 的接口基本和 ThreadPool 相同，但是不同的是，它的每一次新的任务都会启动一个新的线程，但是构造更简单，可以理解为一个不节能版的 ThreadPool，创建 Contractor 的速度更快，初始化快速，比较适合执行任务比较少或者是性能要求不是太严苛的情景。  

相比 ThreadPool，没有 stop 方法执行完就可以随意丢掉，删掉或者是什么都随便，当然还可以继续添加任务，调用 start() 开始一轮新的任务。**但是并不推荐在一次任务执行未完成的时候使用正在执行任务的 Contractor 实例**，这样做的**后果就是，会把之前的任务重新执行一遍**（除非你真的有这样的需求），如果需要动态执行任务，推荐使用 ThreadPool。  

总而言之，Contractor 速度略快于 ThreadPool 但是相比 ThreadPool 的话没有那么稳重与优雅。  
另外一个区别就是，Contractor 返回的结果不包涵任务执行情况和执行任务的线程的状况，如果任务发生异常会把异常和 traceback 的信息作为返回值，如果任务正常执行，则返回函数原来的执行结果。

使用起来的话，两者没有什么感觉，甚至有时候想相互替换只需要把创建实例的类名修改一下就可以了。  

正常使用：

	from g3ar import Contractor

    def testcontractor_function(arg1, arg2='adsf'):
        sleep(1)
        return arg1, arg2

	# 创建 Contractor 实例
    ctr = Contractor(thread_max=30)

	# 添加任务
    for i in range(40):
        ctr.feed(testcontractor_function, arg1=i)
    
	# 执行任务
    ctr.start()

	# 获取结果
    result_gen = ctr.get_result_generator()


    for i in result_gen:
        print i

---
	

    ctr = Contractor(thread_max=50)

	#
	# 测试异常状况
	#
    def testcontractor_function_ecp(arg1, arg2='adsf'):
        sleep(1)
        raise Exception('test exception')
        #return arg1, arg2
    
    for i in range(41, 81):
        ctr.feed(testcontractor_function_ecp, arg1=i)

    ctr.start()

        
    result_queue = ctr.get_result_queue()

    while True:
        try:
            print result_queue.get(timeout=2)
        except Empty:
            print 'End!'
            break

其他方法和属性说明（对比 ThreadPool）：

* 和 ThreadPool 基本相同。
* 新增加 add_task 方法（用法与 feed 方法完全一样）。
* 没有 stop 方法。


## 多进程管理：

在 Python 中，经常要遇到的问题是，线程没有办法强行结束，为什么呢？（为什么 Python 的线程没有提供 kill 或者是 stop 或者 truncate 或者 abort 方法？怎么强行停止一个 Python 线程？）答案在这里 [StackOverFlow 关于杀死 Python 线程的回答](http://stackoverflow.com/questions/323972/is-there-any-way-to-kill-a-thread-in-python)。  
为什么要杀死一个线程呢？（线程那么可爱怎么可以杀死他），其实 Python 是因为共享资源等原因，故意不提供线程的结束方法的。  

但是有些时候我们还是想强行结束某一个任务，但是进程（multiprocessing） 提供了比较好的办法。难道我们每一个任务都要使用 Process？（Process 是进程，Thread 是线程，一般来说进程的开销要比 Thread 大，虽然 Process 可以支持多核执行，但是过多的 Process 并不见得有优势）。那么比较理想的方式其实就是 Process + Thread 来执行你的任务。

那么想要强行结束某一项任务又不违和，就去使用进程吧！  

### TaskBulter 控制&监视你的进程与线程

Bulter 的意思是男管家，当然是用来管理你的任务的，通过什么管理呢？简单来说就是：如果你使用了 TaskBulter 来执行你的任务（任务事先写好-最好使用多线程而不是多进程），你的每一个任务都会专门启动一个进程（Process），在这个进程内，你可以运行多个线程（使用 ThreadPool，或者 Contractor）。TaskBulter 可以监视你的任务执行情况，和任务内的线程的生存状况，如果任务内关键的线程已经挂了，任务还是没有办法自己退出，你就可以调用 TaskBulter 来强行结束掉你的任务，同样的如果只是想了解各个线程的运行状况，TaskBulter 也提供了监控功能。  

当然，性能上并不占优势，但是可以在一定程度上保证稳重与优雅。  

#### 示例代码

	from g3ar import TaskBulter

	def tasktest(arg1):
	    print(arg1)
	    def runforever():
	        while True:
	            pass
	    
	    pool = ThreadPool()
	    pool.start()
	    pool.feed(runforever)
	    pool.feed(runforever)
	    pool.feed(runforever)
	    while True:
	        pass

    bulter = TaskBulter(threads_update_interval=0.3)
        
    bulter.start_task(id='tasktest', target=tasktest, args=(5,))
    task = bulter.get_task_by_id('tasktest')
    print task
    #print bulter.get_task_status()
    sleep(2)
    bulter.destory_task(task)
    #bulter.close()
