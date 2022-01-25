# Making a fully featured NFT on NEAR in Rust
In the previous Quest, we looked at writing a dummy NFT on NEAR using Rust. If you’ve not done that yet, No worries, no coding is involved in this quest. But you have to do that quest if you like to follow our Rust coding quests. 

To create an NFT that qualifies as an NFT on NEAR, we’ll have to stick to some standards that have been defined by the community. If you have come from some Ethereum background, the Ethereum community calls this standard an ERC721. Similarly, the NEAR community has defined what is called NEP171. NEP-171 basically tells the developer what skeleton their contract should follow in order to be able to create an NFT (see https://nomicon.io/Standards/NonFungibleToken/Core.html). If your NFT follows this specification it will be possible to list this NFT on various NFT marketplaces. The NFT marketplaces require all the NFTs listed to have some functions that they can call to facilitate buying and selling of NFTs.

In this quest, we are going to show how to launch your own NFT collection with no coding. Just running stuff in the terminal to understand the process. Just the process, details are coming later.

## The setup:
We will use the code proposed by the NEAR team themselves. Again, it is always better to stick to standards. So, go to your terminal and run this:

``` git clone https://github.com/near-examples/nft-tutorial.git ```

You can see that there are already some instructions there. We are going to do pretty much the same, but with more illustration. Run these commands:

``` cd nft-tutorial ```

``` git switch 6.royalty ```

_WHY?_:
Because this repo was designed for a multistep tutorial by NEAR. In each step, a learner adds a chunk of code, so we switch to this branch to benefit from the full code.
Now, this:

``` rustup target add wasm32-unknown-unknown ```

_WHY?_:
We are going to use WebAssembly for building, so make sure it is included in your Rust toolchain.

``` yarn build ```

_WHY?_:
This is the build command in package.json:
"cd nft-contract && ./build.sh && cd ../..".

This runs build.sh in the directory nft-contract, remember this stuff from the previous quest?

Before moving on to the next subquest, log in to continue.

``` near login ```.

## Initializing the collection:
Now, we need to create a subaccount of our main account. This will be our token account. Let's set those account IDs to variables, we are going to use them a lot.

``` MAIN_ACCOUNT=your-account.testnet ```

``` NFT_CONTRACT_ID=nft-example.your-account.testnet ```

Replace the accounts above with your token account ID and your NEAR testnet account.

And we create a new account:

``` near create-account $NFT_CONTRACT_ID --masterAccount $MAIN_ACCOUNT --initialBalance 10 ```

Now let's deploy:

``` near deploy --accountId $NFT_CONTRACT_ID --wasmFile out/main.wasm ```

_WHAT?_:
Remember that command in build.sh? The result of compilation is main.wasm. This is what gets deployed.

Now we claim ownership:
``` near call $NFT_CONTRACT_ID new_default_meta '{"owner_id": "'$NFT_CONTRACT_ID'"}' --accountId $NFT_CONTRACT_ID ```

_HOW?_:
If you go to src/lib.rs and search for new_default_meta, you find a function by that name. It is not hard to realize what it is doing, right? It sets some necessary metadata. How about you change the name and symbol fields? How would you like to call your NFT collection? 

Now make sure the data is recorded:

``` near view $NFT_CONTRACT_ID nft_metadata ```

## Minting and transferring:
Ok, so how exactly to mint a new token?
You have to run this:

``` near call $NFT_CONTRACT_ID nft_mint '{"token_id": "some id", "metadata": {"title": "some title", "description": "some description", "media": "a link to a media file"}, "receiver_id": "'$MAIN_ACCOUNT'"}' --accountId $MAIN_ACCOUNT --amount 0.1 ```

You can populate the fields above the way you like. Do you see that --amount? You have to pay for storage on NEAR. in the media field, here you put a link for your token's media.

After that, you can make sure that all is recorded:

``` near view $NFT_CONTRACT_ID nft_token '{"token_id": "some id"}' ```

Check that nft_token function in nft_core.rs. Now go to your wallet -> collectibles tab, can you see that? Your token is there!

Now, suppose you want to transfer your precious NFT to someone. You first run this:

``` MAIN_ACCOUNT_2=your-second-account.testnet ```

 You run this command:

``` near call $NFT_CONTRACT_ID nft_transfer '{"receiver_id": "'$MAIN_ACCOUNT_2'", "token_id": "some id", "approval_id" :0, "memo": "some memo"}' --accountId $MAIN_ACCOUNT --depositYocto 1 ```

This function is payable if you look at it. There is a line that requires sending one Yocto along with the call. You will be refunded if that Yocto is more than enough to pay for storage. The last command is more or less clear except for that approval_id. Don't worry about it for now, a topic for another day.

## Done!:
Alright, now you know basically how to launch NEAR NFTs, isn't that cool? Stay tuned for more cool quests to come. Good luck and happy coding!
