(취약점) Critical/High/Medium. 자금 탈취가 가능하거나 컨트랙트가 고장 난다거나.. 악의적인 공격으로 실제 피해가 발생할 가능성이 있는 버그.
예시 1) LP 토큰을 마음대로 mint 해서 Pool에 있는 자산을 거의 다 탈취할 수 있다.
예시 2) 담보의 가격이 떨어졌는데 청산이 제대로 안된다 / 추가 담보 입금 없이 청산을 안 당할 방법이 있다. 아니면 가격이 충분히 내려가지 않았는데 막 청산이 된다거나..

(잠재적 문제점 / 그냥 버그) Low/Informational. 요구사항에 있는 기능이 제대로 작동하지 않거나 사용자 본인만 손해를 보는 버그.
예시 1) Swap 1개는 되는데 10000개를 하니까 Pool에 토큰이 충분한데도 오류가 나거나 수량을 이상하게 적게 줌. (xy=k 공식 때문에 적은 건 정상입니다.)

## 제목

### 설명

해당 코드의 파일명, 함수명, 줄 번호.
뭐가 잘못됐고 결과적으로 의도된 동작과 어떻게 달라지는지 설명.
코드를 어느 정도 이해한 사람이 읽는다는 것을 전제로 하기 때문에 모든 것을 자세히 설명할 필요는 없습니다.
개발자가 이게 왜 버그인지 납득할 수 있게 설명하는 것이 중요합니다.

### 파급력

본인이 생각하는 심각도(Critical/
High : 청산 못함 무이자 대출
/Medium :
/Low/Informational)와 발생할 수 있는 잠재적인 피해에 대한 내용.
심각도는 발생할 수 있는 피해와 발생 가능성(공격 난이도 등)을 종합적으로 고려해서 판단합니다.
(이 부분은 실무에서도 아주 중요한데요. 설명이 충분하지 않으면 심각성이 과소평가될 수 있고 반대로 사실과 다른 내용을 과장해서 적으면 전문성을 의심받기 때문입니다.)

### 해결방안

어떻게 해결하면 좋을지.
그리고 가능하다면 유사한 문제를 방지할 방법.
보통 코드 수정사항이 귀찮을 정도가 (저의 경우 2줄 이상) 되면 직접 고쳐주지는 않습니다.

---

# procfs

## removeLiquidity Y토큰 손해 발생

### 설명

- 파일명 : Dex.sol
- 함수명 : removeLiquidity(197) -> calculateExchangeRate(48)
  - https://github.com/procfs-web3/practice_DEX/blob/master/src/Dex.sol#L197
  - https://github.com/procfs-web3/practice_DEX/blob/master/src/Dex.sol#L48
- Provider가 Pool에 예치한 토큰들을 회수할려고 할 때 Y토큰에 대한 계산이 잘못되어 Provider는 자신이 투자한 Token보다 더 적게 받게 됩니다.

```
tokenYAmount = calculateExchangeRate(tokenXAmount, tokenXBalance, tokenYBalance);
```

- 만약 현재의 Pool의 토큰의 비율이 1:1 어서 40 : 40토큰씩 예치한다고 했을 때 사용자는 22222222222...만큼만 Y토큰을 회수하게 되어 Provider는 손해를 보게 됩니다.

### Low

- Provider가 자기가 투자한 돈에 대한 이익대신 손실이 되기 때문에 Provider의 수가 줄어들며 유동성이 줄어들고 Dex의 기능을 상실하게 될 것 같습니다.

### 해결 방안

- tokenXAmount = tokenXBalance \* lpTokenAmount / lpSum;
- tokenYAmount = tokenYBalance \* lpTokenAmount / lpSum;

---

## removeLiquidity lpSum 코드 개선

### 설명

- 파일명 : Dex.sol
- 함수명 : removeLiquidity()
  - https://github.com/procfs-web3/practice_DEX/blob/master/src/Dex.sol#L211
