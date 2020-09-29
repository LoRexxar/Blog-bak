---
title: “以太坊智能合约编码安全问题”影响分析报告
date: 2018-09-06 14:28:06
tags:
- 智能合约
- checklist
---

这篇扫描报告其实算是一片比较特殊的文章，因为这是checklist唯一被我直接标记为安全问题的一类，其中很多问题虽然特征明显，但修复逻辑却千变万化，所以统计数据一直波动很大，犹豫了很久才发出来，图片的很多涉及到的合约并不一定真的存在问题，但仍然值得注意。

智能合约checklist系列文章:

[“以太坊智能合约规范问题”影响分析报告](https://lorexxar.cn/2018/08/14/haotian-s-1/)
[“以太坊智能合约设计缺陷问题”影响分析报告](https://lorexxar.cn/2018/08/22/haotian-s-2/)

<!--more-->


# 一、简介

在知道创宇404区块链安全研究团队整理输出的《知道创宇以太坊合约审计CheckList》中，把“溢出问题”、“重入漏洞”、“权限控制错误”、“重放攻击”等问题统一归类为“以太坊智能合约编码安全问题”。

“昊天塔(HaoTian)”是知道创宇404区块链安全研究团队独立开发的用于监控、扫描、分析、审计区块链智能合约安全自动化平台。我们利用该平台针对上述提到的《知道创宇以太坊合约审计CheckList》中“以太坊智能合约编码安全”类问题在全网公开的智能合约代码做了扫描分析。详见下文：

# 二、漏洞详情
## 1、溢出问题

以太坊Solidity设计之初就被定位为图灵完备性语言。在solidity的设计中，支持int/uint变长的有符号或无符号整型。变量支持的步长以8递增，支持从uint8到uint256，以及int8到int256。需要注意的是，uint和int默认代表的是uint256和int256。uint8的数值范围与C中的uchar相同，即取值范围是0到2^8-1，uint256支持的取值范围是0到2^256-1。而当对应变量值超出这个范围时，就会溢出至符号位，导致变量值发生巨大的变化。

### (1) 算数溢出

在Solidity智能合约代码中，在余额的检查中如果直接使用了加减乘除没做额外的判断时，就会存在算术溢出隐患

```
contract MyToken {
    mapping (address => uint) balances;
        
    function balanceOf(address _user) returns (uint) { return balances[_user]; }
    function deposit() payable { balances[msg.sender] += msg.value; }
    function withdraw(uint _amount) {
        require(balances[msg.sender] - _amount > 0);  // 存在整数溢出
        msg.sender.transfer(_amount);
        balances[msg.sender] -= _amount;
    }
}
```

在上述代码中，由于没有校验`_amount`一定会小于`balances[msg.sender]`，所以攻击者可以通过传入超大数字导致溢出绕过判断，这样就可以一口气转走巨额代币。

2018年4月24日，SMT/BEC合约被恶意攻击者转走了50,659,039,041,325,800,000,000,000,000,000,000,000,000,000,000,000,000,000,000个SMT代币。恶意攻击者就是利用了[SMT/BEC合约的整数溢出漏洞](https://www.anquanke.com/post/id/106382)导致了这样的结果。

2018年5月19日，以太坊Hexagon合约代币被公开存在[整数溢出漏洞](https://www.anquanke.com/post/id/145520)。

### (2) 铸币烧币溢出问题

作为一个合约代币的智能合约来说，除了有其他合约的功能以外，还需要有铸币和烧币功能。而更特殊的是，这两个函数一般都为乘法或者指数交易，很容易造成溢出问题。

```
function TokenERC20(
    uint256 initialSupply,
    string tokenName,
    string tokenSymbol
) public {
    totalSupply = initialSupply * 10 ** uint256(decimals);  
    balanceOf[msg.sender] = totalSupply;                
    name = tokenName;                                   
    symbol = tokenSymbol;                               
}
```

上述代码未对代币总额做限制，会导致指数算数上溢。

2018年6月21日，Seebug Paper公开了一篇关于整数溢出漏洞的分析文章[ERC20 智能合约整数溢出系列漏洞披露](https://paper.seebug.org/626/)，里面提到很多关于指数上溢的漏洞样例。

## 2、call注入

Solidity作为一种用于编写以太坊智能合约的图灵完备的语言，除了常见语言特性以外，还提供了调用/继承其他合约的功能。在`call`、`delegatecall`、`callcode`三个函数来实现合约之间相互调用及交互。正是因为这些灵活各种调用，也导致了这些函数被合约开发者“滥用”，甚至“肆无忌惮”提供任意调用“功能”，导致了各种安全漏洞及风险。

```
function withdraw(uint _amount) {
    require(balances[msg.sender] >= _amount);
    msg.sender.call.value(_amount)();
    balances[msg.sender] -= _amount;
}
```

上述代码就是一个典型的存在call注入问题直接导致重入漏洞的demo。

2016年7月，[The DAO](https://en.wikipedia.org/wiki/The_DAO_%28organization%29)被攻击者使用重入漏洞取走了所有代笔，损失超过60亿，直接导致了eth的硬分叉，影响深远。

2017年7月20日，Parity Multisig电子钱包版本1.5+的漏洞被发现，使得攻击者从三个高安全的多重签名合约中窃取到超过15万ETH ，其事件原因是由于未做限制的 delegatecall 函数调用了合约初始化函数导致合约拥有者被修改。

2018年6月16日，「隐形人真忙」在先知大会上分享了[「智能合约消息调用攻防」](https://paper.seebug.org/624/)的议题，其中提到了一种新的攻击场景——call注⼊，主要介绍了利用对call调用处理不当，配合一定的应用场景的一种攻击手段。接着于 2018年6月20日，ATN代币团队发布「ATN抵御黑客攻击的报告」，报告指出黑客利用call注入攻击漏洞修改合约拥有者，然后给自己发行代币，从而造成 ATN 代币增发。

2018年6月26日，知道创宇区块链安全研究团队在Seebug Paper上公开了[《以太坊 Solidity 合约 call 函数簇滥用导致的安全风险》](https://paper.seebug.org/633/)。

## 3、权限控制错误

在智能合约中，合约开发者一般都会设置一些用于合约所有者，但如果开发者疏忽写错了函数权限，就有可能导致所有者转移等严重后果。

```
function initContract() public {
    owner = msg.reader;
}
```

上述代码函数就需要设置onlyOwner。

## 4、重放攻击

2018年，DEFCON26上来自 360 独角兽安全团队(UnicornTeam)的 Zhenzuan Bai, Yuwei Zheng 等分享了议题[《Your May Have Paid More than You Imagine：Replay Attacks on Ethereum Smart Contracts》](https://github.com/nkbai/defcon26/blob/master/docs/Replay%20Attacks%20on%20Ethereum%20Smart%20Contracts.md)

在攻击中提出了智能合约中比较特殊的委托概念。

在资产管理体系中，常有委托管理的情况，委托人将资产给受托人管理，委托人支付一定的费用给受托人。这个业务场景在智能合约中也比较普遍。

这里举例子为transferProxy函数，该函数用于当user1转token给user3，但没有eth来支付gasprice，所以委托user2代理支付，通过调用transferProxy来完成。

```
function transferProxy(address _from, address _to, uint256 _value, uint256 _fee,
    uint8 _v, bytes32 _r, bytes32 _s) public returns (bool){
    if(balances[_from] < _fee + _value 
        || _fee > _fee + _value) revert();
    uint256 nonce = nonces[_from];
    bytes32 h = keccak256(_from,_to,_value,_fee,nonce,address(this));
    if(_from != ecrecover(h,_v,_r,_s)) revert();
    if(balances[_to] + _value < balances[_to]
        || balances[msg.sender] + _fee < balances[msg.sender]) revert();
    balances[_to] += _value;
    emit Transfer(_from, _to, _value);
    balances[msg.sender] += _fee;
    emit Transfer(_from, msg.sender, _fee);
    balances[_from] -= _value + _fee;
    nonces[_from] = nonce + 1;
    return true;
}
```
上述代码nonce值可以被预测，而其他变量不变的情况下，可以通过重放攻击来多次转账。

#三、漏洞影响范围

使用Haotian平台智能合约审计功能可以准确扫描到该类型问题。

![image.png-11.8kB][1]

基于Haotian平台智能合约审计功能规则，我们对全网的公开的共42538个合约代码进行了扫描，其中共1852个合约涉及到这类问题。

## 1、溢出问题

截止2018年9月5日，我们发现了391个存在算数溢出问题的合约代码，其中332个仍处于交易状态，其中交易量最高的10个合约情况如下：
![image.png-396kB][2]

截止2018年9月5日，我们发现了1636个存在超额铸币销币问题的合约代码，其中1364个仍处于交易状态，其中交易量最高的10个合约情况如下：

![image.png-458.4kB][3]

## 2、call注入

截止2018年9月5日，我们发现了204个存在call注入问题的合约代码，其中140个仍处于交易状态，其中交易量最高的10个合约情况如下：

![image.png-359.4kB][4]

# 3、重放攻击

截止2018年9月5日，我们发现了18个存在重放攻击隐患问题的合约代码，其中16个仍处于交易状态，其中交易量最高的10个合约情况如下：

![image.png-411.7kB][5]

# 四、修复方式

## 1、溢出问题

### 1) 算术溢出问题 

在调用加减乘除时，通常的修复方式都是使用openzeppelin-safeMath，但也可以通过对不同变量的判断来限制，但很难对乘法和指数做什么限制。
```
function transfer(address _to, uint256 _amount)  public returns (bool success) {
    require(_to != address(0));
    require(_amount <= balances[msg.sender]);
    balances[msg.sender] = balances[msg.sender].sub(_amount);
    balances[_to] = balances[_to].add(_amount);
    emit Transfer(msg.sender, _to, _amount);
    return true;
}
```

### 2）铸币烧币溢出问题

铸币函数中，应对totalSupply设置上限，避免因为算术溢出等漏洞导致恶意铸币增发。

在铸币烧币加上合理的权限限制可以有效减少该问题危害。
```
contract OPL {
    // Public variables
    string public name;
    string public symbol;
    uint8 public decimals = 18; // 18 decimals
    bool public adminVer = false;
    address public owner;
    uint256 public totalSupply;
    function OPL() public {
        totalSupply = 210000000 * 10 ** uint256(decimals);      
        ...                                 
}
```

## 2、call注入

call函数调用时，应该做严格的权限控制，或直接写死call调用的函数。避免call函数可以被用户控制。

在可能存在重入漏洞的代码中，经可能使用transfer函数完成转账，或者限制call执行的gas，都可以有效的减少该问题的危害。

```
contract EtherStore {

    // initialise the mutex
    bool reEntrancyMutex = false;
    uint256 public withdrawalLimit = 1 ether;
    mapping(address => uint256) public lastWithdrawTime;
    mapping(address => uint256) public balances;

    function depositFunds() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdrawFunds (uint256 _weiToWithdraw) public {
        require(!reEntrancyMutex);
        require(balances[msg.sender] >= _weiToWithdraw);
        // limit the withdrawal
        require(_weiToWithdraw <= withdrawalLimit);
        // limit the time allowed to withdraw
        require(now >= lastWithdrawTime[msg.sender] + 1 weeks);
        balances[msg.sender] -= _weiToWithdraw;
        lastWithdrawTime[msg.sender] = now;
        // set the reEntrancy mutex before the external call
        reEntrancyMutex = true;
        msg.sender.transfer(_weiToWithdraw);
        // release the mutex after the external call
        reEntrancyMutex = false; 
    }
 }
```

上述代码是一种用互斥锁来避免递归防护方式。

## 3、权限控制错误

合约中不同函数应设置合理的权限

检查合约中各函数是否正确使用了public、private等关键词进行可见性修饰，检查合约是否正确定义并使用了modifier对关键函数进行访问限制，避免越权导致的问题。

```
function initContract() public OnlyOwner {
    owner = msg.reader;
}
```

## 4、重放攻击

合约中如果涉及委托管理的需求，应注意验证的不可复用性，避免重放攻击。

其中主要的两点在于：
1、避免使用transferProxy函数。采用更靠谱的签名方式签名。
2、nonce机制其自增可预测与这种签名方式违背，导致可以被预测。尽量避免nonce自增。

# 五、一些思考

在完善智能合约审计checklist时，我选取了一部分问题将其归为编码安全问题，这类安全问题往往是开发者疏忽导致合约代码出现漏洞，攻击者利用代码中的漏洞来攻击，往往会导致严重的盗币事件。

在我们使用HaoTian对全网的公开合约进行扫描和监控时，我们发现文章中提到的几个问题涉及到的合约较少。由于智能合约代码公开透明的特性，加上这类问题比较容易检查出，一旦出现就会导致对合约的毁灭性打击，所以大部分合约开发人员都会注意到这类问题。但在不容易被人们发现的未公开合约中，或许还有大批潜在的问题存在。

这里我们建议所有的开发者重新审视自己的合约代码，检查是否存在编码安全问题，避免不必要的麻烦或严重的安全问题。


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/tub5ug8lyrsdyex667gqi1kb/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/sdmjneivtg8sfnmbdgmvyllb/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/mw0nliru74uoj89glmx607b5/image.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/x89qplere2imehe4pwgqbjmt/image.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/v2b81fxsfk3z2r2kslrq2ulb/image.png
