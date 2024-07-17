# Aptos specific protocols and schemes for e2e encryption, self encryption and mail
  
# Abstract
  
## Summary
  
This paper defines 3 interrelated protocols and their reference implementation to achieve e2e encryption, self encryption and simple mailing on Aptos.  
Briefly these protocols are:
* ACES: Aptos Chain Encryption Scheme defines a standard encryption scheme to encrypt and decrypt data.
* ACKSP: Aptos Chain Key Sharing Protocol defines a key sharing protocol for users on Aptos to share public keys.
* ACMP: Aptos Chain Mail Protocol defines a simple mailing protocol for users to exchange e2e encrypted on chain messages

There is also a sample Blockchain-Mail app that uses the above protocol to provide a simple e2e encrypted email like service  

# Introduction
  
*Problem Statement*: Currently their is no native way for users to encrypt/decrypt data in a universal way on Aptos. This severely limits decentralized app development as users cannot have true ownership over private data. Their is also no standardized messaging scheme that would allow for simple messaging passing.  
  
*Objective*: Provide a set of protocols that solve the above issues and in the long term get them standardized  
  
*Target Audience*: Daap developers, Aptos community

# Technical Overview

Note: We are still working on the exact specification of these protocols. The particulars are not fixed and are bound to change depending on community feedback.
  
### ACES
Aptos Chain Encryption Scheme defines a standard encryption scheme to encrypt and decrypt data. It is similar to an ECIES based encryption scheme.
  
*Reference Implementation*: [Link](https://github.com/crestfallnatwork/aces-js)  
*AIP*: Not implemented yet  
*White Paper*: Not implemented yet  
  
The encryption scheme leaves most of the encryption logic to the client having the wallets do only the sensitive ecdh part.  
We propose that the following two features be added to the aptos wallet standard:
```js
ecdh(theirPublicKey: Uint8Array): Promise<Uint8Array
publicEncryptionKey(): Promise<Uint8Array> 
```
*publicEncryptionKey*: this function will return users an x25519 public encryption key which could be used to send them messages. Due to the aptos supporting multiple key types this could not just simply be users public key. Depending on type of keys used by the user, the key can by derived as:  
* secpk1256 keys: take the first 32 bytes of the serialized private key and use it as the private point on Curve25519 and derive the corresponding public key.
* ed25519 keys: map the ed25519 private key onto the montgomery curve and derive the public key from it.
* multisig: as of now there is no support for multisig accounts  
*ecdh*: this function returns the shared secret that is generate by applying the Curve25519 function on a derived private key and the provided Curve25519 public key. The private key derivation is explained in the publicEncryptionKey part.  
For a reference implementation see this [link](https://github.com/crestfallnatwork/aces-js/blob/main/src/ecdh/impl.ts).

Using these feature it is possible to implement a secure encryption scheme  
  
To encrypt a message ACES does the following:  
1. Get the public key(rpk) of the message recipient. 
   * This public key should be a point on the Curve25519 with the recipient having the corresponding private key. 
   * This could be the output from the recipients wallet on calling publicEncryptionKey.
2. Generate an ephemeral key pair(esk, epk) on Curve25519.
3. Use epk and rpk to generate a shared secret(ess)
4. Use wallets ecdh functionality to generate another shared secret(pss)
5. Apply HKDF with secret as (ess | pss), salt as "" and info as "" to derive a 256-bit key.
   * salt and info can be changed depending on application.
6. Generate a 12 byte IV(iv)
7. Use AES in GCM mode using IV as iv to generate ciphertext(cm)
8. Encrypted message := (epk | iv | cm)
  
To decrypt this message follow the steps in reverse.  
For a more comprehensive overview look at the [reference implementation](https://github.com/crestfallnatwork/aces-js)  
  
### ACKSP
Aptos Chain Key Sharing Protocol defines a standard way to publish public and optionally encrypted private keys on the blockchain.  
   
*Reference Implementation*: [Contract](https://github.com/crestfallnatwork/acmp-contracts/blob/main/acmp/sources/acksp.move), [Client](https://github.com/crestfallnatwork/acksp-js)  
*AIP*: Not implemented yet  
*White Paper*: Not implemented yet  
  
Current implementation only supports Ed25519 key pairs but more key types will be supported.  
The contract provides a way to publish keys and retrieve keys. The users are advised to publish their public keys along with their encrypted private keys. Private keys are encrypted using ACES with users own publicEncryptionKey. Look at the reference implementation [here](https://github.com/crestfallnatwork/acmp-contracts/blob/main/acmp/sources/acksp.move)  
The client extends the functionality by providing a way to retrieve keys valid on a particular timestamp along with a way to create an ACES client based on the stored keys. For more information look at the clients reference implementation [here](https://github.com/crestfallnatwork/acksp-js)  

### ACMP
Aptos Chain Mail Protocol defines a standard way to send messages on the blockchain.  
  
*Reference Implementation*: [Contract](https://github.com/crestfallnatwork/acmp-contracts/blob/main/acmp/sources/acmp.move), [Client](https://github.com/crestfallnatwork/acmp-js)  
*AIP*: Not implemented yet  
*White Paper*: Not implemented yet  
  
ACMP uses the blockchain for timestamps and from fields and transportation of the messages and relies on the client to pack and submit messages.  
While the protocol does not necessitate encryption, barring any special reasons, we propose that all users should use ACES for e2e encryption along with ACKSP for key retrieval.  
A message on the blockchain has the following structure:  
```move
struct Message has store , copy
{
    timestamp: u64,
    payload: vector<u8>,
    from: address
}
```
The client packed messages have a special binary encoding. Take a look at the [reference implementation](https://github.com/crestfallnatwork/acmp-js) for further details.  
  
### Aptos-Mail  
Aptos Mail is a simple e2e encrypted email like app that we created to showcase the protocols. It is deployed on the testnet and can be accessed at this [link](https://aptos-mail.vercel.app/). You can also look at the source code [here](https://github.com/crestfallnatwork/aptos-mail)  

Note: Due to no wallets yet supporting the features mentioned in the ACES section, the app requires you to import your private key. Upon import you will be automatically charged APT.  
  
# Further Improvements  
  
* As mentioned before the protocols are not finalized and would benefit great from community engagement. We have already started working on that.
* Apart from this a name service protocol would be a great supplement to this suite. 
* The Aptos-Mail demonstration app lacks a lot of polish due to time constraints.

# Conculsion
  
It is our firm belief that the aptos community can benefit from the standardization of these protocols.  
  
For any queries you can reach out to the authors at:  
[crestfalln](mailto:mail@crestfalln.com)  
[gurshaan](mailto:)  
