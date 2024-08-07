# M^0 NFT custody Security Review

Date: **18.04.24**

Produced by **Kirill Fedoseev** (telegram: [kfedoseev](http://t.me/kfedoseev),
twitter: [@k1rill_fedoseev](http://twitter.com/k1rill_fedoseev))

## Introduction

An independent security review of the M^0 Protocol was conducted by **kfedoseev** from 26.03.24 to 27.03.24. The
fixes review was conducted on 18.04.24. The following methods were used for conducting a security review:

- Manual source code review

## Disclaimer

No security review can guarantee or verify the absence of vulnerabilities. This security review is a time-bound process
where I tried to identify as many potential issues and vulnerabilities as possible, using my personal expertise in the
smart contract development and review.

## About M^0 NFT custody

The M^0 NFT custody is an opt-in solution for wrapping rebasing POWER tokens minted via the core M^0 protocol. POWER
holders not willing to deal with rebasing tokens due to legal or other reasons may use the wrapped NFT to transitively
control their POWER balance though an NFT ownership.

## Observations and Limitations

* M^0 NFT custody is designed to support only the POWER ERC20 token issued by the deployed M^0 protocol. All other
  assets, including other tokens and native ETH, sent to the NFT custody contracts will be irrecoverably lost.

## Severity classification

| **Severity**           | **Impact: High** | **Impact: Medium** | **Impact: Low** |
|------------------------|------------------|--------------------|-----------------|
| **Likelihood: High**   | Critical         | High               | Medium          |
| **Likelihood: Medium** | High             | Medium             | Low             |
| **Likelihood: Low**    | Medium           | Low                | Low             |

**Impact** - the economic, technical, reputation or other damage to the protocol implied from a successful exploit.

**Likelihood** - the probability that a particular finding or vulnerability gets exploited.

**Severity** - the overall criticality of the particular finding.

## Scope summary

Reviewed commits:

* [4e311e29cb68273c4597950fd5f19078656a1ce7](https://github.com/MZero-Labs/nft-custody/commit/4e311e29cb68273c4597950fd5f19078656a1ce7)

Reviewed fixes commits:

* [0693b6d1bed03f244f01542952ad3dbbdf5da989](https://github.com/MZero-Labs/nft-custody/commit/0693b6d1bed03f244f01542952ad3dbbdf5da989)

Reviewed contracts:

- `src/interfaces/**`
- `src/Proxy.sol`
- `src/TokenHolder.sol`
- `src/TokenWrapper.sol`

---

# Findings Summary

| ID     | Title                                                | Severity      | Status |
|--------|------------------------------------------------------|---------------|--------|
| [H-01] | Wrapped `POWER` balances are lost during reset       | High          | Fixed  |
| [L-01] | Address manipulation of deployed proxies             | Low           | Fixed  |
| [I-01] | More-efficient ERC1167 proxies for clone deployments | Informational | Fixed  |
| [I-02] | Redundant `_tokenHolders` storage mapping            | Informational | Fixed  |
| [I-03] | Avoid use of low-level calls                         | Informational | Fixed  |
| [I-04] | Unused imports                                       | Informational | Fixed  |

# Security & Economic Findings

## [H-01] Wrapped `POWER` balances are lost during reset

Core M^0 protocol has a reset functionality that redeploys the POWER token contract to a new address and re-distributes
its initial supply to prior POWER holders.

`TokenWrapper`, however, does not provide a way for NFT owner to access their redeployed POWER balance, while their old
POWER balance will be effectively rendered worthless.

Reset proposals submitted to `ZeroGovernor` during epoch `N`, can be voted for and executed during epochs `N` and `N+1`.
If the threshold is reached and proposal is also executed during epoch `N`, new power token will use balances snapshot
from epoch `N-1` for its bootstrap. In this case, the NFT owner won't have any time to react to the submitted reset
proposal and transfer tokens out of the custody contract.

### Recommendation

Allow NFT owner to access arbitrary ERC20 tokens by adding an extra argument in the `ITokenHolder` interface:

```diff
-function transfer(address recipient, uint256 amount) external returns (bool);
+function transfer(address token, address recipient, uint256 amount) external returns (bool);

-function delegate(address delegatee) external;
+function delegate(address token, address delegatee) external;
```

## [L-01] Address manipulation of deployed proxies

`TokenWrapper` uses `CREATE` to deploy `TokenHolder` proxies. This allows front-running of `wrap` transactions, which
will impact the assigned NFT ownership rights. This is problematic in the following scenario:

1. Alice wraps 1 POWER token to test the system and observes the NFT with id `N1` being minted to her.
2. Bob frontruns the Alice's transaction in a chain reorg, resulting in `N1` ownership being re-assigned to Bob, while
   Alice receives a new `N2` NFT.
3. Alice, while still thinking that `N1` belongs to her, transfers the rest of her POWER tokens to the `TokenHolder`
   controlled by `N1`.
4. Bob takes control over the Alice's POWER as he is the current owner of the `N1` NFT.

### Recommendation

Use `CREATE2` to deploy the proxies at deterministic addresses:

```diff
-function _wrap(uint256 amount_, address to_, address delegatee_) internal returns (uint256 tokenId_) {
+function _wrap(uint256 amount_, address to_, address delegatee_, bytes32 salt_) internal returns (uint256 tokenId_) {
     // ...

-    address tokenHolder_ = address(new Proxy(tokenHolderImplementation));
+    bytes32 salt = keccak256(abi.encode(to_, salt_));
+    address tokenHolder_ = address(new Proxy{salt: salt}(tokenHolderImplementation));
    
    // ...
}
```

## Informational & Gas Optimizations

## [I-01] More-efficient ERC1167 proxies for clone deployments

`TokenWrapper` uses a custom implementation for immutable proxies deployment inspired by ERC1967 standard. However, as
the deployed proxies are designed to be immutable, a more efficient proxy standard exists - ERC1167.

ERC1167 is much cheaper gas-wise both during the proxy deployment and during proxied calls, as less storage is required
for storing contract bytecode or implementation addresses.

Consider deploying ERC1167 proxies instead to save on gas, e.g. by using
a [Clones](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/9558e546d9996d533eeb5151569a52d611eca08b/contracts/proxy/Clones.sol)
library.

```diff
-address tokenHolder_ = address(new Proxy(tokenHolderImplementation));
+address tokenHolder_ = Clones.clone(tokenHolderImplementation);
+// OR `Clones.cloneDeterministic(tokenHolderImplementation, salt)`, according to [L-01]
```

## [I-02] Redundant `_tokenHolders` storage mapping

Storage mapping `tokenHolders` in `TokenWrapper` is effectively used to record key-value
pairs `tokenHolder => tokenHolder`, since `_getTokenId` does not change the low-level value of its argument and returns
the input value unchanged.

Consider removing `_tokenHolders` mapping completely and changing `getTokenHolder` in the following way:

```diff
 function getTokenHolder(uint256 tokenId_) external view returns (address) {
     _requireOwned(tokenId_);

-    return _tokenHolders[tokenId_];
+    return address(uint160(tokenId_));
 }
```

## [I-03] Avoid use of low-level calls

`TokenWrapper` implementation of `_wrap` uses a low-level call that shadows the original revert reason coming from
the `delegate` implementation (e.g. `error VoteEpoch()`). This seems unnecessary, consider making a direct Solidity call
instead.

```diff
-(bool success_, ) = tokenHolder_.call(abi.encodeWithSelector(ITokenHolder.delegate.selector, delegatee_));
-
-if (!success_) revert DelegationFailed();
+ITokenHolder(tokenHolder_).delegate(delegatee_);
```

## [I-04] Unused imports

The following imports are unused and can be removed:

In `TokenHolder.sol`:

```diff
-import { SignatureChecker } from "../lib/common/src/libs/SignatureChecker.sol";
```

In `TokenWrapper.sol`:

```diff
-import { Base64 } from "../lib/openzeppelin-contracts/contracts/utils/Base64.sol";
-import { Strings } from "../lib/openzeppelin-contracts/contracts/utils/Strings.sol";

// ...

-import { IERC20Like } from "./interfaces/IERC20Like.sol";
```
