# Do a private payment on a blockchain with Mobius

## Assumptions

1. We have a `RING_SIZE` of 1
2. Alice wants to send 1ETH to Bob through the Mobius contract

## Purpose

Represent the sequence of actions that Alice and Bob need to run to send a payment through Mobius.

## Commands

1. Generate a master key pair for the recipient (Bob)
  - Run: 
  ```bash
  orbital generate -n 1 > keys.json`
  ```
  - Output:
```javascript
{
  "pubkeys": [
    {
      "x": "0x27eb2f29a14e4306fe9dca39f66bed13c26a97da6ec6d9dc0018ad0f212e8c46",
      "y": "0xb6207bcc9494a11290e564ea4aba700478ed44e374d9aafb9889e493f5f019a"
    }
  ],
  "privkeys": [
    "0x93c31e0558a4c290f177793ebff9dd392c67c68692c0abc00795c62c5f35b71"
  ]
}
```
2. Bob sends his master public key to Alice. along with a secret and a nonce.   _Note:_
  - The Master Public Key (`mpk`) can be sent over an unsecure channel (since it is made to be public)
  - The Nonce (`n`) can also be sent in an open and unsecure channel
  - The Secret (`s`) should not be leaked to anyone. Thus, Alice and Bob could agree on a shared secret by using the Diffie-Hellman key exchange protocol or by unsing whatever secure channel they want.
  It is assumed in the next steps that Alice and Bob have shared the tuple (`mpk`, `n`, `s`).
3. Alice generates a Stealth Public Key (spk) for Bob, using (mpk, n, s)
  - In this example, we assume that the secret is `0x12987c779f1c76fdc01efaa3ce7fe8e730512ee4a12055018a44b8c9044fb860` and that the nonce is set to `0`
  - Run: 
  ```bash
  orbital stealth -o 0 \
  -s 0x12987c779f1c76fdc01efaa3ce7fe8e730512ee4a12055018a44b8c9044fb860 \
  -x 0x27eb2f29a14e4306fe9dca39f66bed13c26a97da6ec6d9dc0018ad0f212e8c46 \ 
  -y 0xb6207bcc9494a11290e564ea4aba700478ed44e374d9aafb9889e493f5f019a
  ```
  - Output:
  ```javascript
    {
        "myPublic": {
            "x": "0x27e073fe3485b7ab97de5813342c3ce3dd19eba467a6b5f61b092813e67325e2",
            "y": "0x639bbbc72ef12bc7449c08a4ac2d7d7a9ed0484e4d5d6ee4f4b6fa5844043c2"
        },
        "theirPublic": {
            "x": "0x27eb2f29a14e4306fe9dca39f66bed13c26a97da6ec6d9dc0018ad0f212e8c46",
            "y": "0xb6207bcc9494a11290e564ea4aba700478ed44e374d9aafb9889e493f5f019a"
        },
        "sharedSecret": "Hf6CAlK4hPwLz8AB0s+yzC8V+EBfyIYn3ysPybZcpcg=",
        "theirStealthAddresses": [
            {
                "public": {
                    "x": "0x23e921b90d4d81265e2646f53d98c22a3ae564072e88986cb27608ca0479863f",
                    "y": "0x21e79eb48b1a888b490f71acf8a954c70cdc4f263aec39e0b5780f1e2820ae72"
                },
                "nonce": 0
            }
        ],
        "myStealthAddresses": [
            {
                "public": {
                    "x": "0x13daec4f0055e8a679391cc8f6612d68d6b85ca8e3489815392607d8aee3a956",
                    "y": "0x1a6fad288c378e0021a1ad0fce2d43bf671194d59f182b231433a81a204082e8"
                },
                "nonce": 0,
                "private": 4999866454118458106557527619043146841915344351946198744737308277429857265076
            }
        ]
    }
  ```
  - Bob stealth public key is the field "theirStealthAddresses":
  ```
    "x": "0x23e921b90d4d81265e2646f53d98c22a3ae564072e88986cb27608ca0479863f"
    "y": "0x21e79eb48b1a888b490f71acf8a954c70cdc4f263aec39e0b5780f1e2820ae72"
  ```
4. In order for Alice to send her funds to Bob, run the following code in the Truffle console after deploying the contracts (truffle compile && truffle deploy --network development && truffle console --network development)
  ```javascript
    var globalResult;
    var mixerReady,
    var ringGuid;
    var ringMessage;

    var bobStealthPublicKeyX = "0x23e921b90d4d81265e2646f53d98c22a3ae564072e88986cb27608ca0479863f";
    var bobStealthPublicKeyY = "0x21e79eb48b1a888b490f71acf8a954c70cdc4f263aec39e0b5780f1e2820ae72";

    // Alice's account (Sender)
    var aliceAccount = "0xc6ffb921da5c5ba9dc3aa4e5f40d7987d2c82027";

    // ------ Call the Deposit function of the Mixer ------ //
    Mixer.deployed().then(function(instance){
        var mixer = instance;
        return mixer.Deposit(0, 1, bobStealthPublicKeyX, bobStealthPublicKeyY, {from: aliceAccount, value: 1})
      }).then(function(result){
        // Save the result in a global variable to access it later
        globalResult = result;

        // Iterate over the events and extracts the args of the MixerReady event
        for (var i = 0; i < result.logs.length; i++) {
          var logEntry = result.logs[i];
          if (log.event == "MixerReady") {
            mixerReady = true;
            ringGuid = log.args["ring_id"];
            ringMessage = log.args["message"];
          }
        }
        console.log(result);
      }).catch(function(error) {
         console.log(error.message);
      })
  ```
Here is an inline version of the code above to Copy/Paste in the Truffle console:
```javascript
var globalResult,mixerReady,ringGuid,ringMessage,bobStealthPublicKeyX="0x23e921b90d4d81265e2646f53d98c22a3ae564072e88986cb27608ca0479863f",bobStealthPublicKeyY="0x21e79eb48b1a888b490f71acf8a954c70cdc4f263aec39e0b5780f1e2820ae72",aliceAccount="0xc6ffb921da5c5ba9dc3aa4e5f40d7987d2c82027";Mixer.deployed().then(function(e){return e.Deposit(0,1,bobStealthPublicKeyX,bobStealthPublicKeyY,{from:aliceAccount,value:1})}).then(function(e){globalResult=e;for(var a=0;a<e.logs.length;a++){e.logs[a];"MixerReady"==log.event&&(mixerReady=!0,ringGuid=log.args.ring_id,ringMessage=log.args.message)}console.log(e)}).catch(function(e){console.log(e.message)});
```
Once the deposit is done, the events `MixerDeposit` and `MixerReady` are triggered. Once Bob handles the event `MixerReady`, he can extract the ring GUID and the message emitted when the ring became full in ordre to claim his money/withdraw his funds.
5. Generate the ring signature by running: 
```bash
orbital input -n 1 -m [messageReturnedByTheRing]
```
In our example, we should run:
```bash
  orbital inputs -n 1 \
  -m 0ab63bea9fdbcc7ec4a4980a6d71df646855126d894c5f24e33474fc36beea7c \
  -k keys.json 
