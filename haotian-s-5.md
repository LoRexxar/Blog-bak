---
title: “以太坊智能合约编码隐患”影响分析报告
date: 2018-11-08 15:58:11
tags:
---

经历了历时2个月伴随着HaoTian的开发，最后一份扫描总算完成了，HaoTian的雏形也基本完成了。在这篇扫描报告告一段落之后，会把之前攒的一些web思路发出来，好久没发web的文章了:>

<!--more-->

# 一、简介

在知道创宇404区块链安全研究团队整理输出的《知道创宇以太坊合约审计CheckList》中，我们把超过10个问题点归结为开发者容易忽略的问题隐患，其中包括“语法特性”、“数据私密性”、“数据可靠性”、“gas消耗优化”、“合约用户”、“日志记录”、“回调函数”、“Owner权限”、“用户鉴权”、 “条件竞争”等，统一归类为“以太坊智能合约编码隐患”。

“昊天塔(HaoTian)”是知道创宇404区块链安全研究团队独立开发的用于监控、扫描、分析、审计区块链智能合约安全自动化平台，目前已经集成了部分基于opcode的审计功能。我们利用该平台针对上述提到的《知道创宇以太坊合约审计CheckList》中“以太坊智能合约编码隐患”类问题在全网公开的智能合约代码做了扫描分析。详见下文：

# 二、漏洞详情

以太坊智能合约是以太坊概念中非常重要的一个概念，以太坊实现了基于solidity语言的以太坊虚拟机（Ethereum Virtual Machine），它允许用户在链上部署智能合约代码，通过智能合约可以完成人们想要的合约。

这次我们提到的问题多数属于智能合约独有问题，与我们常见的各类代码不同，在编写智能合约代码时还需要考虑多种问题。

## 1、语法特性

在智能合约中小心整数除法的向下取整问题

在智能合约中，所有的整数除法都会向下取整到最接近的整数，当我们需要更高的精度时，我们需要使用乘数来加大这个数字。

该问题如果在代码中显式出现，编译器会提出问题警告，无法继续编译，但如果隐式出现，将会采取向下取整的处理方式。

错误样例
```
uint x = 5 / 2; // 2
正确代码

uint multiplier = 10;
uint x = (5 * multiplier) / 2;
```

## 2、数据私密性

在合约中，所有的数据都是公开的。包括私有变量等，不得将任何带有私密性的数据储存在链上。


## 3、数据可靠性

在合约中，许多开发者习惯用时间戳来做判断条件，例如

```
uint someVariable = now + 1;

if (now % 2 == 0) { // now可能被矿工控制

}
```

now、block_timestamp会被矿工所控制，并不可靠。

## 4、gas消耗优化

```
contract EUXLinkToken is ERC20 {
    using SafeMath for uint256;
    address owner = msg.sender;
    mapping (address => uint256) balances;
    mapping (address => mapping (address => uint256)) allowed;
    mapping (address => bool) public blacklist;
    string public constant name = "xx";
    string public constant symbol = "xxx";
    uint public constant decimals = 8;
    uint256 public totalSupply = 1000000000e8;
    uint256 public totalDistributed = 200000000e8;
    uint256 public totalPurchase = 200000000e8;
    uint256 public totalRemaining = totalSupply.sub(totalDistributed).sub(totalPurchase);
    uint256 public value = 5000e8;
    uint256 public purchaseCardinal = 5000000e8;
    uint256 public minPurchase = 0.001e18;
    uint256 public maxPurchase = 10e18;
```

在合约中，涉及到状态变化的代码会消耗更多的，为了经可能优化gas消耗，对于不涉及状态变化的变量应该加constant来限制

## 5、合约用户

合约中，交易目标可能为合约，因此可能会产生的各种恶意利用。

```
contract Auction{
    address public currentLeader;
    uint256 public hidghestBid;
    function bid() public payable {
        require(msg.value > highestBid);
        require(currentLeader.send(highestBid));
        currentLeader = msg.sender;
        highestBid = currentLeader;
    }
}
```

上述合约就是一个典型的没有考虑合约为用户时的情况，这是一个简单的竞拍争夺王位的代码。当交易ether大于合约内的highestBid，当前用户就会成为合约当前的"王"，他的交易额也会成为新的highestBid。

```
contract Attack {
    function () { revert(); }
    function Attack(address _target) payable {
        _target.call.value(msg.value)(bytes4(keccak256("bid()")));
    }
}
```

