# EOSIO Quick Start
This guide is modeled from the Ethereum quick start at [moschetti.org](https://www.moschetti.org/rants/geth1.html). 
We will:
- install the required components
- create a wallet to sign transactions
- run a single node blockchain
- build and run a smart contract to track who owns the bank money
- interact with the blockchain via EOSIO CLI tools
- interact with the blockchain using Go
- maybe try the [Java SDK](https://github.com/EOSIO/eosio-java), although my Java experience is circa 2002
- maybe try the [JS SDK](https://github.com/EOSIO/eosjs), the most popular way to interact by far

For full EOSIO technical documentation, the best resource is the [Developer's Portal](https://developers.eos.io/welcome/latest/getting-started-guide/local-development-environment/index)

For the source code of the smart contract and the scripts, see the ```develop``` branch of this repo. 

### Prerequisites
- OSX (+brew) or Linux
- Windoze users must use Docker

## Step 1. Install EOSIO CDT/SDK and the Node
### OSX
```
brew tap eosio/eosio.cdt
brew install eosio.cdt
brew tap eosio/eosio
brew install eosio
```

### Debian/Ubuntu
> NOTE: these commands install eosio 2.1 and eosio.cdt 1.8.0-1 - the latest as of 05 June 2021
```
UBUNTU_VERSION=20.04   # <-- CHANGE this your version 16.04, 18.04, or 20.04

wget https://github.com/eosio/eosio.cdt/releases/download/v1.8.0/eosio.cdt_1.8.0-1-ubuntu-${UBUNTU_VERSION}_amd64.deb
sudo apt install ./eosio.cdt_1.8.0-1-ubuntu-${UBUNTU_VERSION}_amd64.deb

wget https://github.com/eosio/eos/releases/download/v2.1.0/eosio_2.1.0-1-ubuntu-${UBUNTU_VERSION}_amd64.deb
sudo apt install ./eosio_2.1.0-1-ubuntu-${UBUNTU_VERSION}_amd64.deb
```

For other linux distributions, see the [eosio.cdt](https://github.com/eosio/eosio.cdt) or [eosio](https://github.com/eosio/eos) READMEs.

### Executables
| Name      | Name derived from | Description
| ----------- | ----------- | ------------
| ```nodeos``` | node + eosio   | network node executable, responsible for building blocks, consensus, executing smart contracts, etc. 
| ```keosd```   | keys + eosio + daemon | encrypted keystore, never invoked directly, called from ```cleos```
| ```cleos```   | CLI + eosio | CLI client for interacting with ```nodeos``` and ```keosd```
| ```eosio-cpp```   | eosio + cpp | this smart contract compiler, although calling via CMake tooling is best practice


## Step 2. Start ```nodeos``` for single-node network

EOSIO is primarily based on plugins for design and modularity. The plugins below are the best for getting started. 
> NOTE: the ```delete-all-blocks``` parameters permanently erases the entire chain and creates a freshie on restart (just remove it to keep the chain)
To make it easier to build upon, let's make this a bash script. Paste the below block into your terminal.
``` bash
cat <<EOF > start.sh

nodeos -e -p eosio \
--plugin eosio::producer_plugin \
--plugin eosio::producer_api_plugin \
--plugin eosio::chain_api_plugin \
--plugin eosio::http_plugin \
--plugin eosio::history_plugin \
--plugin eosio::history_api_plugin \
--filter-on="*" \
--access-control-allow-origin='*' \
--contracts-console \
--http-validate-host=false \
--delete-all-blocks \
--verbose-http-errors >> nodeos.log 2>&1 &
EOF
chmod +x start.sh
```
## Step 3. Create a wallet and import the key

``` bash
mkdir ~/eosio && cd ~/eosio
cleos wallet create --file wallet_password
```
### Unlocking the wallet
```keosd``` has automatic wallet locking at about 15 mins of inactivity. If it locks run the following command to unlock it.
``` bash
cleos wallet unlock < wallet_password
```

### Import key
We will use the well-known EOSIO default key pair for simplicity. To create new ones, you may use ```cleos create key```
```
Default EOSIO Key Pair
Public key: EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
Private key: 5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3
```

Use this command to import the key.
``` bash
cleos wallet import --private-key 5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3
```

## Step 4. Create accounts
- The account names must be 1-12 characters, all lowercase for alpha, 1-5 for numeric, and periods/dots are also allowed. (^[a-z1-5.]{1,11}[a-z1-5]$)|(^[a-z1-5.]{12}[a-j1-5]$)
- Common name formats are like ```alice```, ```alice.nba```, or ```1b2j3d1hsl14```
- Accounts require an ```active``` key and ```owner``` key. Permissions are feature rich. For more info, see the [docs](https://developers.eos.io/welcome/v2.0/protocol/accounts_and_permissions)
- Every account may optionally have a smart contract deployed to it; "contract" accounts and "user" accounts are the same construct.
``` bash
cleos create account eosio bob EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
cleos create account eosio alice EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
cleos create account eosio bank EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
```

You can verify that the accounts were created as well as check resource usage:
``` bash
cleos get account <account-name>
```

## Step 5. Write and deploy a smart contract
- The contract will maintain loan balances due to the bank. 
- There will be 3 actions that can be called:
  1. ```borrowed``` - an account has borrowed assets
  2. ```paid``` - an account has paid down their debt
  3. ```chgcp``` - an account has agreed to take on the debt of an existing account's balance
- EOSIO has a standard asset data type that abstracts asset math, storage, etc. It is the same data type and smart contract used for the EOS token on the public network.
- There is no concept of **native** vs **smart contract** tokens, like you see with ETH vs ERC20 or ALGO vs Algorand ASA.
- Most private EOSIO private blockchains do not have a network token at all. Public chains use them to stake for system resources, like CPU, RAM, and Network, which managed independently.

Run the following code block to populate our ```loan.cpp``` file.
``` c++
cat <<EOF > loan.cpp
#include <eosio/eosio.hpp>
#include <eosio/asset.hpp>
#include <eosio/system.hpp>

class [[eosio::contract("loan")]] loan : public eosio::contract
{
public:
   loan(eosio::name self, eosio::name code, eosio::datastream<const char *> ds) : eosio::contract(self, code, ds), loan_t(self, self.value) {}
   ~loan() {}

   // NOTE: the bracketed mark-up (C++11/GNU-style attributes) is used in constructing the ABI, which defines the interface (API) to clients
   struct [[eosio::table, eosio::contract("loan")]] Loan
   {
      eosio::name counterparty;                                         // the account that owes the bank
      eosio::asset amount_due;                                          // asset data type is comprised of an int64, precision, and a 1-7 all capital letter string for the symbol
      eosio::time_point updated = eosio::current_time_point();          // saved as microseconds/epoch time, always UTC
      uint64_t primary_key() const { return counterparty.value; }       // configure the primary key
      uint64_t by_updated() const { return updated.sec_since_epoch(); } // for a 2nd index: allows sorting by last_updated
      uint64_t by_amount() const { return amount_due.amount; }          // for a 3rd index: allows based on amount_due
   };

   typedef eosio::multi_index<eosio::name("loans"), Loan, // multi_index is based on boost::multi_index, which has been around for many many years
                              eosio::indexed_by<eosio::name("byamount"), eosio::const_mem_fun<Loan, uint64_t, &Loan::by_amount>>,
                              eosio::indexed_by<eosio::name("byupdated"), eosio::const_mem_fun<Loan, uint64_t, &Loan::by_updated>>>
       loan_table;

   loan_table loan_t;

   // We will have 3 actions that may be called from a client.
   // - borrowed  - either creates a new record OR increments existing amount_due if exists
   // - paid      - decrements amount_due OR error if not exists
   // - chgcp     - change the counterparty

   [[eosio::action]] void
   borrowed(const eosio::name &counterparty, const eosio::asset &amount_borrowed)
   {
      // the borrower/counterparty must approve new borrowing
      eosio::require_auth(counterparty);                // if not signed by counterparty, error out and rollback
      eosio::check(amount_borrowed.amount > 0, "amount borrowed must be greater than 0");
      adjust_amount_due(counterparty, amount_borrowed); // private function (see below)
   }

   [[eosio::action]] void paid(const eosio::name &counterparty, const eosio::asset &amount_paid)
   {
      // the bank (this contract), a.k.a. "get_self()" must approve payments against the amount_due
      eosio::require_auth(get_self());
      adjust_amount_due(counterparty, amount_paid * -1);
   }

   [[eosio::action]] void chgcp(const eosio::name &current_cp, const eosio::name &new_cp)
   {
      // the new counterparty must authorize responsibility for new debts
      eosio::require_auth(new_cp);

      auto current_cp_itr = loan_t.find(current_cp.value);
      eosio::check(current_cp_itr != loan_t.end(), "counterparty not found");

      // new_cp's debt is created or incremented
      adjust_amount_due(new_cp, current_cp_itr->amount_due);

      // current_cp's debt is erased
      loan_t.erase(current_cp_itr);
   }

private:
   eosio::asset adjust_amount_due(const eosio::name &counterparty, const eosio::asset &adjustment)
   {

      auto loan_itr = loan_t.find(counterparty.value);
      if (loan_itr == loan_t.end())
      { // create a new record
         eosio::check(adjustment.amount > 0, "amount_due on a new loan must be greater than zero");
         loan_t.emplace(get_self(), [&](auto &l)
                        {
                           l.counterparty = counterparty;
                           l.amount_due = adjustment;
                           l.updated = eosio::current_time_point();
                        });
         return adjustment;
      }

      if (adjustment.amount < 0)
      {
         // by erroring out here, we avoid the situation of having to handle over payment amounts
         eosio::check(abs(adjustment.amount) < loan_itr->amount_due.amount, "cannot adjust amount_due below zero");
         if (adjustment == loan_itr->amount_due)
         {
            // loan is paid in full, erase the record
            loan_t.erase(loan_itr);
            return adjustment * 0; // return a valid asset type with an amount of zero
         }
      }

      eosio::asset new_total_due = loan_itr->amount_due + adjustment;
      loan_t.modify(loan_itr, get_self(), [&](auto &l)
                    {
                       l.amount_due = new_total_due;
                       l.updated = eosio::current_time_point();
                    });

      return new_total_due;
   }
};
EOF
```

## Compile contract to WASM
``` bash
eosio-cpp -abigen -o loan.wasm loan.cpp
```

You will see some warnings that our actions do not have [Ricardian contracts](https://eos.io/for-developers/build/ricardian-template-toolkit). These are template documents that explain the transaction to the user in human-readable form. These are displayed in GUI wallets when a user is asked to approve a transaction. It is akin to "terms and conditions" of a smart contract action.

The command creates ```loan.wasm``` and ```loan.abi```.

## Deploy the contract
``` bash
cleos set contract bank . loan.wasm loan.abi
```

If you take a look at the account (see above), you'll see that there is 403.8 KiB of RAM used. Our smart contract WASM and ABI is now stored in RAM within nodeos, ready to be called. 

## Step 6. Interact with the contract using ```cleos```
### Pushing actions
Creating and broadcasting transactions can be done with ```cleos push action``` with the following parameters.
``` bash
cleos push action <contract-account> <action-name> <action-parameters> -p <authorizer>@<permission-level>
```

Alice borrows $100 to create our first loan record. 
``` bash
cleos push action bank borrowed '["alice", "100.00 USD"]' -p alice@active
```

Querying the on-chain tables can be done with ```cleos get table``` with the following parameters.
``` bash
cleos get table <contract-account> <table-scope> <table-name>
```
> Note: the ```table-scope``` is simply a namespace that allows for segregating data. As an example usage, we could store loans for many banks in the same table via scoping. For now, we will just use a table scope that matches the account name, which is the standard practice for tables that only require a single scope.

We can view the ```loan_table``` with the ```cleos get table``` command. 
``` bash
cleos get table bank bank loans
```
The output will look like this: 
``` json
{
  "rows": [{
      "counterparty": "alice",
      "amount_due": "100.00 USD",
      "updated": "2021-06-05T18:47:52.000"
    }
  ],
  "more": false,
  "next_key": "",
  "next_key_bytes": ""
}
```
> Note: the ```more``` and ```next-key``` attributes are used for paginating through the table over multiple requests. 

### Borrow more as Alice
``` bash
cleos push action bank borrowed '["alice", "100.00 USD"]' -p alice@active
```
Check the loan table again and you will see that the amount_due increased.

### Pay a bit off
``` bash
cleos push action bank paid '["alice", "55.00 USD"]' -p bank@active
```
### Change the counter party to bob
``` bash
cleos push action bank chgcp '["alice", "bob"]' -p bob@active
```

### Borrow for Alice again and change counter party again
``` bash 
cleos push action bank borrowed '["alice", "650.00 USD"]' -p alice@active
cleos push action bank chgcp '["alice", "bob"]' -p bob@active
```
Check the table, and you'll see that alice's new loan was erased and bob's incremented accordingly. :thumbsup:

## Step 7. Reset chain script
This script will kill ```nodeos```, restart it using our start script (which deletes all blocks), create the accounts, build the contract, deploy it, create some loans, and print the table.  
> Note: there are many options for test frameworks. I prefer a combination of C++ unit tests and Go integration/function testing. I use Github Actions to run all tests on push, and PRs must pass all tests before merging. This is especially true on larger open source projects where I may not know the developer.
``` bash
cat <<EOF > reset.sh
#!/bin/bash
pkill nodeos

sleep 1s  # give nodeos time to respond to the KILL - adjust as needed, increase duration for slower CPUs

./start.sh

sleep 1s  # give nodeos time to start making blocks - adjust as needed

cleos create account eosio bob EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
cleos create account eosio alice EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
cleos create account eosio bank EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV

eosio-cpp -abigen -o loan.wasm loan.cpp
cleos set contract bank . loan.wasm loan.abi

cleos push action bank borrowed '["alice", "100.00 USD"]' -p alice@active
cleos push action bank borrowed '["bob", "1000.00 USD"]' -p bob@active
cleos get table bank bank loans
EOF

chmod +x reset.sh
./reset.sh
```

### [Learn how to create a Javascript client using this guide](js-client.md)