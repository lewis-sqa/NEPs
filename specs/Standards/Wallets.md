# Wallets

## Summary

TODO: What this standard is about.

## Motivation

TODO: Why we need this standard.

## Terminology

- **Connected**: The state of whether a dApp has access to one or more accounts in the form of `FunctionCall` access keys.

## Injected Wallets

TODO: Description here.

### Namespace

TODO: Description here explaining wallets will use the `window.near` namespace and expose a property to detect the wallet e.g. `isSender` or `isMathWallet`.

### Wallet Interface

Below is a high-level overview of what an injected wallet should look like: 

```ts
import { providers } from "near-api-js";

interface Account {
  accountId: string;
  // Public key related to the underlying FunctionCall access key.
  publicKey: string;
}

interface Network {
  networkId: string;
  nodeUrl: string;
}

interface ConnectParams {
  contractId: string;
  methodNames?: Array<string>;
  maxAccounts?: number;
}

interface SignAndSendTransactionParams {
  signerId?: string;
  receiverId: string;
  // NEAR Actions (plain objects). See "Actions" section for details.
  actions: Array<Action>;
}

interface Transaction {
  signerId?: string;
  receiverId: string;
  // NEAR Actions (plain objects). See "Actions" section for details.
  actions: Array<Action>;
}

interface SignAndSendTransactionsParams {
  transactions: Array<Transaction>;
}

// Allows for other functionality in the future.
// Heavily inspired by https://docs.metamask.io/guide/rpc-api.html#ethereum-json-rpc-methods.
interface Methods {
  // Get accounts exposed to dApp.
  // Empty list of accounts means we aren't connected.
  getAccounts: {
    params: {
      method: "getAccounts";
    };
    response: Array<Account>;
  };
  // Get the currently selected network.
  getNetwork: {
    params: {
      method: "getNetwork";
    };
    response: Network;
  };
  // Request access to one or more accounts (i.e. sign in).
  connect: {
    params: {
      method: "connect";
      params: ConnectParams;
    };
    response: Array<Account>;
  };
  // Remove access to all accounts (i.e. sign out).
  disconnect: {
    params: {
      method: "disconnect";
    };
    response: void;
  };
  // Sign and Send one or more NEAR Actions.
  signAndSendTransaction: {
    params: {
      method: "signAndSendTransaction";
      params: SignAndSendTransactionParams;
    };
    response: providers.FinalExecutionOutcome;
  };
  // Sign and Send one or more NEAR Transactions.
  signAndSendTransactions: {
    params: {
      method: "signAndSendTransactions";
      params: SignAndSendTransactionsParams;
    };
    response: Array<providers.FinalExecutionOutcome>;
  };
}

interface Events {
  accountsChanged: { accounts: Array<Account> };
};

type Unsubscribe = () => void;

interface Wallet {
  request<
    MethodName extends keyof Methods,
    Method extends Methods<MethodName>
  >(
    params: Method["params"]
  ): Promise<Method["response"]>;
  on<EventName extends keyof Events>(
    event: EventName,
    callback: (params: Events<EventName>) => void
  ): Unsubscribe;
  off<EventName extends keyof Events>(
    event: EventName,
    callback?: () => void
  ): void;
}
```

### Actions

Below are the 8 NEAR Actions and used for signing transactions:

```ts
interface CreateAccountAction {
  type: "CreateAccount";
}

interface DeployContractAction {
  type: "DeployContract";
  params: {
    code: Uint8Array;
  };
}

interface FunctionCallAction {
  type: "FunctionCall";
  params: {
    methodName: string;
    args: object;
    gas: string;
    deposit: string;
  };
}

interface TransferAction {
  type: "Transfer";
  params: {
    deposit: string;
  };
}

interface StakeAction {
  type: "Stake";
  params: {
    stake: string;
    publicKey: string;
  };
}

type AddKeyPermission =
  | "FullAccess"
  | {
      receiverId: string;
      allowance?: string;
      methodNames?: Array<string>;
    };

interface AddKeyAction {
  type: "AddKey";
  params: {
    publicKey: string;
    accessKey: {
      nonce?: number;
      permission: AddKeyPermission;
    };
  };
}

interface DeleteKeyAction {
  type: "DeleteKey";
  params: {
    publicKey: string;
  };
}

interface DeleteAccountAction {
  type: "DeleteAccount";
  params: {
    beneficiaryId: string;
  };
}

type Action =
  | CreateAccountAction
  | DeployContractAction
  | FunctionCallAction
  | TransferAction
  | StakeAction
  | AddKeyAction
  | DeleteKeyAction
  | DeleteAccountAction;
```

### Examples

**Connect to the wallet**

```ts
const accounts = await window.near.request({
  method: "connect",
  params: { contractId: "guest-book.testnet" }
});
```

**Get accounts (exposed via `connect`)**

```ts
const accounts = await window.near.request({
  method: "getAccounts"
});
```

**Subscribe to account changes**

```ts
await window.near.on("accountsChanged", (accounts) => {
  console.log("Accounts Changed", accounts);
});
```

**Get network configuration**

```ts
await window.near.request({ method: "getNetwork" });
```

**Sign and send a transaction**

```ts
const result = await window.near.request({
  method: "signAndSendTransaction",
  params: {
    signerId: "test.testnet",
    receiverId: "guest-book.testnet",
    actions: [{
      type: "FunctionCall",
      params: {
        methodName: "addMessage",
        args: { text: "Hello World!" },
        gas: "30000000000000",
        deposit: "10000000000000000000000",
      },
    }]
  }
});
```

## Connecting

The purpose of connecting to a wallet is to give dApps access to one or more accounts - backed by `FunctionCall` access keys. When the Connect flow is triggered, the user will be prompted with an interface similar to this example taken from Math Wallet:

![Connect Prompt](assets/connect-prompt.png)

The list of accounts to select are those that have been imported previously. The user can choose a subset of these accounts to expose to the dApp.

### Considerations

- If there's only one imported account, the flow can be simplified to an approval prompt to connect with the only account.
- If there are problems with the `AddKey` action for any account, we should continue unless none were successful. In the event where only a subset of the selected accounts were connected, the dApp can call `connect` again so the user can modify the list (remove existing accounts and/or add new ones).
- If the dApp would like to restrict the number of accounts (e.g. only one) a user can select, they can pass a `maxAccounts` parameter for the `connect` request method.

### Multiple Accounts

An important concept of this architecture is dApps have access to multiple accounts. This might seem confusing at first because why would a dApp want to sign transactions with multiple accounts? The idea is the dApp might still maintain the concept of a single "active" account, but users won't need to sign in and out of accounts each time. The dApp can just display a switcher and sign transactions with new account without having to further prompt the user, thus improving the UX flow.

TODO: Add references/images around existing wallets and how they have multiple accounts internally, but only expose a single "active" account.

TODO: Talk about how WalletConnect has first-class support for this functionality.

TODO: Reference Ethereum APIs where they also return a list of accounts (though MetaMask always returns an array with one address).

### Locked Status

TODO: Talk about what it means for a wallet to be locked. What you can access etc. 
