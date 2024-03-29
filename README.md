# Spectra audit details
- Total Prize Pool: $36,500 in USDC
  - HM awards: $24,000 in USDC
  - Analysis awards: $1,500 in USDC
  - QA awards: $700 in USDC
  - Bot Race awards: $2,200 in USDC
  - Gas awards: $700 in USDC
  - Judge awards: $4,500 in USDC
  - Lookout awards: $2,400 in USDC 
  - Scout awards: $500 in USDC
 
- Join [C4 Discord](https://discord.gg/code4rena) to register
- Submit findings [using the C4 form](https://code4rena.com/contests/2024-02-spectra/submit)
- [Read our guidelines for more details](https://docs.code4rena.com/roles/wardens)
- Starts February 23, 2024 20:00 UTC
- Ends March 01, 2024 20:00 UTC

## Automated Findings / Publicly Known Issues

The 4naly3er report can be found [here](https://github.com/code-423n4/2024-02-spectra/blob/main/4naly3er-report.md).

Automated findings output for the audit can be found [here](https://github.com/code-423n4/2024-02-spectra/blob/main/bot-report.md) within 24 hours of audit opening.

_Note for C4 wardens: Anything included in this `Automated Findings / Publicly Known Issues` section is considered a publicly known issue and is ineligible for awards._

#### Code Unreachable

Custom errors are implemented for scenarios that theoretically should not occur and could be harmful for the end user. These errors cater to instances where code is considered unreachable. However, in cases of invariant violation, the code might be accessible. An example of this is found in the `_computeYield` method of the `PrincipalTokenUtil` library.

#### Reverting Transactions When PT rate is Zero

The PT rate accounts for a negative rate on the IBT (Interest Bearing Token). A decrease in the PT rate occurs when the IBT experiences a negative rate. The PT rate does not increase; it can only be zero if the IBT rate is also zero, indicating no claimable underlying assets in the IBT (e.g., if the IBT is drained due to a hack), rendering the IBT worthless and prohibiting interaction with the protocol.

#### The Principal Token contract expects a compliant ERC4626 and non malicious IBT 

The protocol relies on trusting the integrated IBT. A malicious actor could create a malicious compliant ([EIP4626](https://eips.ethereum.org/EIPS/eip-4626)) IBT that could break the protocol and might result in a loss of funds for the users.

#### Access Control Trust Model

Certain functions are secured with an OpenZeppelin Access Manager contract, enabling centralized role management distributed to DAO accounts.

Some IBTs are elligible for additional rewards, and the DAO can claim these rewards and redistribute them to the users. The DAO will be able for such IBTs to setup a reward proxy on top of the PT contract to be able to claim the rewards. The DAO is the only entity that can perform these operations.

We aknowledge that a malicious user with a REWARDS_HARVESTER_ROLE could claim the rewards and keep them for himself. However, the DAO is the only entity that can claim the rewards and the DAO is a trusted entity.

We acknoledge that a malicious user with a REWARDS_PROXY_SETTER_ROLE could set the rewards proxy to a malicious contract and drain the protocol via the delegatecall. However, the DAO is the only entity that can set the rewards proxy and the DAO is a trusted entity.

We aknowledge that a malicious user with a FEE_SETTER_ROLE could set the fees to 100% and drain the protocol. However, the DAO is the only entity that can set the fees and the DAO is a trusted entity.


# Overview

Spectra is a permissionless interest rate derivatives protocol for DeFi. The protocol allows to split the yield generated by an Interest Bearing Token (IBT) from the principal asset. The IBT is deposited in the protocol and the user receives Principal Tokens (PT) and Yield Tokens (YT) in return. The PT represents the principal asset and the YT represents the yield generated by the IBT. Holders of the yield token for a specific IBT can claim the yield generated by the corresponding deposited IBTs during the time they hold the YT. 

## Architecture Overview

![Spectra Contracts Architecture](https://github.com/code-423n4/2024-02-spectra/blob/main/spectra_contracts_architecture.png?raw=true)

#### Principal Token

> *[PrincipalToken](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)*

This is the core contract of Spectra. The Principal Token is [EIP-5095](https://eips.ethereum.org/EIPS/eip-5095) and [EIP-2612](https://eips.ethereum.org/EIPS/eip-2612) compliant. Users can deposit an [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626) IBT or the underlying token of that IBT and receive Principal Tokens (PT) and Yield Tokens (YT). The PT contract holds the logic that separates the yield generated from the principal asset deposited in the IBT.

#### Yield Token

> *[YieldToken](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)*

This contract represents the Yield Token (YT). The YT is an [EIP-20](https://eips.ethereum.org/EIPS/eip-20) token and follows the [EIP-2612](https://eips.ethereum.org/EIPS/eip-2612) standard. The same amount of PT and YT is minted upon depositing into the protocol (`PrincipalToken.deposit`, `PrincipalToken.depositIBT`). The YT captures the yield generated by the deposited principal. Holding the YT allows the user to claim the corresponding amount of yield generated by the IBTs deposited in the associated PT contract.

#### Access Manager and Ownable

The Spectra protocol uses the [OpenZeppelin AccessManager](https://docs.openzeppelin.com/contracts/5.x/api/access#accessmanager) to manage the access control of the different protected functions.

We thus modified the [Openzepellin Transparent Proxy](https://docs.openzeppelin.com/contracts/5.x/api/proxy#TransparentUpgradeableProxy), [Beacon Proxy](https://docs.openzeppelin.com/contracts/5.x/api/proxy#BeaconProxy) and [Proxy Admin](https://docs.openzeppelin.com/contracts/5.x/api/proxy#ProxyAdmin) of OpenZeppelin to leverage the access manager instead of the Ownable pattern for the upgrade and admin functions.

## Links

- [Spectra Website](https://spectra.finance/)
- [Developer Documentation](https://dev.spectra.finance/)
- [Twitter](https://twitter.com/spectra_finance)
- [Discord](https://discord.gg/BNZtnNFevH) 


# Scope

| Contract                                        | SLOC | Purpose                                                                                                                                                                                                                                | Libraries used                         |
| ----------------------------------------------- | ---- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------- |
| [src/proxy/AMBeacon.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMBeacon.sol)                      | 24   | Modified from Openzeppelin Beacon using Openzeppelin [Access Manager](https://docs.openzeppelin.com/contracts/5.x/api/access#AccessManager) instead of Ownable                                                                         | openzeppelin                           |
| [src/proxy/AMProxyAdmin.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMProxyAdmin.sol)                  | 14   | Modified from [Openzeppelin ProxyAdmin](https://docs.openzeppelin.com/contracts/5.x/api/proxy#ProxyAdmin) using Openzeppelin [Access Manager](https://docs.openzeppelin.com/contracts/5.x/api/access#AccessManager) instead of Ownable | openzeppelin                           |
| [src/proxy/AMTransparentUpgradeableProxy.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/proxy/AMTransparentUpgradeableProxy.sol) | 42   | Modified from [Openzeppelin TransparentUpgradeableProxy](https://docs.openzeppelin.com/contracts/5.x/api/proxy#TransparentUpgradeableProxy) using AMProxyAdmin instead of   Ownable                                                    | openzeppelin                           |
| [src/tokens/PrincipalToken.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/PrincipalToken.sol)               | 649  | The Principal Token is the main contract of Spectra. Users deposit their IBT in exchange for Principal Token and Yield Token                                                                                                           | openzeppelin, openzeppelin-upgradeable |
| [src/tokens/YieldToken.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/tokens/YieldToken.sol)                   | 73   | Holders of the yield token for a specific IBT can claim the yield generated by the corresponding deposited IBTs                                                                                                                        | openzeppelin                           |
| [src/libraries/PrincipalTokenUtil.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/PrincipalTokenUtil.sol)        | 142  | Utility library for the Principal Token contract                                                                                                                                                                                       | openzeppelin, openzeppelin-upgradeable |
| [src/libraries/RayMath.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/libraries/RayMath.sol)                   | 32   | Library for number conversions from decimals between 6 and 18 to 27 decimals  (ray)                                                                                                                                                 |                                        |


## Out of scope

The files listed above as in scope are the core components of the Spectra protocol. The following files are not in scope for this audit and represent the periphery contracts and libraries that are not directly involved in the core functionality of the protocol:
- [src/factory/Factory.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/factory/Factory.sol)
- [src/Registry.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/Registry.sol)
- [src/router/](https://github.com/code-423n4/2024-02-spectra/blob/main/src/router)
- [src/oracle/RateOracle.sol](https://github.com/code-423n4/2024-02-spectra/blob/main/src/oracle/RateOracle.sol)


Anything in [interfaces](https://github.com/code-423n4/2024-02-spectra/blob/main/src/interfaces), [test](https://github.com/code-423n4/2024-02-spectra/blob/main/test), [mocks](https://github.com/code-423n4/2024-02-spectra/blob/main/src/mocks), or [scripts](https://github.com/code-423n4/2024-02-spectra/blob/main/script) is out of scope.

# Additional Context

- The protocol is expected to interact with any [ERC4626](https://eips.ethereum.org/EIPS/eip-4626) compliant vault. 
- The protocol can only interact with IBTs between 6 and 18 decimals (inclusive).
- IBT must not be [ERC777](https://eips.ethereum.org/EIPS/eip-777)
- The following roles are defined with the [OpenZeppelin AccessManager](https://docs.openzeppelin.com/contracts/5.x/api/access#accessmanager):
  - `ADMIN_ROLE` - roleId `0` - the Access Manager super admin. Can grant and revoke any role. Set by default in the Access Manager constructor.
  - `UPGRADER_ROLE` - roleId `1` - the users who can upgrade the protocol implementations.
  - `PAUSER_ROLE` - roleId `2` - the DAO address that can pause the protocol (in case of emergency).
  - `FEE_SETTER_ROLE` - roleId `3` - the role that can change the fees in the protocol.
  - `REGISTRY_ROLE` - roleId `4` - the users who can call the registry contract to register new contracts addresses.
  - `REWARDS_HARVESTER_ROLE` - roleId `5` - the DAO address that can harvest the additional rewards generated by the IBT holdings of the protocol (redistributed to users).
  - `REWARDS_PROXY_SETTER_ROLE` - roleId `6` - the DAO address that can setup a rewards proxy to be able to claim IBT rewards.
- Spectra may be deployed to Ethereum, Polygon, Arbitrum
- EIP Compliance:
  - `PrincipalToken.sol` : Should comply with `ERC5095` and `ERC2612`
  - `YieldToken.sol` : Should comply with `ERC20` and `ERC2612`

## Attack ideas (Where to look for bugs)

- Decimals imprecisions should always benefit the protocol and no user should be able to extract extra value.
- Proxy Admin and Beacon are a modified version of Openzepelin origin contract replacin OZ Ownable with OZ Access Managed. Check if this modification can be harmful outside of our trust model ([see above](https://www.notion.so/V2-C4-dfec6a803eac4214b2a63e406250a172?pvs=21)).
- Imprecisions and rounding errors.
- Manipulation of the IBT rate.
- Mechanism of negative rates and the impact on the PT rate. [See docs](https://dev.spectra.finance/technical-reference/yield-calculations)


## Main invariants

**IBT rate is only updated upon user interactions with our protocol**

**PT rate is only updated after an accounted negative rate change on the IBT rate**

**PT and its YT should have an equal supply at all times**

**PT rate should not increase**

`ptRateOfUser(u) ≥ ptRate` for all `u` in `users` with `users` being all the users that deposited in the PT.

**Accounted IBT rate cannot decrease without impacting PT rate**

If the protocol records an IBT rate decrease, the PT rate has to decrease to account for the negative rate.

**Principal Token is ERC5095**

All [EIP-5095](https://eips.ethereum.org/EIPS/eip-5095) invariants should hold such as `previewRedeem ≥ redeem`.

**Principal Token deposit**

`previewDeposit ≤ deposit` : the preview of shares minted upon depositing should be less than or equal to the actual shares minted.


## Scoping Details 

```
- If you have a public code repo, please share it here:  
- How many contracts are in scope?:   7
- Total SLoC for these contracts?:  976
- How many external imports are there?: 15  
- How many separate interfaces and struct definitions are there for the contracts within scope?:  2
- Does most of your code generally use composition or inheritance?:   Inheritance
- How many external calls?:   5
- What is the overall line coverage percentage provided by your tests?: 96
- Is this an upgrade of an existing system?: False
- Check all that apply (e.g. timelock, NFT, AMM, ERC20, rollups, etc.): ERC-20 Token
- Is there a need to understand a separate part of the codebase / get context in order to audit this part of the protocol?:   
- Please describe required context:   
- Does it use an oracle?:  No
- Describe any novel or unique curve logic or mathematical models your code uses: 
- Is this either a fork of or an alternate implementation of another project?:   No
- Does it use a side-chain?:
- Describe any specific areas you would like addressed:
```

# Tests

## Installation

Follow [this link](https://book.getfoundry.sh/getting-started/installation) to install Foundry, Forge, Cast and Anvil.

Do not forget to update Foundry regularly with the following command

```properties
foundryup
```

Similarly for forge-std run

```properties
forge update lib/forge-std
```

## Submodules

Run below command to include/update all git submodules like openzeppelin contracts, forge-std etc (`lib/`)

```properties
git submodule update --init --recursive
```

To get the `node_modules/` directory run

```properties
yarn
```

## Compilation

To compile your contracts run

```properties
forge build
```

## Testing

> ⚠️ **Note**: Before running the tests, you need to setup an RPC URL for the Goerli network. You can create a free RPC url at https://www.alchemy.com. Once you have it you can create a `.env` file copying the `.env.example` file (`cp .env.example .env`) and replacing the `GOERLI_RPC_URL` with your own.



Run your tests with

```properties
forge test
```

Find more information on testing with Foundry [here](https://book.getfoundry.sh/forge/tests)

Note: tests might take a long time due to the number of fuzz runs. You can modify the `runs` parameter under the `[fuzz]` section of the `foundry.toml` file as per your needs.


## Slither

Running slither should produce 349 results. We have reviewed all of these results on our end, and have not found them to be issues. Please do not submit slither results as findings unless you have confirmed there is a specific exploitable issue resulting in negative consequences linked to the result.

