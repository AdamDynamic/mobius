# Sending money through the Mobius contract

1. Generate key pair with Orbital
Run `orbital generate -n 1`:
```javascript
{
    "pubkeys": [
        {
            "x": "0x26569781c3ab69ff42834ea67be539bb231fa48730afc3c89f2bba140b2045b2",
            "y": "0xbf75913861d38b5a01b53654daa260856d5dd705af6a24e57622811d485e407"
        }
    ],
    "privkeys": [
        "0x216e142880261d4b743386185c41ae9cf3609f648dbedc15bd0790332b23fb87"
    ]
}
```
2. Generate an address from the private key, using geth:
```bash
echo "216e142880261d4b743386185c41ae9cf3609f648dbedc15bd0790332b23fb87" > privatekey
geth account import privatekey
```
Private key: "216e142880261d4b743386185c41ae9cf3609f648dbedc15bd0790332b23fb87",
Corresponding Address: "fdd1f2b17e5a8eda4b2743da81de881eebbefdfd"
Corresponding passphrase: "test"
3. Get the ABI of the Mixer contract:
```bash
echo "var MixerAbi=`solc --optimize --combined-json abi contracts/Mixer.sol`" > mixerAbi.js
```
4. Copy the Abi into the docker container: `docker cp mixerAbi.js 571ea7fc269a:/home/eth_user`
5. Deploy the contracts with truffle (by specifying the running geth network: `truffle deploy --network development`)
6. get the contract address by looking at the result of `Mixer.deployed()` in the truffle console.
7. In the docker container's runnig  geth instance, run:
```javascript
loadScript('mixerAbi.js')
var mixerContractAbi = MixerAbi.contracts['contracts/Mixer.sol:Mixer'].abi;
var mixerContract = eth.contract(JSON.parse(mixerContractAbi));
var mixerContractAddress = "0x25b9609f6dd1649c426133f0fc98bbfb33e7d090"; // Observed in the Mixer.deployed() output
var MyMixerInstance = mixerContract.at(mixerContractAddress);
```
8. In geth we should have at least 2 accounts:
    - The first one created directly by launching ./entrypoint in the docker container (we call this account A)
    - The other, by running `geth account import privatekey` in step 1. (we call this account B) or just `web3.personal.importRawKey("<PrivateKey>","<Password>")` directly in the geth console