但当新的用户试图成为新的“王”时，当代码执行到`require(currentLeader.send(highestBid));`时，合约中的fallback函数会触发，如果攻击者在fallback函数中加入revert()函数，那么交易就会返回false，即永远无法完成交易，那么当前合约就会一直成为合约当前的"王"。


## 6、日志记录

当合约跑在链上之后，链上的一切数据都难以监控，对于一个健康的智能合约来说，记录合理的event，为了便于运维监控，除了转账，授权等函数以外，其他操作也需要加入详细的事件记录，如转移管理员权限、其他特殊的主功能。

```
fonction transferOwnership(address newOwner) onlyOwner public {
    ownner = newOwner;
    emit OwnershipTransferred(owner, newowner);
    }
```


## 7、回调函数

fallback机制是基于智能合约的特殊性而存在的。对于智能合约来说，任何函数的执行都是通过交易来完成的，但函数的执行过程中可能会遇到各种各样的问题，在交易失败或者交易结束后，就会执行fallback来最后处理结果和返回。

而在合约交易中，执行的每一个操作都会花费巨大的gas，如果gas不足，那么fallback函数也会执行失败。在evm中规定，交易失败时，只有2300gas用于执行fallback函数，而2300gas只允许执行一组字节码指令。一旦遇到极端情况，可能会因为gas不够用导致某种情况发生，导致未知的不可挽回的后果。

例如
```
function() payable { LogDepositReceived(msg.sender); }
function() public payable{ revert();};
```

## 8、Owner权限

避免owner权限过大

部分合约owner权限过大，owner可以随意操作合约内各种数据，包括修改规则，任意转账，任意铸币烧币，一旦发生安全问题，可能会导致严重的结果。

```
  function destroy() onlyOwner public onlyOwner{
    selfdestruct(owner);
  }
```

## 9、用户鉴权问题

合约中不要使用tx.origin做鉴权

tx.origin代表最初始的地址，如果用户a通过合约b调用了合约c，对于合约c来说，tx.origin就是用户a，而msg.sender才是合约b，对于鉴权来说，这是十分危险的，这代表着可能导致的钓鱼攻击。

下面是一个范例:

```
pragma solidity >0.4.24;
// THIS CONTRACT CONTAINS A BUG - DO NOT USE
contract TxUserWallet {
    address owner;
    constructor() public {
        owner = msg.sender;
    }
    function transferTo(address dest, uint amount) public {
        require(tx.origin == owner);
        dest.transfer(amount);
    }
}
```

我们可以构造攻击合约

```
pragma solidity >0.4.24;
interface TxUserWallet {
    function transferTo(address dest, uint amount) external;
}
contract TxAttackWallet {
    address owner;
    constructor() public {
        owner = msg.sender;
    }
    function() external {
        TxUserWallet(msg.sender).transferTo(owner, msg.sender.balance);
    }
}
```

当用户被欺骗调用攻击合约，则会直接绕过鉴权而转账成功，这里应使用msg.sender来做权限判断。

