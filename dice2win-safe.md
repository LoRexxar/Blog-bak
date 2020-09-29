---
title: 智能合约游戏之殇——Dice2win安全分析
date: 2018-10-18 15:33:44
tags:
- 智能合约
---

Dice2win是目前以太坊上很火爆的区块链博彩游戏，其最大的特点就是理论上的公平性保证，每天有超过1000以太币被人们投入到这个游戏中。

[Dice2win官网](https://dice2.win/)

[Dice2win合约代码](https://etherscan.io/address/0xd1ceeeeee83f8bcf3bedad437202b6154e9f5405#code)

dice2win的游戏非常简单，就是一个赌概率的问题。
![image.png-570kB][1]

就相当于猜硬币的正面和反面，只要你猜对了，就可以赢得相应概率的收获。

这就是一个最简单的依赖公平性的游戏合约，只要“庄家”可以保证绝对的公正，那么这个游戏就成立。

2018年9月21日，我在[《以太坊合约审计 CheckList 之“以太坊智能合约编码设计问题”影响分析报告》](https://paper.seebug.org/707/#7_1)中提到了以太坊智能合约中存在一个弱随机数问题，里面提到dice2win的合约中实现了一个很好的随机数生成方案**hash-commit-reveal**。

2018年10月12日，Zhiniang Peng from Qihoo 360 Core Security发表了[《Not a fair game, Dice2win 公平性分析》](https://paper.seebug.org/715/)，里面提到了关于Dice2win的3个安全问题。

在阅读文章的时候，我重新审视了Dice2win的合约代码，在第一次的阅读代码中，我着重寻找了攻击层面的问题，忽略了合约方可能会存在的问题，而且在上次的阅读中对Dice2win的执行流程有所误解，而且Dice2win也在后面的代码中迭代更新了Merkle proof功能，这里我们就重点聊聊这几个问题。

<!--more-->

# Dice2win安全性分析

## 选择中止攻击

让我们来回顾一下dice2win的代码

```
    function placeBet(uint betMask, uint modulo, uint commitLastBlock, uint commit, bytes32 r, bytes32 s) external payable {
        // Check that the bet is in 'clean' state.
        Bet storage bet = bets[commit];
        require (bet.gambler == address(0), "Bet should be in a 'clean' state.");

        // Validate input data ranges.
        uint amount = msg.value;
        require (modulo > 1 && modulo <= MAX_MODULO, "Modulo should be within range.");
        require (amount >= MIN_BET && amount <= MAX_AMOUNT, "Amount should be within range.");
        require (betMask > 0 && betMask < MAX_BET_MASK, "Mask should be within range.");

        // Check that commit is valid - it has not expired and its signature is valid.
        require (block.number <= commitLastBlock, "Commit has expired.");
        bytes32 signatureHash = keccak256(abi.encodePacked(uint40(commitLastBlock), commit));
        require (secretSigner == ecrecover(signatureHash, 27, r, s), "ECDSA signature is not valid.");

        uint rollUnder;
        uint mask;

        if (modulo <= MAX_MASK_MODULO) {
            // Small modulo games specify bet outcomes via bit mask.
            // rollUnder is a number of 1 bits in this mask (population count).
            // This magic looking formula is an efficient way to compute population
            // count on EVM for numbers below 2**40. For detailed proof consult
            // the dice2.win whitepaper.
            rollUnder = ((betMask * POPCNT_MULT) & POPCNT_MASK) % POPCNT_MODULO;
            mask = betMask;
        } else {
            // Larger modulos specify the right edge of half-open interval of
            // winning bet outcomes.
            require (betMask > 0 && betMask <= modulo, "High modulo range, betMask larger than modulo.");
            rollUnder = betMask;
        }

        // Winning amount and jackpot increase.
        uint possibleWinAmount;
        uint jackpotFee;

        (possibleWinAmount, jackpotFee) = getDiceWinAmount(amount, modulo, rollUnder);

        // Enforce max profit limit.
        require (possibleWinAmount <= amount + maxProfit, "maxProfit limit violation.");

        // Lock funds.
        lockedInBets += uint128(possibleWinAmount);
        jackpotSize += uint128(jackpotFee);

        // Check whether contract has enough funds to process this bet.
        require (jackpotSize + lockedInBets <= address(this).balance, "Cannot afford to lose this bet.");

        // Record commit in logs.
        emit Commit(commit);

        // Store bet parameters on blockchain.
        bet.amount = amount;
        bet.modulo = uint8(modulo);
        bet.rollUnder = uint8(rollUnder);
        bet.placeBlockNumber = uint40(block.number);
        bet.mask = uint40(mask);
        bet.gambler = msg.sender;
    }

    // This is the method used to settle 99% of bets. To process a bet with a specific
    // "commit", settleBet should supply a "reveal" number that would Keccak256-hash to
    // "commit". "blockHash" is the block hash of placeBet block as seen by croupier; it
    // is additionally asserted to prevent changing the bet outcomes on Ethereum reorgs.
    function settleBet(uint reveal, bytes32 blockHash) external onlyCroupier {
        uint commit = uint(keccak256(abi.encodePacked(reveal)));

        Bet storage bet = bets[commit];
        uint placeBlockNumber = bet.placeBlockNumber;

        // Check that bet has not expired yet (see comment to BET_EXPIRATION_BLOCKS).
        require (block.number > placeBlockNumber, "settleBet in the same block as placeBet, or before.");
        require (block.number <= placeBlockNumber + BET_EXPIRATION_BLOCKS, "Blockhash can't be queried by EVM.");
        require (blockhash(placeBlockNumber) == blockHash);

        // Settle bet using reveal and blockHash as entropy sources.
        settleBetCommon(bet, reveal, blockHash);
    }
```

主要函数为placeBet和settleBet，其中placeBet函数主要为建立赌博，而settleBet为开奖。最重要的一点就是，这里完全遵守**hash-commit-reveal**方案实现，随机数生成过程在服务端，整个过程如下。

1、用户选择好自己的下注方式，确认好后点击下注按钮。
2、服务端生成随机数reveal，生成本次赌博的随机数hash信息，有效最大blockNumber，并将这些数据进行签名，并将commit和信息签名传给用户。
3、用户将获取到的随机数hash以及lastBlockNumber等信息和下注信息打包，通过Metamask执行placebet函数交易。
4、服务端在一段时间之后，将带有随机数和服务端执行settlebet开奖

在原文中提到，庄家（服务端）接收到用户猜测的数字，可以选择是否中奖，选择部分对自己不利的中止，以使庄家获得更大的利润。

这的确是这类型合约最容易出现的问题，庄家依赖这种方式放大庄家获胜的概率。

上面的流程如下

![image.png-72.4kB][2]

而上面提到的选择中止攻击就是上面图的右边可能会出现的问题

![image.png-75.4kB][3]

整个流程最大的问题，就在于placebet和settlebet有强制的执行先后顺序，否则其中的一项block.number将取不到正确的数字，也正是应为如此，当用户下注，placebet函数执行时，用户的下注信息就可以被服务端获得了，此时服务端有随机数、打包placebet的block.number、下注信息，服务端可以提前计算用户是否中奖，也就可以选择是否中止这次交易。

## 选择开奖攻击

在原文中，提到了一个很有趣的攻击方式，在了解这种攻击方式之前，首先我们需要对区块链共识算法有所了解。

比特币区块链采用Proof of Work（PoW）的机制，这是一个叫做工作量证明的机制，提案者需要经过大量的计算才能找到满足条件的hash，当寻找到满足条件的hash反过来也证明了提案者付出的工作量。但这种情况下，可能会有多个提案者，那么就有可能出现链的分叉。区块链对这种结果的做法是，会选取最长的一条链作为最终结果。

当你计算出来的块被抛弃时，也就意味着你付出的成本白费了。所以矿工会选择更容易被保留的链继续计算下去。这也就意味着如果有人破坏，需要付出大量的经济成本。

借用一张原文中的图
![image.png-16.5kB][4]

在链上，计算出的b2、c5、b5、b6打包的交易都会回退，交易失败，该块不被认可。

回到Dice2win合约上，Dice2win是一个不希望可逆的交易过程，对于赌博来说，单向不可逆是一个很重要的原则。所以Dice2win新添加了MerikleProof方法来解决这个问题。

MerikleProofi方法核心在于，无论是否分叉，该分块是否会被废弃，Dice2win都认可这次交易。当服务端接收到一个下注交易（placebet）时，立刻对该区块开奖。

[MerikleProofi 的commit](https://github.com/dice2-win/contracts/commit/86217b39e7d069636b04429507c64dc061262d9c)

上面这种方法的原理和以太坊的区块结构有关，具体可以看[《Not a fair game, Dice2win 公平性分析》](https://paper.seebug.org/715/)一文中的分析，但这种方法一定程度的确解决了开奖速度的问题，甚至还减少了上面提到的选择中止攻击的难度。

但却出现了新的问题，当placebet交易被打包到分叉的多个区块中，服务端可以通过选择获利更多的那个区块接受，这样可以最大化获得的利益。但这种攻击方式效果有效，主要有几个原因：

1、Dice2win需要有一定算力的矿池才能主动影响链上的区块打包，但大部分算力仍然掌握在公开的矿池手中。所以这种攻击方式不适用于主动攻击。
2、被动的遇到分叉情况并不会太多，尤其是遇到了打包了placebet的区块，该区块的hash只是多了选择，仍然是不可控的，大概率多种情况结果都是一致的。

从这种角度来看，这种攻击方式有效率有限，对大部分玩家影响较小。

## 任意开奖攻击（Merkle proof验证绕过）

在上面的分析中，我们详细分析了我们Merkle proof的好处以及问题所在。但如果Merkle proof机制从根本上被绕过，那么是不是就有更大的问题了。

Dice2win在之前已经出现了这类攻击
[https://etherscan.io/tx/0xd3b1069b63c1393b160c65481bd48c77f1d6f2b9f4bde0fe74627e42a4fc8f81](https://etherscan.io/tx/0xd3b1069b63c1393b160c65481bd48c77f1d6f2b9f4bde0fe74627e42a4fc8f81)

攻击者成功构造攻击合约，通过合约调用placeBet来下赌注，并伪造Merkle proof并调用settleBetUncleMerkleProof开奖，以100%的几率控制赌博成功。

分析攻击合约可以发现该合约中的多个安全问题：

1、Dice2win是一个不断更新的合约，存在多个版本。但其中决定庄家身份的secretSigner值存在多个版本相同的问题，导致同一个签名可以在多个合约中使用。

2、placebet中对于最后一个commitlaskblock的check存在问题

![image.png-330.9kB][5]

用作签名的commitlastblock定义是uint256，但用作签名的只有uint40，也就是说，我们在执行placeBet的时候，可以修改高位的数字，导致某个签名信息始终有效。

3、Merkle proof边界检查不严格。

在最近的一次commit中，dice2win修复了一个漏洞是关于Merkle proofcheck的范围。

[https://github.com/dice2-win/contracts/commit/b0a0412f0301623dc3af2743dcace8e86cc6036b](https://github.com/dice2-win/contracts/commit/b0a0412f0301623dc3af2743dcace8e86cc6036b)

![image.png-93.5kB][6]

这里检查使Merkle proof更严格了

4、settleBet 权限问题

经过我的研究，实际上在Dice2win的游戏逻辑中，settleBet应该是只有服务端才能调用的（只有庄家才能开奖），但在之前的版本中，并没有这样的设置。

在新版本中，settleBet加入了这个限制。
![image.png-61.2kB][7]

这里绕过Merkle proof的方法就不再赘述了，有兴趣可以看看原文。

## refundBet下溢

原文中最后提到了一个refundBet函数的下溢，感谢@Zhiniang Peng from Qihoo 360 Core Security 提出了我这里的问题，最开始理解有所偏差导致错误的结论。

让我们来看看这个函数的代码

![image.png-286.7kB][8]

跟入getDiceWinAmount函数，发现jackpotFee并不可控

![image.png-283.4kB][9]

其中`JACKPOT_FEE = 0.001 ether`，且要保证amount大于0.1 ether，amount来自bet变量

![image.png-196.7kB][10]

而bet变量只有在placebet中可以被设置。

![image.png-186.8kB][11]

但可惜的是，placebet中会进行一次相同的调用

![image.png-125.4kB][12]

所以我们无法构造一个完整的攻击过程。

但我们回到refundBet函数中，我们无法控制jackpotFee，那么我们是不是可以控制jackpotSize呢

首先我们需要理解一下，jackpotSize是做什么，在Dice2win的规则中，除了本身的规则以外，还有一份额外的大奖，是从上次大奖揭晓之后的交易抽成累积下来的。

如果有人中了大奖，那么这个值就会清零。

![image.png-175.7kB][13]

但这里就涉及竞争了，完整的利用流程如下：

1、攻击者a下注placebet，并获得commit
2、某个好运的用户在a下注开奖前拿走了大奖
3、攻击者调用refundBet退款
4、jackpotSize成功溢出



# 总结

在回溯分析完整个Dice2win合约之后，我们不难发现，由于智能合约和传统的服务端逻辑不同，导致许多我们惯用的安全思路遇到了更多问题，区块链的不可信原则直接导致了随机数生成方式的难度加深。目前最为成熟的**hash-commit-reveal**方法是属于通过服务端与智能合约交互实现的，在随机数保密方面完成度很高，可惜的是无法避免服务端获取过多信息的问题。

在**hash-commit-reveal**方法的基础上，只要服务端不能即时响应开奖，选择中止攻击就始终存在。有趣的是Dice2win合约中试图实现的Merkle proof功能初衷是为了更快的开奖，但反而却在一定程度上减少了选择中止攻击的可能性。

任意开奖攻击，是一个针对Merkle proof的攻击方式，应验了所谓的功能越多漏洞越多的问题。攻击方式精巧，是一种很有趣的利用方式。

就目前为止，无论是底层的机制也好，又或是随机数的生成方式也好，智能合约的安全还有很长的路要走。


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/t60mhjw487zp2pgzhi4hurkp/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/5117iq8ahgvgtugkz2cfnzwv/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/lkg0ccut2gpd7nubxyiy5x09/image.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/hg6fijxmxndiqsrxzvl1hxzk/image.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/vomabxeg90j8wbi9fv7bvba2/image.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/wmyluf5z5muk79wt757qdwr4/image.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/3amrflndfl27xgqri5sjdcqz/image.png
  [8]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/h86qkka4ibqmys1dja2n8qrn/image.png
  [9]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/bdtz5kgavla68ebmsmhs8ojs/image.png
  [10]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/zbnajq74l6m01xpfx4z3r9f8/image.png
  [11]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/zq19tbbj6gyd0x3f99nqp014/image.png
  [12]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/1n95tv47fd9b4ouqx3lu7q80/image.png
  [13]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/81lglt14i2u1v3pigmbayq8y/image.png
