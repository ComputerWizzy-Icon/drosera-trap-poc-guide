# Drosera PoC Trap – Complete Beginner Deployment Guide

This guide walks you **step by step** through how to:

* Set up your environment (even if you’ve never done this before)
* Understand how Drosera traps actually work (no wrong mental models)
* Build a **Drosera-correct PoC trap**
* Avoid the most common reviewer-killing mistakes
* Deploy, test, and wire your trap correctly
* Push everything cleanly to GitHub

If you follow this guide **top to bottom**, your trap will run correctly under Drosera operators.

---

## What a Drosera Trap Is (Read This First)

A **Drosera trap is NOT something you trigger manually**.

You do NOT:

* call the trap with ethers.js
* click a button to activate it
* call a responder from inside the trap

Instead, the execution model is:

1. **Drosera operators** deploy your trap on a shadow fork
2. Operators call `collect()` **on-chain** every block
3. Operators pass the collected outputs into `shouldRespond()` **off-chain**
4. If `shouldRespond()` returns `(true, payload)`,
   Drosera calls your **responder contract** (configured in TOML)

Your trap **must fit this model exactly**.

---

## Core Rules (Non-Negotiable)

Your trap **must**:

* Implement Drosera’s `ITrap` interface exactly
* Be **stateless**
* Be **deterministic**
* Never rely on `msg.sender`
* Never rely on constructor arguments
* Never call the responder directly
* Never revert in `collect()`

Breaking **any one** of these rules will cause your trap to be rejected or fail at runtime.

---

## ABI Rule (CRITICAL – READ CAREFULLY)

⚠️ **The `bytes` returned by `shouldRespond()` MUST be produced using `abi.encode(...)`
and MUST match the responder function signature defined in `drosera.toml`.**

Do NOT:

* return `bytes("...")`
* manually pack values
* return arbitrary payloads

Always do this:

```solidity
abi.encode(arg1, arg2, ...)
```

Example:
If your responder function is:

```
respond(int256,uint256)
```

Then your trap **must** return:

```solidity
abi.encode(int256Value, uint256Value)
```

This is the single most common beginner mistake.

---

## Required Interface (Must Match Exactly)

Every Drosera trap must implement this interface **exactly**:

```solidity
interface ITrap {
    function collect() external view returns (bytes memory);
    function shouldRespond(bytes[] calldata data)
        external
        pure
        returns (bool, bytes memory);
}
```

No extra parameters.
No different return types.

---

## 1. System Requirements

### Operating System

* Ubuntu 22.04+ (native or WSL)

### Tools

* Git
* VS Code
* Foundry
* GitHub account
* Basic terminal familiarity

---

## 2. Install System Tools

### Git

```bash
sudo apt update
sudo apt install git -y
git --version
```

### VS Code

```bash
sudo snap install code --classic
```

Install the **Solidity (Juan Blanco)** extension.

---

## 3. Install Foundry

```bash
curl -L https://foundry.paradigm.xyz | bash
foundryup
```

Verify:

```bash
forge --version
cast --version
```

---

## 4. Create the Trap Project

```bash
mkdir drosera-poc-trap
cd drosera-poc-trap
forge init
```

Expected structure:

```
src/
script/
test/
foundry.toml
```

---

## 5. Install Drosera Contracts

```bash
git clone https://github.com/drosera-network/contracts.git lib/drosera-contracts
```

Verify:

```bash
ls lib/drosera-contracts/src/interfaces/ITrap.sol
```

---

## 6. Configure Foundry Remappings (Correct Form)

Edit `foundry.toml`:

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]

remappings = [
  "drosera-contracts/=lib/drosera-contracts/src/"
]
```

This allows imports like:

```solidity
import {ITrap} from "drosera-contracts/interfaces/ITrap.sol";
```

---

## 7. Environment Variables

Create `.env`:

```bash
touch .env
```

Example:

```env
ETH_RPC_URL=https://0xrpc.io/hoodi
PRIVATE_KEY=0xYOUR_PRIVATE_KEY
```

Load:

```bash
source .env
```

---

## 8. Correct Drosera Architecture

A proper Drosera PoC typically looks like:

```
Off-chain detector (optional)
        ↓
