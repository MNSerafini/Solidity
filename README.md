---

# Auction Contract

---
## Author

Martin Nicolas Serafini


---

This repository contains a Solidity smart contract that implements a feature-rich auction system. It allows participants to place bids, enforces bid increments, and handles auction extensions for last-minute bids.

---

## Features

This contract provides a robust auction mechanism with the following key features:

* **Bid Increment Enforcement**: Requires new bids to be at least 5% higher than the current highest bid.
* **Auction Extension**: Bids placed within the last 60 seconds of the auction automatically extend the auction duration by a predefined `extensionTime`.
* **Automatic Refunds (for test purposes)**: Overbid amounts (minus a 2% commission) are immediately refunded to the previous highest bidder.
    * **Note**: For production environments, a pull payment system (where users withdraw their refunds) is generally recommended over automatic refunds for enhanced security and gas efficiency. This contract implements automatic refunds for testing convenience.
* **Time-Based Conclusion**: The auction automatically concludes when the `block.timestamp` surpasses `auctionEndTime`.
* **Commission Handling**: A 2% commission is applied to both overbids and the winning bid. The designated `commissionRecipient` (set during deployment) can claim accumulated commissions via the `claimCommission()` function.
* **Proceeds Claim**: The `proceedsRecipient` (set during deployment) can claim the net proceeds from the winning bid using `claimProceeds()` once the auction has officially ended.
* **Detailed History Tracking**:
    * **Bid History**: Tracks all bids made on the contract in order of completion.
    * **Participant-Specific History**: Maintains detailed records of all bids, refunds, and commissions for each individual participant.

---

## Pending Features

The following features are planned for future development:

* **Optimized Commission Calculation for Recurring Bidders**: In cases where the same bidder places multiple consecutive bids, the commission should ideally be calculated only on the difference between their previous and current bid, rather than the full bid amount.

---

## Contract Details

### `Auction.sol`

This is the main Solidity contract file that defines the auction logic.

**Solidity Version**: `pragma solidity >=0.8.2 <0.9.0;`
**License**: `GPL-3.0`

### Core Variables

* `owner`: The address that deployed the contract, controlling claim functions.
* `commissionRecipient`: The address designated to receive all accumulated commissions.
* `proceedsRecipient`: The address designated to receive the net proceeds from the winning bid.
* `highestBidder`: The address of the current highest bidder.
* `highestBid`: The current highest bid amount in Wei.
* `auctionEndTime`: The current timestamp when the auction is scheduled to end (can be extended).
* `initialEndTime`: The original, fixed end time of the auction.
* `extensionTime`: The duration in seconds by which the auction is extended if a bid is placed in the last 60 seconds.
* `commissionTotal`: The total amount of commission accumulated.
* `ownerProceedsPending`: The proceeds from the winning bid that are pending transfer to the `proceedsRecipient`.

### Structs

* `Bid`: A structure to store details of an individual bid, including `bidder` (address) and `amount` (uint256).

### Events

* `NewBid(address indexed bidder, uint256 amount)`: Emitted when a new bid is successfully placed.
* `AuctionEnded(address winner, uint256 amount)`: Emitted when the auction officially concludes.
* `Refunded(address indexed bidder, uint256 amount)`: Emitted when an overbid amount (minus commission) is refunded to a previous bidder.
* `CommissionClaimed(address indexed recipient, uint256 amount)`: Emitted when accumulated commissions are claimed by the `commissionRecipient`.
* `ProceedsTransferred(address indexed recipient, uint256 amount)`: Emitted when the net proceeds from the winning bid are transferred to the `proceedsRecipient`.

### Modifiers

* `onlyOwner()`: Restricts function execution to the contract owner.
* `auctionOngoing()`: Ensures a function can only be called while the auction is still active.
* `auctionEnded()`: Ensures a function can only be called after the auction has concluded.

### Functions

#### Constructor

```solidity
constructor(
    uint256 _durationSeconds,
    uint256 _extensionSeconds,
    address _commissionRecipient,
    address _proceedsRecipient
)
```

Initializes the auction with:
* `_durationSeconds`: The initial duration of the auction in seconds (must be >= 120 seconds).
* `_extensionSeconds`: The time in seconds to extend the auction if a bid is placed in the last 60 seconds (must be >= 30 seconds).
* `_commissionRecipient`: The address to receive all accumulated commissions.
* `_proceedsRecipient`: The address to receive the net proceeds from the winning bid.

#### `bid()`

```solidity
function bid() external payable auctionOngoing
```

Allows users to place a bid by sending ETH to the contract.
* Requires `msg.value` to be at least 5% higher than the current `highestBid`.
* Automatically attempts to refund the previous highest bidder (minus 2% commission).
* Extends the auction duration if the bid is placed within the last 60 seconds.

#### `claimCommission()`

```solidity
function claimCommission() external onlyOwner auctionEnded
```

Allows the contract `owner` to claim all accumulated commissions, which are then sent to the `commissionRecipient`. Can only be called after the auction has ended.

#### `claimProceeds()`

```solidity
function claimProceeds() external onlyOwner auctionEnded
```

Allows the contract `owner` to claim the net proceeds from the `highestBid`, which are then sent to the `proceedsRecipient`. Can only be called after the auction has ended.

#### Getters (Read-Only Functions)

* `getAllBids() external view returns (Bid[] memory)`: Returns an array of all `Bid` structs recorded in the auction.
* `getBidHistory(address bidder) external view returns (Bid[] memory)`: Returns all bids made by a specific `bidder`.
* `getRefundHistory(address bidder) external view returns (uint256[] memory)`: Returns all refund amounts processed for a specific `bidder`.
* `getCommissionHistory(address bidder) external view returns (uint256[] memory)`: Returns all commission amounts charged to a specific `bidder`.
* `getTimeLeft() external view returns (uint256)`: Returns the number of seconds remaining until the auction ends, or 0 if it has already concluded.
* `getAuctionState() external view returns (bool isEnded, uint256 timeLeft)`: Returns a boolean indicating if the auction has ended and the remaining time in seconds.

---

## Deployment

To deploy this contract, you will need a Solidity development environment (e.g., Hardhat, Foundry, Remix).

1.  **Compile the contract**:
    ```bash
    solc --abi --bin Auction.sol
    ```
    (Or use your chosen framework's compile command)
2.  **Deploy the compiled bytecode** to your desired Ethereum-compatible network, providing the required constructor arguments: `_durationSeconds`, `_extensionSeconds`, `_commissionRecipient`, and `_proceedsRecipient`.

---

## Usage

Once deployed, users can interact with the contract by:

* Calling the `bid()` function and sending Ether to place their bid.
* The `owner` can call `claimCommission()` and `claimProceeds()` after the auction has ended.
* Anyone can view the auction's state and history using the various `get*` functions.

---

## Security Considerations

* **Automatic Refunds**: While implemented for testing, automatic refunds in Solidity can be susceptible to reentrancy attacks if not handled carefully. This contract attempts to mitigate this by updating state before external calls. For production systems, a pull payment system is generally safer.
* **Gas Limits**: Be aware of gas limits when performing transactions, especially for functions involving external calls.

---
