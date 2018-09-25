---
title: “以太坊智能合约编码设计问题”影响分析报告
date: 2018-09-25 15:33:00
tags:
---

以太坊智能合约是以太坊概念中非常重要的一个概念，以太坊实现了基于solidity语言的以太坊虚拟机（Ethereum Virtual Machine），它允许用户在链上部署智能合约代码，通过智能合约可以完成人们想要的合约。

这次我们提到的编码设计问题就和EVM底层的设计有很大的关系，由于EVM的特性，智能合约有很多与其他语言不同的特性，当开发者没有注意到这些问题时，就容易出现潜在的问题。

智能合约checklist系列文章:

[“以太坊智能合约规范问题”影响分析报告](https://lorexxar.cn/2018/08/14/haotian-s-1/)
[“以太坊智能合约设计缺陷问题”影响分析报告](https://lorexxar.cn/2018/08/22/haotian-s-2/)
[“以太坊智能合约编码安全问题”影响分析报告](https://lorexxar.cn/2018/09/06/haotian-s-3/)

<!--more-->



# 一、简介

在知道创宇404区块链安全研究团队整理输出的《知道创宇以太坊合约审计CheckList》中，把“地址初始化问题”、“判断函数问题”、“余额判断问题”、“转账函数问题”、“代码外部调用设计问题”、“错误处理”、“弱随机数问题”等问题统一归类为“以太坊智能合约编码设计问题”。

“昊天塔(HaoTian)”是知道创宇404区块链安全研究团队独立开发的用于监控、扫描、分析、审计区块链智能合约安全自动化平台。我们利用该平台针对上述提到的《知道创宇以太坊合约审计CheckList》中“以太坊智能合约编码设计”类问题在全网公开的智能合约代码做了扫描分析。详见下文：

# 二、漏洞详情

以太坊智能合约是以太坊概念中非常重要的一个概念，以太坊实现了基于solidity语言的以太坊虚拟机（Ethereum Virtual Machine），它允许用户在链上部署智能合约代码，通过智能合约可以完成人们想要的合约。

这次我们提到的编码设计问题就和EVM底层的设计有很大的关系，由于EVM的特性，智能合约有很多与其他语言不同的特性，当开发者没有注意到这些问题时，就容易出现潜在的问题。

## 1、地址初始化问题

在EVM中，所有与地址有关的初始化时，都会赋予初值0。

如果一个address变量与0相等时，说明该变量可能未初始化或出现了未知的错误。

如果开发者在代码中初始化了某个address变量，但未赋予初值，或用户在发起某种操作时，误操作未赋予address变量，但在下面的代码中需要对这个变量做处理，就可能导致不必要的安全风险。

## 2、判断函数问题

在智能合约中，有个很重要的校验概念。下面这种问题的出现主要是合约代币的内部交易。

但如果在涉及到关键判断（如余额判断）等影响到交易结果时，当交易发生错误，我们需要对已经执行的交易结果进行回滚，而EVM不会检查交易函数的返回结果。如果我们使用return false，EVM是无法获取到这个错误的，则会导致在之前的文章中提到的[假充值问题](https://paper.seebug.org/663/#3)。

在智能合约中，我们需要抛出这个错误，这样EVM才能获取到错误触发底层的revert指令回滚交易。

而在solidity扮演这一角色的，正是require函数。而有趣的是，在solidity中，还有一个函数叫做assert，和require不同的是，它底层对应的是空指令，EVM执行到这里时就会报错退出，不会触发回滚。

转化到直观的交易来看，如果我们使用assert函数校验时，assert会消耗掉所有剩余的gas。而require会触发回滚操作。

assert在校验方面展现了强一致性，除了对固定变量的检查以外，require更适合这种情况下的使用。


## 3、余额判断问题

在智能合约中，经常会出现对用户余额的判断，尤其是账户初建时，许多合约都会对以合约创建时余额为0来判断合约的初建状态，这是一种错误的行为。

在智能合约中，永远无法阻止别人向你的强制转账，即使fallback函数throw也不可以。攻击者可以创建带有余额的新合约，然后调用`selfdestruct(victimAddress)`销毁，这样余额就会强制转移给目标，在这个过程中，不会调用目标合约的代码，所以无法从代码层面阻止。

值得注意的是，在打包的过程中，攻击者可以通过条件竞争来在合约创建前转账，这样在合约创建时余额就为0了。


## 4、转账函数问题

在智能合约中，涉及到转账的操作最常见不过了。而在solidity中，提供了两个函数用于转账tranfer/send。

当tranfer/send函数的目标是合约时，会调用合约内的fallback函数。但当fallback函数执行错误时，transfer函数会抛出错误并回滚，而send则会返回false。如果在使用send函数交易时，没有及时做判断，则可能出现转账失败却余额减少的情况。

```
function withdraw(uint256 _amount) public {
    require(balances[msg.sender] >= _amount);
    balances[msg.sender] -= _amount;
    etherLeft -= _amount;
    msg.sender.send(_amount);  
}
```

上面给出的代码中使用 send() 函数进行转账，因为这里没有验证 send() 返回值，如果msg.sender 为合约账户 fallback() 调用失败，则 send() 返回false，最终导致账户余额减少了，钱却没有拿到。

## 5、代码外部调用设计问题

在智能合约的设计思路中，有一个很重要的概念为外部调用。或是调用外部合约，又或是调用其它账户。这在智能合约的设计中是个很常见的思路，最常见的便是转账操作，就是典型的外部调用。

但外部调用本身就是一个容易发生错误的操作，谁也不能肯定在和外部合约/用户交互时能确保顺利，举一个合约代币比较常见的例子
```
contract auction {
    address highestBidder;
    uint highestBid;
    function bid() payable {
        if (msg.value < highestBid) throw;
        if (highestBidder != 0) {
            if (!highestBidder.send(highestBid)) { // 可能会发生错误
                throw;
            }
        }
       highestBidder = msg.sender;
       highestBid = msg.value;
    }
}
```

上述代码当转账发生错误时可能会导致进一步其他的错误，如果碰到循环调用bid函数时，更可能导致循环到中途发生错误，在之前提到的[ddos优化问题](https://paper.seebug.org/679/#1_1)中，这也是一个很典型的例子。

而这就是一个典型的push操作，指合约主动和外部进行交互，这种情况容易出现问题是难以定位难以弥补，导致潜在的问题。

## 6、错误处理

智能合约中，有一些涉及到address底层操作的方法
```
address.call()
address.callcode()
address.delegatecall()
address.send()
```

他们都有一个典型的特点，就是遇到错误并不会抛出错误，而是会返回错误并继续执行。

且作为EVM设计的一部分，下面这些函数如果调用的合约不存在，将会返回True。如果合约开发者没有注意到这个问题，那么就有可能出现问题。

```
call、delegatecall、callcode、staticcall
```

[http://rickgray.me/2018/05/26/ethereum-smart-contracts-vulnerabilities-review-part2/#4-Unchecked-Return-Values-For-Low-Level-Calls](http://rickgray.me/2018/05/26/ethereum-smart-contracts-vulnerabilities-review-part2/#4-Unchecked-Return-Values-For-Low-Level-Calls)

# 7、弱随机数问题

智能合约是借助EVM运行，跑在区块链上的合约代码。其最大的特点就是公开和不可篡改性。而如何在合约上生成随机数就成了一个大问题。

Fomo3D合约在空投奖励的随机数生成中就引入了block信息作为随机数种子生成的参数，导致随机数种子只受到合约地址影响，无法做到完全随机。

```
function airdrop()
    private 
    view 
    returns(bool)
{
    uint256 seed = uint256(keccak256(abi.encodePacked(
        (block.timestamp).add
        (block.difficulty).add
        ((uint256(keccak256(abi.encodePacked(block.coinbase)))) / (now)).add
        (block.gaslimit).add
        ((uint256(keccak256(abi.encodePacked(msg.sender)))) / (now)).add
        (block.number)
    )));
    if((seed - ((seed / 1000) * 1000)) < airDropTracker_)
        return(true);
    else
        return(false);
}
```

上述这段代码直接导致了Fomo3d薅羊毛事件的诞生。真实世界损失巨大，超过数千eth。

[8万笔交易「封死」以太坊网络，只为抢夺Fomo3D大奖？](https://mp.weixin.qq.com/s/5nrgj8sIZ0SlXebG5sWVPw)
[Last Winner ](https://paper.seebug.org/672/)


#三、漏洞影响范围

使用Haotian平台智能合约审计功能可以准确扫描到该类型问题。

![image.png-17kB][1]

基于Haotian平台智能合约扫描功能规则，我们对全网的公开的共42538个合约代码进行了扫描，其中35107个合约存在地址初始化问题，4262个合约存在判断函数问题，173个合约存在余额判断问题，930个合约存在转账函数问题， 349个合约存在弱随机数问题，2300个合约调用了block.timestamp，过半合约涉及到这类安全风险。

## 1、地址初始化问题

截止2018年9月21日，我们发现了35107个存在地址初始化问题的合约代码，存在潜在的安全隐患。

## 2、判断函数问题

截止2018年9月21日，我们发现了4262个存在判断函数问题的合约代码，存在潜在的安全隐患。

## 3、余额判断问题

截止2018年9月21日，我们发现了173个存在余额判断问题的合约代码，其中165个仍处于交易状态，其中交易量最高的10个合约情况如下：

![image.png-410.1kB][2]

## 4、转账函数问题

截止2018年9月21日，我们发现了930个存在转账函数问题的合约代码，其中873个仍处于交易状态，其中交易量最高的10个合约情况如下：

![image.png-437.2kB][3]

## 5、弱随机数问题

截止2018年9月21日，我们发现了349个存在弱随机数问题的合约代码，其中272个仍处于交易状态，其中交易量最高的10个合约情况如下：
![image.png-476.6kB][4]

截止2018年9月21日，我们发现了2300个存在调用了block.timestamp的合约代码，其中2123个仍处于交易状态，其中交易量最高的10个合约情况如下：

![image.png-244.1kB][5]

# 四、修复方式

## 1、地址初始化问题

涉及到地址的函数中，建议加入require(_to!=address(0))验证，有效避免用户误操作或未知错误导致的不必要的损失

## 2、判断函数问题

对于正常的判断来说，优先使用`require`来判断结果。

而对于固定变量的检查，使用assert函数可以避免一些未知的问题，因为他会强制终止合约并使其无效化，在一些固定条件下，assert更适用


## 3、余额判断问题

不要在合约任何地方假设合约的余额，尤其是不要通过创建时合约为0来判断合约初建状态，攻击者可以使用多种方式强制转账。


## 4、转账函数问题

在完成交易时，默认推荐使用transfer函数而不是send完成交易。


## 5、代码外部调用设计问题

对于外部合约优先使用pull而不是push。如上述的转账函数，可以通过赋予提取权限来将主动行为转换为被动行为

```
contract auction {
    address highestBidder;
    uint highestBid;
    mapping(address => uint) refunds;
    function bid() payable external {
        if (msg.value < highestBid) throw;
        if (highestBidder != 0) {
            refunds[highestBidder] += highestBid; // 记录在refunds中
        }
        highestBidder = msg.sender;
        highestBid = msg.value;
    }
    function withdrawRefund() external {
        uint refund = refunds[msg.sender];
        refunds[msg.sender] = 0;
        if (!msg.sender.send(refund)) {
            refunds[msg.sender] = refund; // 如果转账错误还可以挽回
        }
    }
}
```

通过构建withdraw来使用户来执行合约将余额取出。

## 6、错误处理

合约中涉及到call等在address底层操作的方法时，做好合理的错误处理

```
if(!someAddress.send(55)) {
    // Some failure code
}
```
包括目标合约不存在时，也同样需要考虑。

## 7、弱随机数问题

智能合约上随机数生成方式需要更多考量

在合约中关于这样的应用时，考虑更合适的生成方式和合理的利用顺序非常重要。

这里提供一个比较合理的随机数生成方式**hash-commit-reveal**，即玩家提交行动计划，然后行动计划hash后提交给后端，后端生成相应的hash值，然后生成对应的随机数reveal，返回对应随机数commit。这样，服务端拿不到行动计划，客户端也拿不到随机数。

有一个很棒的实现代码是[dice2win](https://etherscan.io/address/0xD1CEeeefA68a6aF0A5f6046132D986066c7f9426)的随机数生成代码。

当然**hash-commit**在一些简单场景下也是不错的实现方式。即玩家提交行动计划的hash，然后生成随机数，然后提交行动计划。



# 五、一些思考

在探索智能合约最佳实践的过程中，逐渐发现，在智能合约中有很多只有智能合约才会出现的问题，这些问题大多都是因为EVM的特殊性而导致的特殊特性，但开发者并没有对这些特性有所了解，导致很多的潜在安全问题诞生。

我把这一类问题归结为编码设计问题，开发者可以在编码设计阶段注意这些问题，可以避免大多数潜在安全问题。


  [1]: http://static.zybuluo.com/LoRexxar/7jrgm6rgesqnj0lk7vu3d0n3/image.png
  [2]: http://static.zybuluo.com/LoRexxar/f7mv3abosn82swuavmxt9b9q/image.png
  [3]: http://static.zybuluo.com/LoRexxar/rvrppg5fwoysxqqtt1x763is/image.png
  [4]: http://static.zybuluo.com/LoRexxar/driyeydgf2m51vfaazjwiktr/image.png
  [5]: http://static.zybuluo.com/LoRexxar/2dhc8l1t7tbh46nw1dgm9hb2/image.png