Feed / Adapter contract (writes data)
        ↓
Trap (reads data in collect())
        ↓
Responder (called by Drosera)
```

Rules:

* Trap is never called directly
* Trap does not store state
* Trap only reads and evaluates
* Responder performs actions

---

## 9. Feed Contract (On-Chain Adapter)

This contract stores observations written by an off-chain detector or operator.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MirageFeed {
    int256 public lastDeltaBps;
    uint256 public lastTimestamp;
    address public owner;

    event ObservationPushed(int256 deltaBps, uint256 timestamp);

    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "NOT_OWNER");
        _;
    }

    function pushObservation(int256 deltaBps)
        external
        onlyOwner
    {
        uint256 ts = block.timestamp;
        lastDeltaBps = deltaBps;
        lastTimestamp = ts;
        emit ObservationPushed(deltaBps, ts);
    }

    function latestObservation() external view returns (int256, uint256) {
        return (lastDeltaBps, lastTimestamp);
    }
}
```

Note:
Observations are written externally, but **once written, they are fully on-chain and deterministic**.

---

## 10. Trap Contract (Drosera-Correct)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ITrap} from "drosera-contracts/interfaces/ITrap.sol";

interface IObservationFeed {
    function latestObservation() external view returns (int256, uint256);
}

contract MirageTrap is ITrap {
    address public constant FEED = 0xYourDeployedMirageFeed;

    int256 public constant THRESHOLD_BPS = 500;
    uint256 private constant ENCODED_LEN = 64; // 2 × 32-byte ABI words

    function collect() external view override returns (bytes memory) {
        uint256 size;
        assembly {
            size := extcodesize(FEED)
        }
        if (size == 0) return bytes("");

        try IObservationFeed(FEED).latestObservation()
            returns (int256 deltaBps, uint256 ts)
        {
            return abi.encode(deltaBps, ts);
        } catch {
            return bytes("");
        }
    }

    function shouldRespond(bytes[] calldata data)
        external
        pure
        override
        returns (bool, bytes memory)
    {
        if (data.length == 0 || data[0].length != ENCODED_LEN) {
            return (false, bytes(""));
        }

        (int256 curDelta, uint256 curTs) =
            abi.decode(data[0], (int256, uint256));

        bool alert =
            (curDelta > THRESHOLD_BPS || curDelta < -THRESHOLD_BPS);

        if (!alert) return (false, bytes(""));

        return (true, abi.encode(curDelta, curTs));
    }
}
```

---

## 11. Responder Contract

Must match the ABI exactly.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract MirageResponder {
    event MirageAlert(int256 deltaBps, uint256 timestamp);

    function respond(int256 deltaBps, uint256 timestamp) external {
        emit MirageAlert(deltaBps, timestamp);
    }
}
```

---

## 12. drosera.toml (Critical Wiring)

```toml
ethereum_rpc = "https://0xrpc.io/hoodi"
drosera_rpc  = "https://relay.hoodi.drosera.io"
eth_chain_id = 560048
drosera_address = "0x91cB447BaFc6e0EA0F4Fe056F5a9b1F14bb06e5D"

[traps.mirage_attack]
path = "out/MirageTrap.sol/MirageTrap.json"
response_contract = "0xYourMirageResponder"
response_function = "respond(int256,uint256)"

block_sample_size = 1
cooldown_period_blocks = 20

min_number_of_operators = 1
max_number_of_operators = 5

private_trap = true
whitelist = ["0xYourOperatorEOA"]
```

---

## 13. Reviewer Red Flags (Avoid These)

🚫 Constructor arguments
🚫 State writes in trap
🚫 Using `msg.sender`
🚫 Calling responder inside trap
🚫 Reverting in `collect()`
🚫 ABI mismatch
🚫 Wrong artifact path
🚫 Confusing byte length with sample size
🚫 Claiming “on-chain detection” while using off-chain logic

---

## 14. Deploy Order

1. Deploy Feed
2. Deploy Responder
3. Update trap constants
4. Deploy Trap
5. Update `drosera.toml`
6. Push to GitHub

---

## Final Mental Model

* Traps **observe**
* Responders **act**
* Operators **decide**
* Determinism beats cleverness
* Simple PoCs beat complex broken ones

---
