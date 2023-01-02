# NM-0063 Rook Finances - Final Report

**Repo:** https://github.com/rookprotocol/Swap/tree/3ececb55af05a2d08229cc838b1419e286372a16


**Blob:** 
NEW:
https://github.com/rookprotocol/Swap/blob/3ececb55af05a2d08229cc838b1419e286372a16/

OLD: https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/

## General

### [Info] Solidity `0.8.6` is not recommended for deployment

**File(s)**: [`contracts/*`](https://github.com/rookprotocol/Swap/tree/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/)

**Description**: `solc` frequently releases new compiler versions. Using an old version prevents access to new Solidity security checks. We also recommend avoiding complex pragma statements.

**Recommendation**: Deploy with [version `0.8.16`](https://github.com/crytic/slither/wiki/Detector-Documentation#incorrect-versions-of-solidity).

**Status**: Unresolved

**Update from the client**:

---

### [Info] Order decay `begin` and `expiry` values are not validated

**File(s)**: [`swap.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/swap.sol), [`orderUtils.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/orderUtils.sol)

**Description**: The function `_decodeData(...)` presented below extracts the order `begin` and `expiry` values from `data`. These values are not validated before being used to calculate the current minimum taker amount. 

When `begin == expiry` the calculated current minimum taker amount is always equal to `takerAmountMin`. When `expiry < begin` the function `_calculateCurrentTakerAmountMin(...)` will revert when it attempts to subtract `begin` from `expiry`, where the Solidity 0.8 overflow check will identify an overflow.

We understand that orders will only come from the HidingBook which is a trusted RookFi entity, but these orders are created by users and it is best practice to immediately validate data as it enters the smart contract regardless of the data source.

```solidity=
function _decodeData(uint256 data, bytes32 orderHash)
    internal
    pure
    returns (LibData.MakerData memory makerdata)
{
    /////////////////////////////////////////////////////////////
    // @audit there is no check if the expiry is greater than begin
    /////////////////////////////////////////////////////////////
    uint64 begin = uint64(data);
    uint64 expiry = uint64(data >> 64);
    bool partiallyFillable = data & 0x100000000000000000000000000000000 !=
        0;
    ...
}
```

**Recommendation(s)**: Consider validating the values of `begin` and `expiry` when they are extracted from `data`.

**Status**: Unresolved 

**Update from the client**:

---
### [Best Practices] General recommendations for saving gas when accessing array elements

**File(s)**: [`contracts/*`](https://github.com/rookprotocol/Swap/tree/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/)

**Description**: Solidity 0.8 (and later versions) have built-in overflow checks. Therefore, in loops every time the iterator variable is incremented, there is a silent overflow check performed even if there is no risk of an overflow. Avoiding the silent overflow check saves gas. This finding lists some actions for reducing gas consumption when dealing with arrays. Most of the suggestions below can be applied throughout the whole project. As general rules: i) use `++i` instead of `i++` in for loops; ii) avoid using `uint` smaller than 256 bits; iii) avoid using `array.length` inside `for-loops`. Instead, when looping over an array, store `array.length` in a temporary variable. The code snippet below shows the example code of a function that goes over the array `order` in the style adopted by the protocol.

```solidity=
function testGasForLoop_Rook( uint256 _n ) public pure returns(uint256)
{
    Order[] memory order = new Order[]( _n );

    //////////////////////////////////////////////////////////////
    // @audit Notice that "order.length" is inside the for-loop
    // @audit Notice the use of "uint8"
    //////////////////////////////////////////////////////////////
    for (uint8 i=0; i<order.length; i++)
    {
        order[i].field = i;
    }
    return order.length;
}
```

We can save gas by rewriting the function as follows.

```solidity=
function testGasForLoop_new( uint256 _n ) public pure returns(uint256)
{
    Order[] memory order = new Order[]( _n );
    
    //////////////////////////////////////////////////////////////
    // @audit We store "order.length" in a temp var
    //////////////////////////////////////////////////////////////
    uint256 temp = order.length;

    //////////////////////////////////////////////////////////////
    // @audit We use the temp var inside the for-loop
    // @audit Also notice the use of "++i" instead of "i++"
    // @audit We also use "uint256". Comparing "uint8" to "uint256"
    //        can lead to unexpected behavior
    //////////////////////////////////////////////////////////////
    for (uint256 i=0; i<temp;)
    {
        order[i].field = i;
        unchecked{
            ++i;
        }
    }
    return order.length;
}
```

The simulation of gas consumption using [Remix](https://remix.ethereum.org/) is presented in the figure below.

![](https://i.imgur.com/elF2iFg.png)

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook1.pdf}}
   \label{fig:rook1}
\end{figure}
-->

Gas savings according to loop interactions is presented below. We notice a linear gain.

![](https://i.imgur.com/LnhZH9n.png)

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook2.pdf}}
   \label{fig:rook2}
\end{figure}
-->

Finally, we also show the percentage gas savings according to the number of interactions in the loop when comparing both strategies. The tendency curve is a second degree polynomial.

![](https://i.imgur.com/N7Ew1i1.png)

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook3.pdf}}
   \label{fig:rook3}
\end{figure}
-->

**Recommendation(s)**: Review the entire code and adopt these practices to reduce gas consumption: a) use `++i` instead of `i++` in `for` loops; b) avoid using `uint` smaller than 256 bits; c) avoid using `array.length` inside `for` loops.

**Status**: Unresolved 

**Update from the client**:

---
### [Best Practices] Greater than zero comparison can be optimized

