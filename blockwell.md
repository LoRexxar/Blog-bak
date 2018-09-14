---
title: blockwell.ai 虚假转账 事件分析
date: 2018-09-14 10:47:34
tags:
---



# 一、背景

2018年9月7日早上1点左右，许多以太坊账户都收到了一种名为`blockwell.ai KYC Casper Token`转账消息，其中有的是收到了这种代币，而有的用户是支出了这种代币。

![image.png-174.9kB][1]

![image.png-24.2kB][2]

得到的用户以为受到了新币种的空投，满心欢喜的打开之后发现并没有获得任何代币。转出的用户着急打开钱包，以为是钱包被盗转走了代币，实际上却毫无损失。

在回过神来看看代币的名字，忍不打开`blockwell.ai`查看原因，一次成功的广告诞生了。

<!--more-->

# 二、事件回顾

回到导致事件发生的blockwell.ai合约中，合约地址为
```
https://etherscan.io/address/0x212d95fccdf0366343350f486bda1ceafc0c2d63
```

实际转账到账户的交易信息
```
https://etherscan.io/token/0x212d95fccdf0366343350f486bda1ceafc0c2d63?a=0xa3fe2b9c37e5865371e7d64482a3e1a347d03acd
```

可以看到通过调用这个合约，发起了一笔代币转账，在event logs里可以看到实际的交易
![image.png-17.8kB][3]

然后具体的交易地址为
```
https://etherscan.io/tx/0x3230f7326ab739d9055e86778a2fbb9af2591ca44467e40f7cd2c7ba2d7e5d35
```

整笔交易花费了244w的gas，价值2.28美元，有针对的从500个用户转账给了500个用户。

![image.png-144.1kB][4]

跟踪到转账的from地址，可以看到一个很有趣的问题

```
https://etherscan.io/address/0xeb7a58d6938ed813f04f36a4ea51ebb5854fa545#tokentxns
```

![image.png-117.7kB][5]

所有的来源账户本身都是不持有这种代币的。

跟踪一下也可以发现，无论是发起交易者还是接受交易者，都没有发生实际代币的变化。

但交易却被记录了下来，这是怎么回事呢？

# 三、漏洞分析

智能合约是一种于1994年被提出的，在没有第三方的情况下进行的可信交易。而以太坊在区块链上实现了一种图灵完备的语言solidity，允许人们在区块链上编写代码来实现智能合约。而智能合约的成熟催生了合约代币的产生，**合约代币中只有遵守以太坊ERC20标准的合约代币才会被承认为ERC20代币，ERC20代币会直接被交易所承认。**

在ERC20标准中规定，transfer函数必须触发Transfer事件，事件会被记录在event log中，而**各大平台与交易所也是通过解析event log来获取交易信息。**

会触发事件的转账代码
```
  function transfer(address _to, uint256 _value) public returns (bool) {
    require(_value <= balances[msg.sender]);
    require(_to != address(0));

    balances[msg.sender] = balances[msg.sender].sub(_value);
    balances[_to] = balances[_to].add(_value);
    emit Transfer(msg.sender, _to, _value);
    return true;
  }
```

“昊天塔(HaoTian)”是知道创宇404区块链安全研究团队独立开发的用于监控、扫描、分析、审计区块链智能合约安全自动化平台。HaoTian全新的引入了对opcode的反编译审计模块，正在逐步应用到对智能合约的审计中。

这里我们使用HaoTian最新的反编译功能针对该智能合约进行简单的恢复源码。

源码如下
```
contract 0x212D95FcCdF0366343350f486bda1ceAfC0C2d63 {

    mapping(address => uint256) balances;
    uint256 public totalSupply;
    mapping (address => mapping (address => uint256)) allowance;
    address public owner;
    string public name;
    string public symbol;
    uint8 public decimals;

    event Approval(address indexed _owner, address indexed _spender, uint256 _value);
    event Transfer(address indexed _from, address indexed _to, uint256 _value);
    event OwnershipRenounced(address indexed previousOwner);
    event TransferOwnership(address indexed old, address indexed new);

    function approve(address _spender, uint256 _value) public returns (bool success) {        
        allowance[msg.sender][_spender] = _value;        
        Approval(msg.sender, _spender, _value);        
        return true;    
    }  

    function transferFrom(address _from, address _to, uint256 _value) public returns (bool success) {
        // 0x841
        require(to != address(0));   
        require(balances[_from] >= _value);
        require(allowance[_from][msg.sender] >= _value);
        balances[_from] = balances[_from].sub(_value);
        balances[_to] = balances[_to].add(_value);
        allowance[_from][msg.sender] =  allowance[_from][msg.sender].sub(_value); 
        Transfer(_from, _to, _value);
        return true;
    }

    function decreaseApproval(address _spender, uint256 _subtractedValue) {
        // 0xc0e
        uint oldValue = allowance[msg.sender][_spender];
        if (_subtractedValue > oldValue) {      
            allowance[msg.sender][_spender] = 0;    
        } else {
            allowance[msg.sender][_spender] = oldValue.sub(_subtractedValue);    
        }    
        Approval(msg.sender, _spender, allowance[msg.sender][_spender]);    
        return true;  
    }

    function balanceOf(address _owner) constant returns (uint256 balance) {       
        // 0xe9f 
        return balances[_owner];    
    }    

    function renounceOwnership() {
        // 0xee7
        require(owner == msg.sender);
        emit OwnershipRenounced(owner);
        owner = address(0);
    }

    function x_975ef7df(address[] arg0, address[] arg1, uint256 arg2) {
        require(owner == msg.sender);
        require(arg0.length > 0, "Address arrays must not be empty");
        require(arg0.length == arg1.length, "Address arrays must be of equal length");
        for (i=0; i < arg0.length; i++) {
            emit Transfer(arg0[i], arg1[i], arg2);
        }
    }

    function transfer(address arg0,uint256 arg1) {
        require(arg0 != address(0x0));
        require(balances[msg.sender] > arg1);
        balances[mag.sender] = balances[msg.sender].sub(arg1);
        balances[arg0] = balances[arg0].add(arg1);
        emit Transfer(msg.sender, arg0, arg1)
        return arg1
    }

    function increaseApproval(address arg0,uint256 arg1) {
        allowance[msg.sender][arg0] = allowance[msg.sender][arg0].add(arg1)
        emit Approval(msg.sender, arg0, arg1)
        return true;
    }

    function transferOwnership(address arg0) {
        require(owner == arg0);
        require(arg0 != adress(0x0));
        emit TransferOwnership(owner, arg0);
        owner = arg0;
    }
}
```