The first account (created by ./entrypoint) should have a lot of ethers (Because it takes all the ethers won by mining). The other one should not have any ether.
9. Retrieve the message sent by the ring when the event MixerReady is triggered. In order to do this, we need to define a callback (see: https://coursetro.com/posts/code/100/Solidity-Events-Tutorial---Using-Web3.js-to-Listen-for-Smart-Contract-Events)
```javascript
var mixerDepositEvent = MyMixerInstance.MixerDeposit();
mixerDepositEvent.watch(function(error, result){
    // Content of the function
});

var mixerReadyEvent = MyMixerInstance.MixerReady();
mixerReadyEvent.watch(function(error, result){
    if (!error) { 
        console.log("Msg: " + result.args.message); 
        console.log(" ID: " + result.args.ring_id); 
    } else {
        console.log(error);
    }
});
```
10. We'll try to transfer some money form A to B: Run:
```javascript
MyMixerInstance.Deposit(0, 1, "0x26569781c3ab69ff42834ea67be539bb231fa48730afc3c89f2bba140b2045b2", "0xbf75913861d38b5a01b53654daa260856d5dd705af6a24e57622811d485e407", {from: "0x7f92d29ac0189660a9e214b021a98acb53c109d7", value: 1, gas: 373259})
```
with 0x57cf324ce11a698adac3ec552fc3ce5e1c8cc41a being the account A, and `0x26569781c3ab69ff42834ea67be539bb231fa48730afc3c89f2bba140b2045b2` and `0xbf75913861d38b5a01b53654daa260856d5dd705af6a24e57622811d485e407` being the (x,y) coordinates of the public key associated with the address B.
11. Generate the ring signature to be able to retrieve the founds by calling `Withdraw` on the contract



SCRIPT:
```javascript
loadScript('mixerAbi.js')
var mixerContractAbi = MixerAbi.contracts['contracts/Mixer.sol:Mixer'].abi;
var mixerContract = eth.contract(JSON.parse(mixerContractAbi));
var mixerContractAddress = "0x23008ebc7221de718178abf454b8d5fc0a4dbeb3";
var MyMixerInstance = mixerContract.at(mixerContractAddress);

var mixerDepositEvent = MyMixerInstance.MixerDeposit();
mixerDepositEvent.watch(function(error, result){
    console.log("Deposit");
});

var mixerReadyEvent = MyMixerInstance.MixerReady();
mixerReadyEvent.watch(function(error, result){
    if (!error) { 
        console.log("Msg: " + result.args.message); 
        console.log(" ID: " + result.args.ring_id); 
    } else {
        console.log(error);
    }
});

MyMixerInstance.Deposit(0, 1, "0x26569781c3ab69ff42834ea67be539bb231fa48730afc3c89f2bba140b2045b2", "0xbf75913861d38b5a01b53654daa260856d5dd705af6a24e57622811d485e407", {from: eth.accounts[0], value: 1, gas: 3273259})
```


Available Accounts
==================
(0) 0xfdd1f2b17e5a8eda4b2743da81de881eebbefdfd
(1) 0x43c719ee71a6212abd2d6d796e589089fdc1e88d

Private Keys
==================
(0) 216e142880261d4b743386185c41ae9cf3609f648dbedc15bd0790332b23fb87
(1) 072f92b9d3fcac40b3e90e84be6b961a3cc472c4ce0ad74da3c6e8af2abb0ca9

Gas Limit
==================
268435455

Msg: 0x2c1ff7110a6507e06904d005afe57c801198de03c021f56d48a93790727f923c
ID: 0x669839e5c7d9e0d94cae7c3a9a6e914cd64b77b7153a9eec89f8e9920367e302

```bash
17:58:18 ~/D/C/g/mobius ❯❯❯ orbital inputs -n 1 -m 2c1ff7110a6507e06904d005afe57c801198de03c021f56d48a93790727f923c -k keys.json
{
    "alice2bob": null,
    "bob2alice": null,
    "message": "LB/3EQplB+BpBNAFr+V8gBGY3gPAIfVtSKk3kHJ/kjw=",
    "ring": [
    {
        "x": "0x26569781c3ab69ff42834ea67be539bb231fa48730afc3c89f2bba140b2045b2",
        "y": "0xbf75913861d38b5a01b53654daa260856d5dd705af6a24e57622811d485e407"
    }
    ],
    "signatures": [
    {
        "tau": {
            "x": "0x8137ac6c15b8e5fec4b3b03de851e5d75f0edbf1f57aae8a92a7b9a3a6ed52b",
            "y": "0x6f21acee7be72a613e98fb271849685ba09cf1640347fad44ef72bbda144c26"
        },
        "ctlist": [
            "0x1dda86403fb27d997e2ccdcbee8ecbf32976db352ccfeaa6ce64c1d6e21d2573",
        "0x212e6d70ca3b30231248bc18e22e1e2b6ff3f8efd96e24c5ae16047adf302cab"
        ]
    }
    ]
}

```


```
var tagX = "0x8137ac6c15b8e5fec4b3b03de851e5d75f0edbf1f57aae8a92a7b9a3a6ed52b";
var tagY = "0x6f21acee7be72a613e98fb271849685ba09cf1640347fad44ef72bbda144c26";

var ctlist = [
    "0x1dda86403fb27d997e2ccdcbee8ecbf32976db352ccfeaa6ce64c1d6e21d2573",
    "0x212e6d70ca3b30231248bc18e22e1e2b6ff3f8efd96e24c5ae16047adf302cab"
];

var ringGuid = "0x669839e5c7d9e0d94cae7c3a9a6e914cd64b77b7153a9eec89f8e9920367e302";

MyMixerInstance.Withdraw(ringGuid, tagX, tagY, ctlist, {from: eth.accounts[0]});
```
