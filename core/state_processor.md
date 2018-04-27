这部分代码负责状态转换执行交易。

**StateProcessor**结构体：
<table>
        <tr>
            <th>变量名</th>
            <th>数据类型</th>
            <th>说明</th>
        </tr>
        <tr>
            <th>config</th>
            <th>*params.ChainConfig</th>
            <th></th>
        </tr>
        <tr>
            <th>bc</th>
            <th>*BlockChain</th>
            <th></th>
        </tr>
        <tr>
            <th>engine</th>
            <th>consensus.Engine</th>
            <th>一致性引擎</th>
        </tr>
</table>

通过**Process**函数具体执行交易，函数设置好接收人Gas日志等信息。主要部分是对区块的交易进行循环调用**ApplyTransaction**，如果有err则返回err。
**ApplyTransaction**完成具体的交易，当gas消耗光的时候返回error。函数首先获取到一个context和environment。然后调用**ApplyMessage**函数，追踪下去发现最后执行的是state_transaction中的**TransitionDb**函数。函数参数只有一个，其结构体如下：
<table>
    <tr>
        <th>
            变量名
        </th>
        <th>
            变量类型
        </th>
        <th>
            说明
        </th>
    </tr>
    <tr>
        <th>
            gp
        </th>
        <th>
            *GasPool（uint64）
        </th>
        <th>
            还有多少gas可用
        </th>
    </tr>
    <tr>
        <th>
            msg
        </th>
        <th>
            Message
        </th>
        <th>
            发送给合约的message
        </th>
    </tr>
    <tr>
        <th>
            gas
        </th>
        <th>
            uint64
        </th>
    </tr>
    <tr>
        <th>
            gasPrice
        </th>
        <th>
            *big.Int
        </th>
        <th>
            gas单价
        </th>
    </tr>
    <tr>
        <th>
            initialGas
        </th>
        <th>
            uint64
        </th>
    </tr>
    <tr>
        <th>
            value
        </th>
        <th>
            *big.Int
        </th>
        <th>
            不知道是啥
        </th>
    </tr>
    <tr>
        <th>
            data
        </th>
        <th>
            []byte
        </th>
        <th>
            不知道是啥
        </th>
    </tr>
    <tr>
        <th>
            state
        </th>
        <th>
            vm.StateDB
        </th>
        <th>
            虚拟机数据库
        </th>
    </tr>
    <tr>
        <th>
            evm
        </th>
        <th>
            *vm.EVM
        </th>
        <th>
            虚拟机(很多玄学bug的根源)
        </th>
    </tr>
</table>

函数首先检查区块的nonce，

调用**preCheck**函数，**preCheck**检查完nonce后开始购买gas（**buyGas**函数），要保证StateTransaction中的msg的nonce和state的nonce一致，不一致则返回error。
执行完perCheck后获取msg sender homestead（版本号） contractCreation等信息。

调用**IntrinsicGas** 支付固有gas。看一下**IntrinsicGas**函数。函数首先设置初始gas，如果是创建合约并且是homestead那么gas=53000否则21000（metamask默认值）。如果交易的data长度大于0则进行下面的操作：

    1. 计算data的非0比特位数，存在 nz中（uint64）
    2. 计算gas会不会超过math.MaxUint64(math.MaxUint64-gas再除以每个非0byte需要的gas)，超过就返回out of gas。
        0 和非0比特位需要的gas是不一样，现在的nz只存了非0的个数，因此算算最大的gas是否够这些比特位消耗。非0比特位消耗gas为68。因此最后的代码为：
            if (math.MaxUint64-gas)/params.TxDataNonZeroGas < nz {
			return 0, vm.ErrOutOfGas
		}
    3.      gas += nz * params.TxDataNonZeroGas
		z := uint64(len(data)) - nz
		if (math.MaxUint64-gas)/params.TxDataZeroGas < z {
			return 0, vm.ErrOutOfGas
		}
		gas += z * params.TxDataZeroGas
        
        gas+=非0比特需要的gas
        z=0比特位数量
        计算下0比特需要的gas，不够就返回ErrOutOfGas
        gas+=比特需要的gas
    4. 最后返回需要的gas