- removeLiqiudity에서 LP 토큰의 total값을 반복문을 통하여 계산하고 있어서 만약 사용자가 많아진다면 많은 가스 비용이 들 거라고 생각됩니다.

```
        for (uint i = 0; i < liquidityProvisions.length; i++) {
            Provision storage p = liquidityProvisions[i];
            lpSum += p.amount;
            if (p.provider == msg.sender) {
                senderLpTokenAmount = p.amount;
                p.amount = 0;
            }
        }
```

### Gas 비용 개선

- Provider가 투자한 토큰을 뺄려고할 때 가스 비용이 투자한 provider의 수 만큼 증가하게 되는 문제가 있다고 생각됩니다.

### 해결 방안

- LPtoken total을 전역 변수로 선언해주고 addLiquidity함수가 실행될 때 발급되는 토큰 양을 저장해두어 removeLiquidity함수를 실행할 때 전역 변수의 total변수를 가져오는 방식이 가스비를 절약할 수 있지 않을까 합니다.

---

## \_depositUsdc에 코드 오류로 인한 코인 복사

### 설명

- 파일명 : Lending.sol
- 함수명 : \_depositUsdc
  - https://github.com/procfs-web3/practice_lending/blob/master/src/Lending.sol#L60
  - https://github.com/procfs-web3/practice_lending/blob/master/src/Lending.sol#L180
- `d.amount = calcPrincipleSum(amount, d.timestamp) + amount;`코드가 복리를 계산한 후 amount를 더함으로 돈이 amount x 2에 해당하는 값이 `d.amount`에 대입되게 됩니다.
- `calcPrincipleSum()`의 함수의 코드는 아래와 같습니다.
- ```
    function _depositUsdc(uint256 amount) internal {
        for (uint i = 0; i < usdcDepositInfos.length; i++) {
            if (usdcDepositInfos[i].provider == msg.sender) {
                DepositInfo storage d = usdcDepositInfos[i];
                d.amount = calcPrincipleSum(amount, d.timestamp);

                d.amount = calcPrincipleSum(amount, d.timestamp) + amount;
                d.timestamp = block.timestamp;
                usdc.transferFrom(msg.sender, address(this), amount);
                return;
            }
        }
        DepositInfo memory d;
        d.provider = msg.sender;
        d.amount = amount;
        d.timestamp = block.timestamp;
        usdcDepositInfos.push(d);
        usdc.transferFrom(msg.sender, address(this), amount);
    }
  ```

  ```
      function _withdrawUsdc(address user, uint256 amount) internal {
        for (uint i = 0; i < usdcDepositInfos.length; i++) {
            DepositInfo storage d = usdcDepositInfos[i];
            if (d.provider == user) {
                uint256 paybackAmount = calcPrincipleSum(d.amount, d.timestamp);
                require(amount <= paybackAmount, "withdraw: excessive withdrawl");
                usdc.transfer(d.provider, amount);
                if (paybackAmount == amount) {
                    usdcDepositInfos[i] = usdcDepositInfos[usdcDepositInfos.length - 1];
                    usdcDepositInfos.pop();
                }
                else {
                    d.timestamp = block.timestamp;
                    d.amount = paybackAmount - amount;
                }
                return;
            }
        }
        require(false, "withdraw: user not found");
    }

  ```

- withdrawUsdc()함수에서 조건을 검사할 때 `require(amount <= paybackAmount, "withdraw: excessive withdrawl");` 코드로 검사하는데 amount값이 paybackAmount(amount x 2)보다 작기만 하면 사용자가 입력한 amount값을 그대로 가져올 수 있게 됩니다.

```
require(amount <= paybackAmount, "withdraw: excessive withdrawl");
payable(d.provider).transfer(amount);
```

- paybackAmount의 값은 `d.amount`의 값을 기준으로 복리를 계산한 값을 가지고 옵니다.

```
uint256 paybackAmount = calcPrincipleSum(d.amount, d.timestamp);
```

- USDC뿐만 아니라 Ether부분에서도 Withdraw() 에서 동일한 로직이 수행되어 Ether를 위와 같이 투자 금액에 2배에 해당하는 금액을 가져올 수 있습니다.

