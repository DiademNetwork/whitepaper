# Diadem Network
## Overview
Bitcoin created an opportunity to transfer money and make deals between people in different places without any intermediaries. Furthermore, blockchain has opened an opportunity to build various applications that can describe distribution of money according to deterministic rules defined as smart contracts. In many cases, those rules are needed to be related to actions of people in external world. Diadem Network has built a bridge between external world and blockchain that allows to prove actions of people, make escrow deposits in different cryptocurrencies, and create communities that can distribute money according to achievements of members.

## Implementation
Prove actions of people. Any person can become a witness to any action of other person and confirm that is has been accomplished. Others can validate signature of witness to check his confirmation.

Make escrow deposits. Escrow deposit can be funded in different cryptocurrencies. Smart contract holds money until witness will confirm accomplishment of associated action and then will release deposit to beneficiary.

Create communities. Different applications built on the protocol enable communities that can validate achievements of members and create escrow deposits that define rules of money distribution according to achievements of members.

### Blockchain
We implemented a solution that allows to create escrow deposits in different blockchains for different applications.

It is designed to provide easy access to all participants and reduce fees.

#### Signatures storage
Digital signatures enabled creation of cryptocurrencies. When person has signed hash of transaction with his private key, his signature can be used to prove that person has decided to spend money.

In similar way, we propose to represent actions of people in the form of unique messages that can be signed by witness. In order to confirm accomplishment of action of another person, witness can sign hash from message that represent the action, and publish his signature. Then his signature can be used to prove that witness has confirmed accomplishment of that action.

Our solution considers following requirements: witness requires an opportunity to publish his signature for free, once per message, before anyone is waiting for it; smart contracts in different blockchains require an opportunity to access and verify signature; smart contracts waiting for signature of specific witness are required to react immediately when witness publishes his signature in free storage; smart contracts should react on signatures previously published.

We implemented decentralized signatures server that satisfies these requirements and mitigate security issues. Witness can publish his signature with decentralized signatures server for free, instead of direct transaction to blockchain that require fees. Server will verify signature, save it in cheap storage, and notify listeners about published signature. Then relayer can build and broadcast transactions to smart contracts waiting for signature in different blockchains.

#### Relayer
Relayer is a service listening to decentralized signatures server that connects it with other blockchains.

When witness publishes his signature, relayer will be notified. Then relayer will build and broadcast transactions to smart contracts waiting for signature in different blockchains. Also relayer listen to requests of smart contracts for previously published signatures.

Relayer can be incentivized to broadcast transactions with fees that can be defined in smart contracts.

#### Witness
Smart contracts for escrow deposits require witness to confirm accomplishment of achievement before deposit will be released.

Creator of deposit can choose any valid address in specific blockchain as a witness. It can be a personal account, or another smart contract. Personal account may have external reputation, for example, associated profiles in social networks or reviews. Smart contract may represent, for example, aragon organization, augur consensus, or even device from Internet of Things.

Witness can be incentivized to participate with fees that can be defined in smart contracts.

#### Protocol
Protocol describes our implementation of decentralized signatures server as smart contracts deployed to Plasma Chain based on Loom Network.

We decided to build on Plasma Chain because it allows other applications deployed to Plasma Chain to interact with protocol directly, and implement custom logic for different usecases. 

##### Messages
Messages.sol contract allows to create messages that represent actions of people.

Messages.sol is deployed to Plasma Chain to allow other applications to create messages on behalf of users.

Message is a structure saved in storage with three mandatory fields(creator, owner, item) and unique hash produced by hashing fields with keccak256 algorithm.

Creator field is supposed to be equal to address of smart contract of application, owner field to be equal address of user, and item field to be equal unique item in context of application (action, achievement, order, goods) 

##### Signatures
Ethereum.sol allows to save signature associated with recovered ethereum address, and Bitcoin.sol allows to save signature associated with recovered bitcoin address. Storage contracts for other blockchains can be implemented and deployed to Plasma Chain as well.

Relayers can listen to RevealedSignature event fired every time someone publishes his signature. It contains recovered address and signature that can be broadcasted to specific blockchain.

Both Ethereum and Bitcoin are based on secpk256 cryptography, but recover methods are different because ethereum and bitcoin generate addresses differently. Ethereum.sol recovers address using ecrecover function in solidity. In difference, Bitcoin.sol recovers bitcoin address with following steps: recover coordinates of public key using special function from precompiled smart contract, calculate compressed public key from coordinates, receive address by hashing public key with hash160 algorithm(ripemd160 from sha256 from public key), convert address to base58 encoding.

##### Escrow
Implementation for escrow deposits in Bitcoin will be based on Hashed Time Locked Contracts.

Escrow.sol is implementation of escrow deposits that can be deployed to Ethereum or other EVM-based blockchain.

