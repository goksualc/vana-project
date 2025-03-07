# vana-project

# Building a Complete DApp on Vana Testnet: Step-by-Step Guide

## 1. Set Up Your Development Environment
Before starting, ensure you have **Node.js (v20 or later)** and **npm** installed. Verify your versions:

```sh
node -v
npm -v
```

If outdated, update using:

```sh
nvm install 20
nvm use 20
```

Or install the latest LTS version from [Node.js official site](https://nodejs.org/en).

## 2. Install Hardhat and Required Dependencies
Create a new project directory and initialize it:

```sh
mkdir vana-dapp && cd vana-dapp
npm init -y
```

Install Hardhat and dependencies:

```sh
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox ethers dotenv express cors
```

Then set up Hardhat:

```sh
npx hardhat
```

Choose **"Create a JavaScript project"** and install dependencies.

## 3. Configure Hardhat for Vana Testnet
Edit `hardhat.config.js` to include Vana's testnet:

```js
require("dotenv").config();
require("@nomicfoundation/hardhat-toolbox");

module.exports = {
  solidity: "0.8.28",
  networks: {
    vana: {
      url: "https://rpc.moksha.vana.org",
      chainId: 14800,
      accounts: [process.env.PRIVATE_KEY],
    },
  },
};
```

## 4. Set Up Environment Variables
Create a `.env` file in the root directory:

```sh
touch .env
```

Add your private key (without `0x` if not needed):

```
PRIVATE_KEY=your_private_key_here
```

Ensure `.gitignore` includes `.env` to keep it secure:

```
.env
```

## 5. Write the Smart Contract (`Lock.sol`)
Create a new Solidity file under `contracts/Lock.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.28;

contract Lock {
    uint public unlockTime;
    address payable public owner;

    constructor(uint _unlockTime) payable {
        require(_unlockTime > block.timestamp, "Unlock time should be in the future");
        unlockTime = _unlockTime;
        owner = payable(msg.sender);
    }
    
    function withdraw() public {
        require(block.timestamp >= unlockTime, "Unlock time has not arrived");
        require(msg.sender == owner, "Only the owner can withdraw");
        payable(owner).transfer(address(this).balance);
    }
}
```

Compile the contract:

```sh
npx hardhat compile
```

## 6. Deploy the Smart Contract
Create a new script under `scripts/deploy.js`:

```js
const { ethers } = require("hardhat");

async function main() {
  const Lock = await ethers.getContractFactory("Lock");
  const unlockTime = Math.floor(Date.now() / 1000) + 3600;
  const lock = await Lock.deploy(unlockTime, { value: ethers.parseEther("1") });
  await lock.waitForDeployment();
  console.log("Contract deployed at:", await lock.getAddress());
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run the deployment script:

```sh
npx hardhat run scripts/deploy.js --network vana
```

## 7. Build the Frontend (React UI)
Initialize a React project:

```sh
npx create-react-app vana-ui
cd vana-ui
```

Install Web3 dependencies:

```sh
npm install ethers
```

Modify `src/App.js`:

```jsx
import { useState } from "react";
import { ethers } from "ethers";
import './App.css';

function App() {
  const [account, setAccount] = useState(null);
  const [contract, setContract] = useState(null);
  
  async function connectWallet() {
    if (window.ethereum) {
      const provider = new ethers.BrowserProvider(window.ethereum);
      const signer = await provider.getSigner();
      setAccount(await signer.getAddress());
      
      const contractAddress = "YOUR_CONTRACT_ADDRESS_HERE";
      const abi = [
        "function unlockTime() view returns (uint256)",
        "function withdraw() public"
      ];
      const lockContract = new ethers.Contract(contractAddress, abi, signer);
      setContract(lockContract);
    } else {
      alert("MetaMask is required");
    }
  }

  async function withdrawFunds() {
    if (contract) {
      try {
        const tx = await contract.withdraw();
        await tx.wait();
        alert("Funds withdrawn successfully");
      } catch (error) {
        console.error(error);
        alert("Withdrawal failed");
      }
    }
  }

  return (
    <div className="App">
      <h1>Vana DApp</h1>
      {account ? (
        <>
          <p>Connected: {account}</p>
          <button onClick={withdrawFunds}>Withdraw Funds</button>
        </>
      ) : (
        <button onClick={connectWallet}>Connect Wallet</button>
      )}
    </div>
  );
}

export default App;
```

Run the frontend:

```sh
npm start
```

## 8. Running Tests
Create a test file `test/Lock.js`:

```js
const { expect } = require("chai");
const { ethers } = require("hardhat");

describe("Lock", function () {
  async function deployLockFixture() {
    const unlockTime = Math.floor(Date.now() / 1000) + 3600;
    const [owner, otherAccount] = await ethers.getSigners();
    const Lock = await ethers.getContractFactory("Lock");
    const lock = await Lock.deploy(unlockTime, { value: ethers.parseEther("1") });
    return { lock, unlockTime, owner, otherAccount };
  }

  it("Should set the right unlockTime", async function () {
    const { lock, unlockTime } = await deployLockFixture();
    expect(await lock.unlockTime()).to.equal(unlockTime);
  });
});
```

Run the tests:

```sh
npx hardhat test
```

## 9. Debugging and Fixing Issues
- **`ethers is not defined`** → Ensure `ethers` is imported.
- **`Invalid JSON-RPC response`** → Check RPC URL.
- **`npm dependency errors`** → Run:

```sh
npm install --legacy-peer-deps
```

By following this guide, you can deploy a complete DApp on the **Vana Testnet** with a fully functional frontend and smart contract integration! 🚀