### Critical

- Lending Contract에 충분한 USDC/ETH 가 있다면 투자한 돈에 두 배만큼 가져올 수 있기 때문에 Critical이라고 생각합니다.

### 해결 방안

- `d.amount = calcPrincipleSum(amount, d.timestamp) + amount;`
  -> `d.amount = calcPrincipleSum(amount, d.timestamp);`

---

# Eunyoung

## \_swap()함수의 수수료를 받지 않는 오류

못

### 설명

- 파일명 : dex.sol
- 함수명 : \_swap
  - https://github.com/Eunyeong2/DEX/blob/master/src/dex.sol#L71
  - 수수료를 빼준 상태에서 토큰을 받기 때문에 결과적으로는 수수료 없이 토큰 swap이 일어납니다.

### Informational

- 수수료를 받지 않는 상태가 되어 provider들은 deposit한 만큼의 이익을 얻지 못하는 상황이 발생해 provider들이 줄어들 것 입니다.

  ```
          uint amount; // 수수료
        amount = I_Amount * 1/1000; //수수료
        I_Amount = I_Amount - amount; //I_Amount - 수수료


        IERC20(O_Token).transfer(msg.sender, __reserve1 - k / (__reserve0 + I_Amount)); //swap 할 때 생기는 가격 변동으로 인해 빠지는 금액
        IERC20(I_Token).transferFrom(msg.sender, address(this), I_Amount); //수수료 뺀 토큰 주기
  ```

### 해결 방안

```
IERC20(I_Token).transferFrom(msg.sender, address(this), I_Amount);
```

- `I_Amount` + 수수료를 해주면 될 것 같습니다.

## repay함수의 잘못된 담보 상환

### 설명

- 파일명 : Lending.sol
- 함수명 : repay()
  - https://github.com/Eunyeong2/Lending/blob/master/src/Lending.sol#L87

```
        IERC20(USDC).transferFrom(msg.sender, address(this), amount);
        borrows[msg.sender][tokenAddress] -= amount;
        IERC20(ETH).transfer(msg.sender, amount);
        mortgages[msg.sender] -= amount;
```

- 빌린 돈을 상환할 때 상환 금액 만큼 담보를 보내고 있습니다. 현재 1 Ether 의 값이 100 USDC라고 가정해 보겠습니다.
- 1 Ehter를 담보로 예치하고 1 Ether \* 100 USDC의 50%인 50 USDC를 빌리게 됩니다.
- repay() 함수를 이용하여 1 USDC를 상환한다고 하면 담보에 대한 USDC의 값을 계산하지 않고 인자(amount) 그대로 Ether를 보내주기 때문에 빌린 돈을 다 갚지 않아도 담보를 가져올 수 있게 된다고 생각합니다.

### Ciritical

- Borrower가 빌린돈을 다 상환하지 않아도 일정 금액으로만 맡긴 담보를 모두 가져올 있기 때문에 빌리고 상환하기를 반복하여 Lending Contract가 가지고 있는 USDC를 모두 가져올 수 있게 됩니다.

### 해결 방안

- Oracle을 통해 Ether토큰과 USDC토큰의 가격을 가져와서 비교한 후 상환하는 금액에 맞게 담보를 돌려주어야 합니다.

---

# woozook

## \_burn함수 visible 취약점

### 설명

- 파일명 : tokenLP.sol
- 함수명 : \_burn
  - https://github.com/dreamwooz3k/wooz3k_dex/blob/master/src/tokenLP.sol#L104
  - `function _burn(address _owner, uint256 _eth) public returns (bool success)`
  - LP토큰의 \_burn함수가 public으로 되어 있습니다. 악의적인 provider가 다른 Provider의 LPtoken을 burn시킬 수 있는 취약점이 존재합니다.

### Critical

- 악의적인 provider가 다른 provider의 LP Token을 burn시켜 자신의 LP Token의 비율을 높여서 Pool에 예치된 다른 provider의 토큰을 가져올 수 있습니다.

