# Wallets

## Summary

TODO: What this standard is about.

## Motivation

TODO: Why we need this standard.

## Terminology

- **Connected**: The state of whether a dApp has access to one or more accounts in the form of `FunctionCall` access keys.

## Injected Wallet Interface

Below is a high-level overview of what an injected wallet should look like: 

```ts
import { providers } from "near-api-js";

interface Account {
  accountId: string;
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

## Actions

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

## Connecting

The purpose of connecting to a wallet is to give the dApp access to one more accounts in the form of `FunctionCall` access keys. The user will be prompted with an interface similar to the example taken from Math Wallet:

![Connect Prompt](assets/connect-prompt.png)

Within this prompt they can select one or more of their imported accounts and have it accessible to the dApp via the `getAccounts` request method.

### Considerations

- If the wallet has only one imported account, the UI could be simplified down to an approval prompt to connect with the account.
- If there are problems with the `AddKey` action for any account, we should continue unless none were successful. In the event where only a subset of the selected accounts were connected, the dApp can call `connect` again where the user could modify the list (remove existing accounts and/or add new ones).
- If it should be a requirement, we could consider a `maxAccounts` parameter for `connect` that restricts the selection to even a single account.

### Multiple Accounts

An important concept to this architecture is dApps having access to multiple accounts. This might seem confusing at first because why would a dApp want to sign transactions with multiple accounts at the same time? The idea is the dApp might still maintain the concept of a single "active" account, but users don't have to sign out and sign in to different accounts each time. The dApp can display a simple switcher and perform transactions with the new account without having to further prompt the user, thus improving the UX flow.

TODO: Add references/images around existing wallets and how they have multiple accounts internally, but only expose a single "active" account.

TODO: Talk about how WalletConnect has first-class support for this functionality.

TODO: Reference Ethereum APIs where they also return a list of accounts (though MetaMask always returns an array with one address).

### Locked Status

TODO: Talk about what it means for a wallet to be locked. What you can access etc. 
