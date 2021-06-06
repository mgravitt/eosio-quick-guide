# EOSIO JS Client Quick Start
This quick start builds on 

### Prerequisites
- First complete the loan smart contract by following the steps found in the README.
- Nodejs
- Install ```eosjs``` using ```npm``` or ```yarn```
    - ```yarn add eosjs```
    - ```npm install eosjs```


## Step 1. Initialize a JS package with dependencies

Create the ```package.json``` file 
``` bash
cat <<EOF > package.json 
{
  "name": "loanjs",
  "version": "1.0.0",
  "description": "JS quick start for interacting with the loan smart contract",
  "main": "index.js",
  "repository": "",
  "author": "max",
  "license": "MIT",
  "private": true,
  "dependencies": {
    "eosjs": "^22.0.0",
    "node-fetch": "^2.6.1",
    "util": "^0.12.4"
  }
}
EOF
yarn install
```

## Step 2. Create script to read loans
Let's create three scripts for our client.
1. Read and print the loans
2. Register a borrowed or paid amount 
3. Change the counter-party of a loan

Let's start with the simple read-only script.
``` bash
cat <<EOF > get-loans.js 
const { JsonRpc } = require('eosjs');
const fetch = require('node-fetch');
const rpc = new JsonRpc('http://127.0.0.1:8888', { fetch });

(async () => {
    console.log(
        await rpc.get_table_rows({
            code: 'bank',    // account name
            scope: 'bank',
            table: 'loans'
        }));
})();
EOF

node get-loans.js
```
This should print the loans in a manner that looks just like the output from ```cleos```. This is because they are simple wrappers around the same nodeos JSON RPC interface. 

As smart contract functionality has improved beyond the EVM, much richer business logic can be orchestrated on-chain. The result is that clients are much 'thinner'. They only need to be responsible for rendering input fields and passing those fields to the smart contract. This is an incredible new design pattern that improves upon the technical debt associated with highly coupled client-server applications. It also allows for more flexible UIs and signer/wallet experiences.

This is the ***passwordless future***.

## Step 3. Create the ```chgcp``` script
The ```chgcp``` action takes two parameters, the current counter-party and the new counter-party. This action must be authorized by the new counter-party.

Paste the below code block into your terminal to create the script.

``` bash 
cat <<EOF > chgcp.js 
const { Api, JsonRpc } = require('eosjs');
const { JsSignatureProvider } = require('eosjs/dist/eosjs-jssig');
const fetch = require('node-fetch');
const { TextDecoder, TextEncoder } = require('util');

const privateKeys = ["5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3"];
const signatureProvider = new JsSignatureProvider(privateKeys);
const rpc = new JsonRpc('http://127.0.0.1:8888', { fetch });
const api = new Api({ rpc, signatureProvider, textDecoder: new TextDecoder(), textEncoder: new TextEncoder() });

(async () => {

    // A little input cleansing, but only for UX rather than security. The contract guarantees 
    // we never reach an invalid or insecure state.
    if (process.argv.length != 4) {
        console.log("Usage: node chgcp.js [current-counter-party] [new-counter-party]")
        console.log("Invalid args: ", process.argv)
        process.exit(1)
    }

    // The parameters to 'transact' are the same as 'cleos push action'
    const result = await api.transact({
        actions: [{
            account: 'bank',        // account storing smart contract
            name: 'chgcp',          // action name
            authorization: [{
                actor: process.argv[3],
                permission: 'active',
            }],
            data: {
                current_cp: process.argv[2],
                new_cp: process.argv[3]
            },
        }]
    }, {
        blocksBehind: 3,    // use the block 3 from the head for TAPoS
        expireSeconds: 30,  // transaction expiration
    });
    console.dir(result);
})();
EOF
```

Then, run the script to move alice's debt to bob.
``` bash
node chgcp.js bob alice
```

If successful, the transaction receipt will be printed. The ```transaction_id``` is helpful for referencing this transaction in the block explorer or query tools. The rest of the receipt contains information about the system resouces used during execution, elapsed time, block time, block number, and so on.

The advanced reporting and query tools built into Erbium allow users to find precisely which state fields were created, updated, or erased on this transaction.

``` json 
{
  transaction_id: '2763f95a9a7a075431bf78e7e0d45cab1e5b2ad40623108202df2799b7771a98',
  processed: {
    id: '2763f95a9a7a075431bf78e7e0d45cab1e5b2ad40623108202df2799b7771a98',
    block_num: 3858,
    block_time: '2021-06-06T02:13:36.000',
    producer_block_id: null,
    receipt: { status: 'executed', cpu_usage_us: 131, net_usage_words: 14 },
    elapsed: 131,
    net_usage: 112,
    scheduled: false,
    action_traces: [ [Object] ],
    account_ram_delta: null,
    except: null,
    error_code: null
  }
}
```

