---
title: “以太坊智能合约规范问题”影响分析报告
date: 2018-08-14 16:24:18
tags:
- eth
- contract
---

最近工作的重点在智能合约上，不仅是整理checklist，还有一个智能合约的自动化审计平台，阅读了比较多的合约代码，我发现，一些开发者在开发过程中的问题，可以算是很典型的一类问题，其实很多不能算是有很大的危害，但确实不得忽视的一个大问题。

<!--more-->

# 一、 简介

在知道创宇404区块链安全研究团队整理输出的《知道创宇以太坊合约审计CheckList》中，把“未触发Transfer事件问题”、“未触发Approval事件问题”、“假充值漏洞”、“构造函数书写错误”等问题统一归类为“以太坊智能合约规范问题”。

“昊天塔(HaoTian)”是知道创宇404区块链安全研究团队独立开发的用于监控、扫描、分析、审计区块链智能合约安全自动化平台。我们利用该平台针对上述提到的《知道创宇以太坊合约审计CheckList》中“以太坊智能合约规范”类问题在全网公开的智能合约代码做了扫描分析。详见下文：

# 二、漏洞详情

ERC20\[1\]是一种代币标准，用于以太坊区块链上的智能合约。ERC20定义了一种以太坊必须执行的通用规则，如果在以太坊发行的代币符合ERC20的标准，那么交易所就可以进行集成，在它们的交易所实现代币的买卖和交易。

ERC20中规定了transfer函数必须触发Transfer事件，transfer函数必须返回bool值，在进行余额判断时，应抛出错误而不是简单的返回错误，approve函数必须触发Approval事件。

## 1、未触发Transfer事件
    
```
function transfer(address _to, uint256 _value) public returns (bool success) {
        require(balanceOf[msg.sender] >= _value);           
        require(balanceOf[_to] + _value >= balanceOf[_to]); 
        balanceOf[msg.sender] -= _value;                            
        balanceOf[_to] += _value;                           
        return true;
    }
```

上述代码在发生交易时未触发Transfer事件，在发生交易时，未产生event事件，不符合ERC20标准，不便于开发人员对合约交易情况进行监控。

## 2、未触发Approval事件
```
    function approve(address _spender, uint256 _value) public
        returns (bool success) {
        allowance[msg.sender][_spender] = _value;
        return true;
    }
```

上述代码在发生交易时未触发Approval事件，在发生交易时，未产生event事件，不符合ERC20标准，不便于开发人员对合约情况进行监控。

## 3、假充值漏洞
```    
    function transfer(address _to, uint256 _amount) returns (bool success) {
        initialize(msg.sender);

        if (balances[msg.sender] >= _amount
            && _amount > 0) {
            initialize(_to);
            if (balances[_to] + _amount > balances[_to]) {

                balances[msg.sender] -= _amount;
                balances[_to] += _amount;

                Transfer(msg.sender, _to, _amount);

                return true;
            } else {
                return false;
            }
        } else {
            return false;
        }
    }
```

上述代码在判断余额时使用了if语句，ERC20标准规定，当余额不足时，合约应抛出错误使交易回滚，而不是简单的返回false。

这种情况下，会导致即使没有真正发生交易，但交易仍然成功，这种情况会影响交易平台的判断结果，可能导致假充值。

2018年7月9日，慢雾安全团队发布了关于假充值的漏洞预警。

2018年7月9日，知道创宇404区块链安全研究团队跟进应急该漏洞，并对此漏洞发出了漏洞预警。

## 4、构造函数书写错误漏洞

Solidity0.4.22版本以前，编译器要求，构造函数名称应该和合约名称保持一致，如果构造函数名字和合约名字大小写不一致，该函数仍然会被当成普通函数，可以被任意用户调用。

Solidity0.4.22中引入了关于构造函数constructor使用不当的问题，constructor在使用中错误的加上了function定义，从而导致constructor可以被任意用户调用，会导致可能的更严重的危害，如Owner权限被盗。

### 构造函数大小写错误漏洞
```
contract own(){
    function Own() {
        owner = msg.sender;
    }
}
```

上述代码错误的将构造函数名大写，导致构造函数名和合约名不一致。这种情况下，该函数被设置为一个普通的public函数，任意用户都可以通过调用该函数来修改自己为合约owner。进一步导致其他严重的后果。

2018年6月22日，MorphToken合约代币宣布更新新的智能合约\[2\]，其中修复了关于大小写错误导致的构造函数问题。

2018年6月22日，知道创宇404区块链安全研究团队跟进应急，并输出了《以太坊智能合约构造函数编码错误导致非法合约所有权转移报告》。

### 构造函数编码错误漏洞
```
    function constructor() public {
        owner = msg.sender;
    }
```

上述代码错误的使用function来作为constructor函数装饰词，这种情况下，该函数被设置为一个普通的public函数，任意用户都可以通过调用该函数来修改自己为合约owner。进一步导致其他严重的后果。

2018年7月14日，链安科技在公众号公布了关于constructor函数书写错误的问题详情\[3\]。

2018年7月15日，知道创宇404区块链安全研究团队跟进应急，并输出了《以太坊智能合约构造函数书写错误导致非法合约所有权转移报告》

