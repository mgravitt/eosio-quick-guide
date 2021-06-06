# EOSIO JS Client Quick Start
This quick start builds on the initial [EOSIO Quickstart](README.md)

### Prerequisites
- First complete the loan smart contract by following the steps found in the README.
- Nodejs and npm or yarn (tested on v14.17.0) - I recommend install these with [Node Version Manager](https://github.com/nvm-sh/nvm)

#### (Optional) Automatic dependency install for nvm, node, npm, and yarn
``` bash
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.38.0/install.sh | bash
source ~/.bashrc  # change to zshrc if you use zsh
nvm install node  # "node" is an alias for the latest version
nvm use node
npm install -g yarn

```

## Steps
1. [Initialize a JS package with dependencies](step-1-initialize-a-js-package-package-with-dependencies)
2. [Create script :memo: to read loans :moneybag:](step-2-create-script-memo-to-read-loans-moneybag-1)
3. [Create the ```chgcp``` script :memo:](step-3-create-the-chgcp-script-memo-1)
4. [Create the loan :moneybag: adjustment script :memo:](step-4-create-the-loan-moneybag-adjustment-script-memo-1)
5. [Adjust a loan :moneybag:](step-5-adjust-a-loan-moneybag-1)
6. [Update the ```reset.sh``` script :link: :arrows_counterclockwise:](step-6-update-the-resetsh-script-link-arrows_counterclockwise-1)
7. [Measure performance :stopwatch:](step-7-measure-performance-stopwatch-1)

## Step 1. Initialize a JS package :package: with dependencies

Create the ```package.json``` file 
``` bash
cat <<EOF > package.json 
{
  "name": "loanjs",
  "description": "JS quick start for interacting with the loan smart contract",
  "dependencies": {
    "eosjs": "^22.0.0",
    "node-fetch": "^2.6.1",
    "util": "^0.12.4"
  }
}
EOF
yarn install

```

## Step 2. Create script :memo: to read loans :moneybag:
Let's create three scripts for our client.
1. Read and print the loans
2. Register a borrowed or paid amount 
3. Change the counter-party :white_haired_main: of a loan :moneybag:

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
This should print the loans in a manner that looks just like the output from ```cleos```. This is because they are simple wrappers around the same ```nodeos``` JSON RPC interface. :thumbsup:

As smart contract :scroll: functionality has improved beyond the EVM, much richer business logic can be orchestrated on-chain :link:. The result is that clients are much 'thinner'. They only need to be responsible for rendering input fields and passing those fields to the smart contract. This is an incredible new design pattern that improves upon the technical debt associated with highly coupled client-server applications. 

It enables higher quality, more flexible :cartwheeling: UIs, and improved signer/wallet :key: user experiences.

This is the ***passwordless future***. :exploding_head: :cowboy_hat_face:

## Step 3. Create the ```chgcp``` script :memo:
The ```chgcp``` action takes two parameters, the current counter-party and the new counter-party. This action must be authorized by the new counter-party.

Paste the below code block into your terminal to create the script. :memo:

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

Then, run the script to move Alice's :red_haired_woman: debt to Bob. :white_haired_man:
``` bash
node chgcp.js bob alice
```

If successful, the transaction receipt :receipt: will be printed. The ```transaction_id``` is helpful for referencing this transaction in the block explorer :astronaut: or query tools :mag_right:. The rest of the receipt contains information about the system resouces used during execution, elapsed time :stopwatch:, block time :clock2:, block number :bricks:, and so on.

The advanced reporting and query tools :toolbox: built into Erbium allow users to find precisely which state fields were created, updated, or erased on this transaction. :stop_sign:
``` bash
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

## Step 4. Create the loan :moneybag: adjustment script :memo:
We could create a script for ```borrowed``` and a second one for ```paid```. However, there is a lot of shared code between these two actions, so let's just create ```adjust.js``` to handle both scenarios.

We only need to adjust the authorizer :key: and the action name between the scenarios. Borrowing requires the counter-party to approve :fountain_pen:, and payments require the bank to approve :fountain_pen:.

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

    var authorizer = process.argv[3]
    if (process.argv[2] == 'paid') {
        authorizer = 'bank'     // if the action is a payment, bank must sign
    }

    const result = await api.transact({
        actions: [{
            account: 'bank',
            name: process.argv[2],      // action name [borrowed|paid]
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

## Step 5. Adjust a loan :moneybag:

To check the usage, simply run the script with no parameters.
``` bash
node adjust.js 
```
And it will print the usage help along with the terminal argument array. :stop_sign:
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

## Step 6. Create ```reset-js.sh``` script :link: :arrows_counterclockwise:

When working on the smart contract, we created a ```reset.sh``` script that abstracts several steps. It restarts nodeos, creates accounts, re-compiles the contract, etc. Let's create another one that uses our JS scripts. 

``` bash
cat <<EOF > reset-js.sh
#!/bin/bash
pkill nodeos

sleep 1s  # give nodeos time to respond to the KILL

./start.sh
cleos wallet unlock < wallet_password # this will show error if wallet is already open, but that is fine

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
EOF
chmod +x reset-js.sh
```

When you run this new reset script :memo:, notice how long it takes :stopwatch:.  This includes 2 full seconds of sleeping :sleeping:, destroying chain :link:, creating a chain :link:, creating 4 accounts :red_haired_woman:, compiling a smart contract :scroll:, deploying the contract :scroll:, 7 transactions :arrow_forward:, and a query :mag_right:.

Compare this speed to Ethereum, Besu, Cosmos, Substrate or any other blockchain :chains: technology. :stopwatch: :astonished:

``` bash 
./reset-js.sh
```

## Step 7. Measure performance :stopwatch:

You can add the below lines BEFORE and AFTER the transactions in the bash scripts to measure performance. :stop_sign:
``` bash
start_time=$(date +%s.%3N)

# TRANSACTIONS GO HERE

end_time=$(date +%s.%3N)
elapsed=$(echo "scale=3; $end_time - $start_time" | bc)
echo ${elapsed}
```

:bulb: I created a script :memo: in the develop branch named ```reset-js-time.sh```.  Elapsed time :stopwatch: on my computer for all of the transaction is about 1.2 seconds. Elapsed time for the entire script, including the 2 sleep seconds, is just over 5 seconds.

:bulb: Also, ```adjust-with-loop.sh``` is worth running. It simply puts the transactions into a loop to execute 50 times and measures that. My local machine is executing these 350 transactions in 46.042 s, or ~8 trx/second. 

Of course, this isn't indicative of network performance, but only anecdotal when compared to DevX of other tech. In prod, transactions would be submitted multi-threaded and there are multiple nodes. The public EOSIO networks process [more transactions per day than any other network](https://blocktivity.info/). 
