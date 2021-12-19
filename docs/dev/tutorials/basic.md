# Basic

In this tutorial we will recap and expand on the `Greeter` example in the [Getting started](../README.md#hello-world-on-zksync) guide.

## Initializing the proejct & deploying smart contract

First, we should initialize the project and install the dependencies. Run the following commands in terminal:

```
mkdir greeter-example
cd greeter-example
yarn init -y
yarn add -D typescript ts-node ethers zksync-web3 hardhat @matterlabs/hardhat-zksync-solc @matterlabs/hardhat-zksync-deploy
```

Please note, that currently typescript is reqiured by zkSync plugins.

Create the `hardhat.config.ts` file and paste the following code there:

```typescript
require("@matterlabs/hardhat-zksync-deploy");
require("@matterlabs/hardhat-zksync-solc");

module.exports = {
  zksolc: {
    version: "0.1.0",
    compilerSource: "docker",
    settings: {
      optimizer: {
        enabled: true,
      },
      experimental: {
        dockerImage: "zksyncrobot/test-build"
      }
    },
  },
  zkSyncDeploy: {
    zkSyncNetwork: "https://z2-dev-api.zksync.dev",
    ethNetwork: "rinkeby",
  },
  solidity: {
    version: "0.8.10"
  }
};
```

Create the `contracts` and `deploy` folders. The former is the place where all the contracts' `*.sol` files should be stored and the latter is the place where all the scripts related to deployment of the contract will be put. 

Create the `contracts/Greeter.sol` contract and insert the following code there:

```solidity
//SPDX-License-Identifier: Unlicense
pragma solidity ^0.8.0;

contract Greeter {
    string private greeting;

    constructor(string memory _greeting) {
        greeting = _greeting;
    }

    function greet() public view returns (string memory) {
        return greeting;
    }

    function setGreeting(string memory _greeting) public {
        greeting = _greeting;
    }
}
```

We can now compile the contracts with the following command:

```
yarn hardhat compile
```

Now, let's create the deployment script in the `deploy/deploy.ts`:

```typescript
import { utils } from 'zksync-web3';
import * as ethers from 'ethers';
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

// An example of a deploy script which will deploy and call a simple contract.
export default async function (hre: HardhatRuntimeEnvironment) {
    console.log(`Running deploy script for the Greeter contract`);

    // Initialize the wallet.
    const wallet = new ethers.Wallet("<WALLET-PRIVATE-KEY>");

    // Create deployer object and load the artifact of the contract we want to deploy.
    const deployer = new Deployer(hre, wallet);
    const artifact = await deployer.loadArtifact("Greeter");

    // Deposit some funds to L2 in order to be able to perform deposits.
    const depositAmount = ethers.utils.parseEther("0.001");
    const depositHandle = await deployer.zkWallet.deposit({
        to: deployer.zkWallet.address,
        token: utils.ETH_ADDRESS,
        amount: depositAmount,
    });
    // Wait until the deposit is processed on zkSync
    await depositHandle.wait();

    // Deploy this contract. The returned object will be of a `Contract` type, similarly to ones in `ethers`.
    // `greeting` is an argument for contract constructor.
    const greeting = "Hi there!";
    const greeterContract = await deployer.deploy(artifact, [greeting]);

    // Show the contract info.
    const contractAddress = greeterContract.address;
    console.log(`${artifact.contractName} was deployed to ${contractAddress}!`);
}
```

Now we can run the script using the following command:

```
yarn hardhat deploy-zksync
```

### Paying fees in ERC-20 tokens

Let's see how we can pay fees in `USDC` token.

```typescript
const USDC_ADDRESS = '0xeb8f08a975ab53e34d8a0330e0d34de942c95926';
const USDC_DECIMALS = 6;

const deploymentFee = await deployer.estimateDeployFee(artifact, [greeting], USDC_ADDRESS);
// Deposit funds to L2.
const depositHandle = await deployer.zkWallet.deposit({
  to: deployer.zkWallet.address,
  token: USDC_ADDRESS,
  // We deposit more than the minimal required amount to have funds 
  // for further iteraction with our smart contract.
  amount: deploymentFee.mul(2),
  // Unlike ETH, ERC-20 tokens require approval in order to deposit to zkSync.
  // You can either set the approval in a separate transaction or provide `approveERC20` flag equal to `true`
  // and the SDK will initiate approval transaction under the hood.
  approveERC20: true
});

// Wait until the deposit is processed by zkSync.
await depositHandle.wait();
```

We can output the fee in human-readable format:

```typescript
const parsedFee = ethers.utils.formatUnits(deploymentFee.toString(), USDC_DECIMALS);
console.log(`The deployment will cost ${parsedFee} USDC`);
```

Please note that the fees on testnet do not correctly represent the fees on the future mainnet release.

Now, we need to pass `USDC` as the `feeToken` to the deployment transaction:

```typescript
const greeterContract = await deployer.deploy(artifact, [greeting], USDC_ADDRESS);
```

In order to pay fees in USDC for smart contract interaction, supply the fee token in the `customData` override:

```typescript
const setNewGreetingHandle = await greeterContract.setGreeting(newGreeting, {
    customData: {
        feeToken: USDC_ADDRESS
    }
});
```

Full example:

```typescript
import { utils } from 'zksync-web3';
import * as ethers from 'ethers';
import { HardhatRuntimeEnvironment } from "hardhat/types";
import { Deployer } from "@matterlabs/hardhat-zksync-deploy";

const USDC_ADDRESS = '0xeb8f08a975ab53e34d8a0330e0d34de942c95926';
const USDC_DECIMALS = 6;

// An example of a deploy script which will deploy and call a simple contract.
export default async function (hre: HardhatRuntimeEnvironment) {
    console.log(`Running deploy script for the Greeter contract`);

    // Initialize the wallet.
    const wallet = new ethers.Wallet("<WALLET-PRIVATE-KEY>");

    // Create deployer object and load the artifact of the contract we want to deploy.
    const deployer = new Deployer(hre, wallet);
    const artifact = await deployer.loadArtifact("Greeter");

    const greeting = "Hi there!";
    const deploymentFee = await deployer.estimateDeployFee(artifact, [greeting], USDC_ADDRESS);

    // Deposit funds to L2
    const depositHandle = await deployer.zkWallet.deposit({
        to: deployer.zkWallet.address,
        token: USDC_ADDRESS,
        amount: deploymentFee.mul(2),
        approveERC20: true
    });
    // Wait until the deposit is processed on zkSync
    await depositHandle.wait();

    // Deploy this contract. The returned object will be of a `Contract` type, similarly to ones in `ethers`.
    // `greeting` is an argument for contract constructor.
    const parsedFee = ethers.utils.formatUnits(deploymentFee.toString(), USDC_DECIMALS);
    console.log(`The deployment will cost ${parsedFee} USDC`);

    const greeterContract = await deployer.deploy(artifact, [greeting], USDC_ADDRESS);

    // Show the contract info.
    const contractAddress = greeterContract.address;
    console.log(`${artifact.contractName} was deployed to ${contractAddress}!`);
}
```

## Front-end integration

Let's create another folder `greeter-front-end`, where we will build the front-end for our dApp. In this tutorial, we will use `Vue` as the framework of our choice, but the process will be quite similar regardless of the framework you use. 

### Setting up the project

To focus on the specifics of using the `zksync-web3` SDK, we created a template, with all the HTML & CSS ready to use.

Clone it:

```
git clone TODO
```

Spin up the project:

```
cd TODO
yarn
yarn serve
```

By default, the page should be running at `http://localhost:8080`. You can open that URL in your browser to see the first page.

We will implement the following functionality:

- The user should be able to get the greeting after the page is loaded.
- The user should be able to select the token he wants to pay the fee with.
- The user should be able to change the greeting on the smart contract. 

### Project structure

All of our code will be written in the `./src/App.vue`. All the front-end code has been written. The only thing left is to fill out the TODO-s:

```javascript
initializeProviderAndSigner() {
    // TODO: initialize provider and signer based on `window.ethereum`
},
async getTokens() {
    // TODO: return the list of tokens in zkSync
    return [];
},
async getGreeting() {
    // TODO: return the current greeting
    return "";
},
async getFee() {
    // TOOD: return formatted fee
    return "";
},
async getBalance() {
    // Return formatted balance
    return ""; 
},
async changeGreeting() {
    this.txStatus = 1;
    try {
        // TODO: Submit the transaction
        this.txStatus = 2;
        
        // TODO: Wait for transaction compilation
        await txHandle.wait();

        this.txStatus = 3;

        // Update greeting
        this.greeting = await this.getGreeting();

        this.retreivingFee = true;
        this.retreivingBalance = true;
        // Update balance and fee
        this.currentBalance = await this.getBalance();
        this.currentFee = await this.getFee();
    } catch (e) {
        alert(JSON.stringify(e));
    }

    this.txStatus = 0;
    this.retreivingFee = false;
    this.retreivingBalance = false;
},
```

### Installing `zksync-web3`

Run the following commands to install `zksync-web3` and `ethers`:

```
yarn add ethers zksync-web3
```

### Getting the ABI and contract address

Open `./src/App.vue` and set the `GREETER_CONTRACT_ADDRESS` constant equal to the address where your greeter contract was deployed. 

To interact with our smart contract, we also need its ABI. 

- Create the `./src/abi.json` file. 
- You can get the contract's abi in the hardhat project folder in the `./artifacts/Greeter.sol/Greeter.json` file. You should copy the `abi` array and paste it into the `abi.json` created in the previous step.
<!-- TODO: hopefully there will be a better way to get the Abi-->

Set the `GREETER_CONTRACT_ABI` to require the api file.

```js
// eslint-disable-next-line
const GREETER_CONTRACT_ADDRESS = '0x...';
// eslint-disable-next-line
const GREETER_CONTRACT_ABI = require("./abi.json");
```


### Working with provider

Go to the `initializeProviderAndSigner` method in `./src/App.vue`. This method is called after the connection to Metamask is successful.

In this method we should:
- Initialize `Web3Provider` and `Signer` objects for interacting with zkSync
- Initialize `Contract` object to interact with the `Greeter` contract.

Unfortunately, `Metamask` does not allow to call methods from `zks_` namespace. That's why we can not use provider obtained from `window.ethereum` for that purpose. We should either have a separate provider based on zkSync node or hardcode the tokens in the website. In this tutorial, we will create a provider, but using the injected `Web3` and hardcoded tokens may be more suitable in some production cases.

Let's add the necessary dependencies:

```javascript
import { Contract, Web3Provider, Provider } from 'zksync-web3';
```

The first two steps can be done the following way:

```javascript
initializeProviderAndSigner() {
    this.provider = new Provider('https://z2-dev-api.zksync.dev');
    // Note that we still need to get the Metamask signer
    this.signer = (new Web3Provider(window.ethereum)).getSigner();

    this.contract = new Contract(
        GREETER_CONTRACT_ADDRESS,
        GREETER_CONTRACT_ABI,
        this.signer
    );
},
```

### Retreiving tokens and greeting

Now we should fill the methods for retrieving the tokens and retrieving the greeting from smart contract:

```javascript
async getTokens() {
    this.tokens = await this.provider.getConfirmedTokens.getTokens();
}
async getGreeting() {
    // Note that smart contract calls work the same way as in `ethers`
    this.greeting = await this.contract.greet();
}
```


The full methods now look the following way:

```javascript
initializeProviderAndSigner() {
    this.provider = new Provider('https://z2-dev-api.zksync.dev');
    // Note that we still need to get the Metamask signer
    this.signer = (new Web3Provider(window.ethereum)).getSigner();

    this.contract = new Contract(
        GREETER_CONTRACT_ADDRESS,
        GREETER_CONTRACT_ABI,
        this.signer
    );
},
async getTokens() {
    return await this.provider.getConfirmedTokens();
},
async getGreeting() {
    return await this.contract.greet();
},
```

Now, when you connect to your metamask you should see the following page:

![img](/start-1.png)

You should now also be able to select token to pay the fee. But no balances are updated, *yet*.

### Retrieving token balance and transaction fee

The easiest way to retrieve the user's balance is to use the `Signer.getBalance` method:

```javascript
async getBalance() {
    // Getting the balance for the signer in the selected token
    const balanceInUnits = await this.signer.getBalance(this.selectedToken.address);
    // To display the number of tokens in the human-readable format, we need to format them,
    // e.g. if balanceInUnits returns 500000000000000000 wei of ETH, we want to display 0.5 ETH the user
    return ethers.utils.formatUnits(balanceInUnits, this.selectedToken.decimals);
},
```

To estimate the fee:

```javascript
async getFee() {
    // Getting the amount of gas (ergs) needed for one transaction
    const feeInGas = await this.contract.estimateGas.setGreeting(this.newGreeting, {
        customData: {
            // We need to pass the fee token to estimate gas correctly
            feeToken: this.selectedToken.address
        }
    });
    // Getting the price of 
    const gasPriceInUnits = await this.provider.getGasPrice();

    // To display the number of tokens in the human-readable format, we need to format them,
    // e.g. if feeInGas*gasPriceInUnits returns 500000000000000000 wei of ETH, we want to display 0.5 ETH the user
    return ethers.utils.formatUnits(feeInGas.mul(gasPriceInUnits), this.selectedToken.decimals);
},
```

Now, when you open the page and select the token to pay fee with, you can see your balance and the expected fee for the transaction. 

The `Refresh` button should be used to recalculate the fee as the fee may depend on the length of the string to be set.

You can even click on the `Change greeting` button, but nothing will be changed, since we don't call the contract yet.

![img](/start-2.png)

### Updating the greeting

Interacting wtth smart contract works absolutely the same way as in `ethers`, but we also need to pass the `customData` override to supply the `feeToken` of the transaction.

```javascript
const txHandle = await this.contract.setGreeting(this.newGreeting, {
    customData: {
        // Passing the token to pay fee with
        feeToken: this.selectedToken.address
    }
});
```

Now we need to wait until the tx is committed:

```javascript
await txHandle.wait();
```

The full method looks the following way:

```javascript
async changeGreeting() {
    this.txStatus = 1;
    try {
        const txHandle = await this.contract.setGreeting(this.newGreeting, {
            customData: {
            // Passing the token to pay fee with
            feeToken: this.selectedToken.address
            }
        });
        this.txStatus = 2;
        // Wait until the tx is committed
        await txHandle.wait();
        
        this.txStatus = 3;

        // Update greeting
        this.greeting = await this.getGreeting();

        this.retreivingFee = true;
        this.retreivingBalance = true;
        // Update balance and fee
        this.currentBalance = await this.getBalance();
        this.currentFee = await this.getFee();
    } catch (e) {
        alert(JSON.stringify(e));
    }

    this.txStatus = 0;
    this.retreivingFee = false;
    this.retreivingBalance = false;
},
```

### Complete app.

You should now be able to update the greeting. 

Type the new greeting in the input box and click on the `Change greeting` button:

![img](/start-3.png)

Since the `feeToken` was supplied, the transaction to be sent is of the `eip712` type:

![img](/start-4.png)

Click "Sign".

After the transaction is processed, the page updates your balances and you can see the new greeting:

![img](/start-5.png)

### Learn more

- To learn more about `zksync-web` SDK, check out its [docuemntation](../../api/js).