# workerpool项目介绍

https://github.com/gammazero/workerpool

## 简介
限制goroutine同时运行的最大数量
## 安装
$ go get github.com/gammazero/workerpool
## 使用
在goroutine开启数量过多 导致goroutine堆积过多的情况下使用
![avatar](Imgs/1.png)
## 示例
    package main

	import (
		"fmt"
		"github.com/gammazero/workerpool"
	)

	func main() {
		wp := workerpool.New(2)
		requests := []string{"alpha", "beta", "gamma", "delta", "epsilon"}

	for _, r := range requests {
		r := r
		wp.Submit(func() {
			fmt.Println("Handling request:", r)
		})
	}

	wp.StopWait()
	}
	

## 源码分析
### 1 用chan限制goroutine
![avatar](Imgs/2.png)

	var taskLimit = make(chan struct{}, 10000)

	func handleTask1(ctx context.Context, f func()) {
		select {
		case taskLimit <- struct{}{}:
		case <-ctx.Done():
			return
		}
		defer func() {<-taskLimit}()
		f()
	}


### 2 实现一个简易go协程池
![avatar](Imgs/3.png)

	type Pool struct {
		maxWorkers int
		waitChannel chan func()
	}

	func NewPool(maxWorkers int) *Pool {
		p := Pool{
			maxWorkers:   maxWorkers,
			waitChannel:  make(chan func(), 100),
		}

		go p.run()
		return &p
	}

	func (p *Pool) run() {
		for i := 0; i < p.maxWorkers; i++ {
			go p.worker()
		}
	}

	func (p *Pool) worker() {
		for task := range p.waitChannel {
			task()
		}
	}

	func (p *Pool)Submit(task func()) {
		p.waitChannel <- task
	}

	func main() {

		p := NewPool(3)

		for i := 0; i < 1000; i++ {
			p.Submit(func() {
				fmt.Println(time.Now())
			})
		}

		time.Sleep(time.Second * 10)
	}

### 2.1 如何确定go协程已经被处理完了

- 可以约定好当启动的工作协程遇到nil就退出

#
![avatar](Imgs/4.png)

	type Pool struct {
		maxWorkers int
		waitChannel chan func()
		stopChannel chan struct{}
	}

	func (p *Pool) worker() {
		for task := range p.waitChannel {
			// 可以打卡下班跑路了
			if task == nil {
				// 打卡
				p.stopChannel <- struct{}{}
				return
			}
			task()
		}
	}

	func (p *Pool) Stop() {
		for i := 0; i < p.maxWorkers; i++ {
			p.Submit(nil)
		}
		for i := 0; i < p.maxWorkers; i++ {
			// 等待所有工作协程退出
			<-p.stopChannel
		}
		close(p.waitChannel)
	}
	

### 2.2 协程池需求
- 1 限制最大协程同时工作的数量
- 2 提交任务
- 3 暂停处理任务
- 4 停止所有协程工作(自由选择提交的任务是否全部完成)

#### 简易版问题

- 1 不能根据当前需要处理的任务数量 自动调整 所开启的work数量
- 2 没有实现暂停等需求

### 3 workerpool设计与实现

![avatar](Imgs/5.png)
### 3.1 按需生成减少worker

- 根据最大maxWorkers限制生成的worker

		if workerCount < p.maxWorkers {
			go startWorker(task, p.workerQueue)
			workerCount++
		} else {
			p.waitingQueue.PushBack(task)
			atomic.StoreInt32(&p.waiting, int32(p.waitingQueue.Len()))
		}

- 如果一段时间没有任务就清除一个worker

		case task, ok := <-p.taskQueue:
			// 其他操作
			idle = false
		case <-timeout.C:
			if idle && workerCount > 0 {
				if p.killIdleWorker() {
					workerCount--
				}
			}
			idle = true
			timeout.Reset(idleTimeout)
		}


- 清除一个worker 通过给他发送一个nil

		for task := range workerQueue {
			if task == nil {
				return
			}
			task()
		}
- 那killIdleWorker方法是不是就是 p.workerQueue <- nil
- 之前的假设是一段时间没有接受到新的消息了 就开始渐渐的关闭worker
- 但是可能存在这种情况 处理消息的能力不够 阻塞在等待队列中 如图所示
![avatar](Imgs/6.png)

		select {
			case p.workerQueue <- nil:
				return true
			default:
				return false
		}

### 3.2 如何加快接受处理的task

- 就好比平时去买票一样 要是根本就没什么人 自然不存在排队这一说
- 直接去窗口处理就好了 如同p.workerQueue <- task:
- 需要处理的请求多了 就渐渐的开启窗口 	如同 go startWorker(task, p.workerQueue)
- 到达设定的窗口范围就加入排队队伍中 如同 	p.waitingQueue.PushBack(task)

		case task, ok := <-p.taskQueue:
			if !ok {
				break Loop
			}
			select {
				case p.workerQueue <- task:
				default:
					if workerCount < p.maxWorkers {
						go startWorker(task, p.workerQueue)
					} else {
						p.waitingQueue.PushBack(task)
					}
				}

- 想想春运的时候 人山人海 那里还看得见什么窗口 甚至于排队到大厅外了
- 基于等待队列处理的任务多 则很可能进来的任务多的假设 workerpool做了这样一个判断
- 需要处理任务过多 就直接跳过后面的判断 每次只判断等待处理任务队列

		if p.waitingQueue.Len() != 0 {
			if !p.processWaitingQueue() {
				break Loop
			}
			continue
		}
- processWaitingQueue的实现如下 task进来了或者准备好处理新的task了
![avatar](Imgs/7.png)

		select {
			// tsak进来了
			case task, ok := <-p.taskQueue:
				if !ok {
					return false
				}
			p.waitingQueue.PushBack(task)
			// 准备好处理新的task了
		case p.workerQueue <- p.waitingQueue.Front().(func()):
			p.waitingQueue.PopFront()
		}

		return true


### 3.3 如何结束work
![avatar](Imgs/8.png)

- 1 任务结束了 关闭接受任务的chan并等待系统通知

		close(p.taskQueue)
	
		<-p.stoppedChan
- 2 清理各种资源 如创建的worker 这次可以直接传入nil 因为不会有需要处理的task了

		for workerCount > 0 {
			p.workerQueue <- nil
			workerCount--
		}

- 3 通知任务完成

		defer close(p.stoppedChan)
![avatar](Imgs/9.png)

- 4 确定要不要选择完成全部任务在关闭 
- 因为workerQueue是无缓冲的chan 当waitingQueue为空的时候
- 就确保了pool中的worker已经接受到最后一个需要处理的task

		if p.wait {
			p.runQueuedTasks()
		}

		func (p *WorkerPool) runQueuedTasks() {
			for p.waitingQueue.Len() != 0 {
				p.workerQueue <- p.waitingQueue.PopFront().(func())
			}
		}

## 总结

- 设计一般都是从通用开始的 然后针对一些特殊场景进行假设优化
- 如本文中基于有任务挤压就可能会有多个任务马上要进来的场景进行优化处理
- 协程池还有一个麻烦点在于不清楚限制多少个worker处理好 
- 希望可以增加一个自动测试worker数量的功能 程序自动测试worker从2-100 然后在对应的场景中收集各种处理情况方便别人确定设置worker的数量 

本文由「CDS」完成，「xxx」进行校对。属于 [Go语言中文网](https://studyggolang.com)旗下 「Go 开源资讯」组织创作。
