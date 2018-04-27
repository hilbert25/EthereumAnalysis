这部分代码主要实现了挖矿这一过程。

**miner**矿工的结构体：
<table>
        <tr>
            <th>变量名</th>
            <th>数据类型</th>
            <th>说明</th>
        </tr>
        <tr>
            <th>mux</th>
            <th>*event.TypeMux</th>
            <th>TypeMux用来注册时间接收者</th>
        </tr>
        <tr>
            <th>worker</th>
            <th>*worker</th>
            <th>见miner/worker.go</th>
        </tr>
        <tr>
            <th>coinbase</th>
            <th>common.Address</th>
            <th>矿工地址</th>
        </tr>
        <tr>
            <th>mining</th>
            <th>int32</th>
            <th>不知道是啥</th>
        </tr>
        <tr>
            <th>eth</th>
            <th>Backend</th>
            <th>前边定义了一个Backend接口，包含了mining需要的全部方法</th>
        </tr>
        <tr>
            <th>engine</th>
            <th>consensus.Engine</th>
            <th>见consensus/Engine</th>
        </tr>
        <tr>
            <th>canStart</th>
            <th>int32</th>
            <th>是否可以开始挖矿</th>
        </tr>
        <tr>
            <th>shouldStart</th>
            <th>int32</th>
            <th>同步后是否应该开始</th>
        </tr>
</table>

**new**

    func New(eth Backend, config *params.ChainConfig, mux *event.TypeMux, engine consensus.Engine) *Miner {
        miner := &Miner{
            eth:      eth,
            mux:      mux,
            engine:   engine,
            worker:   newWorker(config, engine, common.Address{}, eth, mux),
            canStart: 1,
        }
        miner.Register(NewCpuAgent(eth.BlockChain(), engine))
        go miner.update()

        return miner
    }
根据指定参数创建一个矿工，矿工注册一个agent并更新。这里涉及到了**Register**和**update**两个函数

    func (self *Miner) Register(agent Agent) {
        if self.Mining() {
            agent.Start()
        }
        self.worker.register(agent)
    }
另一个函数update稍微复杂一点。用来追踪下载事件。是一种快照类型的更新循环。只执行一次，一旦广播出了Done或者Failed，事件就不被注册，跳出循环。 

**update**首先注册一个events，遍历events，根据event的data做出不同的行为。一共三种事件：**StartEvent，DoneEvent，FailedEvent**。**StartEvent**意味者此时本节点正在从其他节点下载新区块，这时miner会立即停止进行中的挖掘工作，并继续监听。**oneEvent**或**FailEvent**时，意味本节点的下载任务已结束-无论下载成功或失败-此时都可以开始挖掘新区块，并且此时会退出Downloader事件的监听。

