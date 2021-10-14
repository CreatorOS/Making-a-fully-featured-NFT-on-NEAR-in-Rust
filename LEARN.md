# Making a fully featured NFT on NEAR in Rust
In the previous Quest, we looked at writing a dummy NFT on NEAR using Rust. If you’ve not done that yet, you should probably do that first since we’ll be building on top of what we did in the previous quest. 

It’s a pre-requisite that you’ve completed the previous quest on Making a Dummy NFT from [https://questbook.app](https://questbook.app) 

To create an NFT that qualifies as an NFT on near, we’ll have to stick to some standards that have been defined by the community. If you have come from some Ethereum background, Ethereum community calls this standard an ERC721. Similarly, NEAR community has defined what is called an NEP4. NEP-4 basically tells the developer what skeleton their contract should follow in order to be able to create an NFT. If your NFT follows this specification it will be possible to list this NFT on various NFT marketplaces. The NFT marketplaces require all the NFTs listed to have some functions that they can call to facilitate buying and selling of NFTs.

That is actually one of the most important parts of Blockchain programming - composability. You always try to write code in such a way that it is easy for other applications to be built on top. By exposing functions and adhering to the right specifications, makes app development using your app much easier.

Now, on to building …
## Defining the functions we need
For this we will define an interface (called trait in rust) that adheres to the NEP4 definitions. 

Open up your `lib.rs` from your previous quest and add the following to it.

```

pub trait NEP4 {

    fn grant_access(&mut self, escrow_account_id: AccountId);

    fn revoke_access(&mut self, escrow_account_id: AccountId);

    fn transfer_from(&mut self, owner_id: AccountId, new_owner_id: AccountId, token_id: TokenId);

    fn transfer(&mut self, new_owner_id: AccountId, token_id: TokenId);

    // Returns `true` or `false` based on caller of the function (`predecessor_id) having access to the token

    fn check_access(&self, account_id: AccountId) -> bool;

    // Get an individual owner by given `tokenId`.

    fn get_token_owner(&self, token_id: TokenId) -> String;

}

pub type TokenId = u64;

pub type AccountIdHash = Vec;

```

Let’s look at what these functions are for. 

1. Grant_access : if you are the owner of the NFT, you can grant some account so that they can operate on this NFT on behalf of the owner.
2. Revoke_access: Revoke the access you might have given above. A user can revoke the access they’ve granted earlier. The user calling this function must be the owner of this account. 
3. Transfer_from : A user can transfer ownership to someone else. Thereby giving up all the privileges on this NFT. This is possible only if the user calling this function has been granted access to this NFT by the owner.
4. Transfer : If the owner intends to transfer the NFT, they can directly transfer the NFT without having to go through the process of granting access and then transferring 
5. Check_access : see if the owner has granted access to the said account. 
6. Get_token_owner : get the account ID of the owner of this NFT, identified by a TokenID

Lastly we define two new datatypes

1. TokenId : Token id is an alias for u64. Meaning that the NFTs which are identified by a unique number will be a 64bit unsigned integer
2. AccountIdHash : An AccountId is how a user’s account is identified. However, each account may be represented by a variable number of digits. E.g. madhavan.near or madhavanmalolan.near. A hash of the accountId will always be of specific length. This hash is a list of 8bit unsigned integers.
## Storing on Blockchain
Let’s first understand a little bit about how the NFTs work.

What we’ll create is an NFT collection. Each collection will have many NFTs of that type. 

Here’s the world’s most expensive crypto collection, with each NFT worth millions of dollars. 

[https://opensea.io/collection/cryptopunks](https://opensea.io/collection/cryptopunks)

When we’re creating an NFT collection, we need to maintain the list of valid NFTs and the owner of each of these NFTs.

```

\#\[near_bindgen\]

\#\[derive(BorshDeserialize, BorshSerialize)\]

pub struct NonFungibleTokenBasic {

    pub token_to_account: UnorderedMap,

    pub account_gives_access: UnorderedMap>,

}

```

Firstly, we’ll have a mapping of token_to_account. Instead of String to String that we had previously written. That is because the tokenId of the NFT is always a number and the account id is a Hash. This records who is the owner of which NFT. 

Then the second account_gives_access is a map that stores the accountIds of people who are given access to the NFTs owned by the `owner`. This is usually used when a user wants to manage their NFTs in an organized way or wants to manage them using a smart contract. Remember that an account can also have upto 1 contract stored in it. So when a user gives access to a particular account, they are giving access to the contract inside that account to manage this NFT.

Again, as before, we need to set the constructor for this struct we’ve created

```

\#\[near_bindgen\]

impl Default for NonFungibleTokenBasic {

    fn default() -> Self {

        Self {

            token_to_account: UnorderedMap::new(b"tta".to_vec()),

            account_gives_access: UnorderedMap::new(b"aga".to_vec()),

        }

    }

}

```
## Implementing functions
```