### 해결 방안

- LP Token의 `_burn` 함수를 Dex Contract만 사용할 수 있게 owner 검사를 해줘야 합니다.

---

# rkdnd

## \_swap함수의 잘못된 계산

### 설명

- 파일명 : DEX.sol
- 함수명 : \_swap
  - https://github.com/rkdnd/DEX/blob/master/src/DEX.sol#L74
  - outputAmount의 값을 구할 때 `outputAmount = reserveGet - constK / (reserveTo + inputAmount);` 해당 식을 이용하는데 `constK / (reserveTo + inputAmount)` 가 0이 되어서 reserveGet의 해당하는 값을 모두 가져올 수 있습니다.
  - `uint256 constK = (reserveTo / 1000) * (reserveGet / 1000);`

### Critical

- swap을 할 때 적은 토큰으로도 swap하고 싶은 토큰의 전체 수량을 가져올 수 있게 됩니다.

### 해결 방안

- X토큰 -> Y토큰으로 바꾼다고 가정했을 때 `Pool_Y - (K / Pool_X + inputXAmount)`의 식을 이용하여 Swap해줄 Y토큰의 값을 계산하여야 합니다.

## \_transfer의 잘못된 visible

### 설명

- 파일명 : ERC20.sol
- 함수명 : \_transfer
  - https://github.com/rkdnd/lending/blob/master/src/ERC20.sol
  - \_transfer의 함수의 가시성이 public으로 되어 있어서 누구나 \_transfer를 사용할 수 있게 됩니다.

### Critical

- 담보나 Lending Contract에 들어있는 토큰들을 모두 가져올 수 있습니다.

### 해결 방안

- visible을 privte으로 변경해주고 Contract에서 사용을 할 때 transfer()함수를 사용하도록 해야 합니다.

# getFCLiquidationList()

### 설명

- 파일명 : Lending.sol
- gdtFCLiquidationList()
  - https://github.com/rkdnd/lending/blob/master/src/Lending.sol#L164
 ```
     function updateFCLiquidationList() internal{
        uint256 etherPrice = DreamOracle(oracle).getPrice(address(USDC));

        for(uint i = 0; i < marketPriceList.length; i++){
            if((marketPriceList[i] * 75 / 100) >= etherPrice){
                for(uint k = 0; k < borrowerMarketPrice[marketPriceList[i]].length; k++)
                    FCLiquidationList.push(address(borrowerMarketPrice[marketPriceList[i]][k]));
            }
        }
    }

    function getFCLiquidationList() public returns(address[] memory list){
        updateFCLiquidationList();
        list= FCLiquidationList;
    }
 ```
    
  - public으로 visible이 되어 있어서 `updateFCLiquidationList()` 함수를 실행하게 되는데 `FCLiquidationList.push(address(borrowerMarketPrice[marketPriceList[i]][k]));` 코드를 계속 실행하게 되면 `FCLiquidationList`를 무한히 늘릴 수 있게 됩니다.
  - `findArray()` `FCLiquiExistCheck()`는 FCLiquidationList를 반복문으로 사용하게 되어 길이를 무한정으로 늘려버리면 가스 사용을 급격히 늘릴 수 있게 됩니다.
```
    function FCLiquiExistCheck(address user) public view returns (bool) {
        for (uint i = 0; i < FCLiquidationList.length; i++) {
            if (FCLiquidationList[i] == user) {
                return true;
            }
        }

        return false;
    }

    function findArray(address user) public returns (uint i){
        for (uint i = 0; i < FCLiquidationList.length; i++) {
            if (FCLiquidationList[i] == user) {
                return i;
            }
        }
    }
 ```
### High

- `liquidate()`함수가 실행되어질 때 리스트의 길이가 길어져 가스 limit을 초과시켜 담보를 청산 못시키게 할 수 있습니다.

### 해결 방안

- visible을 public이 아닌 private으로 변경해주면 될 것 같습니다.
