# Beginners guide to setting up a Solana validator (Agave)

1. [Introduction](#1-introduction)
2. [Hardware requirements](#2-hardware-requirements)
3. [Install Ubuntu server](#3-install-ubuntu-server)
4. [Create an unprivileged user](#4-create-an-unprivileged-user)
5. [Install and configure Agave](#5-install-and-configure-agave)
6. [Create keypairs](#6-create-keypairs)
7. [Create startup script](#7-create-startup-script)
8. [Create validator service](#8-create-validator-service)
9. [Tune system settings](#9-tune-system-settings)
10. [Set up log rotation](#10-set-up-log-rotation)
11. [Start the validator](#11-start-the-validator)
12. [Monitor the validator](#12-monitor-the-validator)


## 1. Introduction

This tutorial is for Solana beginners. It is a rewritten and updated version of my original [devnet validator tutorial](https://github.com/agjell/sol-tutorials/blob/master/setting-up-a-solana-devnet-validator.md). I've adapted this one for the Agave validator client, and have tested it on a fresh install of Ubuntu 24.04.2 LTS. The process is applicable for mainnet, testnet and devnet, but must be adapted by the user accordingly. I have used **mainnet** as an example throughout. Please post an issue if something is not working as expected!


## 2. Hardware requirements

Abide by the hardware requirements in the [official docs](https://docs.anza.xyz/operations/requirements) for mainnet, testnet and devnet, respectively. You may also have a look at the community maintained [Solana Hardware Compatibility List](https://solanahcl.org/).


## 3. Install Ubuntu server

First you need to [install Ubuntu server](https://documentation.ubuntu.com/server/tutorial/basic-installation/). Remember to check “Install OpenSSH server” during the Ubuntu installation if you want to access your validator remotely. If you forget it during the main installation you can follow [this](https://documentation.ubuntu.com/server/how-to/security/openssh-server/index.html) guide afterwards. You can read about how to connect to the server remotely over SSH [here](https://www.howtogeek.com/311287/how-to-connect-to-an-ssh-server-from-windows-macos-or-linux/).

The first thing I do after logging in to a fresh install is to update it:
```bash
sudo apt update && sudo apt upgrade --assume-yes
```

Then you'll need to partition, format and mount any extra SSDs. If you don't know how, the internet is your friend. There are tons of guides out there. The process usually involve the following tools: [`lsblk`](https://manpages.ubuntu.com/manpages/noble/en/man8/lsblk.8.html),  [`fdisk`](https://manpages.ubuntu.com/manpages/noble/en/man8/fdisk.8.html), [`mkfs`](https://manpages.ubuntu.com/manpages/noble/en/man8/mkfs.8.html), [`blkid`](https://manpages.ubuntu.com/manpages/noble/en/man8/blkid.8.html), [`fstab`](https://manpages.ubuntu.com/manpages/noble/en/man5/fstab.5.html) and [`mount`](https://manpages.ubuntu.com/manpages/noble/en/man8/mount.8.html).


## 4. Create an unprivileged user

During the Ubuntu installation you created a user with root privileges; a user that can perform `sudo` commands. This user is needed to administer the system. However, the Agave validator client does not require root privileges, and it is considered poor practice to run Agave as “root" or as another privileged user. You should therefore create an unprivileged user to run the validator client. For example the user "sol":
```bash
sudo adduser sol
```

Then you can switch to the new user account by running:
```bash
sudo su - sol
```

To get back to the admin user account you simply run:
```bash
exit
```

Using the `su` command makes it easy to switch between users without having to log out and back in. You can also open two separate terminal windows, each logged in as separate users. I have highlighted the applicable user account for each chapter below.


## 5. Install and configure Agave

```diff
! Perform as user “sol”
```

You can either install the prebuilt binaries or build your own binaries. In this tutorial we're installing the prebuilt binaries, as this is the easiest way to get going. If you prefer building your own binaries from source you can check out my tutorial [here](https://github.com/agjell/sol-tutorials/blob/master/building-solana-from-source.md).

You should first check the officially recommended version in Discord for either [mainnet](https://discord.com/channels/428295358100013066/669406841830244375), [testnet](https://discord.com/channels/428295358100013066/594138785558691840) or [devnet](https://discord.com/channels/428295358100013066/749059399875690557). Replace the version number in the command below with the recommendation from Discord. You can also find the most recent installation command in the [Agave docs](https://docs.anza.xyz/cli/install), but the version provided there is usually **not recommended** for mainnet.
```bash
sh -c "$(curl -sSfL https://release.anza.xyz/v2.1.16/install)"
```

After the installation is complete, close and reopen the terminal or log out and in again (as “sol”). This is done to load the environment variable that was added to `~/.profile` during the installation, which enables "sol" to run the `solana` and `agave` commands from any directory.

Then configure the Agave client to target a cluster. You can specify "testnet" or "devnet" as well.
```bash
solana config set --url mainnet-beta
```

Verify that the cluster is reachable by running a command. For example:
```bash
solana gossip
```
It should return a list of validators.

You can get information about any command by running `command --help` or `command subcommand --help`. For example
```bash
solana --help
```
```bash
solana config --help
```

## 6. Create keypairs

```diff
! Perform as user “sol”
```

To run a validator you need three keypairs. These represent accounts that the validator use to interact with Solana. All accounts have a public key associated with them, which identifies the account in the system. The keypairs needed are

- a wallet keypair, which
  - is used to transfer voting funds to the validator
  - is used to withdraw rewards from the vote account
- a validator identity keypair, which
  - represent your validators identity
  - pays voting fees
  - signs transactions
- a voting account keypair, which
  - is used for voting by the validator
  - receives validator rewards

When you run the command to create a keypair, you are prompted to enter a passphrase. The client then returns a pubkey (public key) and a seed phrase in the terminal. You **must** store both the passphrase and the seed phrase securely, as these are required to recreate the keypair in the future. Do not give your passphrase, seed phrase or keypair to anyone!

### Wallet

First create the wallet keypair:
```bash
solana-keygen new --outfile ~/wallet-keypair.json
```

### Validator identity

Second, create the validator identity keypair:
```bash
solana-keygen new --outfile ~/validator-keypair.json
```

Which can also be appended to the client configuration:
```bash
solana config set --keypair ~/validator-keypair.json
```

Because the validator pays the voting fees, you will need to transfer some SOL to its identity account (pubkey). There are many ways to do this. This is an example of how to transfer 1 SOL from the wallet to the validator, where the the transaction fee is deducted from the wallet:
```bash
solana transfer --allow-unfunded-recipient \
  --fee-payer ~/wallet-keypair.json \
  --from ~/wallet-keypair.json ~/validator-keypair.json 1
```

You can check the account balance by running:
```bash
solana balance ~/validator-keypair.json
```

### Vote account

Third, create the vote account keypair:
```bash
solana-keygen new --outfile ~/vote-account-keypair.json
```

Then you need to register the account as a vote account on the blockchain, and set an authorized withdrawer. In this example the transaction fee is deducted from the validator keypair, and the wallet is set as the authorized withdrawer:
```bash
solana create-vote-account \
  --fee-payer ~/validator-keypair.json \
  ~/vote-account-keypair.json ~/validator-keypair.json ~/wallet-keypair.json
```
That's it for keypairs!

## 7. Create startup script

Next up is to create a shell script that contains all the flags and options needed for your validator. The script will be executed by a system service, which you're going to set up later.

```diff
! Perform as user “sol”
```

Create the `start-validator.sh` script inside the home directory (`~/`) of “sol”:
```bash
tee ~/start-validator.sh > /dev/null <<EOT
#!/bin/bash
exec agave-validator \\
  --identity ~/validator-keypair.json \\
  --vote-account ~/vote-account-keypair.json \\
  --entrypoint entrypoint.mainnet-beta.solana.com:8001 \\
  --entrypoint entrypoint2.mainnet-beta.solana.com:8001 \\
  --entrypoint entrypoint3.mainnet-beta.solana.com:8001 \\
  --entrypoint entrypoint4.mainnet-beta.solana.com:8001 \\
  --entrypoint entrypoint5.mainnet-beta.solana.com:8001 \\
  --known-validator 7Np41oeYqPefeNQEHSv1UDhYrehxin3NStELsSKCT4K2 \\
  --known-validator GdnSyH3YtwcxFvQrVVJMm1JhTS4QVX7MFsX56uJLUfiZ \\
  --known-validator DE1bawNcRJB9rVm3buyMVfr8mBEoyyu73NBovf2oXJsJ \\
  --known-validator CakcnaRDHka2gXyfbEd2d3xsvkJkqsLw2akB3zsN1D2S \\
  --expected-genesis-hash 5eykt4UsFv8P8NJdTREpY1vzqKqZKvdpKuc147dw2N9d \\
  --only-known-rpc \\
  --rpc-port 8899 \\
  --private-rpc \\
  --dynamic-port-range 8000-8020 \\
  --wal-recovery-mode skip_any_corrupted_record \\
  --ledger /mnt/ledger \\
  --accounts /mnt/accounts \\
  --snapshots /mnt/snapshots \\
  --log ~/log/validator.log \\
  --limit-ledger-size
EOT
```

**Note 1:** The script is targeted to mainnet validators. I have listed the relevant flags for [testnet](#testnet-values-official-docs) and [devnet](#devnet-values-official-docs) at the **bottom of this chapter** for convenience.

**Note 2:** "ledger", "accounts" and "snapshots" are all pointing to separate directories. The assumption is that every directory under `/mnt` has a dedicated SSD mounted to it.

You need to make the script executable, or else it will not launch:
```bash
chmod +x ~/start-validator.sh
```

Adapt the script to your needs with any text editor. For example with `nano`:
```bash
nano ~/start-validator.sh
```

You can study all the validator options by running `agave-validator --help`. For example, to store the minimum amount of ledger you have to set `--limit-ledger-size 50000000`. If you are starting an RPC, you should add the `--no-voting` flag. Press **Ctrl+S** to save the file and **Ctrl+X** to exit if you are using Nano.




Note that the script points to a “log” directory inside the home directory. Let’s create it:
```bash
mkdir ~/log
```

For convenience we can also create symbolic links to the "ledger", "accounts" and "snapshots" directories inside `~/`:
```bash
ln --symbolic /mnt/ledger ~/ledger
```
```bash
ln --symbolic /mnt/accounts ~/accounts
```
```bash
ln --symbolic /mnt/snapshots ~/snapshots
```

### Testnet values ([official docs](https://docs.anza.xyz/clusters/available#testnet))
```
  --entrypoint entrypoint.testnet.solana.com:8001 \
  --entrypoint entrypoint2.testnet.solana.com:8001 \
  --entrypoint entrypoint3.testnet.solana.com:8001 \
  --known-validator 5D1fNXzvv5NjV1ysLjirC4WY92RNsVH18vjmcszZd8on \
  --known-validator dDzy5SR3AXdYWVqbDEkVFdvSPCtS9ihF5kJkHCtXoFs \
  --known-validator Ft5fbkqNa76vnsjYNwjDZUXoTWpP7VYm3mtsaQckQADN \
  --known-validator eoKpUABi59aT4rR9HGS3LcMecfut9x7zJyodWWP43YQ \
  --known-validator 9QxCLckBiJc783jnMvXZubK4wH86Eqqvashtrwvcsgkv \
  --expected-genesis-hash 4uhcVJyU9pJkvQyS88uRDiswHXSCkY3zQawwpjk2NsNY \
```

### Devnet values ([official docs](https://docs.anza.xyz/clusters/available#devnet))
```
  --entrypoint entrypoint.devnet.solana.com:8001 \
  --entrypoint entrypoint2.devnet.solana.com:8001 \
  --entrypoint entrypoint3.devnet.solana.com:8001 \
  --entrypoint entrypoint4.devnet.solana.com:8001 \
  --entrypoint entrypoint5.devnet.solana.com:8001 \
  --known-validator dv1ZAGvdsz5hHLwWXsVnM94hWf1pjbKVau1QVkaMJ92 \
  --known-validator dv2eQHeP4RFrJZ6UeiZWoc3XTtmtZCUKxxCApCDcRNV \
  --known-validator dv3qDFk1DTF36Z62bNvrCXe9sKATA6xvVy6A798xxAS \
  --known-validator dv4ACNkpYPcE3aKmYDqZm9G5EB3J4MRoeE7WNDRBVJB \
  --expected-genesis-hash EtWTRABZaYq6iMfeYKouRu166VU2xqa1wcaWoxPkrZBG \
```

## 8. Create validator service

```diff
! Perform as user with root privileges
```

Running the validator as a system service makes it easy to start it automatically at boot. The system can also restart services automatically if they crash, as the service manager monitors all services continuously. Create the `validator.service` file like this:
```bash
sudo tee /etc/systemd/system/validator.service > /dev/null <<EOT
[Unit]
Description=Solana Validator
After=network.target network-online.target
Wants=network-online.target
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
LimitNOFILE=2000000
LogRateLimitIntervalSec=0
User=sol
Environment=SOLANA_METRICS_CONFIG=host=https://metrics.solana.com:8086,db=mainnet-beta,u=mainnet-beta_write,p=password
ExecStart=/home/sol/start-validator.sh

[Install]
WantedBy=multi-user.target
EOT
```

**Note:** Remember to change the value for `User` and the path in `ExecStart` if you created a user with a different name.

Then reload the service manager to make the system aware of the new service:
```bash
sudo systemctl daemon-reload
```
The validator service is now almost ready to run.

## 9. Tune system settings

```diff
! Perform as user with root privileges
```

To ensure that the validator service is able to operate smoothly, you should apply some basic tuning before starting it. Copy and paste these into the terminal, and press **Enter**:
```bash
sudo tee /etc/security/limits.d/90-solana-nofiles.conf > /dev/null <<EOT
# Increase process file descriptor count limit
#<domain> <type> <item> <value>
* - nofile 2000000
EOT
```

```bash
sudo tee /etc/sysctl.d/20-solana.conf > /dev/null <<EOT
# Minimize swapping
vm.swappiness=0

# Increase UDP buffer size
net.core.rmem_max=134217728
net.core.rmem_default=134217728
net.core.wmem_max=134217728
net.core.wmem_default=134217728

# Increase memory mapped files limit
vm.max_map_count=2000000

# Maximize number of file-handles a process can allocate
fs.nr_open=2147483584
EOT
```

Then load the new settings.
```bash
sudo sysctl --system
```

Just one more step before take-off.

## 10. Set up log rotation

```diff
! Perform as user with root privileges
```

`agave-validator` produces **big** logs. To make the logs easier to handle it's best to rotate them every day. “Logrotate” takes care of the log rotation for us. It automatically creates a new log at 00:00 and deletes the excess. This configuration will retain 7 days worth of logs:
```bash
sudo tee /etc/logrotate.d/solana > /dev/null <<EOT
/home/sol/log/validator.log {
  su sol sol
  daily
  rotate 7
  missingok
  postrotate
    systemctl kill -s USR1 validator.service
  endscript
}
EOT
```

Restart the logrotate service to load the new configuration:
```bash
sudo systemctl restart logrotate
```
Log rotation is now active.

## 11. Start the validator

```diff
! Perform as user with root privileges
```

After completing all the steps above I usually reboot (`sudo reboot`), although I suppose it’s not really necessary. It's still nice to verify that all SSDs are mounted correctly on boot. When you're ready you can start the service:
```bash
sudo systemctl enable --now validator.service
```
Then check if it started successfully and is running:
```bash
sudo systemctl status validator.service
```
It should say “active (running)”.

## 12. Monitor the validator

```diff
! Perform as user “sol”
```

After starting the validator service I switch to "sol" again and start monitoring the start-up:
```bash
agave-validator --ledger ~/ledger monitor
```
The monitoring tool tells us what the validator is doing. It can take a very long time to start up the validator for the first time. The validator does roughly this:
1. Looks for an available RPC node (can take a long time)
1. Downloads snapshot (~90 GB for mainnet)
1. Loads ledger state from snapshot
1. Catches up to the cluster

It's not uncommon for this to take at least half an hour on mainnet. When everything is up and running the reply from the monitor command should look similar to this:
```
$ agave-validator --ledger ~/ledger monitor
Ledger location: /home/sol/ledger
Identity: NordEHiwa6wT5TCjdeWJzpsA7DSmWQPqfSS7m2b6cv3
Genesis Hash: 5eykt4UsFv8P8NJdTREpY1vzqKqZKvdpKuc147dw2N9d
Version: 2.1.16
Shred Version: 50093
Gossip Address: 123.456.789.10:8000
TPU Address: 64.130.53.58:8006
⠲ 00:20:12 | Processed Slot: 329829774 | Confirmed Slot: 329829774 | Finalized Slot: 329829742 | Full Snapshot Slot: 329827302 | Incremental Snapshot Slot: 329829614 | Transactions: 388451705620 | ◎0.707313645
```

You can then run the command below to see if the validator has caught up to the cluster. If the validator is not caught up, it will display the progress. The command will fail if the validator is still loading the ledger from snapshot.
```bash
solana catchup ~/validator-keypair.json --our-localhost
```

If you need to troubleshoot you can look for warnings and errors in the log:
```bash
grep --extended-regexp 'ERROR|WARN' ~/log/validator.log
```
Fell free to post an issue or give me feedback through Discord. All constructive feedback is welcome!
