# Staking on Solana

 1. [Introduction](#introduction)
 2. [Create wallet (optional)](#create-wallet-optional)
 3. [Create stake account](#create-stake-account)
 4. [Delegate stake](#delegate-stake)
 5. [Deactivate stake](#deactivate-stake)
 6. [Withdraw stake](#withdraw-stake)
 7. [Merge stake accounts](#merge-stake-accounts)
 8. [More staking resources](#more-staking-resources)


## Introduction

A staker is someone that has SOL and wants to put them to work to collect rewards. They can do this by staking SOL on a validator. A validator, on the other hand, needs SOL staked on it to earn rewards for itself (or its' owner, to be precise). More stake equals more rewards, because more stake means a validator is chosen to write new transactions to the ledger more often. This means that stakers and validators can benefit from each other. You can read more about staking in the [docs](https://docs.solana.com/staking).

This guide is only intended for people operating in Solana devnet or testnet. Although the procedure will be somewhat similar for mainnet, my experience and knowledge of mainnet operations is too limited to say.

The process of staking involves creating a stake account, transferring SOL to it and delegating it to a validator. SOL placed in a stake account is simply referred to as "stake".


## Create wallet (optional)

This step is optional, as you may already have a wallet you can use. For the sake of demonstration I will create one:
```bash
solana-keygen new -o ~/staker-wallet-keypair.json
```

I save the password and seed phrase somewhere securely. Then I airdrop 1 SOL into it, that I will stake on a validator later:
```bash
solana airdrop 1 ~/staker-wallet-keypair.json
```

Solana replies with the balance, but we can verify the balance manually like this:
```bash
solana balance ~/staker-wallet-keypair.json
```


## Create stake account

Now for the stake account. We don't need to set passwords for stake accounts, as they are fully controlled by their assigned "withdraw"- and "stake authorities". I start out by creating a regular system account:
```bash
solana-keygen new --no-passphrase -o ~/stake-account-keypair.json
```

Solana replies with the public key of the account. I copy it and save it somewhere safe. Then I use the keypair I just made as a base to create the stake account, and transfer 0.5 SOL to it from my wallet. I also assign my wallet as the withdraw authority and the stake authority of the account:
```bash
solana create-stake-account \
  --fee-payer ~/staker-wallet-keypair.json \
  --stake-authority ~/staker-wallet-keypair.json \
  --withdraw-authority ~/staker-wallet-keypair.json \
  --from ~/staker-wallet-keypair.json \
  ~/stake-account-keypair.json 0.5
```

If the transaction succeeds I may delete the keypair file, provided I've taken note of the stake account public key. If I haven't got the public key I must retrieve it and save it before I delete the keypair:
```bash
solana address -k ~/stake-account-keypair.json
```
```bash
rm ~/stake-account-keypair.json
```
Note that if you loose access to the withdraw authority account, you also loose control of the stake account and its' contents!


## Delegate stake

Lastly I need to delegate the stake. This is done to the vote account of a validator, *not* the validator identity. I do it like this:
```bash
solana delegate-stake \
  --fee-payer ~/staker-wallet-keypair.json \
  --stake-authority ~/staker-wallet-keypair.json \
  <STAKE_ACCOUNT_PUBKEY> <VOTE_ACCOUNT_PUBKEY>
```

We can view information about the stake account by running:
```bash
solana stake-account <STAKE_ACCOUNT_PUBKEY>
```
There is a "warmup"-period for activating stake. It's not clear how long that period is (cf. [docs](https://docs.solana.com/implemented-proposals/staking-rewards#stake-warmup-cooldown-withdrawal)), but in my limited experience it has been a few days or so.


## Deactivate stake

At some point I may want to deactivate the stake. Either because I want to delegate it to another validator, or because I want to withdraw it back to my wallet. I do it by running:
```bash
solana deactivate-stake \
  --fee-payer ~/staker-wallet-keypair.json \
  --stake-authority ~/staker-wallet-keypair.json \
  <STAKE_ACCOUNT_PUBKEY>
```
Just like there is a warmup period to activate stake, there is a "cooldown" period to deactivate stake. That means I'll have to wait to redelegate or withdraw my stake until they are released.


## Withdraw stake

When the cooldown period is over I can withdraw my stake. First I'll check the balance of the stake account:
```bash
solana balance <STAKE_ACCOUNT_PUBKEY>
```

Then I'll withdraw 0.5 SOL back to my wallet:
```bash
solana withdraw-stake \
  --fee-payer ~/staker-wallet-keypair.json \
  --withdraw-authority ~/staker-wallet-keypair.json \
  <STAKE_ACCOUNT_PUBKEY> ~/staker-wallet-keypair.json 0.5
```


## Merge stake accounts

We can merge two stake accounts on certain conditions (see [here](https://docs.solana.com/staking/stake-accounts#merging-stake-accounts)). To merge stake accounts that have active stake (live accounts) they both need to

 - have the same stake authority
 - be associated with the same vote account
 - have identical vote credits (being active on the same vote account for at least one epoch)

If the conditions are met I can merge two stake accounts by running:
```bash
solana merge-stake \
  --fee-payer ~/staker-wallet-keypair.json \
  --stake-authority ~/staker-wallet-keypair.json \
  <MERGE_TO_STAKE_ACCOUNT_PUBKEY> <MERGE_FROM_STAKE_ACCOUNT_PUBKEY>
```


## More staking resources

There is more to staking than what I have shown above. The following Solana docs contains a lot of useful information:

 - [Staking on Solana](https://docs.solana.com/staking)
 - [Delegate stake](https://docs.solana.com/cli/delegate-stake)
 - [Manage stake accounts](https://docs.solana.com/cli/manage-stake-accounts)
 - [Validator stake](https://docs.solana.com/running-validator/validator-stake)
 - [Staking rewards](https://docs.solana.com/implemented-proposals/staking-rewards)
