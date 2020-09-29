---
title: “以太坊智能合约设计缺陷问题”影响分析报告
date: 2018-08-22 10:15:09
tags:
- contract
- eth
---

在审计各种智能合约之后，我发现了一类很有趣的问题，这类问题出现的原因不只是由于开发者的疏忽，也同样是因为智能合约本身的一些设计缺陷，在开发者不了解这些问题的基础上，就容易出现问题。

智能合约checklist系列文章:

[“以太坊智能合约规范问题”影响分析报告](https://lorexxar.cn/2018/08/14/haotian-s-1/)

<!--more-->

# 一、 简介

在知道创宇404区块链安全研究团队整理输出的《知道创宇以太坊合约审计CheckList》中，把“条件竞争问题”、“循环dos问题”等问题统一归类为“以太坊智能合约设计缺陷问题”。

“昊天塔(HaoTian)”是知道创宇404区块链安全研究团队独立开发的用于监控、扫描、分析、审计区块链智能合约安全自动化平台。我们利用该平台针对上述提到的《知道创宇以太坊合约审计CheckList》中“以太坊智能合约设计缺陷”类问题在全网公开的智能合约代码做了扫描分析。详见下文：

# 二、漏洞详情

## 1、条件竞争

2016年11月29号，Mikhail Vladimirov和Dmitry Khovratovich公开了一篇《ERC20 API: An Attack Vector on Approve/TransferFrom Methods》\[1\]，在文章中提到了一个在ERC20标准中存在的隐患问题，**条件竞争**。

这里举一个approve函数中会出现的比较典型的例子，approve一般用于授权，比如授权别人可以取走自己的多少代币，整个流程是这样的：

- 1、用户A授权用户B 100代币的额度
- 2、用户A觉得100代币的额度太高了，再次调用approve试图把额度改为50
- 3、用户B在待交易处（打包前）看到了这笔交易
- 4、用户B构造一笔提取100代币的交易，通过条件竞争将这笔交易打包到了修改额度之前，成功提取了100代币
- 5、用户B发起了第二次交易，提取50代币，用户B成功拥有了150代币

想要理解上面这个条件竞争的原理，首先我们得对以太坊的打包交易逻辑有基础认识。

[https://medium.com/blockchannel/life-cycle-of-an-ethereum-transaction-e5c66bae0f6e](https://medium.com/blockchannel/life-cycle-of-an-ethereum-transaction-e5c66bae0f6e)

简单来说就是
- 1、只有当交易被打包进区块时，他才是不可更改的
- 2、区块会优先打包gasprice更高的交易

所以当用户B在待打包处看到修改的交易时，可以通过构造更高gasprice的交易来竞争，将这笔交易打包到修改交易之前，就产生了问题。

以下代码就存在条件竞争的问题
```
function approve(address _spender, uint256 _value) public returns (bool success){
    allowance[msg.sender][_spender] = _value;
    return true
```


## 2、循环Dos问题

在以太坊代码中，循环是一种很常见的结构，但由于以太坊智能合约的特殊性，在循环也有很多需要特别注意的点，
存在潜在的合约问题与安全隐患。

### 1) 循环消耗问题

在以太坊中，每一笔交易都会消耗一定的gas，而交易的复杂度越高，则该交易的gasprice越高。而在区块链上，每个区块又有最大gas消耗值限制，且在矿工最优化收益方案中，如果一个交易的gas消耗过大，就会倾向性把这个交易排除在区块外，从而导致交易失败。

所以，对于合约内的循环次数不宜过大，在循环中的代码不宜过于复杂。
```
struct Payee {
    address addr;
    uint256 value;
}
Payee payees[];
uint256 nextPayeeIndex;

function payOut() {
    uint256 i = nextPayeeIndex;
    while (i < payees.length && msg.gas > 200000) {
      payees[i].addr.send(payees[i].value);
      i++;
    }
    nextPayeeIndex = i;
}
```

如果上述代码地址列表过长，就有可能导致交易失败。

2018年7月23日，seebug paper发表的[《首个区块链 token 的自动化薅羊毛攻击分析》](https://paper.seebug.org/646/)\[3\]中攻击合约就提到了这种gas优化方式。

### 2) 循环安全问题

在以太坊中，应该尽量避免循环次数受到用户控制，攻击者可能会使用过大的循环来完成Dos攻击。

```
function distribute(address[] addresses) onlyOwner {
    for (uint i = 0; i < addresses.length; i++) {
        // transfer code
    }
}
```

当攻击者通过不断添加address列表长度，来迫使该函数执行循环次数过多，导致合约无法正常维护，函数无法执行。

2016年，GovernMental合约代币被爆出恶意攻击\[4\]，导致地址列表过长无法执行，超过1100 ETH被困在了合约中。


# 三、漏洞影响范围

使用Haotian平台智能合约审计功能可以准确扫描到该类型问题。

![image.png-21.6kB][1]

基于Haotian平台智能合约审计功能规则，我们对全网的公开的共39548 个合约代码进行了扫描，其中共24791个合约涉及到这类问题。

## 1、 条件竞争

截止2018年8月10日为止，我们发现了22981个存在approve条件竞争的合约代码，其中15325个合约仍处于交易状态，其中交易量最高的10个合约情况如下：
![image.png-467.4kB][2]


## 2、 循环Dos问题

截止2018年8月10日为止，我们发现了1810个存在潜在循环dos问题的合约代码，其中1740个合约仍处于交易状态，其中交易量最高的10个合约情况如下：

![image.png-464.8kB][3]

# 四、修复方式


## 1）条件竞争

关于这个问题的修复方式讨论很多，由于这属于底层特性的问题，所以很难在智能合约层面做解决，在代码层面，我们建议在approve函数中加入
```
require((_value == 0) || (allowance[msg.sender][_spender] == 0));
```

将这个条件加入，在每次修改权限时，将额度修改为0，再将额度改为对应值。

在这种情况下，合约管理者可以通过日志或其他手段来判断是否有条件竞争发生，从风控的角度警醒合约管理者注意该问题的发生。范例代码如下：

```
    function approve(address _spender, uint256 _value) isRunning validAddress returns (bool success) {
        require(_value == 0 || allowance[msg.sender][_spender] == 0);
        allowance[msg.sender][_spender] = _value;
        Approval(msg.sender, _spender, _value);
        return true;
    }

```

## 2）循环Dos问题

在面临循环Dos问题产生的场景中，最为常见的就是向多个用户转账这个功能。

这里推荐代码中尽量避免用户可以控制循环深度，如果无法避免的话，尽量使用类似withdrawFunds这种函数，循环中只分发用户提币的权限，让用户来提取属于自己的代币，通过这种操作可以大幅度节省花费的gas开支，也可以一定程度避免可能导致的问题。代码如下所示：

```
function distribute(address[] addresses) onlyOwner {
    for (uint i = 0; i < addresses.length; i++) {
        if (address_claimed_tokens[addresses[i]] == 0) {
            balances[owner] -= transferAmount;
            balances[addresses[i]] += transferAmount;
            address_claimed_tokens[addresses[i]] += transferAmount;
            Transfer(owner, addresses[i], transferAmount);
        }
    }
}
```


# 五、一些思考

在分析了许多智能合约已有的漏洞以及合约以后，我发现有一类问题比较特殊，这些问题的诞生根本原因都是因为以太坊智能合约本身的设计缺陷，再加上开发者对此没有清晰的认识，导致了合约本身的一些隐患。

文章中提到的条件竞争是个比较特殊的问题，这里的条件竞争涉及到了智能合约底层实现逻辑，本身打包逻辑存在条件竞争，我们无法在代码层面避免这个问题，但对于开发者来说，比起无缘无故的因为该问题丢失代币来说，更重要的是合约管理者可以监控到每一笔交易的结果，所以我们加入置0的操作来提醒合约管理者、代币持有者该问题，尽量避免这样的操作发生。

而循环Dos问题就是一个针对开发者的问题，每一次操作就是一次交易，每次交易就要花费gas，交易越复杂花费的gas越多，而在区块链上，每个区块又有最大gas消耗值限制，且在矿工最优化收益方案中，如果一个交易的gas消耗过大，就会倾向性把这个交易排除在区块外，从而导致交易失败。这也就直接导致了在交易中，我们需要尽可能的优化gas花费，避免交易失败。

我们在对全网公开的合约代码进行扫描和监控时容易发现，有很大一批开发人员并没有注意到这些问题，其中条件竞争问题甚至影响广泛，有超过一半以上的公开代码都受到影响。

这里我们建议所有的开发者重新审视自己的合约代码，检查是否存在设计缺陷问题，避免不必要的麻烦以及安全问题。

# 六、REF

- \[1\] ERC20 API: An Attack Vector on Approve/TransferFrom Methods
[https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit](https://docs.google.com/document/d/1YLPtQxZu1UAvO9cZ1O2RPXBbT0mooh4DYKjA_jp-RLM/edit)

- \[2\] Life Cycle of an Ethereum Transaction
[https://medium.com/blockchannel/life-cycle-of-an-ethereum-transaction-e5c66bae0f6e](https://medium.com/blockchannel/life-cycle-of-an-ethereum-transaction-e5c66bae0f6e)

- \[3\] 首个区块链 token 的自动化薅羊毛攻击分析
[https://paper.seebug.org/646/](https://paper.seebug.org/646/)

- \[4\] GovernMental's 1100 ETH ：
[https://www.reddit.com/r/ethereum/comments/4ghzhv/governmentals_1100_eth_jackpot_payout_is_stuck/](https://www.reddit.com/r/ethereum/comments/4ghzhv/governmentals_1100_eth_jackpot_payout_is_stuck/)
 


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/bo0boybdj7nd5mgmt6jfs3tz/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/e51ihtpievj0strjnya86j4r/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/w2tvf02hnjezjceobziak6k6/image.png
