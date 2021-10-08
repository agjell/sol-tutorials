# Setting up a Solana devnet validator

 1. [Introduction](#introduction)
 2. [Hardware requirements](#hardware-requirements)
 3. [Install Ubuntu server](#install-ubuntu-server)
 4. [Create RAM disk and swap spillover (optional)](#create-ram-disk-and-swap-spillover-optional)
 5. [Install Solana](#install-solana)
 6. [Configure Solana](#configure-solana)
 7. [Create startup script and system services](#create-startup-script-and-system-services)
 8. [Set up log rotation (optional)](#set-up-log-rotation-optional)
 9. [Starting the validator](#starting-the-validator)
 10. [Useful commands](#useful-commands)


## Introduction

Setting up a devnet validator was the first thing I did to become familiar with Solana. I started by following the [official documentation](https://docs.solana.com/running-validator), and asked people in the [Solana Discord](https://discordapp.com/invite/pquxPsq) when I stubled into issues. The official docs were a bit outdated when I set up for the first time. This is to be expected in a rapidly developing project, as code maintenance is usually prioritized higher than documentation. To simplify the setup process for myself later, I decided to make this detailed step-by-step tutorial. Hopefully it can be useful to others as well.

If you have general questions about running a validator, you may find answers to them in [this FAQ](https://github.com/agjell/sol-tutorials/blob/master/solana-validator-faq.md).


## Hardware requirements

Running a devnet validator does not require a high end system. This makes devnet an ideal playground to experiment and get to know Solana, before considering a mainnet setup. Hardware requirements for mainnet are _a lot_ higher, and can be found [here](https://docs.solana.com/running-validator/validator-reqs). My devnet validator is running on an old computer with a quad core CPU, 16 GB RAM and a 256 GB SATA SSD.

You _will_ need an SSD, as a mechanical drive will not be able to handle the amount of IOPS (input/output operations per second) that Solana demands. SATA SSDs are only usable in devnet. Running testnet and mainnet validators requires NVMe SSDs with high speed, high IOPS and high endurance ratings.

If you have a very old CPU you may need to build Solana from source. Be aware of this if the prebuilt binaries crash at launch. The prebuilt binaries should run in most modern systems without problems, though.


## Install Ubuntu server

First we need to [install Ubuntu server](https://ubuntu.com/tutorials/install-ubuntu-server). The installation can be downloaded from [their website](https://ubuntu.com/download/server). You can use [Rufus](https://rufus.ie/) to create a bootable USB drive for installing Ubuntu (tutorial [here](https://ubuntu.com/tutorials/create-a-usb-stick-on-windows)).

Remember to tick off on “Install OpenSSH server” during the Ubuntu installation process if you want to access your validator remotely. If you forget it during the main installation you can follow [this](https://ubuntu.com/server/docs/service-openssh) guide afterwards. You can read about how to connect to the server remotely over SSH [here](https://www.howtogeek.com/311287/how-to-connect-to-an-ssh-server-from-windows-macos-or-linux/).

The first thing I do after logging into a fresh installation is to install updates:
```bash
sudo apt update && sudo apt upgrade --assume-yes
```

Solana does not require root privileges, and It’s considered poor practice to run Solana as “root". It’s also considered good to practice the principle of least privilege. This basically means that no user, process or application should ever have higher privileges than they need to fulfill their intended purpose. I therefore create a new user, “sol", which will be running the validator service as a regular user:
```bash
sudo adduser sol
```

After creating the new user, I use that account as my “base”. By that I mean that I always log in as “sol” initially. If I need to run a command that require root privileges, I switch to an account with those privileges by running `su - username` from the console. Replace “username” with whatever username you assigned during the Ubuntu installation. To get back to the “sol” account I run `exit` in the console. This makes it easy to switch between users without having to log out and back in. Some of the instructions below require root privileges, but I have written where that applies.


## Create RAM disk and swap spillover (optional)

As I mentioned above, Solana require a high amount of IOPS. This is especially true for processing accounts data. To reduce wear on my SSD I prefer storing accounts in a [RAM disk](https://en.wikipedia.org/wiki/RAM_drive). Especially since I’m already using an old and worn SSD, which may fail at any time.

The steps in this section are optional, and only concern storage for the accounts data (contrary to the ledger data, also known as “rocksdb”). If you would rather store accounts on an SSD, skip to “Install Solana” further below.

### Create RAM disk

```diff
! Perform as user with root privileges
```

The RAM disk will act as the primary storage for accounts. To create it, I first have to create a [mount point](https://help.ubuntu.com/community/Mount):
```bash
sudo mkdir /mnt/ramdisk
```

Second, I append the RAM disk configuration in [fstab](https://help.ubuntu.com/community/Fstab) to make it permanent (i.e. appear on system boot). Access is limited to the user “sol”, and size set to 16 GB, as this is currently more than enough for devnet accounts data:
```bash
echo 'tmpfs /mnt/ramdisk tmpfs rw,noexec,nodev,nosuid,noatime,size=16G,user=sol 0 0' | \
  sudo tee --append /etc/fstab > /dev/null
```

Then I reload fstab and mount all entries:
```bash
sudo mount --all --verbose
```

### Create a swap spillover file

```diff
! Perform as user with root privileges
```

In some cases my system may need more RAM than I have available. For those cases I need a [swap](https://help.ubuntu.com/community/SwapFaq) file that can act as reserve. If RAM utilization expands beyond my physical RAM, the excess data will spill over into swap and ensure the validator can continue running.

Any spillover into swap should be temporary, so I usually monitor if the RAM disk and swap space starts filling up. If that happens there is likely something wrong.

Ubuntu server creates a swap file by default, but I want a bigger one to ensure I have enough space. So I delete the existing one and create a new one. To do this, I first fetch a list of the current swap files in the system:
```bash
sudo swapon --show
```

My system replies:
```
NAME      TYPE SIZE USED PRIO
/swap.img file   4G   0B   -2
```

I can see there's a 4 GB swap file called `/swap.img`. Let's disable it:
```bash
sudo swapoff /swap.img
```

And delete the leftover file:
```bash
sudo rm /swap.img
```

I also need to delete delete the reference to `swap.img` from fstab:
```bash
sudo sed --in-place '/swap.img/d' /etc/fstab
```

Then I create a new swap file that will hold 16 GB:
```bash
sudo dd if=/dev/zero of=/swapfile bs=1M count=16K
```

I need to set permissions for it to ensure access is limited to “root”:
```bash
sudo chmod 600 /swapfile
```

And configure the file as a swap area:
```bash
sudo mkswap /swapfile
```

For the swap area to appear on every system boot I need to add it to fstab:
```bash
echo '/swapfile none swap sw 0 0' | sudo tee --append /etc/fstab > /dev/null
```

To ensure the system is using RAM as much as possible, and swap as little as possible, I decrease “[swappiness](https://help.ubuntu.com/community/SwapFaq#What_is_swappiness_and_how_do_I_change_it.3F)” to 1 and reload `sysctl`. This is probably not needed in a modern system. I only do this to protect my worn SSD.
```bash
echo 'vm.swappiness=1' | sudo tee --append /etc/sysctl.conf > /dev/null
```
```bash
sudo sysctl --load
```

Finally I enable all swaps from fstab by running:
```bash
sudo swapon --all --verbose
```
I have now created the RAM disk and swap file, and can move to the installation phase.

## Install Solana

As I mentioned above, you can either install the prebuilt binaries or build your own binaries. I have only included instructions to install the prebuilt binaries in this tutorial, as this will work for most people. However, I have also made a tutorial on building binaries from source, which you can find [here](https://github.com/agjell/sol-tutorials/blob/master/building-solana-from-source.md).

### Install prebuilt binaries

```diff
! Perform as user “sol”
```

This is by far the easiest way to install Solana. Check for the most recent version on [GitHub](https://github.com/solana-labs/solana/releases/latest), and replace the version number in the command below. You can also copy the most recent installation command from the [Solana docs](https://docs.solana.com/cli/install-solana-cli-tools#use-solanas-install-tool).
```bash
sh -c "$(curl -sSfL https://release.solana.com/v1.8.0/install)"
```

After the installation is complete I close and reopen the terminal, or log out and in again (as “sol”). I do this to enable the environment variables that were added to `~/.profile` during the installation. These tell the system to look for binaries in the Solana installation directory, so I can run the Solana commands from any directory in the system.

## Configure Solana

In this section I’ll connect the validator to the Solana network and set up the necessary accounts.

### Joining devnet

First I configure Solana to connect to devnet:
```bash
solana config set --url devnet
```

Then I verify that the cluster is reachable by running a random command. For example:
```bash
solana gossip
```
It should reply with a list of validators.

### Setting up accounts

The account structure can be a bit confusing, but this is how I understand it (I may be wrong). There are four account types: system accounts, voting accounts, staking accounts and nonce accounts. Each account type has different properties, that I won’t go into here (mostly because I have no clue). The system account is the default type. All accounts have a public key associated with them, which identifies them in the system. For a new devnet validator I usually create two system accounts and a voting account (I won't cover staking accounts or nonce accounts). Specifically:

-   a wallet (system account), which
	- simulates the owners' wallet
	- is used to transfer voting funds to the validator
	- is used to withdraw rewards from the vote account
-  a validator identity (system account), which
	- represents the validators identity
	- pays voting fees
	- signs transactions
-   a voting account, which
	- is used for voting by the validator
	- receives validator rewards

When creating system accounts and voting accounts we need to provide a password. Each account creation produce a keypair file, and Solana provides us with the accounts' public key and seed phrase. The seed phrase is needed to recover a lost account (e.g. if you delete the keypair or lose the password). I usually store everything securely in a [KeePass](https://keepass.info/ "https://keepass.info/") database.

#### Wallet

Let’s get started. I first create the wallet:
```bash
solana-keygen new --outfile ~/wallet-keypair.json
```

Then I airdrop 1 SOL into it (this black magic is not possible on mainnet, for obvious reasons):
```bash
solana airdrop 1 ~/wallet-keypair.json
```

Solana replies with the balance, but we can verify the balance manually like this:
```bash
solana balance ~/wallet-keypair.json
```

#### Validator identity

Second is the validator identity:
```bash
solana-keygen new --outfile ~/validator-keypair.json
```

Which I need to assign to my Solana configuration:
```bash
solana config set --keypair ~/validator-keypair.json
```

Because the validator pays the voting fees, I need to give it some SOL. I can either airdrop or transfer to it. To simulate a real life operation I will transfer 0.5 SOL from the wallet:
```bash
solana transfer --allow-unfunded-recipient \
  --fee-payer ~/wallet-keypair.json \
  --from ~/wallet-keypair.json ~/validator-keypair.json 0.5
```

Then I can check the validator account balance. Note that we don't need to provide the account identity here, because Solana uses the one from the configuration (that we just set):
```bash
solana balance
```

#### Vote account

Third is the vote account:
```bash
solana-keygen new --outfile ~/vote-account-keypair.json
```

Then I tell Solana that this is a vote account, so it inherits the properties of that account type. I also set my wallet as an authorized withdrawer:
```bash
solana create-vote-account \
  ~/vote-account-keypair.json ~/validator-keypair.json ~/wallet-keypair.json
```
That's it for accounts!

## Create startup script and system services

To launch Solana I’m using a shell script that contains all the flags and options needed for my configuration (a “wrapper” script). The script is executed by the Solana service, which make sure Solana starts automatically on every system boot. Solana also calls upon a system tuner daemon, to make recommended adjustments to some system settings. Below I will take you through how I set all that up.

### Create wrapper script

```diff
! Perform as user “sol”
```

I’m using Nano to create and edit the script (proving I’m a pleb). You’ll recognize a lot of the file and directory references in the launch script from the sections above. Remember this if you chose other names for your files and directories.

I create the file `start-validator.sh` inside the home directory (`~/`) of “sol”:
```bash
nano ~/start-validator.sh
```

Nano will create and open an empty file. Inside it I paste the following:
```bash
#!/bin/bash
exec solana-validator \
 --entrypoint entrypoint.devnet.solana.com:8001 \
 --entrypoint entrypoint2.devnet.solana.com:8001 \
 --entrypoint entrypoint3.devnet.solana.com:8001 \
 --entrypoint entrypoint4.devnet.solana.com:8001 \
 --entrypoint entrypoint5.devnet.solana.com:8001 \
 --known-validator dv1ZAGvdsz5hHLwWXsVnM94hWf1pjbKVau1QVkaMJ92 \
 --known-validator dv2eQHeP4RFrJZ6UeiZWoc3XTtmtZCUKxxCApCDcRNV \
 --known-validator dv4ACNkpYPcE3aKmYDqZm9G5EB3J4MRoeE7WNDRBVJB \
 --known-validator dv3qDFk1DTF36Z62bNvrCXe9sKATA6xvVy6A798xxAS \
 --expected-genesis-hash EtWTRABZaYq6iMfeYKouRu166VU2xqa1wcaWoxPkrZBG \
 --dynamic-port-range 8000-8010 \
 --rpc-port 8899 \
 --only-known-rpc \
 --wal-recovery-mode skip_any_corrupted_record \
 --identity ~/validator-keypair.json \
 --vote-account ~/vote-account-keypair.json \
 --log ~/log/validator.log \
 --accounts /mnt/ramdisk/solana-accounts \
 --ledger ~/ledger \
 --limit-ledger-size
```

I won’t go through the flags and options, but you can study them by running `solana-validator --help`. I will say this, though: prepending `solana-validator` with `exec` is necessary to make log rotation work properly. More about that later. Also, to store the minimum amount of ledger you have to set `--limit-ledger-size 50000000`. Press **Ctrl+S** to save the file and **Ctrl+X** to exit Nano.

We need to make the script executable, or else it won’t launch:
```bash
chmod +x ~/start-validator.sh
```

Note that my script points to a “log” directory inside the home folder. Let’s create it:
```bash
mkdir ~/log
```

### Create system services

I need to create two system services; the validator service (that runs the script) and the tuner service. By running them as services they can be configured to start at boot. The system can also restart services automatically if they crash.

#### Validator service

```diff
! Perform as user with root privileges
```

I create the file `validator.service` inside systemd:
```bash
sudo nano /etc/systemd/system/validator.service
```

Inside the file I paste the following:
```
[Unit]
Description=Solana Validator
After=network.target
Wants=systuner.service
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=on-failure
RestartSec=1
LimitNOFILE=1000000
LogRateLimitIntervalSec=0
User=sol
Environment=PATH=/bin:/usr/bin:/home/sol/.local/share/solana/install/active_release/bin
Environment=SOLANA_METRICS_CONFIG=host=https://metrics.solana.com:8086,db=devnet,u=scratch_writer,p=topsecret
ExecStart=/home/sol/start-validator.sh

[Install]
WantedBy=multi-user.target
```

Save and exit.

#### System tuner service

```diff
! Perform as user with root privileges
```

Next I create the file `systuner.service` inside systemd:
```bash
sudo nano /etc/systemd/system/systuner.service
```

Inside the file I paste the following:
```
[Unit]
Description=Solana System Tuner
After=network.target

[Service]
Type=simple
Restart=on-failure
RestartSec=1
LogRateLimitIntervalSec=0
ExecStart=/home/sol/.local/share/solana/install/active_release/bin/solana-sys-tuner --user sol

[Install]
WantedBy=multi-user.target
```

Save and exit. To let the system know about the new services I reload the service manager:
```bash
sudo systemctl daemon-reload
```
The services are now ready to run. Just one more (optional) step before take-off.

## Set up log rotation (optional)

```diff
! Perform as user with root privileges
```

This step is optional, but recommended. Solana is a log hog. Just one days' worth of devnet logging is 1 GB+. Still, logs are nice to have for trouble shooting, and required if seeking help from the team. To make the logs easier to handle I rotate them every day. I also keep a weeks' worth of logs, in case I need to trouble shoot.

“Logrotate” takes care of the log rotation for us. It automatically creates a new log at 00:00 and deletes the excess. To activate logrotate for my validator log I need to create a configuration file for it:
```bash
sudo nano /etc/logrotate.d/solana
```

And paste the following inside it:
```
/home/sol/log/validator.log {
  su sol sol
  daily
  rotate 7
  missingok
  postrotate
    systemctl kill -s USR1 validator.service
  endscript
}
```

To load the new configuration I need to restart the logrotate service:
```bash
sudo systemctl restart logrotate
```
Log rotation is ready to roll.

## Starting the validator

After completing all the steps above I usually reboot my server (`sudo reboot`), although I suppose it’s not really necessary. It's still nice to verify that I have set up `fstab` correctly. After rebooting I do some quick checks before I start the services.

### Verify swap file and RAM disk presence

```diff
! Perform as user “sol”
```

I first check if the swap file and RAM disk is present. Mostly to confirm that they survived the reboot, which would mean I configured them correctly. I check the swap file by running:
```bash
swapon --show
```

My system replies:
```
NAME      TYPE SIZE USED PRIO
/swapfile file  16G   0B   -2
```

I look for the “name” and “size” to correspond to what I set previously. Then I use `mount` to check the RAM disk. I utilize `grep` to only print lines with “ramdisk” in their name:
```bash
mount | grep ramdisk
```

My system replies:
```
tmpfs on /mnt/ramdisk type tmpfs (rw,nosuid,nodev,noexec,noatime,size=16777216k,user=sol)
```

I typically look at the size and the user association.

### Start the services

```diff
! Perform as user with root privileges
```

Finally, it's time to start the services! The systuner service is first, since the validator application calls upon it:
```bash
sudo systemctl enable --now systuner.service
```

Then I check if it started successfully and is running:
```bash
sudo systemctl status systuner.service
```

It should say “active (running)”. I repeat the same steps with the validator service:
```bash
sudo systemctl enable --now validator.service
```
```bash
sudo systemctl status validator.service
```

### Check if the validator is running

```diff
! Perform as user “sol”
```

After starting the services I switch to "sol" again and start monitoring the start-up:
```bash
solana-validator --ledger ~/ledger monitor
```
The monitoring tool tells us what the validator is doing. It can take several minutes to start up the validator for the first time. When everything is up and running the reply from the monitor command should look similar to this:
```
Ledger location: /home/sol/ledger
Identity: vJPHDtC1UrdRy2NpNTtHQp14m6ZDUJA7CEDNsiYRA7k
Genesis Hash: EtWTRABZaYq6iMfeYKouRu166VU2xqa1wcaWoxPkrZBG
Version: 1.8.0
Shred Version: 23305
Gossip Address: IP_ADDRESS:8000
TPU Address: IP_ADDRESS:8003
⠋ 00:07:59 | Max Slot: 86087616 | Processed Slot: 86087615 | Confirmed Slot: 86087613 | Finalized Slot: 86087571 | Snapshot Slot: 86087510 | Transactions: 2211802703 | ◎0.498529086
```
If something different is showing, or if there is any red text, I usually give it a few minutes and check again (press **Ctrl+C** to exit). We can also run a simple command to see if the validator is caught up with the cluster. If the validator is not caught up, it will display the progress:
```bash
solana catchup ~/validator-keypair.json --our-localhost
```
If I get errors running these commands after a good wait (~10 minutes), the trouble shooting begins.

## Useful commands

Here are some commands I use to monitor my validator. You should run `solana --help` and explore!
To display log entries which contain “error” or “warn” I can run:
```bash
grep --ignore-case --extended-regexp 'error|warn' ~/log/validator.log
```

Show block production and skipped slots for my node:
```bash
solana block-production | grep $(solana address)
```

Export my leader schedule for the current epoch to a text file:
```bash
solana leader-schedule | grep $(solana address) \
  > ~/leader-schedule-epoch-$(solana epoch)-$(solana address).txt
```

Show info about the current epoch:
```bash
solana epoch-info
```

List all validators:
```bash
solana validators
```

I can also check live metrics for the network and my node here: [https://metrics.solana.com:3000/](https://metrics.solana.com:3000/)