Escrow.sol allows to create escrow deposits with following mandatory fields: message hash, witness, beneficiary, expiration time, witness fee, relayer fee.

Creator of deposit has to send funds that will cover fees and funds that beneficiary will receive.
When witness will sign message hash and publish his signature to Plasma Chain, then relayer will broadcast signature in transaction to Escrow.sol contract that will verify signature and release funds to beneficiary.
During execution of the same transaction, both relayer and witness also will receive their fees.
In the case, when deposit has expired before it was released, funds will be refunded back to creator.

##### Applications
Application deployed to Plasma Chain can interact with protocol contracts and cover fees for users.

Each application interpret messages differently, and implement different methods to create messages and publish signatures.

Diadem.sol allows users to publish their achievements, compose chain of achievements, and verify achievements of others.

### Client
We designed interaction with blockchain with two goals in mind: give users easy access to blockchain and keep them secure. 

#### Wallet
We implemented client-side wallet that supports multiple cryptocurrencies.

When user open wallet first time, it generates mnemonic and addresses for supported cryptocurrencies.

It leverages local storage to save generated private keys associated with addresses, and allows user to visit and use it any time.

It allows to export private keys and restore wallet on another device.

#### SDK
We developed library based on CAL (Chain Abstraction Layer) that allows to interact with supported blockchains using common interface.

It allows to build transactions to different blockchains and broadcast them using remote RPC nodes.

It provides methods to interact with protocol smart contracts on Plasma Chain and with escrow contracts on other blockchains.

Example:

```
const { Client, providers } = require("diadem")
const { LoomWalletProvider, EthereumWalletProvider, DiademWalletProvider } = providers

const privateKey = localStorage.getItem('privateKey')

const client = new Client()
client.addProvider(new LoomWalletProvider({ privateKey }))
client.addProvider(new EthereumWalletProvider({ privateKey })

client.createMessage({ owner, item }).then(({ loomTransaction, messageHash }) => {
    console.log(loomTransaction)
    return client.createDeposit({
        beneficiary, witness, messageHash, value, expirationTime
    }).then(({ ethTransaction, depositHash }) => {
      console.log(ethTransaction)
      return client.confirm({ messageHash }).then(({ loomTransaction }) => {
        console.log(loomTransaction)
        return client.listenWithdrawal({ ethereumWithdrawalTransaction }).then(() => {
          console.log(ethereumWithdrawalTransaction)  
        })
      })
    })
})
```

#### Boilerplate
Boilerplate application implements interface for Diadem.sol smart contract. 

It supports configurable wallet, (will) leverage SDK to interact with chosen blockchains, and allows user to connect his profile in social network. 

It allows to build in days applications that previously may require months.

### Bridge
Currently between blockchain and real users exists a gap because of required technical skills. 

We are using following approaches to build a bridge and give simple users an access to blockchain economy.

#### Relayers
One of the most hard obstacles on the path to blockchain, is that user needs to acquire cryptocurrency before he can make any transaction.

We solve this problem and allow to make transactions without need to buy cryptocurrency as described above, through applications that cover fees to Plasma Chain and relayers that connect Plasma Chain with other blockchains.

#### Fiat exchange
Currently, in order to buy or sell any amount of cryptocurrency, user needs to register on exchange and confirm his identity. It may be hard to learn interface of exchange, and may take a lot of time.

Instead, we allow user to withdraw cryptocurrency from their wallet to their credit card instantly and without fees, using technology of atomic swaps. 

#### Feeds reducer
Usually transactions take a time, but user needs instant feedback and simple UX.

We implemented service that listen to blockchains and build realtime feeds for every user, that is displayed in boilerplate.

#### Social networks
Over two billions persons have account in social network and many of them use it daily. 

Through integration with social networks, we make our solution available for everyone. 

Social networks provide great infrastructure for communication and collaboration of people that can be used for negotiations before creating escrow deposit. 

Social networks provides trust layer and reputation for owner of profile in social network.

Currently we support authorization via Facebook. Later we will allow to make transactions directly in social networks like Facebook and messengers like Telegram. Currently this step may compromise security, but we believe that later social networks will become decentralized and integration will be easier.

## Usecases
We define three types of escrow deposits in terms of value: business, charity, and personal.

Currently, business deposits is most essential. But we believe, that charity deposits and personal deposits also will gain usage and lead us to better society.

### Business
Business deposit has a goal for creator to receive direct benefits from execution of action by beneficiary.

For example, buyer may create a deposit for seller that will be released when seller will deliver goods.

### Charity
Charity deposit has a goal to deliver benefits for another person.

For example, sponsor who love children, may create a deposit that will be released to person who will save life of child in Africa.

### Personal
Personal deposit has a goal related to specific person.

For example, employer who want to motivate his employees, may create reward that will be released to specific person when his colleagues confirm that he quit smoking.   