# 三、漏洞影响范围

使用Haotian平台智能合约审计功能可以准确扫描到该类型问题。

![image.png-27.5kB][1]

基于Haotian平台智能合约审计功能规则，我们对全网的公开的共39548 个合约代码进行了扫描，其中共14978个合约涉及到这类问题。

## 1、 未触发Transfer事件

截止2018年8月10日为止，我们发现了4604个存在未遵循ERC20标准未触发Transfer事件的合约代码，其中交易量最高的10个合约情况如下：
![image.png-274.8kB][2]

## 2、 未触发Approval事件

截止2018年8月10日为止，我们发现了5231个存在未遵循ERC20标准未出发Approval事件的合约代码，其中交易量最高的10个合约情况如下：

![image.png-295.3kB][3]

## 3、假充值漏洞

2018年7月9日，知道创宇404区块链安全研究团队在跟进应急假充值漏洞时，曾对全网公开合约代码进行过一次扫描，当时发现约3141余个存在假充值问题的合约代码，其中交易量最高的10个合约情况如下：

![image.png-284.5kB][4]

截止2018年8月10日为止，我们发现了5027个存在假充值问题的合约代码，其中交易量最高的10个合约情况如下：

![image.png-260.3kB][5]

## 4、构造函数书写问题

### 构造函数大小写错误漏洞

2018年6月22日，知道创宇404区块链安全研究团队在跟进应急假充值漏洞时，全网中存在该问题的合约约为16个。

截止2018年8月10日为止，我们发现了90个存构造函数大小写错误漏洞的合约代码，其中交易量最高的10个合约情况如下：
![image.png-294.7kB][6]

### 构造函数编码问题

截止2018年8月10日为止，我们发现了24个存在构造函数书写问题的合约代码，比2018年7月14日对该漏洞应急时只多了一个合约，其中交易量最高的10个合约情况如下：

![image.png-309.6kB][7]

# 四、修复方式


## 1）transfer函数中应触发Tranfser事件
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
## 2）approve函数中应触发Approval事件
```
  function approve(address _spender, uint256 _value) public returns (bool) {
    allowed[msg.sender][_spender] = _value;
    emit Approval(msg.sender, _spender, _value);
    return true;
  }
```

## 3）transfer余额验证时应使用require抛出错误
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
## 4）0.4.22版本之前，构造函数应和合约名称一致
```
contract ownable {
	function ownable() public {
    owner = msg.sender;
  }
```

## 5）0.4.22版本之后，构造函数不应用function修饰
```
constructor() public {
        owner = msg.sender;
    }
```

# 五、一些思考

上面这些问题算是我在回顾历史漏洞中经常发现的一类问题，都属于开发人员没有遵守ERC20标准而导致的，虽然这些问题往往不会直接导致合约漏洞的产生，但却因为这些没有遵守标准的问题，在后期对于合约代币的维护时，会出现很多问题。

如果没有在transfer和approve时触发相应的事件，开发人员就需要更复杂的方式监控合约的交易情况，一旦发生大规模盗币时间，甚至没有足够的日志提供回滚。

如果转账时没有抛出错误，就有可能导致假充值漏洞，如果平台方在检查交易结果时是通过交易状态来判断的，就会导致平台利益被损害。

如果开发人员在构造函数时，没有注意不同版本的编译器标准，就可能导致合约所有权被轻易盗取，导致进一步更严重的盗币等问题。

我们在对全网公开的合约代码进行扫描和监控时容易发现，有很大一批开发人员并没有注意到这些问题，甚至构造函数书写错误这种低级错误，在漏洞预警之后仍然在发生，考虑到大部分合约代码没有公开，可能还有很多开发者在不遵守标准的情况下进行开发，还有很多潜在的问题需要去考虑。

这里我们建议所有的开发者重新审视自己的合约代码，检查是否遵守了ERC20合约标准，避免不必要的麻烦以及安全问题。

# 六、REF

- \[1\] ERC标准
[https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-20.md)

- \[2\] Morpheus官方公告
[https://medium.com/@themorpheus/new-morpheus-network-token-smart-contract-91
b80dbc7655](https://medium.com/@themorpheus/new-morpheus-network-token-smart-contract-91
b80dbc7655)

- \[3\] 构造函数书写问题漏洞详情：
[https://mp.weixin.qq.com/s/xPwhanev-cjHhc104Wmpug](https://mp.weixin.qq.com/s/xPwhanev-cjHhc104Wmpug)
 


  [1]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/l7co6wfvbf1w3rnuu5ertukd/image.png
  [2]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/yaabnbccin5hpi2sf3enmor8/image.png
  [3]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/l6hdzoyhnta75mi8ogi48y0n/image.png
  [4]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/bd5r1ksfuigyq6evpkvhwbte/image.png
  [5]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/mhf4q18kv0c5d7uzqu1n36v0/image.png
  [6]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/zqar0nfpu38j9eakx5iclyeh/image.png
  [7]: https://lorexxar-blog.oss-cn-shanghai.aliyuncs.com/zybuluo-backup/LoRexxar/a8s5awhte8a2ah5cfx6s7kk4/image.png
