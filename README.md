# Drosera PoC Trap – Complete Beginner Deployment Guide

This guide walks you **step by step** through how to:

* Set up your environment (even if you’ve never done this before)
* Understand how Drosera traps actually work (no wrong mental models)
* Build a **Drosera-correct PoC trap**
* Avoid every common mistake reviewers flag
* Deploy, test, and wire your trap properly
* Push everything cleanly to GitHub

If you follow this guide **top to bottom**, your trap will run under Drosera operators.

---

## What a Drosera Trap Is (Important First Read)

A **Drosera trap is NOT something you “trigger” manually**.

You do NOT:

* call the trap with ethers.js
* click a button to activate it
* call a responder from inside the trap

Instead:

1. **Drosera operators** deploy your trap on a shadow fork.
2. Operators call `collect()` **on-chain** every block.
3. Operators pass the outputs into `shouldRespond()` **off-chain**.
4. If `shouldRespond()` returns `(true, payload)`,
   Drosera calls your **responder contract** (configured in TOML).

Your trap must fit this model exactly.

---

## Core Rules (Non-Negotiable)

Your trap **must**:

* Implement **Drosera’s ITrap interface**
* Be **stateless**
* Be **deterministic**
* Never rely on `msg.sender`
* Never rely on constructor arguments
* Never call the responder directly
* Never revert in `collect()`

If you break any of these, the trap will not run.

---

## Required Interface (Must Match Exactly)

Every Drosera trap must implement:

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

* Ubuntu 22.04+
  (native or WSL on Windows)

### Tools You Need

* Git
* VS Code
* Foundry
* A GitHub account
* Basic terminal access

---

## 2. Install System Tools

### Install Git

```bash
sudo apt update
sudo apt install git -y
```

Verify:

```bash
git --version
```

---

### Install VS Code

```bash
sudo snap install code --classic
```

Inside VS Code, install:

* **Solidity** extension (Juan Blanco)

---

## 3. Install Foundry

Foundry is the Solidity toolchain used by Drosera.

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

## 4. Create Your Trap Project

```bash
mkdir drosera-poc-trap
cd drosera-poc-trap
forge init
```

You should now have:

```
src/
script/
test/
foundry.toml
```

---

## 5. Install Drosera Contracts

You **must** use Drosera’s official interfaces.

```bash
git clone https://github.com/drosera-network/contracts.git lib/drosera-contracts
```

Verify:

```bash
ls lib/drosera-contracts/src/interfaces/ITrap.sol
```

---

## 6. Configure Foundry Remappings

Edit `foundry.toml`:

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]

[profile.default.remappings]
drosera-contracts = "lib/drosera-contracts/src/"
```

⚠️ Very common mistake
Do **not** use `/=` here. Use `=`.

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

Load it:

```bash
source .env
```

---

## 8. The Correct Trap Architecture

A **proper Drosera PoC trap** usually has three parts:

```
Off-chain detector (optional)
        ↓
Feed / Adapter contract (writes data)
        ↓
Trap (reads data in collect())
        ↓
Responder (called by Drosera)
```

### Important:

* The **trap never gets called directly**
* The **trap never stores state**
* The **trap only reads**
* The **responder reacts**

---

## 9. Example: Mirage Attack PoC (Correct Shape)

### Feed Contract (On-Chain Adapter)

This is what your Node.js detector (or you) writes to.

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

    function pushObservation(int256 deltaBps, uint256 timestamp)
        external
        onlyOwner
    {
        lastDeltaBps = deltaBps;
        lastTimestamp = timestamp;
        emit ObservationPushed(deltaBps, timestamp);
    }

    function latestObservation() external view returns (int256, uint256) {
        return (lastDeltaBps, lastTimestamp);
    }
}
```

---

## 10. Trap Contract (Drosera-Correct)

Key points this trap satisfies:

* No constructor args
* No state writes
* Defensive `collect()`
* Planner-safe `shouldRespond()`

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import {ITrap} from "drosera-contracts/interfaces/ITrap.sol";

interface IObservationFeed {
    function latestObservation() external view returns (int256, uint256);
}

contract MirageTrap is ITrap {
    // PoC: hardcode feed address
    address public constant FEED =
        0xYourDeployedMirageFeed;

    int256 public constant THRESHOLD_BPS = 500;
    uint256 private constant SAMPLE_SIZE = 64;

    function collect() external view override returns (bytes memory) {
        // Safety: FEED must exist
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
        if (data.length == 0 || data[0].length < SAMPLE_SIZE) {
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

The responder must match **exactly** what the trap returns.

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

This file connects:

* the trap artifact
* the responder
* operator rules

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

## 13. Common Red Flags (Reviewer Kill Switches)

🚫 Trap has constructor arguments
🚫 Uses `msg.sender`
🚫 Stores state
🚫 Calls responder inside trap
🚫 collect() can revert
🚫 ABI mismatch between trap, responder, and TOML
🚫 Wrong artifact path
🚫 Watching `address(this)` instead of a target
🚫 Monitoring EOAs but claiming “contract anomalies”
🚫 block_sample_size doesn’t match logic
🚫 cooldown = 0 without edge guards

Avoid these and you’re good.

---

## 14. Deploy Order

1. Deploy Feed
2. Deploy Responder
3. Update trap constants
4. Deploy Trap
5. Update `drosera.toml`
6. Push to GitHub

---

## 15. GitHub Push

```bash
git add .
git commit -m "Drosera PoC trap with full setup guide"
git push -u origin master
```

---

## Final Mental Model (Remember This)

* Traps **observe**, they don’t act
* Responders **act**, traps don’t
* Operators decide, not you
* Determinism > cleverness
* Simple PoCs beat complex broken ones

---
