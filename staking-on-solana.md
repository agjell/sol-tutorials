# Staking on Solana

## Introduction

A staker is someone that has SOL and wants to put them to work to collect rewards. They can do this by staking SOL on a validator. A validator, on the other hand, needs SOL staked on it to earn rewards for itself (or its' owner, to be precise). More stake equals more rewards, because more stake means a validator is chosen to write new transactions to the ledger more often. This means that stakers and validators can benefit from each other. You can read more about staking in the [docs](https://docs.solana.com/staking).

The process of staking involves creating a stake account, transferring SOL to it and delegating it to a validator. SOL placed in a stake account is simply referred to as "stake".

## Create wallet (optional)

This step is optional, as you may already have a wallet you can use. For the sake of demonstration I will create one:
```bash
solana-keygen new -o ~/staker-wallet-keypair.json
```

I save the password and seed phrase somewhere securely. Then I airdrop 1 SOL into it (only possible on devnet and testnet), that I will stake on a validator later:
```bash
solana airdrop 1 ~/staker-wallet-keypair.json
```

Solana replies with the balance, but we can verify the balance manually like this:
```bash
solana balance ~/staker-wallet-keypair.json
```

## Create stake account

Now for the stake account. We don't need to set passwords for stake accounts, as they are fully controlled by their assigned "withdraw authority" and "stake authority". These can be the same or different keys, depending on your preference. I start out by creating a regular system account:
```bash
solana-keygen new --no-passphrase -o ~/stake-account-keypair.json
```

Solana replies with the public key of the account. I copy and save it somewhere safe. Then I create a stake account from the keypair I just made, and transfer 0.5 SOL to it from my wallet. I also assign my wallet as the withdraw authority and the stake authority of the account:
```bash
solana create-stake-account \
  --fee-payer ~/staker-wallet-keypair.json \
  --stake-authority ~/staker-wallet-keypair.json \
  --withdraw-authority ~/staker-wallet-keypair.json \
  --from ~/staker-wallet-keypair.json \
  ~/stake-account-keypair.json 0.5
```

If the transaction succeeds I can delete the stake account keypair, provided I've taken note of the stake account public key. If I haven't got the public key I must retrieve it and save it before I delete the keypair:
```bash
solana address -k ~/stake-account-keypair.json
```
```bash
rm ~/stake-account-keypair.json
```
Note that if you lose access to the withdraw authority account, you also lose control of the stake account and its' contents. It is very important to manage your keys securely!

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
There is a "warmup"-period for activating stake. It's not clear how long that period is (cf. [docs](https://docs.solana.com/implemented-proposals/staking-rewards#stake-warmup-cooldown-withdrawal)), but in my limited experience it takes a few days.

## Deactivate stake

At some point I may want to deactivate the stake. Either because I want to delegate it to another validator, or because I want to withdraw the funds to my wallet. I do it by running:
```bash
solana deactivate-stake \
  --fee-payer ~/staker-wallet-keypair.json \
  --stake-authority ~/staker-wallet-keypair.json \
  <STAKE_ACCOUNT_PUBKEY>
```
Just like there is a "warmup" period to activate stake, there is a "cooldown" period to deactivate stake. That means I'll have to wait to redelegate or withdraw my stake until the funds are released.

## Withdraw stake

When the cooldown period is over I can withdraw my stake. First I'll check the balance of the stake account:
```bash
solana balance <STAKE_ACCOUNT_PUBKEY>
```

Then I withdraw 0.5 SOL to my wallet (you can also use 'ALL' to withdraw everything):
```bash
solana withdraw-stake \
  --fee-payer ~/staker-wallet-keypair.json \
  --withdraw-authority ~/staker-wallet-keypair.json \
  <STAKE_ACCOUNT_PUBKEY> ~/staker-wallet-keypair.json 0.5
```

## Merging stake accounts

We can merge two stake accounts on certain conditions (see [here](https://docs.solana.com/staking/stake-accounts#merging-stake-accounts)). If we're merging stake accounts that has activated stake (live accounts) they both need to

 - have the same stake authority
 - be delegated to the same vote account
 - have identical vote credits (being actively staked on the same validator for at least one epoch)

If the conditions are met I can merge my stake accounts by running:
```bash
solana merge-stake \
  --fee-payer ~/staker-wallet-keypair.json \
  --stake-authority ~/staker-wallet-keypair.json \
  <TO_STAKE_ACCOUNT_PUBKEY> <FROM_STAKE_ACCOUNT_PUBKEY>
```


## More staking resources

There is much more to staking than what I have shown above. The following Solana docs contains a lot of useful information:

 - [Staking on Solana](https://docs.solana.com/staking)
 - [Delegate stake](https://docs.solana.com/cli/delegate-stake)
 - [Manage stake accounts](https://docs.solana.com/cli/manage-stake-accounts)
 - [Validator stake](https://docs.solana.com/running-validator/validator-stake)
 - [Staking rewards](https://docs.solana.com/implemented-proposals/staking-rewards)
