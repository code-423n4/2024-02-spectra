# Report


## Gas Optimizations


| |Issue|Instances|
|-|:-|:-:|
| [GAS-1](#GAS-1) | `a = a + b` is more gas effective than `a += b` for state variables (excluding arrays and mappings) | 2 |
| [GAS-2](#GAS-2) | Using bools for storage incurs overhead | 1 |
| [GAS-3](#GAS-3) | State variables should be cached in stack variables rather than re-reading them from storage | 9 |
| [GAS-4](#GAS-4) | Use calldata instead of memory for function arguments that do not get mutated | 2 |
| [GAS-5](#GAS-5) | For Operations that will not overflow, you could use unchecked | 60 |
| [GAS-6](#GAS-6) | Avoid contract existence checks by using low level calls | 6 |
| [GAS-7](#GAS-7) | State variables only set in the constructor should be declared `immutable` | 2 |
| [GAS-8](#GAS-8) | Using `private` rather than `public` for constants, saves gas | 2 |
| [GAS-9](#GAS-9) | `internal` functions not called by the contract should be removed | 6 |
### <a name="GAS-1"></a>[GAS-1] `a = a + b` is more gas effective than `a += b` for state variables (excluding arrays and mappings)
This saves **16 gas per instance.**

*Instances (2)*:
```solidity
File: src/tokens/PrincipalToken.sol

714:         unclaimedFeesInIBT += _feesInIBT;

715:         totalFeesInIBT += _feesInIBT;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="GAS-2"></a>[GAS-2] Using bools for storage incurs overhead
Use uint256(1) and uint256(2) for true/false to avoid a Gwarmaccess (100 gas), and to avoid Gsset (20000 gas) when changing from ‘false’ to ‘true’, after having been ‘true’ in the past. See [source](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/58f635312aa21f947cae5f8578638a85aa2519f5/contracts/security/ReentrancyGuard.sol#L23-L27).

*Instances (1)*:
```solidity
File: src/tokens/PrincipalToken.sol

47:     bool private ratesAtExpiryStored;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="GAS-3"></a>[GAS-3] State variables should be cached in stack variables rather than re-reading them from storage
The instances below point to the second+ access of a state variable within a function. Caching of a state variable replaces each Gwarmaccess (100 gas) with a much cheaper stack read. Other less obvious fixes/optimizations include having local memory caches of state variable structs, or having local caches of state variable contracts/addresses.

*Saves 100 gas per instance*

*Instances (9)*:
```solidity
File: src/tokens/PrincipalToken.sol

136:         __ERC20_init(name, NamingUtil.genPTSymbol(_ibtSymbol, expiry));

155:             NamingUtil.genYTName(_ibtSymbol, expiry),

156:             NamingUtil.genYTSymbol(_ibtSymbol, expiry)

182:         IERC20(_asset).safeIncreaseAllowance(ibt, assets);

183:         uint256 ibts = IERC4626(ibt).deposit(assets, address(this));

312:         IERC20(ibt).safeTransfer(receiver, ibts);

499:         return IERC4626(ibt).previewRedeem(IERC4626(ibt).balanceOf(address(this)));

628:         IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);

903:         if (IERC4626(ibt).totalAssets() == 0 && IERC4626(ibt).totalSupply() != 0) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="GAS-4"></a>[GAS-4] Use calldata instead of memory for function arguments that do not get mutated
When a function with a `memory` array is called externally, the `abi.decode()` step has to use a for-loop to copy each index of the `calldata` to the `memory` index. Each iteration of this for-loop costs at least 60 gas (i.e. `60 * <mem_array>.length`). Using `calldata` directly bypasses this loop. 

If the array is passed to an `internal` function which passes the array to another internal function where the array is modified and therefore `memory` is used in the `external` call, it's still more gas-efficient to use `calldata` when the `external` function uses modifiers, since the modifiers may prevent the internal functions from being called. Structs have the same overhead as an array of length one. 

 *Saves 60 gas per instance*

*Instances (2)*:
```solidity
File: src/proxy/AMProxyAdmin.sol

43:         bytes memory data

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol)

```solidity
File: src/tokens/PrincipalToken.sol

394:     function claimRewards(bytes memory _data) external restricted {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="GAS-5"></a>[GAS-5] For Operations that will not overflow, you could use unchecked

*Instances (60)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

5: import "../interfaces/IYieldToken.sol";

6: import "../interfaces/IPrincipalToken.sol";

7: import "../interfaces/IRegistry.sol";

8: import "openzeppelin-contracts/interfaces/IERC4626.sol";

9: import "openzeppelin-math/Math.sol";

10: import "../libraries/RayMath.sol";

18:     uint256 private constant SAFETY_BOUND = 100; // used to favour the protocol in case of approximations

19:     uint256 private constant FEE_DIVISOR = 1e18; // equivalent to 100% fees

75:             newYieldInIBTRay = ibtOfPTInRay.mulDiv(_ibtRate - _oldIBTRate, _ibtRate);

85:                             _oldPTRate - _ptRate,

88:                         ) +

91:                             _ibtRate - _oldIBTRate,

100:                         _oldPTRate - _ptRate,

105:                         ibtOfPTInRay * (_oldIBTRate - _ibtRate),

111:                         : actualNegativeYieldInAssetRay - expectedNegativeYieldInAssetRay;

129:         return _userYieldIBT + newYieldInIBTRay.fromRay(IERC20Metadata(_yt).decimals());

166:                     FEE_DIVISOR - IRegistry(_registry).getFeeReduction(_pt, msg.sender),

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

```solidity
File: src/libraries/RayMath.sol

21:         uint256 decimals_ratio = 10 ** (27 - _decimals);

40:         uint256 decimals_ratio = 10 ** (27 - _decimals);

58:         uint256 decimals_ratio = 10 ** (27 - _decimals);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol)

```solidity
File: src/proxy/AMBeacon.sol

7: import {IBeacon} from "openzeppelin-contracts/proxy/beacon/IBeacon.sol";

8: import "openzeppelin-contracts/access/manager/AccessManaged.sol";

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol)

```solidity
File: src/proxy/AMProxyAdmin.sol

7: import {IAMTransparentUpgradeableProxy} from "./AMTransparentUpgradeableProxy.sol";

8: import "openzeppelin-contracts/access/manager/AccessManaged.sol";

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol)

```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

6: import {ERC1967Utils} from "openzeppelin-contracts/proxy/ERC1967/ERC1967Utils.sol";

7: import {ERC1967Proxy} from "openzeppelin-contracts/proxy/ERC1967/ERC1967Proxy.sol";

8: import {IERC1967} from "openzeppelin-contracts/interfaces/IERC1967.sol";

9: import {AMProxyAdmin} from "./AMProxyAdmin.sol";

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)

```solidity
File: src/tokens/PrincipalToken.sol

15: import "../libraries/PrincipalTokenUtil.sol";

16: import "../libraries/NamingUtil.sol";

17: import "../libraries/RayMath.sol";

18: import "../interfaces/IRegistry.sol";

19: import "../interfaces/IPrincipalToken.sol";

20: import "../interfaces/IYieldToken.sol";

21: import "../interfaces/IRewardsProxy.sol";

48:     address private ibt; // address of the Interest Bearing Token 4626 held by this PT vault

49:     address private _asset; // the asset of this PT vault (which is also the asset of the IBT 4626)

50:     address private yt; // YT corresponding to this PT, deployed at initialization

51:     uint256 private ibtUnit; // equal to one unit of the IBT held by this PT vault (10^decimals)

55:     uint256 private ptRate; // or PT price in asset (in Ray)

56:     uint256 private ibtRate; // or IBT price in asset (in Ray)

57:     uint256 private unclaimedFeesInIBT; // unclaimed fees

58:     uint256 private totalFeesInIBT; // total fees

59:     uint256 private expiry; // date of maturity (set at initialization)

60:     uint256 private duration; // duration to maturity

62:     mapping(address => uint256) private ibtRateOfUser; // stores each user's IBT rate (in Ray)

63:     mapping(address => uint256) private ptRateOfUser; // stores each user's PT rate (in Ray)

64:     mapping(address => uint256) private yieldOfUserInIBT; // stores each user's yield generated from YTs

107:         _disableInitializers(); // using this so that the deployed logic contract cannot later be initialized

133:         expiry = _duration + block.timestamp;

151:         ibtUnit = 10 ** _ibtDecimals;

576:             _yieldOfUserInIBT -= PrincipalTokenUtil._computeYieldFee(_yieldOfUserInIBT, registry);

628:         IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);

650:         return _convertIBTsToSharesPreview(_ibts - tokenizationFee);

702:         (uint256 _ptRate, uint256 _ibtRate) = _getPTandIBTRates(true); // to round up the shares, the PT rate must round down

714:         unclaimedFeesInIBT += _feesInIBT;

715:         totalFeesInIBT += _feesInIBT;

762:         shares = _convertIBTsToShares(_ibts - tokenizationFee, false);

856:             yieldInIBT -= yieldFeeInIBT;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

22:         _disableInitializers(); // using this so that the deployed logic contract later cannot be initialized.

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="GAS-6"></a>[GAS-6] Avoid contract existence checks by using low level calls
Prior to 0.8.10 the compiler inserted extra code, including `EXTCODESIZE` (**100 gas**), to check for contract existence for external function calls. In more recent solidity versions, the compiler will not insert these checks if the external call has a return value. Similar behavior can be achieved in earlier versions by using low-level calls, since low level calls never check for contract existence

*Instances (6)*:
```solidity
File: src/tokens/PrincipalToken.sol

399:         (bool success, ) = rewardsProxy.delegatecall(_data);

499:         return IERC4626(ibt).previewRedeem(IERC4626(ibt).balanceOf(address(this)));

588:         return IERC4626(ibt).balanceOf(address(this));

871:             uint256 ytBalance = IYieldToken(yt).balanceOf(_user);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

124:         return (block.timestamp < IPrincipalToken(pt).maturity()) ? super.balanceOf(account) : 0;

129:         return super.balanceOf(account);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="GAS-7"></a>[GAS-7] State variables only set in the constructor should be declared `immutable`
Variables only set in the constructor and never edited afterwards should be marked as immutable, as it would avoid the expensive storage-writing operation in the constructor (around **20 000 gas** per variable) and replace the expensive storage-reading operations (around **2100 gas** per reading) to a less expensive value reading (**3 gas**)

*Instances (2)*:
```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

86:         _admin = address(new AMProxyAdmin(initialAuthority));

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)

```solidity
File: src/tokens/PrincipalToken.sol

106:         registry = _registry;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="GAS-8"></a>[GAS-8] Using `private` rather than `public` for constants, saves gas
If needed, the values can be read from the verified contract source code, or if there are multiple values there can be a single getter function that [returns a tuple](https://github.com/code-423n4/2022-08-frax/blob/90f55a9ce4e25bceed3a74290b854341d8de6afa/src/contracts/FraxlendPair.sol#L156-L178) of the values of all currently-public constants. Saves **3406-3606 gas** in deployment gas due to the compiler not having to create non-payable getter functions for deployment calldata, not having to store the bytes of the value outside of where it's used, and not adding another entry to the method ID table

*Instances (2)*:
```solidity
File: src/libraries/RayMath.sol

12:     uint256 public constant RAY_UNIT = 1e27;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol)

```solidity
File: src/proxy/AMProxyAdmin.sol

23:     string public constant UPGRADE_INTERFACE_VERSION = "5.0.0";

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol)

### <a name="GAS-9"></a>[GAS-9] `internal` functions not called by the contract should be removed
If the functions are required by an interface, the contract should inherit from that interface and use the `override` keyword

*Instances (6)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

157:     function _computeTokenizationFee(

178:     function _computeYieldFee(uint256 _amount, address _registry) internal view returns (uint256) {

188:     function _computeFlashloanFee(

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

```solidity
File: src/libraries/RayMath.sol

20:     function fromRay(uint256 _a, uint256 _decimals) internal pure returns (uint256 b) {

35:     function fromRay(

57:     function toRay(uint256 _a, uint256 _decimals) internal pure returns (uint256 b) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol)


## Non Critical Issues


| |Issue|Instances|
|-|:-|:-:|
| [NC-1](#NC-1) | Replace `abi.encodeWithSignature` and `abi.encodeWithSelector` with `abi.encodeCall` which keeps the code typo/type safe | 2 |
| [NC-2](#NC-2) | `constant`s should be defined rather than using magic numbers | 5 |
| [NC-3](#NC-3) | Control structures do not follow the Solidity Style Guide | 4 |
| [NC-4](#NC-4) | Events that mark critical parameter changes should contain both the old and the new value | 2 |
| [NC-5](#NC-5) | Function ordering does not follow the Solidity style guide | 2 |
| [NC-6](#NC-6) | Functions should not be longer than 50 lines | 58 |
| [NC-7](#NC-7) | Interfaces should be defined in separate files from their usage | 1 |
| [NC-8](#NC-8) | Lack of checks in setters | 2 |
| [NC-9](#NC-9) | NatSpec is completely non-existent on functions that should have them | 1 |
| [NC-10](#NC-10) | Incomplete NatSpec: `@param` is missing on actually documented functions | 24 |
| [NC-11](#NC-11) | Incomplete NatSpec: `@return` is missing on actually documented functions | 19 |
| [NC-12](#NC-12) | Use a `modifier` instead of a `require/if` statement for a special `msg.sender` actor | 8 |
| [NC-13](#NC-13) | Consider using named mappings | 3 |
| [NC-14](#NC-14) | Adding a `return` statement when the function defines a named return variable, is redundant | 5 |
| [NC-15](#NC-15) | `require()` / `revert()` statements should have descriptive reason strings | 2 |
| [NC-16](#NC-16) | Take advantage of Custom Error's return value property | 36 |
| [NC-17](#NC-17) | Contract does not follow the Solidity style guide's suggested layout ordering | 2 |
| [NC-18](#NC-18) | Use Underscores for Number Literals (add an underscore every 3 digits) | 2 |
| [NC-19](#NC-19) | Internal and private variables and functions names should begin with an underscore | 18 |
| [NC-20](#NC-20) | Event is missing `indexed` fields | 2 |
| [NC-21](#NC-21) | Constants should be defined rather than using magic numbers | 3 |
### <a name="NC-1"></a>[NC-1] Replace `abi.encodeWithSignature` and `abi.encodeWithSelector` with `abi.encodeCall` which keeps the code typo/type safe
When using `abi.encodeWithSignature`, it is possible to include a typo for the correct function signature.
When using `abi.encodeWithSignature` or `abi.encodeWithSelector`, it is also possible to provide parameters that are not of the correct type for the function.

To avoid these pitfalls, it would be best to use [`abi.encodeCall`](https://solidity-by-example.org/abi-encode/) instead.

*Instances (2)*:
```solidity
File: src/tokens/PrincipalToken.sol

398:         _data = abi.encodeWithSelector(IRewardsProxy(rewardsProxy).claimRewards.selector, _data);

732:                 abi.encodeWithSelector(

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="NC-2"></a>[NC-2] `constant`s should be defined rather than using magic numbers
Even [assembly](https://github.com/code-423n4/2022-05-opensea-seaport/blob/9d7ce4d08bf3c3010304a0476a785c70c0e90ae7/contracts/lib/TokenTransferrer.sol#L35-L39) can benefit from using readable constants instead of hex/numeric literals

*Instances (5)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

141:         if (success && encodedDecimals.length >= 32) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

```solidity
File: src/libraries/RayMath.sol

21:         uint256 decimals_ratio = 10 ** (27 - _decimals);

40:         uint256 decimals_ratio = 10 ** (27 - _decimals);

58:         uint256 decimals_ratio = 10 ** (27 - _decimals);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol)

```solidity
File: src/tokens/PrincipalToken.sol

151:         ibtUnit = 10 ** _ibtDecimals;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="NC-3"></a>[NC-3] Control structures do not follow the Solidity Style Guide
See the [control structures](https://docs.soliditylang.org/en/latest/style-guide.html#control-structures) section of the Solidity Style Guide

*Instances (4)*:
```solidity
File: src/tokens/PrincipalToken.sol

143:         if (

595:         if (_token != ibt) revert AddressError();

615:         if (_amount > maxFlashLoan(_token)) revert FlashLoanExceedsMaxAmount();

624:         if (_receiver.onFlashLoan(msg.sender, _token, _amount, fee, _data) != ON_FLASH_LOAN)

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="NC-4"></a>[NC-4] Events that mark critical parameter changes should contain both the old and the new value
This should especially be done if the new value is not required to be different from the old value

*Instances (2)*:
```solidity
File: src/tokens/PrincipalToken.sol

340:     function updateYield(address _user) public override returns (uint256 updatedUserYieldInIBT) {
             (uint256 _ptRate, uint256 _ibtRate) = _updatePTandIBTRates();
     
             uint256 _oldIBTRateUser = ibtRateOfUser[_user];
             if (_oldIBTRateUser != _ibtRate) {
                 ibtRateOfUser[_user] = _ibtRate;
             }
             uint256 _oldPTRateUser = ptRateOfUser[_user];
             if (_oldPTRateUser != _ptRate) {
                 ptRateOfUser[_user] = _ptRate;
             }
     
             // Check for skipping yield update when the user deposits for the first time or rates decreased to 0.
             if (_oldIBTRateUser != 0) {
                 updatedUserYieldInIBT = PrincipalTokenUtil._computeYield(
                     _user,
                     yieldOfUserInIBT[_user],
                     _oldIBTRateUser,
                     _ibtRate,
                     _oldPTRateUser,
                     _ptRate,
                     yt
                 );
                 yieldOfUserInIBT[_user] = updatedUserYieldInIBT;
                 emit YieldUpdated(_user, updatedUserYieldInIBT);

420:     function setRewardsProxy(address _rewardsProxy) external restricted {
             // Note: address zero is allowed in order to disable the claim proxy
             emit RewardsProxyChange(rewardsProxy, _rewardsProxy);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="NC-5"></a>[NC-5] Function ordering does not follow the Solidity style guide
According to the [Solidity style guide](https://docs.soliditylang.org/en/v0.8.17/style-guide.html#order-of-functions), functions should be laid out in the following order :`constructor()`, `receive()`, `fallback()`, `external`, `public`, `internal`, `private`, but the cases below do not follow this pattern

*Instances (2)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

1: 
   Current order:
   internal _convertToSharesWithRate
   internal _convertToAssetsWithRate
   external _computeYield
   external _tryGetTokenDecimals
   internal _computeTokenizationFee
   internal _computeYieldFee
   internal _computeFlashloanFee
   
   Suggested order:
   external _computeYield
   external _tryGetTokenDecimals
   internal _convertToSharesWithRate
   internal _convertToAssetsWithRate
   internal _computeTokenizationFee
   internal _computeYieldFee
   internal _computeFlashloanFee

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

```solidity
File: src/tokens/PrincipalToken.sol

1: 
   Current order:
   external initialize
   external pause
   external unPause
   external deposit
   public deposit
   external deposit
   external depositIBT
   public depositIBT
   external depositIBT
   public redeem
   external redeem
   public redeemForIBT
   external redeemForIBT
   public withdraw
   external withdraw
   public withdrawIBT
   external withdrawIBT
   external claimFees
   public updateYield
   public claimYield
   public claimYieldInIBT
   external beforeYtTransfer
   external claimRewards
   public storeRatesAtExpiry
   external setRewardsProxy
   public previewDeposit
   external previewDepositIBT
   external maxDeposit
   external previewWithdraw
   public previewWithdrawIBT
   public maxWithdraw
   public maxWithdrawIBT
   public previewRedeem
   public previewRedeemForIBT
   public maxRedeem
   external convertToPrincipal
   public convertToUnderlying
   public totalAssets
   public decimals
   external underlying
   external maturity
   external getDuration
   external getIBT
   external getYT
   external getIBTRate
   external getPTRate
   external getIBTUnit
   external getUnclaimedFeesInIBT
   external getTotalFeesInIBT
   external getCurrentYieldOfUserInIBT
   public maxFlashLoan
   public flashFee
   public getTokenizationFee
   external flashLoan
   internal _previewDepositIBT
   internal _convertSharesToIBTs
   internal _convertIBTsToShares
   internal _convertIBTsToSharesPreview
   internal _updateFees
   internal _deployYT
   internal _depositIBT
   internal _withdrawShares
   internal _beforeRedeem
   internal _beforeWithdraw
   internal _claimYield
   internal _maxBurnable
   internal _updatePTandIBTRates
   internal _getCurrentPTandIBTRates
   internal _getPTandIBTRates
   
   Suggested order:
   external initialize
   external pause
   external unPause
   external deposit
   external deposit
   external depositIBT
   external depositIBT
   external redeem
   external redeemForIBT
   external withdraw
   external withdrawIBT
   external claimFees
   external beforeYtTransfer
   external claimRewards
   external setRewardsProxy
   external previewDepositIBT
   external maxDeposit
   external previewWithdraw
   external convertToPrincipal
   external underlying
   external maturity
   external getDuration
   external getIBT
   external getYT
   external getIBTRate
   external getPTRate
   external getIBTUnit
   external getUnclaimedFeesInIBT
   external getTotalFeesInIBT
   external getCurrentYieldOfUserInIBT
   external flashLoan
   public deposit
   public depositIBT
   public redeem
   public redeemForIBT
   public withdraw
   public withdrawIBT
   public updateYield
   public claimYield
   public claimYieldInIBT
   public storeRatesAtExpiry
   public previewDeposit
   public previewWithdrawIBT
   public maxWithdraw
   public maxWithdrawIBT
   public previewRedeem
   public previewRedeemForIBT
   public maxRedeem
   public convertToUnderlying
   public totalAssets
   public decimals
   public maxFlashLoan
   public flashFee
   public getTokenizationFee
   internal _previewDepositIBT
   internal _convertSharesToIBTs
   internal _convertIBTsToShares
   internal _convertIBTsToSharesPreview
   internal _updateFees
   internal _deployYT
   internal _depositIBT
   internal _withdrawShares
   internal _beforeRedeem
   internal _beforeWithdraw
   internal _claimYield
   internal _maxBurnable
   internal _updatePTandIBTRates
   internal _getCurrentPTandIBTRates
   internal _getPTandIBTRates

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="NC-6"></a>[NC-6] Functions should not be longer than 50 lines
Overly complex code can make understanding functionality more difficult, try to further modularize your code to ensure readability 

*Instances (58)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

137:     function _tryGetTokenDecimals(address _token) external view returns (uint8) {

178:     function _computeYieldFee(uint256 _amount, address _registry) internal view returns (uint256) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

```solidity
File: src/libraries/RayMath.sol

20:     function fromRay(uint256 _a, uint256 _decimals) internal pure returns (uint256 b) {

57:     function toRay(uint256 _a, uint256 _decimals) internal pure returns (uint256 b) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol)

```solidity
File: src/proxy/AMBeacon.sol

46:     function implementation() public view virtual returns (address) {

63:     function upgradeTo(address newImplementation) public virtual restricted {

75:     function _setImplementation(address newImplementation) private {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol)

```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

18:     function upgradeToAndCall(address, bytes calldata) external payable;

94:     function _proxyAdmin() internal virtual returns (address) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)

```solidity
File: src/tokens/PrincipalToken.sol

171:     function deposit(uint256 assets, address receiver) external override returns (uint256 shares) {

201:     function depositIBT(uint256 ibts, address receiver) external override returns (uint256 shares) {

329:     function claimFees() external override returns (uint256 assets) {

340:     function updateYield(address _user) public override returns (uint256 updatedUserYieldInIBT) {

369:     function claimYield(address _receiver) public override returns (uint256 yieldInAsset) {

377:     function claimYieldInIBT(address _receiver) public override returns (uint256 yieldInIBT) {

385:     function beforeYtTransfer(address _from, address _to) external override {

394:     function claimRewards(bytes memory _data) external restricted {

409:     function storeRatesAtExpiry() public override afterExpiry {

420:     function setRewardsProxy(address _rewardsProxy) external restricted {

430:     function previewDeposit(uint256 assets) public view override returns (uint256) {

436:     function previewDepositIBT(uint256 ibts) external view override returns (uint256) {

441:     function maxDeposit(address) external pure override returns (uint256) {

454:     function previewWithdrawIBT(uint256 ibts) public view override whenNotPaused returns (uint256) {

460:     function maxWithdraw(address owner) public view override whenNotPaused returns (uint256) {

466:     function maxWithdrawIBT(address owner) public view override whenNotPaused returns (uint256) {

471:     function previewRedeem(uint256 shares) public view override returns (uint256) {

483:     function maxRedeem(address owner) public view override returns (uint256) {

488:     function convertToPrincipal(uint256 underlyingAmount) external view override returns (uint256) {

493:     function convertToUnderlying(uint256 principalAmount) public view override returns (uint256) {

498:     function totalAssets() public view override returns (uint256) {

503:     function decimals() public view override(IERC20Metadata, ERC20Upgradeable) returns (uint8) {

508:     function underlying() external view override returns (address) {

513:     function maturity() external view override returns (uint256) {

518:     function getDuration() external view override returns (uint256) {

523:     function getIBT() external view override returns (address) {

528:     function getYT() external view override returns (address) {

533:     function getIBTRate() external view override returns (uint256) {

539:     function getPTRate() external view override returns (uint256) {

545:     function getIBTUnit() external view override returns (uint256) {

550:     function getUnclaimedFeesInIBT() external view override returns (uint256) {

555:     function getTotalFeesInIBT() external view override returns (uint256) {

583:     function maxFlashLoan(address _token) public view override returns (uint256) {

594:     function flashFee(address _token, uint256 _amount) public view override returns (uint256) {

602:     function getTokenizationFee() public view override returns (uint256) {

701:     function _convertIBTsToSharesPreview(uint256 ibts) internal view returns (uint256 shares) {

713:     function _updateFees(uint256 _feesInIBT) internal {

724:     function _deployYT(string memory _name, string memory _symbol) internal returns (address _yt) {

805:     function _beforeRedeem(uint256 _shares, address _owner) internal nonReentrant whenNotPaused {

828:     function _beforeWithdraw(uint256 _assets, address _owner) internal whenNotPaused nonReentrant {

848:     function _claimYield() internal returns (uint256 yieldInIBT) {

866:     function _maxBurnable(address _user) internal view returns (uint256 maxBurnable) {

879:     function _updatePTandIBTRates() internal returns (uint256 _ptRate, uint256 _ibtRate) {

901:     function _getCurrentPTandIBTRates(bool roundUpPTRate) internal view returns (uint256, uint256) {

921:     function _getPTandIBTRates(bool roundUpPTRate) internal view returns (uint256, uint256) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

42:     function burnWithoutUpdate(address from, uint256 amount) external override {

50:     function mint(address to, uint256 amount) external override {

116:     function getPT() public view virtual override returns (address) {

128:     function actualBalanceOf(address account) public view override returns (uint256) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="NC-7"></a>[NC-7] Interfaces should be defined in separate files from their usage
The interfaces below should be defined in separate files, so that it's easier for future projects to import them, and to avoid duplication later on if they need to be used elsewhere in the project

*Instances (1)*:
```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

17: interface IAMTransparentUpgradeableProxy is IERC1967 {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)

### <a name="NC-8"></a>[NC-8] Lack of checks in setters
Be it sanity checks (like checks against `0`-values) or initial setting checks: it's best for Setter functions to have them

*Instances (2)*:
```solidity
File: src/tokens/PrincipalToken.sol

420:     function setRewardsProxy(address _rewardsProxy) external restricted {
             // Note: address zero is allowed in order to disable the claim proxy
             emit RewardsProxyChange(rewardsProxy, _rewardsProxy);
             rewardsProxy = _rewardsProxy;

713:     function _updateFees(uint256 _feesInIBT) internal {
             unclaimedFeesInIBT += _feesInIBT;
             totalFeesInIBT += _feesInIBT;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="NC-9"></a>[NC-9] NatSpec is completely non-existent on functions that should have them
Public and external functions that aren't view or pure should have NatSpec comments

*Instances (1)*:
```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

18:     function upgradeToAndCall(address, bytes calldata) external payable;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)

### <a name="NC-10"></a>[NC-10] Incomplete NatSpec: `@param` is missing on actually documented functions
The following functions are missing `@param` NatSpec comments.

*Instances (24)*:
```solidity
File: src/tokens/PrincipalToken.sol

170:     /** @dev See {IPrincipalToken-deposit}. */
         function deposit(uint256 assets, address receiver) external override returns (uint256 shares) {

175:     /** @dev See {IPrincipalToken-deposit}. */
         function deposit(
             uint256 assets,
             address ptReceiver,
             address ytReceiver

187:     /** @dev See {IPrincipalToken-deposit}. */
         function deposit(
             uint256 assets,
             address ptReceiver,
             address ytReceiver,
             uint256 minShares

200:     /** @dev See {IPrincipalToken-depositIBT}. */
         function depositIBT(uint256 ibts, address receiver) external override returns (uint256 shares) {

205:     /** @dev See {IPrincipalToken-depositIBT}. */
         function depositIBT(
             uint256 ibts,
             address ptReceiver,
             address ytReceiver

215:     /** @dev See {IPrincipalToken-depositIBT}. */
         function depositIBT(
             uint256 ibts,
             address ptReceiver,
             address ytReceiver,
             uint256 minShares

228:     /** @dev See {IPrincipalToken-redeem}. */
         function redeem(
             uint256 shares,
             address receiver,
             address owner

239:     /** @dev See {IPrincipalToken-redeem}. */
         function redeem(
             uint256 shares,
             address receiver,
             address owner,
             uint256 minAssets

252:     /** @dev See {IPrincipalToken-redeemForIBT}. */
         function redeemForIBT(
             uint256 shares,
             address receiver,
             address owner

264:     /** @dev See {IPrincipalToken-redeemForIBT}. */
         function redeemForIBT(
             uint256 shares,
             address receiver,
             address owner,
             uint256 minIbts

277:     /** @dev See {IPrincipalToken-withdraw}. */
         function withdraw(
             uint256 assets,
             address receiver,
             address owner

289:     /** @dev See {IPrincipalToken-withdraw}. */
         function withdraw(
             uint256 assets,
             address receiver,
             address owner,
             uint256 maxShares

302:     /** @dev See {IPrincipalToken-withdrawIBT}. */
         function withdrawIBT(
             uint256 ibts,
             address receiver,
             address owner

315:     /** @dev See {IPrincipalToken-withdrawIBT}. */
         function withdrawIBT(
             uint256 ibts,
             address receiver,
             address owner,
             uint256 maxShares

339:     /** @dev See {IPrincipalToken-updateYield}. */
         function updateYield(address _user) public override returns (uint256 updatedUserYieldInIBT) {

368:     /** @dev See {IPrincipalToken-claimYield}. */
         function claimYield(address _receiver) public override returns (uint256 yieldInAsset) {

376:     /** @dev See {IPrincipalToken-claimYieldInIBT}. */
         function claimYieldInIBT(address _receiver) public override returns (uint256 yieldInIBT) {

384:     /** @dev See {IPrincipalToken-beforeYtTransfer}. */
         function beforeYtTransfer(address _from, address _to) external override {

393:     /** @dev See {IPrincipalToken-claimRewards}. */
         function claimRewards(bytes memory _data) external restricted {

419:     /** @dev See {IPrincipalToken-setRewardsProxy}. */
         function setRewardsProxy(address _rewardsProxy) external restricted {

606:     /**
          * @dev See {IERC3156FlashLender-flashLoan}.
          */
         function flashLoan(
             IERC3156FlashBorrower _receiver,
             address _token,
             uint256 _amount,
             bytes calldata _data

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

41:     /** @dev See {IYieldToken-burnWithoutUpdate} */
        function burnWithoutUpdate(address from, uint256 amount) external override {

49:     /** @dev See {IYieldToken-mint} */
        function mint(address to, uint256 amount) external override {

57:     /** @dev See {IYieldToken-burn} */
        function burn(uint256 amount) public override {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="NC-11"></a>[NC-11] Incomplete NatSpec: `@return` is missing on actually documented functions
The following functions are missing `@return` NatSpec comments.

*Instances (19)*:
```solidity
File: src/tokens/PrincipalToken.sol

170:     /** @dev See {IPrincipalToken-deposit}. */
         function deposit(uint256 assets, address receiver) external override returns (uint256 shares) {

175:     /** @dev See {IPrincipalToken-deposit}. */
         function deposit(
             uint256 assets,
             address ptReceiver,
             address ytReceiver
         ) public override returns (uint256 shares) {

187:     /** @dev See {IPrincipalToken-deposit}. */
         function deposit(
             uint256 assets,
             address ptReceiver,
             address ytReceiver,
             uint256 minShares
         ) external override returns (uint256 shares) {

200:     /** @dev See {IPrincipalToken-depositIBT}. */
         function depositIBT(uint256 ibts, address receiver) external override returns (uint256 shares) {

205:     /** @dev See {IPrincipalToken-depositIBT}. */
         function depositIBT(
             uint256 ibts,
             address ptReceiver,
             address ytReceiver
         ) public override returns (uint256 shares) {

215:     /** @dev See {IPrincipalToken-depositIBT}. */
         function depositIBT(
             uint256 ibts,
             address ptReceiver,
             address ytReceiver,
             uint256 minShares
         ) external override returns (uint256 shares) {

228:     /** @dev See {IPrincipalToken-redeem}. */
         function redeem(
             uint256 shares,
             address receiver,
             address owner
         ) public override returns (uint256 assets) {

239:     /** @dev See {IPrincipalToken-redeem}. */
         function redeem(
             uint256 shares,
             address receiver,
             address owner,
             uint256 minAssets
         ) external override returns (uint256 assets) {

252:     /** @dev See {IPrincipalToken-redeemForIBT}. */
         function redeemForIBT(
             uint256 shares,
             address receiver,
             address owner
         ) public override returns (uint256 ibts) {

264:     /** @dev See {IPrincipalToken-redeemForIBT}. */
         function redeemForIBT(
             uint256 shares,
             address receiver,
             address owner,
             uint256 minIbts
         ) external override returns (uint256 ibts) {

277:     /** @dev See {IPrincipalToken-withdraw}. */
         function withdraw(
             uint256 assets,
             address receiver,
             address owner
         ) public override returns (uint256 shares) {

289:     /** @dev See {IPrincipalToken-withdraw}. */
         function withdraw(
             uint256 assets,
             address receiver,
             address owner,
             uint256 maxShares
         ) external override returns (uint256 shares) {

302:     /** @dev See {IPrincipalToken-withdrawIBT}. */
         function withdrawIBT(
             uint256 ibts,
             address receiver,
             address owner
         ) public override returns (uint256 shares) {

315:     /** @dev See {IPrincipalToken-withdrawIBT}. */
         function withdrawIBT(
             uint256 ibts,
             address receiver,
             address owner,
             uint256 maxShares
         ) external override returns (uint256 shares) {

328:     /** @dev See {IPrincipalToken-claimFees}. */
         function claimFees() external override returns (uint256 assets) {

339:     /** @dev See {IPrincipalToken-updateYield}. */
         function updateYield(address _user) public override returns (uint256 updatedUserYieldInIBT) {

368:     /** @dev See {IPrincipalToken-claimYield}. */
         function claimYield(address _receiver) public override returns (uint256 yieldInAsset) {

376:     /** @dev See {IPrincipalToken-claimYieldInIBT}. */
         function claimYieldInIBT(address _receiver) public override returns (uint256 yieldInIBT) {

606:     /**
          * @dev See {IERC3156FlashLender-flashLoan}.
          */
         function flashLoan(
             IERC3156FlashBorrower _receiver,
             address _token,
             uint256 _amount,
             bytes calldata _data
         ) external override returns (bool) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="NC-12"></a>[NC-12] Use a `modifier` instead of a `require/if` statement for a special `msg.sender` actor
If a function is supposed to be access-controlled, a `modifier` should be used instead of a `require/if` statement for more readability.

*Instances (8)*:
```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

102:         if (msg.sender == _proxyAdmin()) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)

```solidity
File: src/tokens/PrincipalToken.sol

330:         if (msg.sender != IRegistry(registry).getFeeCollector()) {

386:         if (msg.sender != yt) {

624:         if (_receiver.onFlashLoan(msg.sender, _token, _amount, fee, _data) != ON_FLASH_LOAN)

806:         if (_owner != msg.sender) {

829:         if (_owner != msg.sender) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

43:         if (msg.sender != pt) {

51:         if (msg.sender != pt) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="NC-13"></a>[NC-13] Consider using named mappings
Consider moving to solidity version 0.8.18 or later, and using [named mappings](https://ethereum.stackexchange.com/questions/51629/how-to-name-the-arguments-in-mapping/145555#145555) to make it easier to understand the purpose of each mapping

*Instances (3)*:
```solidity
File: src/tokens/PrincipalToken.sol

62:     mapping(address => uint256) private ibtRateOfUser; // stores each user's IBT rate (in Ray)

63:     mapping(address => uint256) private ptRateOfUser; // stores each user's PT rate (in Ray)

64:     mapping(address => uint256) private yieldOfUserInIBT; // stores each user's yield generated from YTs

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="NC-14"></a>[NC-14] Adding a `return` statement when the function defines a named return variable, is redundant

*Instances (5)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

21:     /** @dev See {IPrincipalToken-convertToSharesWithRate}. */
        function _convertToSharesWithRate(
            uint256 _assets,
            uint256 _rate,
            uint256 _ibtUnit,
            Math.Rounding _rounding
        ) internal pure returns (uint256 shares) {
            if (_rate == 0) {
                revert IPrincipalToken.RateError();
            }
            return _assets.mulDiv(_ibtUnit, _rate, _rounding);

34:     /** @dev See {IPrincipalToken-convertToAssetsWithRate}. */
        function _convertToAssetsWithRate(
            uint256 _shares,
            uint256 _rate,
            uint256 _ibtUnit,
            Math.Rounding _rounding
        ) internal pure returns (uint256 assets) {
            return _shares.mulDiv(_rate, _ibtUnit, _rounding);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

```solidity
File: src/tokens/PrincipalToken.sol

844:     /**
          * @dev Internal function for handling the claims of caller's unclaimed yield
          * @return yieldInIBT The unclaimed yield in IBT that is about to be claimed
          */
         function _claimYield() internal returns (uint256 yieldInIBT) {
             yieldInIBT = updateYield(msg.sender);
             if (yieldInIBT == 0) {
                 return 0;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

63:     /**
         * @dev See {IERC20-transfer}.
         *
         * Requirements:
         *
         * - `to` cannot be the zero address.
         * - the caller must have a balance of at least `amount`.
         */
        function transfer(
            address to,
            uint256 amount
        ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {
            IPrincipalToken(pt).beforeYtTransfer(msg.sender, to);
            return super.transfer(to, amount);

79:     /**
         * @dev See {IERC20-transferFrom}.
         *
         * Emits an {Approval} event indicating the updated allowance. This is not
         * required by the EIP. See the note at the beginning of {ERC20}.
         *
         * NOTE: Does not update the allowance if the current allowance
         * is the maximum `uint256`.
         *
         * Requirements:
         *
         * - `from` and `to` cannot be the zero address.
         * - `from` must have a balance of at least `amount`.
         * - the caller must have allowance for ``from``'s tokens of at least
         * `amount`.
         */
        function transferFrom(
            address from,
            address to,
            uint256 amount
        ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {
            IPrincipalToken(pt).beforeYtTransfer(from, to);
            return super.transferFrom(from, to, amount);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="NC-15"></a>[NC-15] `require()` / `revert()` statements should have descriptive reason strings

*Instances (2)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

29:             revert IPrincipalToken.RateError();

126:                 revert IPrincipalToken.RateError();

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

### <a name="NC-16"></a>[NC-16] Take advantage of Custom Error's return value property
An important feature of Custom Error is that values such as address, tokenID, msg.value can be written inside the () sign, this kind of approach provides a serious advantage in debugging and examining the revert details of dapps such as tenderly.

*Instances (36)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

29:             revert IPrincipalToken.RateError();

126:                 revert IPrincipalToken.RateError();

147:         revert AssetDoesNotImplementMetadata();

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

104:                 revert ProxyDeniedAdminAccess();

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)

```solidity
File: src/tokens/PrincipalToken.sol

87:             revert PTExpired();

95:             revert PTNotExpired();

104:             revert AddressError();

126:             revert AddressError();

129:             revert RateError();

148:             revert InvalidDecimals();

196:             revert ERC5143SlippageProtectionFailed();

224:             revert ERC5143SlippageProtectionFailed();

248:             revert ERC5143SlippageProtectionFailed();

273:             revert ERC5143SlippageProtectionFailed();

298:             revert ERC5143SlippageProtectionFailed();

324:             revert ERC5143SlippageProtectionFailed();

331:             revert UnauthorizedCaller();

387:             revert UnauthorizedCaller();

396:             revert NoRewardsProxySet();

401:             revert ClaimRewardsFailed();

411:             revert RatesAtExpiryAlreadyStored();

595:         if (_token != ibt) revert AddressError();

615:         if (_amount > maxFlashLoan(_token)) revert FlashLoanExceedsMaxAmount();

625:             revert FlashLoanCallbackFailed();

665:             revert RateError();

686:             revert RateError();

704:             revert RateError();

727:             revert BeaconNotSet();

764:             revert RateError();

788:             revert RateError();

807:             revert UnauthorizedCaller();

810:             revert UnsufficientBalance();

830:             revert UnauthorizedCaller();

840:             revert UnsufficientBalance();

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

44:             revert CallerIsNotPtContract();

52:             revert CallerIsNotPtContract();

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="NC-17"></a>[NC-17] Contract does not follow the Solidity style guide's suggested layout ordering
The [style guide](https://docs.soliditylang.org/en/v0.8.16/style-guide.html#order-of-layout) says that, within a contract, the ordering should be:

1) Type declarations
2) State variables
3) Events
4) Modifiers
5) Functions

However, the contract(s) below do not follow this ordering

*Instances (2)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

1: 
   Current order:
   UsingForDirective.Math
   UsingForDirective.RayMath
   ErrorDefinition.AssetDoesNotImplementMetadata
   VariableDeclaration.SAFETY_BOUND
   VariableDeclaration.FEE_DIVISOR
   FunctionDefinition._convertToSharesWithRate
   FunctionDefinition._convertToAssetsWithRate
   FunctionDefinition._computeYield
   FunctionDefinition._tryGetTokenDecimals
   FunctionDefinition._computeTokenizationFee
   FunctionDefinition._computeYieldFee
   FunctionDefinition._computeFlashloanFee
   
   Suggested order:
   UsingForDirective.Math
   UsingForDirective.RayMath
   VariableDeclaration.SAFETY_BOUND
   VariableDeclaration.FEE_DIVISOR
   ErrorDefinition.AssetDoesNotImplementMetadata
   FunctionDefinition._convertToSharesWithRate
   FunctionDefinition._convertToAssetsWithRate
   FunctionDefinition._computeYield
   FunctionDefinition._tryGetTokenDecimals
   FunctionDefinition._computeTokenizationFee
   FunctionDefinition._computeYieldFee
   FunctionDefinition._computeFlashloanFee

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

1: 
   Current order:
   FunctionDefinition.upgradeToAndCall
   VariableDeclaration._admin
   ErrorDefinition.ProxyDeniedAdminAccess
   FunctionDefinition.constructor
   FunctionDefinition._proxyAdmin
   FunctionDefinition._fallback
   FunctionDefinition._dispatchUpgradeToAndCall
   
   Suggested order:
   VariableDeclaration._admin
   ErrorDefinition.ProxyDeniedAdminAccess
   FunctionDefinition.upgradeToAndCall
   FunctionDefinition.constructor
   FunctionDefinition._proxyAdmin
   FunctionDefinition._fallback
   FunctionDefinition._dispatchUpgradeToAndCall

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)

### <a name="NC-18"></a>[NC-18] Use Underscores for Number Literals (add an underscore every 3 digits)

*Instances (2)*:
```solidity
File: src/tokens/PrincipalToken.sol

48:     address private ibt; // address of the Interest Bearing Token 4626 held by this PT vault

49:     address private _asset; // the asset of this PT vault (which is also the asset of the IBT 4626)

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="NC-19"></a>[NC-19] Internal and private variables and functions names should begin with an underscore
According to the Solidity Style Guide, Non-`external` variable and function names should begin with an [underscore](https://docs.soliditylang.org/en/latest/style-guide.html#underscore-prefix-for-non-external-functions-and-variables)

*Instances (18)*:
```solidity
File: src/libraries/RayMath.sol

20:     function fromRay(uint256 _a, uint256 _decimals) internal pure returns (uint256 b) {

35:     function fromRay(

57:     function toRay(uint256 _a, uint256 _decimals) internal pure returns (uint256 b) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol)

```solidity
File: src/tokens/PrincipalToken.sol

46:     address private rewardsProxy;

47:     bool private ratesAtExpiryStored;

48:     address private ibt; // address of the Interest Bearing Token 4626 held by this PT vault

50:     address private yt; // YT corresponding to this PT, deployed at initialization

51:     uint256 private ibtUnit; // equal to one unit of the IBT held by this PT vault (10^decimals)

55:     uint256 private ptRate; // or PT price in asset (in Ray)

56:     uint256 private ibtRate; // or IBT price in asset (in Ray)

57:     uint256 private unclaimedFeesInIBT; // unclaimed fees

58:     uint256 private totalFeesInIBT; // total fees

59:     uint256 private expiry; // date of maturity (set at initialization)

60:     uint256 private duration; // duration to maturity

62:     mapping(address => uint256) private ibtRateOfUser; // stores each user's IBT rate (in Ray)

63:     mapping(address => uint256) private ptRateOfUser; // stores each user's PT rate (in Ray)

64:     mapping(address => uint256) private yieldOfUserInIBT; // stores each user's yield generated from YTs

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

18:     address private pt;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="NC-20"></a>[NC-20] Event is missing `indexed` fields
Index event fields make the field more quickly accessible to off-chain tools that parse events. However, note that each index field costs extra gas during emission, so it's not necessarily best to index the maximum allowed per event (three fields). Each event should use three indexed fields if there are three or more fields, and gas usage is not particularly of concern for the events in question. If there are fewer than three fields, all of the fields should be indexed.

*Instances (2)*:
```solidity
File: src/tokens/PrincipalToken.sol

68:     event Redeem(address indexed from, address indexed to, uint256 amount);

69:     event Mint(address indexed from, address indexed to, uint256 amount);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="NC-21"></a>[NC-21] Constants should be defined rather than using magic numbers

*Instances (3)*:
```solidity
File: src/libraries/RayMath.sol

21:         uint256 decimals_ratio = 10 ** (27 - _decimals);

40:         uint256 decimals_ratio = 10 ** (27 - _decimals);

58:         uint256 decimals_ratio = 10 ** (27 - _decimals);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol)


## Low Issues


| |Issue|Instances|
|-|:-|:-:|
| [L-1](#L-1) | Some tokens may revert when zero value transfers are made | 7 |
| [L-2](#L-2) | `decimals()` is not a part of the ERC-20 standard | 6 |
| [L-3](#L-3) | `decimals()` should be of type `uint8` | 3 |
| [L-4](#L-4) | Fallback lacking `payable` | 2 |
| [L-5](#L-5) | Initializers could be front-run | 11 |
| [L-6](#L-6) | Prevent accidentally burning tokens | 6 |
| [L-7](#L-7) | Solidity version 0.8.20+ may not work on other chains due to `PUSH0` | 6 |
| [L-8](#L-8) | `symbol()` is not a part of the ERC-20 standard | 1 |
| [L-9](#L-9) | Unsafe ERC20 operation(s) | 2 |
| [L-10](#L-10) | Upgradeable contract is missing a `__gap[50]` storage variable to allow for new storage variables in later versions | 16 |
| [L-11](#L-11) | Upgradeable contract not initialized | 30 |
### <a name="L-1"></a>[L-1] Some tokens may revert when zero value transfers are made
Example: https://github.com/d-xo/weird-erc20#revert-on-zero-value-transfers.

In spite of the fact that EIP-20 [states](https://github.com/ethereum/EIPs/blob/46b9b698815abbfa628cd1097311deee77dd45c5/EIPS/eip-20.md?plain=1#L116) that zero-valued transfers must be accepted, some tokens, such as LEND will revert if this is attempted, which may cause transactions that involve other tokens (such as batch operations) to fully revert. Consider skipping the transfer if the amount is zero, which will also save gas.

*Instances (7)*:
```solidity
File: src/tokens/PrincipalToken.sol

181:         IERC20(_asset).safeTransferFrom(msg.sender, address(this), assets);

211:         IERC20(ibt).safeTransferFrom(msg.sender, address(this), ibts);

260:         IERC20(ibt).safeTransfer(receiver, ibts);

312:         IERC20(ibt).safeTransfer(receiver, ibts);

380:             IERC20(ibt).safeTransfer(_receiver, yieldInIBT);

621:         IERC20(ibt).safeTransfer(address(_receiver), _amount);

628:         IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="L-2"></a>[L-2] `decimals()` is not a part of the ERC-20 standard
The `decimals()` function is not a part of the [ERC-20 standard](https://eips.ethereum.org/EIPS/eip-20), and was added later as an [optional extension](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/IERC20Metadata.sol). As such, some valid ERC20 tokens do not support this interface, so it is unsafe to blindly cast all tokens to this interface, and then call this function.

*Instances (6)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

69:             IYieldToken(_yt).decimals()

113:                         IERC4626(IPrincipalToken(IYieldToken(_yt).getPT()).underlying()).decimals()

129:         return _userYieldIBT + newYieldInIBTRay.fromRay(IERC20Metadata(_yt).decimals());

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

```solidity
File: src/tokens/PrincipalToken.sol

141:         _ibtDecimals = IERC4626(_ibt).decimals();

504:         return IERC4626(ibt).decimals();

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

112:         return IERC20Metadata(pt).decimals();

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="L-3"></a>[L-3] `decimals()` should be of type `uint8`

*Instances (3)*:
```solidity
File: src/libraries/RayMath.sol

20:     function fromRay(uint256 _a, uint256 _decimals) internal pure returns (uint256 b) {

37:         uint256 _decimals,

57:     function toRay(uint256 _a, uint256 _decimals) internal pure returns (uint256 b) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol)

### <a name="L-4"></a>[L-4] Fallback lacking `payable`

*Instances (2)*:
```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

101:     function _fallback() internal virtual override {

109:             super._fallback();

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)

### <a name="L-5"></a>[L-5] Initializers could be front-run
Initializers could be front-run, allowing an attacker to either set their own values, take ownership of the contract, and in the best case forcing a re-deployment

*Instances (11)*:
```solidity
File: src/tokens/PrincipalToken.sol

120:     function initialize(

124:     ) external initializer {

136:         __ERC20_init(name, NamingUtil.genPTSymbol(_ibtSymbol, expiry));

137:         __ERC20Permit_init(name);

138:         __Pausable_init();

139:         __ReentrancyGuard_init();

140:         __AccessManaged_init(_initialAuthority);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

31:     function initialize(

35:     ) external initializer {

36:         __ERC20_init(_name, _symbol);

37:         __ERC20Permit_init(_name);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="L-6"></a>[L-6] Prevent accidentally burning tokens
Minting and burning tokens to address(0) prevention

*Instances (6)*:
```solidity
File: src/tokens/PrincipalToken.sol

766:         _mint(_ptReceiver, shares);

796:         _burn(_owner, shares);

820:         _burn(_owner, _shares);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

46:         _burn(from, amount);

54:         _mint(to, amount);

60:         _burn(msg.sender, amount);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="L-7"></a>[L-7] Solidity version 0.8.20+ may not work on other chains due to `PUSH0`
The compiler for Solidity 0.8.20 switches the default target EVM version to [Shanghai](https://blog.soliditylang.org/2023/05/10/solidity-0.8.20-release-announcement/#important-note), which includes the new `PUSH0` op code. This op code may not yet be implemented on all L2s, so deployment on these chains will fail. To work around this issue, use an earlier [EVM](https://docs.soliditylang.org/en/v0.8.20/using-the-compiler.html?ref=zaryabs.com#setting-the-evm-version-to-target) [version](https://book.getfoundry.sh/reference/config/solidity-compiler#evm_version). While the project itself may or may not compile with 0.8.20, other projects with which it integrates, or which extend this project may, and those projects will have problems deploying these contracts/libraries.

*Instances (6)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

3: pragma solidity 0.8.20;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

```solidity
File: src/libraries/RayMath.sol

2: pragma solidity =0.8.20;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol)

```solidity
File: src/proxy/AMBeacon.sol

5: pragma solidity 0.8.20;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol)

```solidity
File: src/proxy/AMProxyAdmin.sol

5: pragma solidity 0.8.20;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol)

```solidity
File: src/tokens/PrincipalToken.sol

3: pragma solidity 0.8.20;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

3: pragma solidity 0.8.20;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="L-8"></a>[L-8] `symbol()` is not a part of the ERC-20 standard
The `symbol()` function is not a part of the [ERC-20 standard](https://eips.ethereum.org/EIPS/eip-20), and was added later as an [optional extension](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/token/ERC20/extensions/IERC20Metadata.sol). As such, some valid ERC20 tokens do not support this interface, so it is unsafe to blindly cast all tokens to this interface, and then call this function.

*Instances (1)*:
```solidity
File: src/tokens/PrincipalToken.sol

134:         string memory _ibtSymbol = IERC4626(_ibt).symbol();

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="L-9"></a>[L-9] Unsafe ERC20 operation(s)

*Instances (2)*:
```solidity
File: src/tokens/YieldToken.sol

76:         return super.transfer(to, amount);

101:         return super.transferFrom(from, to, amount);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="L-10"></a>[L-10] Upgradeable contract is missing a `__gap[50]` storage variable to allow for new storage variables in later versions
See [this](https://docs.openzeppelin.com/contracts/4.x/upgradeable#storage_gaps) link for a description of this storage variable. While some contracts may not currently be sub-classed, adding the variable now protects against forgetting to add it in the future.

*Instances (16)*:
```solidity
File: src/proxy/AMProxyAdmin.sol

7: import {IAMTransparentUpgradeableProxy} from "./AMTransparentUpgradeableProxy.sol";

41:         IAMTransparentUpgradeableProxy proxy,

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol)

```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

17: interface IAMTransparentUpgradeableProxy is IERC1967 {

60: contract AMTransparentUpgradeableProxy is ERC1967Proxy {

84:             "AMTransparentUpgradeableProxy: initialAuthority is zero address"

103:             if (msg.sig != IAMTransparentUpgradeableProxy.upgradeToAndCall.selector) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)

```solidity
File: src/tokens/PrincipalToken.sol

30:     ERC20PermitUpgradeable,

31:     AccessManagedUpgradeable,

32:     ReentrancyGuardUpgradeable,

33:     PausableUpgradeable,

503:     function decimals() public view override(IERC20Metadata, ERC20Upgradeable) returns (uint8) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

15: contract YieldToken is IYieldToken, ERC20PermitUpgradeable {

74:     ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {

99:     ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {

109:         override(IYieldToken, ERC20Upgradeable)

123:     ) public view override(IYieldToken, ERC20Upgradeable) returns (uint256) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)

### <a name="L-11"></a>[L-11] Upgradeable contract not initialized
Upgradeable contracts are initialized via an initializer function rather than by a constructor. Leaving such a contract uninitialized may lead to it being taken over by a malicious user

*Instances (30)*:
```solidity
File: src/proxy/AMProxyAdmin.sol

7: import {IAMTransparentUpgradeableProxy} from "./AMTransparentUpgradeableProxy.sol";

41:         IAMTransparentUpgradeableProxy proxy,

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol)

```solidity
File: src/proxy/AMTransparentUpgradeableProxy.sol

17: interface IAMTransparentUpgradeableProxy is IERC1967 {

60: contract AMTransparentUpgradeableProxy is ERC1967Proxy {

84:             "AMTransparentUpgradeableProxy: initialAuthority is zero address"

103:             if (msg.sig != IAMTransparentUpgradeableProxy.upgradeToAndCall.selector) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol)

```solidity
File: src/tokens/PrincipalToken.sol

30:     ERC20PermitUpgradeable,

31:     AccessManagedUpgradeable,

32:     ReentrancyGuardUpgradeable,

33:     PausableUpgradeable,

107:         _disableInitializers(); // using this so that the deployed logic contract cannot later be initialized

120:     function initialize(

124:     ) external initializer {

136:         __ERC20_init(name, NamingUtil.genPTSymbol(_ibtSymbol, expiry));

137:         __ERC20Permit_init(name);

138:         __Pausable_init();

139:         __ReentrancyGuard_init();

140:         __AccessManaged_init(_initialAuthority);

503:     function decimals() public view override(IERC20Metadata, ERC20Upgradeable) returns (uint8) {

733:                     IYieldToken(address(0)).initialize.selector,

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

```solidity
File: src/tokens/YieldToken.sol

15: contract YieldToken is IYieldToken, ERC20PermitUpgradeable {

22:         _disableInitializers(); // using this so that the deployed logic contract later cannot be initialized.

31:     function initialize(

35:     ) external initializer {

36:         __ERC20_init(_name, _symbol);

37:         __ERC20Permit_init(_name);

74:     ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {

99:     ) public virtual override(IYieldToken, ERC20Upgradeable) returns (bool success) {

109:         override(IYieldToken, ERC20Upgradeable)

123:     ) public view override(IYieldToken, ERC20Upgradeable) returns (uint256) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)


## Medium Issues


| |Issue|Instances|
|-|:-|:-:|
| [M-1](#M-1) | Contracts are vulnerable to fee-on-transfer accounting-related issues | 3 |
| [M-2](#M-2) | Fees can be set to be greater than 100%. | 1 |
| [M-3](#M-3) | `increaseAllowance/decreaseAllowance` won't work on mainnet for USDT | 1 |
| [M-4](#M-4) | Library function isn't `internal` or `private` | 2 |
### <a name="M-1"></a>[M-1] Contracts are vulnerable to fee-on-transfer accounting-related issues
Consistently check account balance before and after transfers for Fee-On-Transfer discrepancies. As arbitrary ERC20 tokens can be used, the amount here should be calculated every time to take into consideration a possible fee-on-transfer or deflation.
Also, it's a good practice for the future of the solution.

Use the balance before and after the transfer to calculate the received amount instead of assuming that it would be equal to the amount passed as a parameter. Or explicitly document that such tokens shouldn't be used and won't be supported

*Instances (3)*:
```solidity
File: src/tokens/PrincipalToken.sol

181:         IERC20(_asset).safeTransferFrom(msg.sender, address(this), assets);

211:         IERC20(ibt).safeTransferFrom(msg.sender, address(this), ibts);

628:         IERC20(ibt).safeTransferFrom(address(_receiver), address(this), _amount + fee);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="M-2"></a>[M-2] Fees can be set to be greater than 100%.
There should be an upper limit to reasonable fees.
A malicious owner can keep the fee rate at zero, but if a large value transfer enters the mempool, the owner can jack the rate up to the maximum and sandwich attack a user.

*Instances (1)*:
```solidity
File: src/tokens/PrincipalToken.sol

713:     function _updateFees(uint256 _feesInIBT) internal {
             unclaimedFeesInIBT += _feesInIBT;
             totalFeesInIBT += _feesInIBT;

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="M-3"></a>[M-3] `increaseAllowance/decreaseAllowance` won't work on mainnet for USDT
On mainnet, the mitigation to be compatible with `increaseAllowance/decreaseAllowance` isn't applied: https://etherscan.io/token/0xdac17f958d2ee523a2206206994597c13d831ec7#code, meaning it reverts on setting a non-zero & non-max allowance, unless the allowance is already zero.

*Instances (1)*:
```solidity
File: src/tokens/PrincipalToken.sol

182:         IERC20(_asset).safeIncreaseAllowance(ibt, assets);

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)

### <a name="M-4"></a>[M-4] Library function isn't `internal` or `private`
In a library, using an external or public visibility means that we won't be going through the library with a DELEGATECALL but with a CALL. This changes the context and should be done carefully.

*Instances (2)*:
```solidity
File: src/libraries/PrincipalTokenUtil.sol

55:     function _computeYield(

137:     function _tryGetTokenDecimals(address _token) external view returns (uint8) {

```
[Link to code](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)

