### Flow Cold Storage Proxy Contract

For security, one might partition accounts using hot and cold keys.
Unfortunately, Flow’s transaction expiry window is not long enough for
transactions to be signed by cold keys.

As a workaround, we can use a “meta transaction” of sorts. Cold keys can sign an
“inner transaction” payload which authorize the transfer of funds. The related
nonce and signature for this “inner transaction” are then passed in as
parameters of a `FlowColdStorageProxy.Vault#transfer` call within the “outer
transaction”.

This approach should enable cold keys to take however long they need to sign the
“inner transaction” and, once this is done, a “controller” account can then
construct, sign, and submit the “outer transaction”.

The FlowColdStorageProxy contract implements a Vault that wraps the underlying
FlowToken Vault. Accounts which need to have their funds protected by a cold key
are created using the following transaction script:

```
import FlowColdStorageProxy from 0x{{.Contracts.FlowColdStorageProxy}}

transaction(publicKey: String) {
    prepare(payer: AuthAccount) {
        // Create a new account with a Proxy Vault.
        FlowColdStorageProxy.setup(payer: payer, publicKey: publicKey.decodeHex())
    }
}
```

And funds are transferred from these accounts using the following transaction
script:

```
import FlowColdStorageProxy from 0x{{.Contracts.FlowColdStorageProxy}}

transaction(sender: Address, receiver: Address, amount: UFix64, nonce: Int64, sig: String) {
    prepare(payer: AuthAccount) {
    }
    execute {
        // Get a reference to the sender's FlowColdStorageProxy.Vault.
        let acct = getAccount(sender)
        let vault = acct.getCapability(FlowColdStorageProxyVault.VaultCapabilityPublicPath).borrow<&FlowColdStorageProxy.Vault>()!

        // Transfer tokens to the receiver.
        vault.transfer(receiver: receiver, amount: amount, nonce: nonce, sig: sig.decodeHex())
    }
}
```

The intent of the code is that:

* Funds held within a FlowColdStorageProxy Vault resource should only be
  transferrable if the `FlowColdStorageProxy.Vault#transfer` “inner transaction”
  has been signed by the publicKey specified during the
  `FlowColdStorageProxy.setup` call.

* Funds are protected against any replay attacks.

* Funds must not be transferrable in any other way.

* It should not be possible to move or destroy a FlowColdStorageProxy Vault
  resource, and thus the underlying funds.

* If the required minimum account balance requirement is ever increased, it
  should be possible to deposit funds to the default FlowToken Vault by using
  `/public/defaultFlowTokenReceiver` and ensure continued operations of an account
  owning a particular FlowColdStorageProxy Vault instance.