\#\[near_bindgen\]

impl NEP4 for NonFungibleTokenBasic {

}

```

Let’s start implementing the functions one by one!

```

    fn grant_access(&mut self, escrow_account_id: AccountId) {

        let escrow_hash = env::sha256(escrow_account_id.as_bytes());

        let predecessor = env::predecessor_account_id();

        let predecessor_hash = env::sha256(predecessor.as_bytes());

        let mut access_set = match self.account_gives_access.get(&predecessor_hash) {

            Some(existing_set) => {

                existing_set

            },

            None => {

                UnorderedSet::new(b"new-access-set".to_vec())

            }

        };

        access_set.insert(&escrow_hash);

        self.account_gives_access.insert(&predecessor_hash, &access_set);

    }

```

In near the person sending the functioncall is called the predecessor. You can get the caller of the function using the environment variable env::predecessor_account_id(). 

The escrow_account_id is to whom we wish to grant access of this NFT to.

We’ll check if the user already has the access mapping set, if not we will create a new UnorderedSet with the passed escrow_account_id as the only account to which escrow has been granted. If the access mapping already exists, we simply add the new escrow_account_id to that set. 

By using `predecessor_account_id()` instead of using the account id of the owner as a function parameter, we ensure that the `account_id` that we get from this function is actually the owner of the account. NEAR runtime does the verification for us under the hood by checking the transaction’s signature.

```

   fn revoke_access(&mut self, escrow_account_id: AccountId) {

        let predecessor = env::predecessor_account_id();

        let predecessor_hash = env::sha256(predecessor.as_bytes());

        let mut existing_set = match self.account_gives_access.get(&predecessor_hash) {

            Some(existing_set) => existing_set,

            None => env::panic(b"Access does not exist.")

        };

        let escrow_hash = env::sha256(escrow_account_id.as_bytes());

        if existing_set.contains(&escrow_hash) {

            existing_set.remove(&escrow_hash);

            self.account_gives_access.insert(&predecessor_hash, &existing_set);

            env::log(b"Successfully removed access.")

        } else {

            env::panic(b"Did not find access for escrow ID.")

        }

    }

```

Revoke access, simply removes the account_id from the set of accessors. 

```

    fn transfer(&mut self, new_owner_id: AccountId, token_id: TokenId) {

        let token_owner_account_id = self.get_token_owner(token_id);

        let predecessor = env::predecessor_account_id();

        if predecessor != token_owner_account_id {

            env::panic(b"Attempt to call transfer on tokens belonging to another account.")

        }

        self.token_to_account.insert(&token_id, &new_owner_id);

    }

```

While transferring we need to check if the person calling this function is actually the owner of the NFT that they’re trying to transfer. If yes, we’ll update our token_to_account mapping with the new account_id to whom the transfer has been made. 

```

    fn check_access(&self, account_id: AccountId) -> bool {

        let account_hash = env::sha256(account_id.as_bytes());

        let predecessor = env::predecessor_account_id();

        if predecessor == account_id {

            return true;

        }

        match self.account_gives_access.get(&account_hash) {

            Some(access) => {

                let predecessor = env::predecessor_account_id();

                let predecessor_hash = env::sha256(predecessor.as_bytes());

                access.contains(&predecessor_hash)

            },

            None => false

        }

    }

    fn get_token_owner(&self, token_id: TokenId) -> String {

        match self.token_to_account.get(&token_id) {

            Some(owner_id) => owner_id,

            None => env::panic(b"No owner of the token ID specified")

        }

    }

```

Check access and get token owner are read only functions that access the current state of ownership. Nothing new here :)

With this, we’re now compliant with NEP4!
## Making the NFTs mintable
We’ve written the functions on how we can transfer the NFTs from one owner to another, but how do we create the NFTs? How does the first person get the NFT?

This is called minting - bringing into existence a new NFT. 

Let’s extend the NonFungibleTokenBasic a little further

```

\#\[near_bindgen\]

impl NonFungibleTokenBasic {

    pub fn mint_token(&mut self, owner_id: String, token_id: TokenId) {

        let token_check = self.token_to_account.get(&token_id);

        if token_check.is_some() {

            env::panic(b"Token ID already exists.")

        }

        self.token_to_account.insert(&token_id, &owner_id);

    }

}

```

Here we’ll let the users create a new NFT and make it their own as long as it has not been claimed by someone else already. Maybe this will start a race among the users to claim the NFTs with important numbers like 42, 100, 1000 etc!
## Deploy & mint nft
```

$ env 'RUSTFLAGS=-C link-arg=-s'

$ cargo build --target wasm32-unknown-unknown --release

```

```

near dev-deploy --wasmFile target/wasm32-unknown-unknown/release/nearnft.wasm

```

Just like before :)
## What Next?
Create an NFT for yourself and transfer that to a friend using the `near call` commandline tool just like we used it in the previous quest to create NFTs.