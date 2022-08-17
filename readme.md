<p align="center">
<img src="./logo_svg.svg" width=150>
</p>

# Meteor Public Repo

This repository is for all public communication around Meteor Wallet. This includes any feedback that users might have concerning it- please feel free to open an issue or discussion if you'd like to give us some feedback!

# Integrate Meteor into your dapp

The easiest way to integrate Meteor Wallet is to make use of the [Near Wallet Selector](https://github.com/near/wallet-selector) in your dApp.

You can install the required packages from the NPM registry, the wallet requires `near-api-js` v0.44.2 or above:

```bash
# Using Yarn
yarn add near-api-js@^0.44.2 @near-wallet-selector/core @near-wallet-selector/meteor-wallet

# Using NPM.
npm install near-api-js@^0.44.2 @near-wallet-selector/core @near-wallet-selector/meteor-wallet
```

Then use it in your dApp:

```ts
import { setupWalletSelector } from "@near-wallet-selector/core";
import { setupMeteorWallet } from "@near-wallet-selector/meteor-wallet";

const meteorWallet = setupMeteorWallet();

const selector = await setupWalletSelector({
  network: "testnet",
  modules: [meteorWallet],
});

const wallet = await selector.wallet();
const accounts = await wallet.signIn({ contractId: "test.testnet" });
```

You can also implement other wallets in a similar manner inside the `modules` array option here.

You can find the full API of the `selector` [here](https://github.com/near/wallet-selector/blob/main/packages/core/docs/api/selector.md).

More important is [the Wallet API](https://github.com/near/wallet-selector/blob/main/packages/core/docs/api/wallet.md) (the value returned from `await selector.wallet()`). This is what you will be using for logging into contracts and signing transactions.

Should you wish to have a quick-and-easy default UI, you can also make use of the Near Wallet Selector's Modal UI package (you will need to install the `@near-wallet-selector/modal-ui` package first):

```ts
import { setupModal } from "@near-wallet-selector/modal-ui";

const selector = // create the selector variable as above

const modal = setupModal(selector, {
  contractId: "test.testnet",
});

modal.show();
```

This UI will allow your users to easily choose which wallet to sign into, or switch between different wallets.