``` 
assuming that the message returned by the ring is; `0ab63bea9fdbcc7ec4a4980a6d71df646855126d894c5f24e33474fc36beea7c`.
The output should resemble:
  ```javascript
{
    "alice2bob": null,
    "bob2alice": null,
    "message": "CrY76p/bzH7EpJgKbXHfZGhVEm2JTF8k4zR0/Da+6nw=",
    "ring": [
        {
        "x": "0x23e921b90d4d81265e2646f53d98c22a3ae564072e88986cb27608ca0479863f",
        "y": "0x21e79eb48b1a888b490f71acf8a954c70cdc4f263aec39e0b5780f1e2820ae72"
        }
    ],
    "signatures": [
        {
            "tau": {
                "x": "0xb476e010627828070e215bd8a21c7e0990414c5d2c2335416ebda6a9e4bc2b6",
                "y": "0x174888b0a2eb3364d01b871d4075113e9f729d932475ca3feaa2f3e0244d190c"
            },
            "ctlist": [
                "0x17a3479fa95fc6cafbde8ac99d27d367f850f50e5def2e21fdc23f65a6090b38",
                "0x23907211830081c7d372bdba2b49b9506f7dc1d066c58f5f8ab45c78fd741e4"
            ]
        }
    ]
}
```
6. Now Bob can claim his funds on the ring (ringGuid), and by providing the signature of the `ringMessage` variable, by running:
```javascript
var tagX = "0xb476e010627828070e215bd8a21c7e0990414c5d2c2335416ebda6a9e4bc2b6";
var tagY = "0x174888b0a2eb3364d01b871d4075113e9f729d932475ca3feaa2f3e0244d190c";

var ctlist = [
  "0x17a3479fa95fc6cafbde8ac99d27d367f850f50e5def2e21fdc23f65a6090b38",
  "0x23907211830081c7d372bdba2b49b9506f7dc1d066c58f5f8ab45c78fd741e4"
];

// ------ Call the Withdraw function of the Mixer ------ //
var withdrawalResult;
Mixer.deployed().then(function(instance){
    var mixer = instance;
    return mixer.Withdraw(ringID, tagX, tagY, ctlist)
  }).then(function(result){
    // Save the result in a global variable to access it later
    withdrawalResult = result;
    if (result) {
      console.log("Successful Withdrawal");
    } else {
      console.log("Error while Withdrawing");
    }
  }).catch(function(error) {
    console.log(error.message);
  })
```
Here is the inline javascript to Copy and Paste in the Truffle console:
```javascript
var withdrawalResult,tagX="0xb476e010627828070e215bd8a21c7e0990414c5d2c2335416ebda6a9e4bc2b6",tagY="0x174888b0a2eb3364d01b871d4075113e9f729d932475ca3feaa2f3e0244d190c",ctlist=["0x17a3479fa95fc6cafbde8ac99d27d367f850f50e5def2e21fdc23f65a6090b38","0x23907211830081c7d372bdba2b49b9506f7dc1d066c58f5f8ab45c78fd741e4"];Mixer.deployed().then(function(a){return a.Withdraw(ringID,tagX,tagY,ctlist)}).then(function(a){withdrawalResult=a,a?console.log("Successful Withdrawal"):console.log("Error while Withdrawing")}).catch(function(a){console.log(a.message)});
```
7. Running this javascript code extract should result in a "Successful Withdrawal" message printed in the console, meaning that Alice's 1ETH has correctly been withdrew by Bob.

## Verification

In order to verify that Bob's account has been credited of Alice's ETH, one can run:


## Privacy verification

// TODO: Do a small blockchain crawler to try to analyze the blockchain and try to infer who received the payment from the contract (see JS function witten in GETH).