[https://solidity.readthedocs.io/en/develop/security-considerations.html#tx-origin](https://solidity.readthedocs.io/en/develop/security-considerations.html#tx-origin)

## 10、条件竞争

在智能合约中，经常容易出现对交易顺序的依赖，如占山为王规则、或最后一个赢家规则。都是对交易顺序有比较强的依赖的设计规则，但以太坊本身的底层规则是基于矿工利益最大法则，在一定程度的极限情况下，只要攻击者付出足够的代价，他就可以一定程度控制交易的顺序。开发者应避免这个问题。

真实世界事件

[智能合约游戏之殇——类 Fomo3D 攻击分析](https://paper.seebug.org/681/)


#三、漏洞影响范围

使用Haotian平台智能合约审计功能可以准确扫描到该类型问题。

![image.png-60.4kB][1]

基于Haotian平台智能合约扫描功能规则，我们对全网的公开的共47305个合约代码进行了扫描。

其中存在数据可靠问题的合约共2732个，
存在int型变量gas优化问题的合约共18285个，
存在string型变量gas优化问题的合约共194个，
存在Owner权限过大或合约后门的合约共1194个，
存在tx.origin 鉴权问题问题的合约共52个。


## 1、数据可靠性

截止2018年10月31日，我们发现了2732个存在数据可靠问题的合约代码，存在潜在的安全隐患。其中交易量最高的10个合约情况如下：

![image.png-428.4kB][2]

## 2、gas消耗优化

截止2018年10月31日，我们发现了18285个存在int型变量gas优化问题的合约代码，存在潜在的安全隐患。其中交易量最高的10个合约情况如下：

![image.png-380.7kB][3]

截止2018年10月31日，我们发现了194个存在string型变量gas优化问题的合约代码，存在潜在的安全隐患。其中交易量最高的10个合约情况如下：

![image.png-385.9kB][4]

## 3、回调函数

截止2018年10月31日，我们发现了8321个存在复杂回调的合约代码，存在潜在的安全隐患。其中交易量最高的10个合约情况如下：

![image.png-401.9kB][5]

## 4、Owner权限

截止2018年10月31日，我们发现了1194个存在Owner权限过大或合约后门，其中交易量最高的10个合约情况如下：

![image.png-366.3kB][6]


## 5、tx.origin 鉴权问题

截止2018年10月31日，我们发现了52个存在tx.origin 鉴权问题，其中交易量最高的10个合约情况如下：

![image.png-365.8kB][7]

# 四、修复方式

## 1、语法特性

在智能合约中小心整数除法的向下取整问题，可以通过先乘积为整数再做处理。
```
uint multiplier = 10;
uint x = (5 * multiplier) / 2;
```

## 2、数据私密问题

在处理一些隐私数据是尽量保留在服务端，可以通过hash-commit的方式来check变量值。


## 3、数据可靠性

尽量使合约内容不依赖时间顺序，如果需要外部变量影响，那尽量采用block.height和block.hash等这类难以控制的变量。

## 4、gas消耗优化

对于某些不涉及状态变化的函数和变量可以加constant来避免gas的消耗

## 5、合约用户

合约中，应尽量考虑交易目标为合约时的情况，避免因此产生的各种恶意利用。

## 6、日志记录

关键事件应有Event记录，为了便于运维监控，除了转账，授权等函数以外，其他操作也需要加入详细的事件记录，如转移管理员权限、其他特殊的主功能。

```
fonction transferOwnership(address newOwner) onlyOwner public {
    ownner = newOwner;
    emit OwnershipTransferred(owner, newowner);
    }
```

## 7、回调函数

合约中定义Fallback函数，并使Fallback函数尽可能的简单。尽量避免在回调函数中调用transfer、call等涉及状态变化的操作，避免gas不够用直接导致未知情况发生。

## 8、Owner权限问题

部分合约owner权限过大，owner可以随意操作合约内各种数据，包括修改规则，任意转账，任意铸币烧币，一旦发生安全问题，可能会导致严重的结果。

关于owner权限问题，应该遵循几个要求： 
1、合约创造后，任何人不能改变合约规则，包括规则参数大小等 
2、只允许owner在合约销毁前，从合约中提取余额
3、owner不能在未限制的情况下操作其他用户的余额等

## 9、用户鉴权

在需要用户鉴权的时刻，尽量使用msg.sender作为目标方。
[https://solidity.readthedocs.io/en/develop/security-considerations.html#tx-origin](https://solidity.readthedocs.io/en/develop/security-considerations.html#tx-origin)

## 10、条件竞争

在智能合约的设计中，避免对交易顺序的依赖，或者想办法强制要求交易顺序。


# 五、一些思考

在这一次整理合约编码隐患的过程中，对智能合约本身的特殊性进行了深入了解。和每个语言一样，智能合约有基于区块链这个大前提在，许多代码都出现了新的问题，如果开发者没有注意到这些隐患，一旦出现问题，这些隐患就可能导致更大的问题发生。

截止2018年10月31日，以太坊合约审计Checklist的所以问题完成了第一轮扫描，第一轮扫描针对以太坊公开的所有合约，其中超过80%的智能合约存在1个以上的安全隐患问题。在接下来的扫描报告中，我们会公开《以太坊合约审计Checklist》并使用HaoTian对以太坊公链上的所有智能合约进行基于opcode的扫描分析。


  [1]: http://static.zybuluo.com/LoRexxar/cglm3fw5hgo8v3yxlw2e9030/image.png
  [2]: http://static.zybuluo.com/LoRexxar/lgqxp0s6918fy0cnycirpld1/image.png
  [3]: http://static.zybuluo.com/LoRexxar/t9cc1ex2st3asqyv9ohqz5ce/image.png
  [4]: http://static.zybuluo.com/LoRexxar/gu1fsetyt2gunyzt0g987i10/image.png
  [5]: http://static.zybuluo.com/LoRexxar/pmnh8qg8rivp13j6xu27egh4/image.png
  [6]: http://static.zybuluo.com/LoRexxar/9m8dxvzfo8a22cm2qv605kfp/image.png
  [7]: http://static.zybuluo.com/LoRexxar/th4gf0bmpdyvm5j1pc2rzjhi/image.png
