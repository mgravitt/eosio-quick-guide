# Prerequisites
- OSX or Linux only
- cmake 

# Install EOSIO node & SDK
## OSX
```
brew tap eosio/eosio.cdt
brew install eosio.cdt
brew tap eosio/eosio
brew install eosio
```

## Debian/Ubuntu
> NOTE: these commands install eosio 2.1 and eosio.cdt 1.8.0-1 - the latest as of 05 June 2021
```
UBUNTU_VERSION=20.04   # <-- CHANGE this your version 16.04, 18.04, or 20.04

wget https://github.com/eosio/eosio.cdt/releases/download/v1.8.0/eosio.cdt_1.8.0-1-ubuntu-${UBUNTU_VERSION}_amd64.deb
sudo apt install ./eosio.cdt_1.8.0-1-ubuntu-${UBUNTU_VERSION}_amd64.deb

wget https://github.com/eosio/eos/releases/download/v2.1.0/eosio_2.1.0-1-ubuntu-${UBUNTU_VERSION}_amd64.deb
sudo apt install ./eosio_2.1.0-1-ubuntu-${UBUNTU_VERSION}_amd64.deb
```

For other linux distributions, see the READMEs eosio.cdt[https://github.com/eosio/eosio.cdt] or eosio[https://github.com/eosio/eos]

For more complete environment setup instructions, see the Developer's Portal[https://developers.eos.io/welcome/latest/getting-started-guide/local-development-environment/index]


## Component Overview
| Component      | Derived from | Description
| ----------- | ----------- | ------------
| ```nodeos``` | node + eosio   | network node executable, responsible for building blocks, consensus, executing smart contracts, etc. 
| ```keosd```   | keys + eosio + daemon | encrypted keystore, never invoked directly, called from ```cleos```
| ```cleos```   | CLI + eosio | CLI client for interacting with ```nodeos``` and ```keosd```
| ```eosio-cpp```   | eosio + cpp | this smart contract compiler, although calling via CMake tooling is best practice


# Create a wallet/key
``` bash
mkdir ~/eosio && cd ~/eosio
cleos wallet create --file wallet_password
```
### Unlocking the wallet
```keosd``` has automatic wallet locking at about 15 mins of inactivity. If it locks run the following command to unlock it.
``` bash
cleos wallet unlock < wallet_password
```

## Import key
We will use the EOSIO default key pair for simplicity.
```
Public key: EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
Private key: 5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3
```

Use this command to import the key.
``` bash
cleos wallet import --private-key 5KQwrPbwdL6PhXujxW37FSSQZ1JiwsST4cqQzDeyXtP79zkvFD3
```

# Start the node
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

# Create accounts
- the account names must be 1-12 characters, all lowercase for alpha, 1-5 for numeric, and periods/dots are also allowed. (^[a-z1-5.]{1,11}[a-z1-5]$)|(^[a-z1-5.]{12}[a-j1-5]$)
- common name formats are things like ```alice```, ```alice.nba```, or ```1b2j3d1hsl14```
- accounts require an ```active``` key and ```owner``` key. Permissions are feature rich- for more info, see the docs[https://developers.eos.io/welcome/v2.0/protocol/accounts_and_permissions]
- every account may optionally have a smart contract deployed to it; "contract" accounts and "user" accounts are the same construct
``` bash
cleos wallet
cleos create account eosio bob EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
cleos create account eosio alice EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
cleos create account eosio bank EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
```

You can verify that the accounts were created as well as check resource usage:
``` bash
cleos get account <account-name>
```


# Write a smart contract
- modeling the tutorial on moschetti.org[https://www.moschetti.org/rants/geth1.html], we'll store 3 pieces of data to represent an outstanding balance due to the bank.
- EOSIO has a standard asset data type that abstracts asset math, storage, etc. It is the exact SAME type and smart contract used for the EOS token on the public network.
- There is no concept of "native" vs "smart contract" tokens, like you see with ETH vs ERC20 or ALGO vs Algorand ASA.
- Most private EOSIO private blockchains do not have a network token. Public chains use them to stake for system resources, like CPU, RAM, and Network, which are all priced separately.

``` c++
cat <<EOF > loan.cpp
#include <eosio/eosio.hpp>
using namespace eosio;

   CONTRACT loan : public contract
   {
   public:
      using contract::contract;

    // the bracketed mark-up (C++11/GNU-style attributes) is used in constructing the ABI, which defines the interface to clients
    struct [[eosio::table, eosio::contract("loan")]] Loan
      {
         eosio::name    counterparty;           // the account that owes the bank 
         eosio::asset   amount_due;             // asset data type is comprised of an int64, precision, and a 1-7 all capital letter string for the symbol
         eosio::time_point  last_updated;       // saved as microseconds/epoch time, always UTC
         uint64_t primary_key() const { return counterparty.value; }    // configure the primary key
         uint64_t by_updated() const { return created_date.sec_since_epoch(); } // for a 2nd index: allows sorting by last_updated
         uint64_t by_amount() const { return amount_due.amount } // for a 3rd index: allows based on amount_due
      };

        typedef eosio::multi_index<eosio::name("loans"), Loan,   // multi_index is based on boost::multi_index, which has been around for many many years
         eosio::indexed_by<eosio::name("byamount"), eosio::const_mem_fun<Loan, uint64_t, &Loan::by_amount>>,
         eosio::indexed_by<eosio::name("byupdated"), eosio::const_mem_fun<Loan, uint64_t, &BorrowOffer::by_updated>>
      > loan_table;


    // We will have 3 actions that may be called from a client

     [[eosio::action]]   // bracketed markup is used for ABI construction (allows function to be called directly from a client)
     public void borrowed (const eosio::name &counterparty, const eosio::asset &amount_due) {

     }

     [[eosio::action]]
     public void paid (const eosio::name &counterparty, const eosio::asset &amount_due) {

     }
     
     [[eosio::action]]
     public void chgcp(const eosio::name &current_cp, const eosio::name &newcp) {

     }
   };
}
EOF
```

## Compile contract to WASM
``` bash
eosio-cpp -abigen -o loan.wasm loan.cpp
1``

You will see some warnings that our actions do not have Ricardian contracts[https://eos.io/for-developers/build/ricardian-template-toolkit/]. These are template documents that explain the transaction to the user in human-readable form. These are displayed in GUI wallets when a user is asked to approve a transaction. It is akin to "terms and conditions" of a smart contract action.

The output is ```loan.wasm``` and ```loan.abi```.

## Deploy the contract
``` bash
cleos set contract bank . loan.wasm loan.abi
```

If you take a look at the account (see above), you'll see that there is 403.8 KiB of RAM used. Our smart contract now sits in RAM within nodeos, ready to be called. 

## Create a loan
Creating and broadcasting transactions can be done with ```push action```
``` bash
cleos push action <contract-account> <action-name> <action-parameters> -p <authorizer>@<permission-level>
```
Alice borrowing $100
``` bash
cleos push action bank borrowed '["alice", "100.00 USD"]' -p alice@active
```

We can view the ```loan_table``` with ```get table```
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

## Borrow more as Alice
``` bash
cleos push action bank borrowed '["alice", "100.00 USD"]' -p alice@active
```
Check the loan table again and you will see that the amount_due increased.

## Pay a bit off
``` bash
cleos push action bank paid '["alice", "55.00 USD"]' -p bank@active
```
## Change the counter party to bob
``` bash
cleos push action bank chgcp '["alice", "bob"]' -p bob@active
```

## Borrow for Alice again and change county party again
``` bash 
cleos push action bank borrowed '["alice", "650.00 USD"]' -p alice@active
cleos push action bank chgcp '["alice", "bob"]' -p bob@active
```
Then check the table, and you'll see that alice's new loan was erased and bob's incremented accordingly. :thumbsup:

Here are some interesting things to try:

## Reset chain script
This script will kill nodeos, restart it using our start script (which deletes all blocks), create the accounts, build the contract, deploy it, create some loans, and print the table. 
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