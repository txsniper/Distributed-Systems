## MapReduce流程解析
MapReduce主要包含两种模式：本地，分布式，分别调用 master.go 中的 Sequential和Distributed方法，这里只解析分布式的流程

## 分布式模式
wg.go中的main函数首先启动master服务端，再启动worker， master和worker逻辑上运行在不同的机器上，两者通过RPC交互，master会启动一个RPC服务端，等待workder连接，worker会连接master的服务并注册自己，表明自己已经准备好接受任务调度

## master进程

```
func Distributed(jobName string, files []string, nreduce int, master string) (mr *Master) {

        // step1 : 创建新的master 
	mr = newMaster(master)
        // step2 : 启动RPC服务
	mr.startRPCServer()
        // step3 : 启动任务
	go mr.run(jobName, files, nreduce, mr.schedule, func() {
		mr.stats = mr.killWorkers()
		mr.stopRPCServer()
	})
	return
}
```

### master 
master基于unix套接字来提供RPC服务
```
rpcs := rpc.NewServer()
rpcs.Register(mr) // 注册自己的方法
os.Remove(mr.address) // only needed for "unix"
l, e := net.Listen("unix", mr.address)  // unix套接字，套接字和文件绑定
if e != nil {
	log.Fatal("RegstrationServer", mr.address, " error: ", e)
}
mr.l = l
```
整个master的执行流有下面几个部分：
1. 主流程 ：启动管理master其他流程的主流程，它会启动master的RPC服务，启动调度流程
2. RPC服务 ：
主流程会创建一个goroutine 来提供RPC 服务，服务提供的RPC接口主要有：
    - Shutdown (master_rpc.go) : 关闭master的RPC服务 
    - Register (master.go) : worker注册时调用的接口
3. 调度流程 ： 同样由主流程创建goroutine执行，主要将工作分配给worker完成，并且合并结果数据，之后让worker退出，并且调用RPC接口关闭RPC服务

```
func Distributed(jobName string, files []string, nreduce int, master string) (mr *Master) {
	mr = newMaster(master)
	mr.startRPCServer()
	go mr.run(jobName, files, nreduce, mr.schedule, func() {
		mr.stats = mr.killWorkers()
		mr.stopRPCServer()
	})
	return
}
```
### worker
worker分为两个流程，主流程负责提供RPC服务，等待 master的调度启动goroutine来执行具体的任务；具体的 map 或者 reduce任务由主流程创建的 goroutine来完成

worker流程：
1. 主流程 ：启动worker的RPC服务, 然后调用 master 的 Register接口注册自己，之后等待 master调度，当有新任务到达时会创建 goroutine 来执行
2. 计算流程 ：执行具体的 map/reduce 任务

```
func RunWorker(MasterAddress string, me string,
	MapFunc func(string, string) []KeyValue,
	ReduceFunc func(string, []string) string,
	nRPC int,
) {
	debug("RunWorker %s\n", me)
	wk := new(Worker)
	wk.name = me
	wk.Map = MapFunc
	wk.Reduce = ReduceFunc
	wk.nRPC = nRPC
        // 启动 worker RPC服务
	rpcs := rpc.NewServer()
	rpcs.Register(wk)
	os.Remove(me) // only needed for "unix"
	l, e := net.Listen("unix", me)
	if e != nil {
		log.Fatal("RunWorker: worker ", me, " error: ", e)
	}
	wk.l = l

        // 向 master 注册自己
	wk.register(MasterAddress)

	// DON'T MODIFY CODE BELOW
	for {
		wk.Lock()
		if wk.nRPC == 0 {
			wk.Unlock()
			break
		}
		wk.Unlock()

                // 等待 master 的调度
		conn, err := wk.l.Accept()
		if err == nil {
			wk.Lock()
			wk.nRPC--
			wk.Unlock()
                        // 新建一个goroutine 来执行具体的任务
			go rpcs.ServeConn(conn)
			wk.Lock()
			wk.nTasks++
			wk.Unlock()
		} else {
			break
		}
	}
	wk.l.Close()
	debug("RunWorker %s exit\n", me)
}
```