**File(s)**: [`contracts/*`](https://github.com/rookprotocol/Swap/tree/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/)

**Description**: When checking if some unsigned integer is greater than zero, using `x != 0` is more gas efficient than `x > 0` while providing the same result.

**Recommendation**: Consider using `x != 0` instead of `x > 0` throughout the codebase to improve gas efficiency.

**Status**: Unresolved

**Update from the client**:

---
### [Best Practices] Missing event emission

**File(s)**: [`contracts/*`](https://github.com/rookprotocol/Swap/tree/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/)

**Description**: There are certain functions that do not emit events but make state changes that may be relevant to users or developers. Emitting events is a good practice and can improve protocol monitoring. A list of functions which we consider should emit events are listed below:

- Constructor in `owner.sol` should emit an event with the `owner`.
- Constructor in `whitelist.sol` should emit an event with the `owner`.
- Function `withdrawEther(...)` in `assetManagement.sol` should emit withdrawal details.

**Recommendation(s)**: Consider emitting events in the suggested functions above.

**Status**: Unresolved

**Update from the client**:

---
### [Info] `swap.sol` exceeds 24576 bytes

**File(s)**: [`contracts/*`](https://github.com/rookprotocol/Swap/tree/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/)

**Description**: The contract `swap.sol` exceeds 24576 bytes. This contract may not be deployable on mainnet. There are several measures that can be adopted for reducing the contract size. Here we summarize the most suitable ones for your project. As a first recommendation, avoid public variables. By declaring public variables, the Solidity compiler automatically generates the corresponding `getter()` function, which increases the contract size. Declaring the proper visibility of functions and state variables plays an important role. Functions or variables that are only called from other contracts must be `external` instead of `public` (`external` functions also reduce gas consumption compared to `public`). Functions or variables called only from within the contract must be `private` or `internal`. For more information on downsizing contracts, check [this reference](https://ethereum.org/en/developers/tutorials/downsizing-contracts-to-fight-the-contract-size-limit/#taking-on-the-fight).

**Recommendation**: This finding does not highlight an issue, but as contract size is a concern for the `RookSwap` contract these recommendations may help to reduce the contract size.

**Status**: Unresolved 

**Update from the client**:

---
### [Best Practices] Tautology in boolean comparisons

**File(s)**: [`swap.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/swap.sol), [`whitelist.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/whitelist.sol)

**Description**: The following boolean comparisons can be evaluated as `true` or `false` directly and don't need to be compared with some `true` or `false` value. For example: `(value)` and `(value == true)` are equivalent. Simplifying this comparison can reduce gas costs.

```solidity=
// whitelist.sol L80, L133
if (value == true)

// swap.sol L95
require(
    isWhitelistedKeeper(keeperTaker) == true,
    "RS:E21"
);

// swap.sol L151
require(
    isWhitelistedDexAggKeeper(msg.sender) == true,
    "RS:E22"
);
```

**Recommendation(s)**: Remove the tautologies.

**Status**: Unresolved 

**Update from the client**:

---
### [Best Practices] Unused named return variables

**File(s)**: [`contracts/*`](https://github.com/rookprotocol/Swap/tree/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/)

**Description**: There are multiple functions where return variables are declared but not used. As an example, we reproduce the code for the function  `isValidOrderSigner(...)`.

```solidity=
function isValidOrderSigner(address maker, address signer)
    public view returns (bool isValid) 
{
    ...
    
    //////////////////////////////////////////////////////////////////
    // @audit-issue named return `isValid` declared but not used
    //////////////////////////////////////////////////////////////////
    return orderSignerRegistry[maker][signer];
}
```

This issue can also be found in functions `_finalizeSwapExecution_dexAggKeeper(...)` and `swapDexAggKeeper_54P(...)`.

**Recommendation(s)**: Assign return values to named variables or remove them.

**Status**: Unresolved 

**Update from the client**:

---

## `contracts/owner.sol`

### [Low] Missing safety features for ownership transfer

**File(s)**: [`owner.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/owner.sol)

**Description**: Changing the owner with the function `transferOwnership(...)` is an atomic process. Once changed, only the new owner can make any further changes. If the current owner passes an incorrect value then ownership of the contract would be lost. 

**Recommendation(s)**: Consider changing the process of setting a new owner to set-then-claim to ensure that the owner has set the correct address. Note that if you intend to renounce the ownership by setting the owner to `address(0)` a dedicated function `renounceOwnership(...)` is recommended for that case.

**Status**: Unresolved 

**Update from the client**:

---
### [Best Practice] `transferOwnership(...)` can be `external`

**File(s)**: [`owner.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/owner.sol)

**Description**: The function `transferOwnership(...)` is not used inside the contract, and can have `external` visibility which will reduce gas cost.

```solidity=
/////////////////////////////////////////////////////
// @audit Function can be "external"
/////////////////////////////////////////////////////
function transferOwnership( address newOwner ) public onlyOwner
{
    ...
}
```

**Recommendation(s)**: Consider changing the visibility of the function `transferOwnership(...)` from `public` to `external`.

**Status**: Unresolved 

**Update from the client**:

---

## `contracts/whitelist.sol`

### [Info] Updating DexAgg whitelist emits an incorrect event

**File(s)**: [`whitelist.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/whitelist.sol)

**Description**: The function `whitelistDexAggKeepers(...)` emits a `WhitelistKeeper` event instead of a `whitelistDexAggKeeper` event.

```solidity=
function whitelistDexAggKeepers(address[] memory keepersToUpdate, bool value)
{
    ...
    ///////////////////////////////////////////////////////////////////////////
    // @audit Wrong event emitted. Should have emitted `WhitelistDexAggKeeper`.
    ///////////////////////////////////////////////////////////////////////////
    emit WhitelistKeeper(keepersToUpdate[i], value);
    ...
}
```

**Recommendation(s)**: Consider changing the emitted event to `whitelistDexAggKeeper`.

**Status**: Unresolved 

**Update from the client**:

---
### [Info] Whitelist constructor repeats logic in Ownable constructor

**File(s)**: [`whitelist.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/whitelist.sol)

**Description**: The whitelist constructor sets the owner address as `msg.sender`, but this action is already done inside the constructor for the `Ownable` contract.

**Recommendation(s)**: Remove the unnecessary `owner` assignment in the constructor.

**Status**: Unresolved 

**Update from the client**:

---
### [Best Practices] Whitelist search algorithmic complexity can be reduced to O(1) 

**File(s)**: [`whitelist.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/whitelist.sol)

**Description**: The functions `whitelistKeepers(...)` and `whitelistDexAggKeepers(...)` use a mapping to query whether an address is whitelisted and an array to contain the currently whitelisted addresses (as mappings are not enumerable). In the current whitelist implementation the mappings map an address to a boolean indicating whether the given address is whitelisted, these are shown below:

```solidity=
mapping(address => bool) keepers;
mapping(address => bool) dexAggKeepers;
```

By using the present implementation, the query time is O(1). However, searching for an element inside the whitelist array is O(n). This implementation has a query time of O(1), however searching for an address when modifying the current whitelisted addresses has a time complexity of O(n). This search time can be improved to O(1) by changing the mappings to hold the position of the address within the array. Thus, the mappings become:

```solidity=
mapping(address => uint256) keepers;
mapping(address => uint256) dexAggKeepers;
```

When a given address does not exist in the array the mapping will return zero. If the mapping returns a non-zero value then it represents the position where the address is stored in the array. 

When we add an address to the whitelist, we simply query the mapping to find the position in the array. If a non-zero value is returned then nothing needs to be done. If the mapping returns zero we can simply append the address to the end of the array. This approach requires an add and remove function for both whitelist types. **An example implementation for a function that adds to the whitelist is presented below**.

```solidity=
function _addToWhitelistKeepers( address[] memory keepersToUpdate ) internal
{
    uint256 size = keepersToUpdate.length;
    for (uint256 i; i < size;)
    {
        address keeper = keepersToUpdate[i];
        // get the position of the "keepersToUpdate[i]" from the mapping "keepers"
        uint256 posKeeper = keepers[ keeper ];

        // if the "keepersToUpdate[i]" is in the array, just skip this iteration
        if (posKeeper != 0)
        {
            emit WhitelistKeeper(keeper, true);
            continue;
        }

        // otherwise, get the last position of the "KeepersArray"
        // here we will store the "keepersToUpdate[i]"
        uint256 pos = keepersArray.length;

        // store the position+1 in the mapping "keepers"
        // to save storage, the mapping will always hold "pos+1", instead of "pos"
        keepers[ keeper ] = pos+1;

        // store "keepersToUpdate[i]" in the array "keepersArray"
        keepersArray.push( keeper );

        emit WhitelistKeeper(keeper, true);

        unchecked{
            ++i;
        }
    }
}
```

When we compare this implementation to the original implementation existing in the code, we notice the following results. The figure below compares the gas consumption for both strategies when adding new elements to the whitelist. The `x-axis` shows the number of elements added to the whitelist, while the `y-axis` shows the gas consumption. We notice the new strategy requires less gas to complete the same task.

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook4.pdf}}
   \label{fig:rook4}
\end{figure}
-->

![](https://i.imgur.com/t5TgAEy.png)

Gas savings are plotted in the figure below, where we notice a linear ratio of gas savings. By adding 30 new elements, we can save almost 40,000 gas units.

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook5.pdf}}
   \label{fig:rook5}
\end{figure}
-->

![](https://i.imgur.com/rMfKVe8.png)

The next figure shows the percentage of gas savings when we compare the two approaches for performing the same activity. When adding one element, the new strategy can save 19% of gas. However, when the array size increases, the percentage of gas savings follows a logarithmic pattern, since the savings become small when compared to the cost of updating the storage variables.

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook6.pdf}}
   \label{fig:rook6}
\end{figure}
-->

![](https://i.imgur.com/9qMD4iK.png)

Similarly, when we must remove an element from the whitelist array, we query the mapping. If the mapping returns `0`, then the element is not part of the array and nothing has to be done. In case the mapping returns a non-zero value, then we have the position where the element is located inside the array. All we have to do is switch this element with the last one, update the mapping entry related to the last element to point to its new position (this is the position initially returned by the mapping for the element to be removed), and then we pop the last element from the array, which is also O(1). **An example implementation for a function that removes from the whitelist is presented below:**

```solidity=
function _removeWhitelistKeepers( address[] memory keepersToUpdate ) public
{
    uint256 size = keepersToUpdate.length;
    for (uint256 i; i < size;)
    {
        address keeper = keepersToUpdate[i];

        // get the position of the keepersToUpdate[i] from the mapping keepers
        uint256 posKeeper = keepers[ keeper ];

        // if the position is zero, the keeper is not in the array. So we just skip this iteration
        if (posKeeper == 0)
        {
            emit WhitelistKeeper( keeper, false );
            continue;
        }

        // otherwise, we need to remove the keeper from the array

        // we know that keeper is in the position "posKeeper"
        // get the length of "keepersArray"
        uint256 posLastKeeper = keepersArray.length - 1;

        // get the keeper address stored in "posLastKeeper"
        address lastKeeper = keepersArray[ posLastKeeper ];

        // copy the "lastKeeper" to the position of "keepersToUpdate[i]"
        // remember that we store "posKeeper" increased by 1 in the mapping
        keepersArray[ posKeeper - 1 ] = keepersArray[ posLastKeeper ];

        // update the mapping "keepers" with the new position of the last keeper
        keepers[ lastKeeper ] = posKeeper;

        // update the mapping "keepers" with zero as the new position of the removed keeper
        keepers[ keeper ] = 0;

        // pop the last element of the array
        keepersArray.pop();

        emit WhitelistKeeper(keeper, false);

        unchecked {
            ++i;
        }
   }
}
```

The gas consumption for the removal operation is presented in the next plot. The `x-axis` shows the number of elements removed from the whitelist, while the `y-axis` shows the gas consumption.

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook7.pdf}}
   \label{fig:rook7}
\end{figure}
-->

![](https://i.imgur.com/G4WAqhj.png)

The figure below plots the gas consumption difference between both strategies, and we notice a polynomial relation between the number of elements removed and the gas saving. The more elements we remove in the same batch, the more gas we save.

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook8.pdf}}
   \label{fig:rook8}
\end{figure}
-->

![](https://i.imgur.com/zDIbia6.png)

In the last figure, we compare the percentage of gas savings by adopting the new strategy. We also notice a polynomial ascending curve, where saving varies from 10% up to 30% when we remove from 1 up to 30 elements in the same batch.

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook9.pdf}}
   \label{fig:rook9}
\end{figure}
-->
![](https://i.imgur.com/zDAdQ1b.png)

Finally, to preserve the actual structure of the code, the function `whitelistKeepers(...)` must be adapted to deal with this new scenario. In case the input parameter `value` is `true`, we must call the function `_addToWhitelistKeepers(...)`. Otherwise, we must call the function `_removeWhitelistKeepers(...)`. An illustrative example is presented below:

```solidity=
function whitelistKeepers(address[] memory keepersToUpdate, bool value) external onlyOwner
{
    if (value)
    {
        _addToWhitelistKeepers(keepersToUpdate);
    }
    else
    {
        _removeWhitelistKeepers(keepersToUpdate);
    }
    return;
}
```

Please note that the query functions `isWhitelistedKeeper(...)` and `isWhitelistedDexAggKeeper(...)` will have to be changed to return `true` on a non-zero value rather than returning the value directly, otherwise a query to a whitelisted address will return its position in the array rather than a boolean. This additional check will incur a higher gas cost on keepers compared to the implementation value as a comparison must be done before returning.

**As a disclaimer, the code snippets presented here serve only the purpose of explaining the proposed solution**.

**Recommendation(s)**: Consider developing a solution where mappings hold the position of an address inside the whitelist array to reduce array modification search times from O(n) to O(1).

**Status**: Unresolved 

**Update from the client**:

---

## `contracts/assetManagement.sol`

### [Low] ERC20 Approve Race in function `manuallySetAllowance(...)`

**File(s)**: [`assetManagement.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/assetManagement.sol)

**Description**: The function `manuallySetAllowance(...)` uses the `IERC20.approve(...)`. Beware that changing an allowance with this method brings the risk that someone may use both the old and the new allowance depending on the transaction ordering. One possible solution to mitigate this race condition is to first reduce the spender’s allowance to 0 and set the desired value afterward, as discussed [here](https://github.com/ethereum/EIPs/issues/20#issuecomment-263524729). You can also rely on the `increaseAllowance(...)` and `decreaseAllowance(...)` functions to achieve the same result. The code with audit comments is reproduced below.

```solidity=
function manuallySetAllowance( IERC20 token, address spender, uint256 value ) 
external onlyOwner
{
    ///////////////////////////////////////////////////////// 
    // @audit ERO20 race condition. Set the "approval"
    //        to zero before setting to the new value or
    //        use "increase/decrease allowance"
    /////////////////////////////////////////////////////////
    token.approve(spender, value);
}
```

**Recommendation(s)**: Consider setting the approval to zero before approving the new value or using `increaseAllowance(...)` and `decreaseAllowance(...)`.

**Status**: Unresolved 

**Update from the client**:

---
### [Best Practices] Inconsistency in input validation

**File(s)**: [`assetManagement.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/assetManagement.sol)

**Description**: `withdrawToken(...)` checks if the token is a zero address, while `manuallySetAllowance(...)` does not.

**Recommendation(s)**: Input validation for tokens should be consistent across all functions to prevent incorrect assumptions.

**Status**: Unresolved 

**Update from the client**:

---

## `contracts/signing.sol`

### [Info] Unnecessary conversion from `bytes32` to `bytes` when using `preSignature` mapping

**File(s)**: [`signing.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/signing.sol)

**Description**: The `preSignature` mapping uses a key value of type `bytes`. The key values to be used for this mapping are order hashes, which are type `bytes32`. When pre-signing in `batchSetPreSignature(...)` the `bytes32` orderhash is converted to the dynamic type `bytes` and then used as a key. The extra conversion from `bytes32` to `bytes` is unnecessary as it leads to higher gas costs.

**Recommendation(s)**: Consider changing `preSignature` mapping from `(bytes => uint256)` to `(bytes32 => uint256)` to save gas on mapping key construction. Please note that all accesses to `preSignature` will have to be changed to use `bytes32` instead of `bytes`

**Status**: Unresolved 

**Update from the client**:

---
### [Best Practices] Allowed signers cannot pre-sign orders on behalf of a maker

**File(s)**: [`signing.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/signing.sol)

**Description**: The rook protocol allows orders to be signed not only by the maker but also by an allowed address. Allowances are stored in a mapping `orderUtils.sol::orderSignerRegistry`. Order signed by allowed addresses are validated in `_validateOrderSignatureAndGetOrderRelevantState(...)` with the code fragment presented below:

```solidity=
address signer = _recoverOrderSignerFromOrderDigest(orderInfo.orderHash, makerdata.signingScheme, order.signature);

isSignatureValid =
    (order.maker == signer) ||
    isValidOrderSigner(order.maker, signer);

require(
    !doRevertOnFailure || isSignatureValid,
    "RS:E7"
);
```

From the code above we can reason that the signer may either be the order maker or an address allowed by the order maker to sign on their behalf. To sign transactions with the pre-sign scheme users can call `batchSetPreSignature(...)`, however, this function can only be successfully called by the order maker and the allowed signers can't pre-sign the orders with this function. Below we present the requirement in `batchSetPreSignature(...)` that prohibits the allowed signer from signing orders:

```solidity=
require(orders[i].maker == msg.sender, "RS:E16");
```

**Recommendation(s)**: Keep the role's permissions consistent and explicitly document permissions for each role of the protocol. It is currently unclear whether an address allowed to sign on behalf of another maker should be allowed to use the pre-sign signature scheme.

**Status**: Unresolved 

**Update from the client**:

---

## `contracts/orderUtils.sol`

### [Medium] `address(0)` can be an approved signer address

**File(s)**: [`orderUtils.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/orderUtils.sol)

**Description**: The function `registerAllowedOrderSigner(...)` does not check if the argument `signer` is not `address(0)`, allowing `address(0)` to be considered a valid signer. When an EIP1271 or pre-sig signature scheme recovery fails within `recoverOrderSignerFromDigest(...)`, then `address(0)` is returned. This allows failed signatures for `EIP1271` or `pre-signature` schemes to be considered valid if a user has allowed `address(0)` to sign on their behalf, as the return value from the signer recovery is also `address(0)`. Consider the following scenario:

1. Bob calls `registerALlowedOrderSigner(0, true)`.
2. Eve (a malicious keeper) incorrectly signs an order with an `EIP1271` or `pre-signature` scheme with Bob's address as the order maker.
3. Eve can execute an order without Bob's permission, as the recovered signed returns zero, which is considered an allowed order signer for Bob.

**Recommendation(s)**: Consider reverting if `signer` is `address(0)` in the function `registerAllowedOrderSigner(...)`.

**Status**: Unresolved 

**Update from the client**:

---
### [Low] `_getActualFillableMakerAmount` may return misleading information

**File(s)**: [`orderUtils.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/orderUtils.sol)

**Description**: The function `_getActualFillableMakerAmount` will return early with the zero value if the `order.takerAmountMin` is zero.

```solidity=
if (order.makerAmount == 0 || order.takerAmountMin == 0)
{
    // Empty order  
    return 0;
}
```

`Orders` may be created with `takerAmountMin` equal to zero, but with a `decay` configuration that makes `takerAmount` greater than zero until expiration. In these cases, the function will return that the `Order` is not fillable while it actually can be filled. This could cause unexpected results in external components relying on this function.

**Recommendation(s)**: Remove the `order.takerAmountMin == 0` from the condition.
```diff
- if (order.makerAmount == 0 || order.takerAmountMin == 0)
+ if (order.makerAmount == 0)
```

**Status**: Unresolved

**Update from the client**:

---
### [Info] Unnecessary double storage access

**File(s)**: [`orderUtils.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/orderUtils.sol)

**Description**: The storage mapping variable `makerAmountFilled[orderInfo.orderHash]` is accessed twice, even though it is cached in the `orderInfo.makerFilledAmount`. The second access using the storage map could use the `orderInfo.makerFilledAmount` from memory instead to save gas costs. The code is shown below:

```solidity=
    function _validateOrderSignatureAndGetOrderRelevantState(
        Order calldata order,
        bytes32 orderHash,
        LibData.MakerData memory makerdata,
        bool doRevertOnFailure,
        bool doGetActualFillableMakerAmount
    )
        ...
        orderInfo.orderHash = orderHash;
        orderInfo.makerFilledAmount = makerAmountFilled[orderInfo.orderHash];
        ///////////////////////////////////////////////////////////////////////////
        // @audit The second mapping access could use `orderInfo.makerFilledAmount`
        ///////////////////////////////////////////////////////////////////////////
        if (makerAmountFilled[orderInfo.orderHash] & HIGH_BIT != 0) {
            orderInfo.status = OrderStatus.CANCELLED;
        }
    }
```

**Recommendation(s)**: Consider using the memory variable `orderInfo.markeFilledAmount` instead of the storage mapping to save gas costs.

**Status**: Unresolved 

**Update from the client**:

---
### [Info] `batchGetOrderRelevantStates` may return misleading information to external parties

**File(s)**: [`orderUtils.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/orderUtils.sol)

**Description**: The function `batchGetOrderRelevantStates` can be used to return the fillable maker amounts for a batch of orders. Querying this function with a batch that contains two orders with the same maker address and same maker token can incorrectly calculate the number of spendable tokens assuming that both orders would be executed in the same batch. The function `_getSpendableERC20BalanceOf` returns the spendable balance for each order, but does not take into account the amount that could have been spent in a previous order in the same batch.

Consider the scenario where there are two orders with the same maker address and same maker token. The first order has a `makerAmount` of 60 and the second order has a `makerAmount` of 50. If the maker's spendable balance is 100, both orders will return that the actual fillable amount is equal to its `makerAmount` as 100 is greater than 50 and 60. However, if this batch were to be executed with `swapKeeper_eD` the first order would use 60 of the user's balance leaving a balance of 40 and the second order would fail as it attempts to transfer 50 tokens.

**Recommendation**: As this issue does not cause direct harm to the protocol as it is not used internally (in a worst case scenario a keeper will blindly accept this data and their order execution will revert), this edge case could be documented so that external parties are made aware. Alternatively, the function logic can be changed so that orders from the same maker address with the same maker tokens take into account "hypothetically spent" tokens from previous orders.

**Status**: Unresolved 

**Update from the client**:

---

## `contracts/swap`

### [Medium] Missing additional checks for fund losses after a swap

**File(s)**: [`swap.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/swap.sol)

**Description**: The function `_finalizeSwapExecution_dexAggKeeper(...)` is designed to ensure that the `RookSwap` contract hasn't lost any funds after swaps have been executed. However, it only ensures that the balance of the surplus tokens has not decreased. All other maker and taker tokens in the order batch are not checked if their balance has decreased which can lead to loss of `RookSwap` contract value without reverting. The following are some scenarios where non-surplus tokens can be taken from the `RookSwap` contract.

a) The `makerAmountsToSpend` array contains maker amounts related to orders. However, its values are never checked.  If `makerAmountsToSpend[i] == 0`, the `_prepareSwapExecution(...)` function transfers zero amount of `makerToken` from the maker to the contract, and even though the maker can receive the taker token amount(`takerAmountFilled`).  In this scenario, the threat comes if `partiallyFillable` is `true` and the contract has the required amount (`orders[i].makerAmount`) to begin the swap execution in `_beginDexAggSwapExecution(...)`. The swap completes its execution using the contract's maker tokens. 

b) The custom token output distribution logic doesn't validate so it is possible to distribute more tokens than were swapped, allowing for tokens that belong to `RookSwap` to be sent to makers.

c) Infinite allowances are given to routers. This opens the potential for routers to use more tokens than expected, including tokens that belong to the `RookSwap` contract.

**Recommendation(s)**: To prevent the `RookSwap` contract from losing surplus tokens after swaps have been executed, the pre and post-trade balances are compared. This should also be applied to all approved tokens in the current order batch as well as all taker tokens.

**Status**: Unresolved

**Update from the client**:

---
### [Medium] `DexAggKeepers` can perform arbitrary calls

**File(s)**: [`swap.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/swap.sol)

**Description**: `DexAggKeepers` are not in the scope for this audit, although they are considered as trusted entities by the protocol. The plan is to have only one `DexAggKeeper`, which is designed by the RookFi development team. The team also claims that there is no intention of allowing other keepers to be part of this special category of keepers. However, **the `DexAggKeeper` has the capacity to exploit the protocol** in many ways as it is able to execute arbitrary calls. This can allow behavior such as taking all the funds from the contract (also taking funds from users who have provided allowance to the protocol).

**Recommendation(s)**: Consider limiting the actions that can be done by this category of keepers, such as whitelisting valid `DexAgg` routers to prevent arbitrary calls to any address. Also consider inspecting the code of new `DexAgg` keepers to ensure they will behave correctly.

**Status**: Unresolved

**Update from the client**:

---
### [Medium] `DexAgg` router allowances are never revoked

**File(s)**: [`swap.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/swap.sol)

**Description**: When starting the swap execution on DexAggs, the function `_dexAggKeeperSwap(...)` approves the DexAgg to spend unlimited tokens and never revokes this allowance. Consequently, the addresses set as `routers` can potentially transfer all the tokens of the contract. 

```solidity=
function _dexAggKeeperSwap(LibSwap.DexAggSwap[] memory swaps)
    private
    returns (uint256 tradeOutputAmount)
{
    ...
    //////////////////////////////////////////////////////////////
    // @audit Unlimited allowance can lead to loss of funds
    //////////////////////////////////////////////////////////////
    IERC20(swaps[i].approveToken).approveMaxAllowanceOnlyIfNeeded(
        swaps[i].router
    );
    ...
}
```

**Recommendation(s)**: Revoke the allowance when the swap execution is finalized.

**Status**: Unresolved

**Update from the client**:

---
### [Low] Minimum `takerAmountFilled` is rounded down

**File(s)**: [`swap.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/swap.sol)

**Description**: The function `_finalizeSwapExecution(...)` must check if the amount sent by the taker to the maker after the swap is enough to satisfy the order requirements. This is checked with the following code:

```solidity=
require(
    takerAmountFilled >= currentTakerAmountMin * makerAmountsToSpend[i] / orders[i].makerAmount,
    "RS:E23"
);
```

The expression `currentTakerAmountMin * makerAmountsToSpend[i] / orders[i].makerAmount` rounds the value down. If there is a numerical error in the division, the maker won't receive the total value they deserve. Because the keeper is the actor who decides the values to send, this should be rounded against it, favoring the maker. The loss value would be low most of the time, but there may exist tokens where the minimal unit still has a non-negligible value. The existence of these tokens added to the possibility of amplifying the loss by filling the `Order` partially many times until it is completely filled could cause the loss of relevant value for makers.

**Recommendation(s)**: Avoid divisions to prevent numerical errors.

```diff=
- require(
-     takerAmountFilled >= currentTakerAmountMin * makerAmountsToSpend[i] / orders[i].makerAmount,
-     "RS:E23"
- );
+ require(
+     takerAmountFilled * orders[i].makerAmount >= currentTakerAmountMin * makerAmountsToSpend[i],
+     "RS:E23"
+ );
```

**Status**: Unresolved 

**Update from the client**:

---
### [Low] `surplusToken` is not checked to be a valid address

**File(s)**: [`swap.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/swap.sol)

**Description**: The function `swapDexAggKeeper_54P(...)` does not ensure that the passed surplus token address is one of the tokens in the batched orders. 

```solidity=
function swapDexAggKeeper_54P(...) external nonReentrant returns (uint256 surplusAmount) {
    
    require(isWhitelistedDexAggKeeper(msg.sender) == true, "RS:E22");        
    /////////////////////////////////////////////////////////
    // @audit Not checking if "surplusToken" is a valid address
    /////////////////////////////////////////////////////////
    LibData.ContractData memory contractData = LibData.ContractData(
        IERC20(metaData.surplusToken).balanceOf(address(this)),
        0
        );
    ...
}
```

**Recommendation(s)**: Check if `metaData.surplusToken` is one of the tokens included in the batch.

**Status**: Unresolved 

**Update from the client**:

---
### [Info] `swapDexAggKeeper_54P` is missing order batch validation

**File(s)**: [`swap.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/swap.sol)

**Description**: In communications with the RookFi team, it was described that `swapDexAggKeeper` function only supports batches of `Orders` involving only two different tokens. This is not enforced by the RookSwap contract, and processing batches that do not meet this requirement could cause unexpected results. It is worth mentioning that this function is only callable by one keeper built by the RookFi team, and currently there is no intention to add any other keeper to this category. However, it is still possible that these invalid batches can be passed into this function.

**Recommendation(s)**: Check in the `swapDexAggKeeper` function that the batches of orders submitted meet the expected requirements.

**Status**: Unresolved

**Update from the client**:

---
### [Best Practices] Gas saving opportunity in `_calculateCurrentTakerAmountMin(...)`

**File(s)**: [`swap.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/swap.sol)

**Description**: The function `_calculateCurrentTakerAmountMin(...)` calculates the current minimum amount of taker tokens, taking price decay into account. The price decay is an optional feature, which depends on the variable `takerAmountDecayRate` being zero or not. However, if the price decay feature is not being used, the complex computation in `_calculateCurrentTakerAmountMin(...)` is still done but with `takerAmountDecayRate` being `0` which does not affect the current taker tokens amount. This causes unnecessary gas consumption.

**Recommendation(s)**: Consider calling `_calculateCurrentTakerAmountMin(...)` only if `takerAmountDecayRate` is greater than `0` to save gas.

**Status**: Unresolved 

**Update from the client**:

---
### [Best Practices] Off-chain sorting of orders can improve time complexity of `prepareSwapExecution(...)`. 

**File(s)**: [`swap.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/swap.sol)

**Description**: The function `_prepareSwapExecution(...)` uses a nested loop to ensure the following two conditions are met for a batch of orders: **a) orders from the same `maker` cannot have the same `takerToken`; b) the `takenToken` from an order cannot be the same as the `makerToken` from another order with the same `maker`**.

The nested loop that checks for these conditions has a time complexity of O(n^2), but can be reduced to O(n) with off-chain order sorting. The orders can be primarily sorted by `maker` and secondarily sorted by `takerToken`. The following shows an example of such a sort:

```shell=
// Unsorted
Order:    Maker = 0x1    takerToken = 0x4
Order:    Maker = 0x3    takerToken = 0x1
Order:    Maker = 0x2    takerToken = 0x1
Order:    Maker = 0x1    takerToken = 0x1
Order:    Maker = 0x1    takerToken = 0x2
Order:    Maker = 0x3    takerToken = 0x2

// Sorted
Order:    Maker = 0x1    takerToken = 0x1
Order:    Maker = 0x1    takerToken = 0x2
Order:    Maker = 0x1    takerToken = 0x4
Order:    Maker = 0x2    takerToken = 0x1
Order:    Maker = 0x3    takerToken = 0x1
Order:    Maker = 0x3    takerToken = 0x2
```

When a batch of orders is sorted as described above, we only need to check the elements consecutively. If there are two orders with the same `maker` and `takerToken` it is easy to identify as you only need to check the next element in the order array. If two orders from the same `maker` have the same `takerToken` then the contract should revert. This satisfies the first condition.

The second condition exists to prevent `takerTokenBalance_before` values from being incorrect. If a `maker` creates two orders where the `takerToken` of one order matches the `makerToken` of the other, the recorded `takerTokenBalance_before` can be incorrect. Consider the following scenario:

```shell=
Order1: Alice swaps 1000 DAI -> 1 WETH
Order2: Alice swaps 2 WETH   -> 2000 USDC 
```

When looping through Alice's Order1, her `takerTokenBalance_before` (WETH) is recorded. On the loop through Alice's Order2, the `makerToken` is WETH so it is transferred out. This means that the `takerTokenBalance_before` value that was recorded for Order1 is now incorrect.

This problem (and therefore the second condition as a whole) can be avoided by recording the `takerToken` balances after the loop has already finished, as all token transfers will already be done. Continuing with the same scenario as above, recording the balances after all transfers would result in the correct `takerTokenBalance_before` values. The code snippet below shows a potential approach to checking the balances after token transfers:

```solidity=
// Loop with the transfers
for (uint8 i = 0; i < orders.length; i++) {
    ...
    IERC20(orders[i].makerToken).safeTransferFrom(
        orders[i].maker,
        makerTokenRecipient,
        makerAmountsToSpend[i]
    );
    ...
}

// Check the balances after all transfers have been done
for (uint8 i = 0; i < orders.length; i++) {
    makerData[i].takerTokenBalance_before = IERC20(orders[i].takerToken).balanceOf(orders[i].maker);
}
```

The gas comparison is presented in the figure below.

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook12.pdf}}
   \label{fig:rook12}
\end{figure}
-->

![](https://i.imgur.com/hKK21Sw.png)


**Recommendation(s)**: Require keepers to sort the order batches off-chain and validate on-chain if the orders are sorted.

**Helper Code**: Below we present the test code used to validate the proposal.

```solidity=
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.6;

contract LoopsGasComparing {

    struct Order {
        address maker;
        address takerToken;
    }

    function check(
        Order[] calldata orders,
        uint256 checkOption
    ) external returns (uint256 gasUsed) {
        uint256 startGas = gasleft();
        if (checkOption == 0) {
            _prepareSwapExecution(orders);
        } else {
            _prepareSwapExecution2(orders);
        }
        gasUsed = startGas - gasleft();
    }

    function _prepareSwapExecution(
        Order[] calldata orders
    )
        private
    {
        for (uint8 i = 0; i < orders.length; i++)
        {
            for (uint8 j = 0; j < orders.length; j++)
            {
                if (i != j && orders[i].maker == orders[j].maker && orders[i].takerToken == orders[j].takerToken)
                {
                    revert("RS:E15");
                }
            }
        }
    }

    function _prepareSwapExecution2(
        Order[] calldata orders
    )
        private
    {
        for (uint8 i = 0; i < orders.length; i++)
        {
            // We will check i with i-1 so if i == 0, just skip
            if (i != 0) {
                // Make sure orders list is sorted
                // Using < instead of <= will also check the equality of them.
                require(orders[i].maker < orders[i-1].maker || 
                    (orders[i].maker == orders[i-1].maker && orders[i].takerToken < orders[i-1].takerToken), 
                    "RS:E15"
                );
            }
        }
    }
}
```

```typescript=
import { ethers } from "hardhat";
import { LoopsGasComparing } from "../typechain-types";

function getRandomEthereumAddress(): string {
  const length: number = 40;
  const number: string = [...Array(length)]
    .map(() => {
      return Math.floor(Math.random() * 16).toString(16);
    })
    .join("");
  return "0x" + number;
}

function getRandomOrdersList(length: number): LoopsGasComparing.OrderStruct[] {
  let orders = [];
  for (let i = 0; i < length; i++) {
    orders.push({
      maker: getRandomEthereumAddress(),
      takerToken: getRandomEthereumAddress() 
    });
  }
  return orders;
}

function sortOrdersList(orders: LoopsGasComparing.OrderStruct[]): LoopsGasComparing.OrderStruct[] {
  return orders.sort((n1, n2) => {
    if (n1.maker == n2.maker && n1.takerToken == n2.takerToken) {
      return 0;
    }
    if (n1.maker < n2.maker || (n1.maker == n2.maker && n1.takerToken < n2.takerToken)) {
      return 1;
    }
    return -1;
  });
}

describe("LoopsGasComparing", function () {
  
  let contract: LoopsGasComparing;

  before("Deploy", async () => {
    const Contract = await ethers.getContractFactory("LoopsGasComparing");
    contract = await Contract.deploy();
  });

  it("Test", async function () {

    let lengths = [1, 2, 3, 4, 5, 10, 20, 50, 100];
    for (let i = 0; i < lengths.length; i++) {
      let length = lengths[i];
      let orders = getRandomOrdersList(length);
      let gasConsumption0 = (await contract.callStatic.check(orders, 0));
      orders = sortOrdersList(orders);
      let gasConsumption1 = (await contract.callStatic.check(orders, 1));
      
      console.log(length, gasConsumption0.toString(), gasConsumption1.toString());
    };
  });
});
```

**Status**: Unresolved

**Update from the client**:

---
### [Best Practices] `_calculateCurrentTakerAmountMin(...)` can be optimized for gas consumption

**File(s)**: [`swap.sol`](https://github.com/rookprotocol/Swap/blob/519ab5ba70e982565d443619b8ace8bf7c78168c/contracts/swap.sol)

**Description**: The function `_calculateCurrentTakerAmountMin(...)` is responsible for computing the minimal amount required by the users using a decay formula in the format of a Dutch auction, which refers to a type of auction in which an auctioneer starts with a very high price, incrementally lowering the price until someone places a bid. The original code is reproduced below.

```solidity=
function _calculateCurrentTakerAmountMin( uint256 takerAmountMin, uint256 takerAmountDecayRate, LibData.MakerData memory makerData)
    private view returns (uint256 currentTakerAmountMin)
{
    if (block.timestamp <= uint256(makerData.begin))
    {
        currentTakerAmountMin = takerAmountMin +
            ((uint256(makerData.expiry) - uint256(makerData.begin)) * takerAmountDecayRate);
    }
    else if (block.timestamp > uint256(makerData.expiry))
    {
        currentTakerAmountMin = takerAmountMin;
    }
    else
    {
        currentTakerAmountMin =
            takerAmountMin +
            ((uint256(makerData.expiry) - uint256(makerData.begin)) * takerAmountDecayRate) -
            ((block.timestamp - uint256(makerData.begin)) * takerAmountDecayRate);
    }
    return currentTakerAmountMin;
}
```

The function can be simplified to the code presented below.

```solidity=
function _new_calculateCurrentTakerAmountMin(uint256 takerAmountMin, uint256 takerAmountDecayRate, LibData.MakerData memory makerData)
    private internal returns (uint256 currentTakerAmountMin)
{
    uint256 timestamp = block.timestamp >= uint256(makerData.begin) ? block.timestamp : uint256(makerData.begin);
    uint256 valueToAdd  = block.timestamp < uint256(makerData.expiry) ? uint256(makerData.expiry) - timestamp: 0;

    currentTakerAmountMin = takerAmountMin + takerAmountDecayRate * valueToAdd;
    return (currentTakerAmountMin);
}
```

The figure below presents the comparison of gas costs using [Remix](https://remix.ethereum.org/). We consider three scenarios: a) `block.timestamp` smaller than `makerData.begin`; b) `block.timestamp` inside the dutch auction decay process; c) `block.timestamp` greater than `makerData.expiry`.

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook10.pdf}}
   \label{fig:rook10}
\end{figure}
-->

![](https://i.imgur.com/M3bKO1z.png)

We notice that the new implementation consumes 471 more gas units than the original implementation presented in the protocol. However, during the decay process, the new strategy saves 517 gas units. Finally, after the expiry date is reached, the new strategy saves 83 gas units per call. Below we present the figure summarizing the gas relations.

<!--
\begin{figure}[H]
   \centering
   \fbox{\includegraphics[trim=0cm 0cm 0cm 0cm, clip=true, width=1\textwidth]{img/rook11.pdf}}
   \label{fig:rook11}
\end{figure}
-->

![](https://i.imgur.com/JsUi2qH.png)

The proposed function has been tested for over 200,000 inputs having as scenarios:
- `makerData_begin < block.timestamp`
- `makerData_begin < block.timestamp and makerData_expiry > makerData_begin`
- `makerData_begin=0 and makerData_expiry<block.timestamp`
- `makerData_begin=0 and makerData_expiry>block.timestamp`
- `takerAmountMin=0 and makerData_begin=0 and makerData_expiry>block.timestamp`
- random inputs

In case the new function does not meet the requirements of the protocol, you can simply choose to simplify the formula of the last branch. When price decay is active, the formula used for computing `currentTakerAmountMin` is
```solidity=
currentTakerAmountMin = takerAmountMin +
    ((uint256(makerData.expiry) - uint256(makerData.begin)) * takerAmountDecayRate) -
    ((block.timestamp - uint256(makerData.begin)) * takerAmountDecayRate);
```

This is equivalent to the formula shown below, which removes redundant operations involving `uint256(makerData.begin)`.

```solidity=
currentTakerAmountMin = takerAmountMin +
    (uint256(makerData.expiry) - block.timestamp) * takerAmountDecayRate
```

**Recommendation(s)**: The proposed code has lower readability than the original code presented in the protocol. Also, the new code reverts transactions having `makerData.expiry` smaller than `makerData.begin`. The development team also must consider the frequency of scenarios. In the case of the scenario (a) (`block.timestamp` smaller than `makerData.begin`) tends to be more frequent, then adopting the new code will incur higher gas usage. Thus, care must be taken before replacing the original code for this one. Evaluate if the new code passes all tests and if it is worth adopting such implementation. In case you don't consider a good change, just use the simplified version of the formula for the last branch in the original code.

**Helper Code**: Below we present the test code developed for validating this proposal.

```javascript=
const { time, loadFixture, } = require("@nomicfoundation/hardhat-network-helpers");
const { anyValue }           = require("@nomicfoundation/hardhat-chai-matchers/withArgs");
const { expect }             = require("chai");
const helpers                = require("@nomicfoundation/hardhat-network-helpers");
const chai                   = require('chai');
const BN                     = require('bn.js');

chai.use(require('chai-bn')(BN));

describe("calc", function () 
{
  async function deployTokenFixture() 
  {
    const ContractFactory = await ethers.getContractFactory("CrisWhitelist");
    const contract        = await ContractFactory.deploy({});
    return {ContractFactory, contract};
  }

  it("testing makerData_begin < block.timestamp", async function () 
  {
    const { ContractFactory, contract } = await loadFixture(deployTokenFixture);
    console.log("i lastBlock takerAmountMin takerAmountDecayRate makerData_begin makerData_expiry value1 value2");
    for (let i=1; i<=50000; i++)
    {
      let lastBlock = 1669844628;
      let takerAmountMin = Math.floor(Math.random() * 1000000000);
      let takerAmountDecayRate = Math.floor(Math.random() * 1000000000);
      let makerData_begin  = Math.floor(Math.random() * 1669844628);
      let makerData_expiry = Math.floor(Math.random() * 1669844628);
    
      value1 = await contract._calculateCurrentTakerAmountMin(takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry);
      value2 = await contract._new_calculateCurrentTakerAmountMin(takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry);
      console.log(i,lastBlock,takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry,value1,value2);
  
      expect(value1).to.be.equal(value2);
      expect(value1).to.be.equal(takerAmountMin);
    }
  });

  it("testing makerData_begin < block.timestamp and makerData_expiry > makerData_begin", async function () 
  {
    const { ContractFactory, contract } = await loadFixture(deployTokenFixture);
    console.log("i lastBlock takerAmountMin takerAmountDecayRate makerData_begin makerData_expiry value1 value2");
    for (let i=1; i<=50000; i++)
    {
      let lastBlock = 1669844628;
      let takerAmountMin = Math.floor(Math.random() * 1000000000);
      let takerAmountDecayRate = Math.floor(Math.random() * 1000000000);
      let makerData_begin  = lastBlock - Math.floor(Math.random() * lastBlock);
      let makerData_expiry = makerData_begin - Math.floor(Math.random() * 1000);
    
      value1 = await contract._calculateCurrentTakerAmountMin(takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry);
      value2 = await contract._new_calculateCurrentTakerAmountMin(takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry);
      console.log(i,lastBlock,takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry,value1,value2);
  
      expect(value1).to.be.equal(value2);
      expect(value1).to.be.equal(takerAmountMin);
    }
  });

  it("testing makerData_begin=0 and makerData_expiry<block.timestamp", async function () 
  {
    const { ContractFactory, contract } = await loadFixture(deployTokenFixture);
    console.log("i lastBlock takerAmountMin takerAmountDecayRate makerData_begin makerData_expiry value1 value2");
    for (let i=1; i<=50000; i++)
    {
      let lastBlock = 1669844628;
      let takerAmountMin = Math.floor(Math.random() * 1000000000);
      let takerAmountDecayRate = Math.floor(Math.random() * 1000000000);
      let makerData_begin  = 0
      let makerData_expiry = Math.floor(Math.random() * lastBlock)-1;
    
      value1 = await contract._calculateCurrentTakerAmountMin(takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry);
      value2 = await contract._new_calculateCurrentTakerAmountMin(takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry);
      console.log(i,lastBlock,takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry,value1,value2);
  
      expect(value1).to.be.equal(value2);
      expect(value1).to.be.equal(takerAmountMin);
    }
  });

  it("testing makerData_begin=0 and makerData_expiry>block.timestamp", async function () 
  {
    const { ContractFactory, contract } = await loadFixture(deployTokenFixture);
    console.log("i lastBlock takerAmountMin takerAmountDecayRate makerData_begin makerData_expiry value1 value2");
    for (let i=1; i<=50000; i++)
    {
      let lastBlock = 1669844628;
      let takerAmountMin = Math.floor(Math.random() * 1000000000);
      let takerAmountDecayRate = Math.floor(Math.random() * 1000000000);
      let makerData_begin  = 0
      let makerData_expiry = lastBlock + Math.floor(Math.random() * lastBlock);
    
      value1 = await contract._calculateCurrentTakerAmountMin(takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry);
      value2 = await contract._new_calculateCurrentTakerAmountMin(takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry);
      console.log(i,lastBlock,takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry,value1,value2);
  
      expect(value1).to.be.equal(value2);
      expect(value1).to.be.above(takerAmountMin);
    }
  });

  it("testing takerAmountMin=0 and makerData_begin=0 and makerData_expiry>block.timestamp", async function () 
  {
    const { ContractFactory, contract } = await loadFixture(deployTokenFixture);
    console.log("i lastBlock takerAmountMin takerAmountDecayRate makerData_begin makerData_expiry value1 value2");
    for (let i=1; i<=50000; i++)
    {
      let lastBlock = 1669844628;
      let takerAmountMin = 0
      let takerAmountDecayRate = Math.floor(Math.random() * 1000000000);
      let makerData_begin  = 0
      let makerData_expiry = lastBlock + Math.floor(Math.random() * lastBlock);
    
      value1 = await contract._calculateCurrentTakerAmountMin(takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry);
      value2 = await contract._new_calculateCurrentTakerAmountMin(takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry);
      console.log(i,lastBlock,takerAmountMin,takerAmountDecayRate,makerData_begin,makerData_expiry,value1,value2);
  
      expect(value1).to.be.equal(value2);
    }
  });
});
```

**Status**: Unresolved 

**Update from the client**:

---


