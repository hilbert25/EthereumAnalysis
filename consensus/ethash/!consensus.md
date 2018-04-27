这部分代码与验证及挖矿难度有关。
挖矿难度有三个版本的函数：
**calcDifficultyByzantium**，**calcDifficultyHomestead**，**calcDifficultyFrontier**
三个函数计算公式不同
不过整体来说有一些共同之处：

    diff = parent_diff + adjustment + bomb
    adjustment = f(parent_diff,parent.timestamp,block_timestamp)
    bomb = 2^(number/100000 - 2)
三个公式主要区别在**adjustment**
- **calcDifficultyByzantium**参数为当前时间的时间戳+父区块头。
计算公式：

	diff = 
    (parent_diff
    +(parent_diff / 2048 * max((2 if len(parent.uncles) else 1) - ((timestamp - parent.timestamp) // 9), -99))
    ) + 2^(periodCount - 2)

- **calcDifficultyHomestead**参数为当前时间的时间戳+父区块头。
计算公式：

    diff = 
    (parent_diff
    +(parent_diff / 2048 * max(1 - (block_timestamp - parent_timestamp) // 10, -99))) + 2^(periodCount - 2)
　其中//为整数除法运算符，a//b = a/b向下取整

- **calcDifficultyFrontier**参数为当前时间的时间戳+父区块头。

计算公式：

    diff = 
    (parent_diff
    +(parent_diff / 2048 * max(1 - (block_timestamp - parent_timestamp) // 10, -99))) + 2^(periodCount - 2)
　其中//为整数除法运算符，a//b，即先计算a/b，然后取不大于a/b的最大整数
解释下**periodCount**
<pre>
    <code>
        periodCount := new(big.Int).Add(parent.Number, big1)
	periodCount.Div(periodCount, expDiffPeriod)
    </code>
</pre>
preiodCount = 父区块的编号+1，再除以expDiffPeriod（=100000）

在 **CalcDifficulty**中规定了什么时候用哪种计算公式：

        func CalcDifficulty(config *params.ChainConfig, time uint64, parent *types.Header) *big.Int {
            next := new(big.Int).Add(parent.Number, big1)
            switch {
            case config.IsByzantium(next):
                return calcDifficultyByzantium(time, parent)
            case config.IsHomestead(next):
                return calcDifficultyHomestead(time, parent)
            default:
                return calcDifficultyFrontier(time, parent)
            }
        }
首先获取当前区块的number，然后作为参数传入IsByzantium或IsHomestead。

    func (c *ChainConfig) IsByzantium(num *big.Int) bool {
        return isForked(c.ByzantiumBlock, num)
    }
    
    func (c *ChainConfig) IsConstantinople(num *big.Int) bool {
        return isForked(c.ConstantinopleBlock, num)
    }
他们都调用了

    func isForked(s, head *big.Int) bool {
        if s == nil || head == nil {
            return false
        }
        return s.Cmp(head) <= 0
    }
具体的**ByzantiumBlock** **ConstantinopleBlock**的值在params/config.go可以看到：
<table>
    <tr>
        <td>
            参数/网络
        </td>
        <td>
            MainnetChain
        </td>
        <td>
            TestnetChain
        </td>
        <td>
            RinkebyChain
        </td>
    </tr>
    <tr>
        <td>
            ByzantiumBlock
        </td>
        <td>
            4370000
        </td>
        <td>
            1700000
        </td>
        <td>
            1035301
        </td>
    </tr>
    <tr>
        <td>
            HomesteadBlock
        </td>
        <td>
            1150000
        </td>
        <td>
            0
        </td>
        <td>
            1
        </td>
    </tr>
</table>