从代码中可以很明显的看到一个特殊的函数`x_975ef7df`，这是唯一一个涉及到数组操作，且会触发Tranfser事件的函数。
```
    function x_975ef7df(address[] arg0, address[] arg1, uint256 arg2) {
        require(owner == msg.sender);
        require(arg0.length > 0, "Address arrays must not be empty");
        require(arg0.length == arg1.length, "Address arrays must be of equal length");
        for (i=0; i < arg0.length; i++) {
            emit Transfer(arg0[i], arg1[i], arg2);
        }
    }
```

从代码中可以很清晰的看到， 在对地址列表的循环中，只触发了Transfer事件，没有任何其余的操作。

是不是说明平台和交易所在获取ERC20代币交易信息，是通过event log事件获取的呢？我们来测试一下。

# 事件复现

首先我们需要编写一个简单的ERC20标准的代币合约

```
contract MyTest {

    mapping(address => uint256) balances;
    uint256 public totalSupply;
    mapping (address => mapping (address => uint256)) allowance;
    address public owner;
    string public name;
    string public symbol;
    uint8 public decimals = 18;

    event Transfer(address indexed _from, address indexed _to, uint256 _value);

    function MyTest() {
        name = "we are ruan mei bi";
        symbol = "RMB";
        totalSupply = 100000000000000000000000000000000000;
    }

    function mylog(address arg0, address arg1, uint256 arg2) public {
        Transfer(arg0, arg1, arg2);
    }

}
```

合约代币需要规定好代币的名称等信息，然后我们定义一个mylog函数。

这里我们通过remix进行部署(由于需要交易所获得提示信息，所以我们需要部署在公链上)
![image.png-303kB][6]

测试合约地址
```
https://etherscan.io/address/0xd69381aec4efd9599cfce1dc85d1dee9a28bfda2
```

然后直接发起交易
![image.png-81.2kB][7]

然后我们的imtoken提示了消息

![image.png-60.7kB][8]

回看余额可以发现没有实际转账诞生。

# 五、总结

回顾前面的测试，可以发现这是一件有趣的利用ERC20标准本身被信任的问题，攻击者花费了约2.28美元的gas，对1000个用户有针对的发送了广告，花费小额的gas完成了广告这个过程。

事件的核心就在于，ERC20作为标准要求要遵守才可以被承认，但交易所/平台却盲目信任符合ERC20标准的合约，将平台本身原理上的bug利用到发放小广告上，是一次比较特别的体验。

可以说，这是智能合约的一次极为特殊的漏洞利用，本身不涉及盗币，但却会对现在的交易所已经平台造成严重的危害，而且其本身底层逻辑bug难以从底层修复，只能在上层做修复，但黑名单的过滤方式很难真正奏效，一个属于区块链广告的黑暗时代到来了...


  [1]: http://static.zybuluo.com/LoRexxar/40wsh583au1b52wrq7h1zlkj/image.png
  [2]: http://static.zybuluo.com/LoRexxar/bt6ksztaddgua93zahn21ayp/image.png
  [3]: http://static.zybuluo.com/LoRexxar/irjva9hnvl1qgdl6dipr9aff/image.png
  [4]: http://static.zybuluo.com/LoRexxar/6mn1iox0cj2ewy6wa8zyelot/image.png
  [5]: http://static.zybuluo.com/LoRexxar/sohrof7ak2rfqs7opei5rr2y/image.png
  [6]: http://static.zybuluo.com/LoRexxar/u52nsbgp67spco536illmsoo/image.png
  [7]: http://static.zybuluo.com/LoRexxar/7g45lkyt0eal5fm1qe77oexi/image.png
  [8]: http://static.zybuluo.com/LoRexxar/zgzt4w4x9vfa7huu76e76usx/image.png