## Step 4. Create the loan adjustment script
We could create a script for ```borrowed``` and a second one for ```paid```. However, there is a lot of shared code between these two actions so let's just create ```adjust.js``` to handle both scenarios.

We only need to adjust the authorizer and the action name between the scenarios. Borrowing requires the counter-party to approve, and payments require the bank.

``` bash
cat <<EOF > adjust.js 
const { Api, JsonRpc } = require('eosjs');
const { JsSignatureProvider } = require('eosjs/dist/eosjs-jssig');
const fetch = require('node-fetch');
const { TextDecoder, TextEncoder } = require('util');

const privateKeys = ["5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3"];
const signatureProvider = new JsSignatureProvider(privateKeys);
const rpc = new JsonRpc('http://127.0.0.1:8888', { fetch });
const api = new Api({ rpc, signatureProvider, textDecoder: new TextDecoder(), textEncoder: new TextEncoder() });

(async () => {

    if (process.argv.length != 5) {
        console.log("Usage: node adjust.js [borrowed|paid] [counter-party] [amount]")
        console.log("Invalid args: ", process.argv)
        process.exit(1)
    }

    var authorizer = process.argv[4]
    if (process.argv[3] == 'paid') {
        authorizer = 'bank'     // if the action is a payment, bank must sign
    }

    const result = await api.transact({
        actions: [{
            account: 'bank',
            name: process.argv[3],      // action name [borrowed|paid]
            authorization: [{
                actor: authorizer,
                permission: 'active',
            }],
            data: {
                counterparty: process.argv[3],
                amount: process.argv[4]
            },
        }]
    }, {
        blocksBehind: 3,
        expireSeconds: 30,
    });
    console.dir(result);
})();
EOF
```

## Step 5. Adjust a loan

To check the usage, simply run the script with no parameters.
``` bash
node adjust.js 
```
And it will print the usage help along with the terminal argument array.

``` bash
❯ node adjust.js
Usage: node adjust.js [borrowed|paid] [counter-party] [amount]
... Invalid args ...
```

Based on this, we'll use the arguments below. Let's also call ```get-loans.js``` so that we can see the adjustments that were made. 
``` bash
node adjust.js paid alice "53.12 USD" && node get-loans.js
```

This command will execute the transaction, print the receipt, and then retrieve the list of loans to print.


## Step 6. Update the ```reset.sh``` script

When working on the smart contract, we created a ```reset.sh``` script that abstracts several steps. It restarts nodeos, creates accounts, re-compiles the contract, etc. Let's create another one that uses our JS scripts. 

``` bash
cat <<EOF > reset-js.sh
#!/bin/bash
start_time='$(date +%s.%6N)'
pkill nodeos

sleep 1s  # give nodeos time to respond to the KILL

./start.sh

sleep 1s  # give nodeos a few seconds to start making blocks

# we can create accounts and deploy contracts from JS also, but we didn't make that script yet
cleos create account eosio bob EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
cleos create account eosio alice EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
cleos create account eosio charlie EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
cleos create account eosio bank EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV

eosio-cpp -abigen -o loan.wasm loan.cpp
cleos set contract bank . loan.wasm loan.abi

node adjust.js borrowed alice "1298.12 USD"
node adjust.js borrowed bob "120.12 USD"
node adjust.js borrowed charlie "99987.12 USD"
node adjust.js paid alice "53.12 USD"
node adjust.js paid charlie "5333.12 USD"
node chgcp.js alice bob
node adjust.js borrowed alice "1.09 USD"

node get-loans.js

end_time='$(date +%s.%6N)'
elapsed='$(echo "scale=6; $end_time - $start_time" | bc)'
echo $elapsed
EOF
chmod +x reset-js.sh
```

When you run this new reset script, notice how long it takes.  This includes 2 full seconds of sleeping, destroying chain, creating a chain, creating 4 accounts, compiling a smart contract, deploying the contract, 6 transactions, and a query.

Compare this speed to Ethereum, Besu, or any other blockchain technology. :stopwatch: :astonished:

``` bash 
./reset-js.sh
```

## Step 6a. Measure performance

You can add the below lines into the bash scripts to measure performance:
``` bash
start_time=$(date +%s.%3N)

# TRANSACTIONS GO HERE

end_time=$(date +%s.%3N)
elapsed=$(echo "scale=3; $end_time - $start_time" | bc)
echo ${elapsed}
```

I've created a script in the develop branch named ```reset-js-time.sh```.