回到**TransitionDb**，**IntrinsicGas**会返回需要的gas和err。
如果err不为空就返回err，然后判断下**useGas**函数会不会返回err，如果有err继续返回err。这个是用来判断提供的gas够不够。

如果需要创建合约，那么就调用evm中的**Create**函数创建一个合约。看一下**Create**函数。检查evm栈深度，超过1024报错："max call depth exceeded"。（不知道栈深度如何确定）.检查余额够不够，不够报错“insufficient balance for transfer”。

获取caller的地址的nonce，然后设置evm.StateDB的nonce为nonce+1.
**为了防止交易重播，ETH（ETC）节点要求每笔交易必须有一个nonce数值。每一个账户从同一个节点发起交易时，这个nonce值从0开始计数，发送一笔nonce对应加1。当前面的nonce处理完成之后才会处理后面的nonce。（这段抄的）**。根据caller的地址和nonce创建一个合约的地址，根据合约地址获取合约的hash。做个判断，以下条件符合一个就报err：contract address collision。
    
    1.合约地址的nonce不为0
    2.hash不为空也不为Keccak256Hash（nil）
创建account，创建合约。
如果是debug模式或者evm深度为0就……开始调用**run**函数启动evm。来看一下**run**函数，run执行一个给定的合约，负责运行预编译，并返回到字节码解释器。

    func run(evm *EVM, contract *Contract, input []byte) ([]byte, error) {
        if contract.CodeAddr != nil {
            precompiles := PrecompiledContractsHomestead
            if evm.ChainConfig().IsByzantium(evm.BlockNumber) {
                precompiles = PrecompiledContractsByzantium
            }
            if p := precompiles[*contract.CodeAddr]; p != nil {
                return RunPrecompiledContract(p, input, contract)
            }
        }
        return evm.interpreter.Run(contract, input)
    }
    首先确定下预编译的参数，跟以太坊版本有关。如果precompiles不为空，则RunPrecompiledContract。RunPrecompiledContract会计算你需要的gas，超了报错否则run代码。
    
如果合约创建执行成功且没有偏差，计算存储代码需要的gas。如果代码因为gas不够不能被存储，那么根据下边的错误检查条件处理一下：
    
    if err == nil && !maxCodeSizeExceeded {
		createDataGas := uint64(len(ret)) * params.CreateDataGas
		if contract.UseGas(createDataGas) {
			evm.StateDB.SetCode(contractAddr, ret)
		} else {
			err = ErrCodeStoreOutOfGas
		}
	}
    这段代码主要是判断gas有没有超。超了就返回ofg的错误。
当evm返回错误或者设置创建代码，将恢复到快照并消耗剩余的gas。如果是homestead，这对于代码存储错误也很重要。如果一下条件符合任意一项：

    1. 超过最大代码限制（24576）
    2. （err不为空且是Homestead）或没有“contract creation code storage out of gas”
回到快照。后边的代码处理maxCodeSizeExceeded和Debug问题。

回到**TransitionDb**，如果不创建合约则setNonce，直接call。如果
evm有错误，则输出一下err。evm可能报的错误有下（看一下evm.go里的call）：

    1.ErrDepth "max call depth exceeded" evm栈深度超过1024
    2.ErrInsufficientBalance "insufficient balance for transfer" 余额不足
执行 **refundGas**退回多的gas


    refund := st.gasUsed() / 2
	if refund > st.state.GetRefund() {
		refund = st.state.GetRefund()
	}
	st.gas += refund

	remaining := new(big.Int).Mul(new(big.Int).SetUint64(st.gas), st.gasPrice)
	st.state.AddBalance(st.msg.From(), remaining)

	st.gp.AddGas(st.gas)
以下节选自黄皮书：

    最终要退回发送者的燃料g<sup>*</sup>等于当前剩余的燃料g<sup>′</sup>, 再加一个补偿, 这个补偿是A<sub>r</sub> 和总使用燃料量的一半中的小者。
关于A<sub>r</sub>的定义：

    Finally there is Ar, the refund balance,
    increased through using the SSTORE instruction in order
    to reset contract storage to zero from some non-zero value.Though not immediately refunded, it is allowed to partially offset the total execution costs.
    用sstore指令把非0合约存储空间置0时Ar会增加。

    

