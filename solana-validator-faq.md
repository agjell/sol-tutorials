# Solana validator FAQ (unofficial)
_Written by Andrebo, with assistance from Zantetsu._

Every day we get lots of similar questions about validating in the Solana discord. In this FAQ we have tried to address most of them. Before asking in the discord, please try to find answers to your questions below.

1. [Can I mine SOL?](#1)
2. [Can anyone become a validator?](#2)
3. [Do I need any skills?](#3)
4. [Can I validate on a Windows computer?](#4)
5. [What are the hardware requirements?](#5)
   * [Are the requirements for mainnet, testnet and devnet the same? Can I get by with less?](#5a)
   * [How big is the ledger? How much storage space do I need for my validator?](#5b)
6. [What are the network requirements?](#6)
7. [Can I run a validator from my home?](#7)
8. [How do I set up a validator?](#8)
9. [Does it cost anything to validate?](#9)
10. [How do I make my rewards as a validator?](#10)
11. [How much stake is needed to be profitable, and can future rewards be calculated?](#11)
12. [How can I attract stake to my validator?](#12)
13. [Can I get stake from the Solana foundation?](#13)
14. [What is Tour de SOL and how can I participate?](#14)
15. [Where can I find more information about Solana?](#15)


<div id="1"></div>

## 1. Can I mine SOL?

No. SOL cannot be mined. Miners are part of proof-of-work blockchains. Solana is based on proof-of-stake, where a “validator” is roughly the equivalent of a miner (but still very different). You’ll find lots of information about these concepts on YouTube, Wikipedia and other sites. You should know the basics before reading any further.


<div id="2"></div>

## 2. Can anyone become a validator?

Yes. The Solana network is permissionless, so anyone can become a validator. But please continue reading this FAQ to get more knowledge about running a validator node.


<div id="3"></div>

## 3. Do I need any skills?

Yes. You need to be comfortable with command line interfaces and Linux administration. These skills are necessary for operating a validator node day-to-day. All validators have a responsibility to run their node in a secure manner, which requires sufficient knowledge about the operating system, the hardware, the network configuration and the relevant risks. There is no way around this!

One way to learn is by [setting up a validator on devnet](https://github.com/agjell/sol-tutorials/blob/master/setting-up-a-solana-devnet-validator.md) or testnet, and run it like you would a mainnet validator. Important tasks are securing and maintaining your validator, keeping the Solana software up to date in accordance with the Discord announcements, and monitoring your validator performance to ensure the highest uptime possible. Spend enough time learning before you think about becoming a mainnet validator.


<div id="4"></div>

## 4. Can I validate on a Windows computer?

No (well, sort of). The Solana validation runtime does not yet work on Windows. Technically you can validate through a Linux virtual machine on a Windows computer, but you would still need the Linux skills mentioned above, in addition to the skills needed to run a virtual machine securely. You would also lose some performance to the host operating system.


<div id="5"></div>

## 5. What are the hardware requirements?

You can find the official hardware requirements in the [Solana docs](https://docs.solana.com/running-validator/validator-reqs).

*Note: The validator software runs better on high clock frequencies, so it's not only core count that matters. High boost frequencies matter just as much. A 32-core CPU @ 2 GHz would probably struggle, while a 16-core CPU @ 3 GHz can perform well. This is partly because one thread (running [proof of history](https://medium.com/solana-labs/proof-of-history-a-clock-for-blockchain-cf47a61a9274)) will run at full capacity all the time. Older server CPUs with lower clocks are therefore less likely to be good performers, even if they have many cores.*


<div id="5a"></div>

### 5a. Are the requirements for mainnet, testnet and devnet the same? Can I get by with less?

#### Mainnet

Mainnet validators should follow the official requirements. Running a mainnet validator comes with responsibilities. Real people with real assets rely on the continuous operation and well being of mainnet. To ensure this, it's critical that the network and its validators have enough performance headroom. The team is also working to increase performance, which is not possible if there is no headroom to scale into. It is worth mentioning that some mainnet validators run successfully with 64 GB RAM, although the recommendation states ≥128 GB.

#### Testnet

Officially the hardware requirements for testnet are the same as for mainnet. This makes sense, as testnet is where the team test updates, functionality, performance and stability ([more info here](https://docs.solana.com/clusters#testnet)). This means the load on testnet can be higher than on mainnet. However, since testnet is somewhat of a playground, with non-real assets at play, there is room to experiment.

The network is permissionless, so anyone can set up with the hardware they have and see how it fares. 8-core CPUs like the AMD Ryzen 7 3700X and the Intel i9-9900K have proved sufficient in the past. You can likely get by with 32 GB RAM, but 64 GB would be more ideal. The official SSD recommendations from the docs are firm; at least one NVMe SSD with high IOPS and TBW ratings is needed. It's worth noting that previously working configurations may be outdated in the near future. As Solana is in rapid development, testnet demand can change rapidly. This is another reason to abide by the official recommendation.

#### Devnet

Again, the requirements are officially the same as for mainnet. Though devnet usually has much less transaction throughput than mainnet or testnet. If you're looking to become a validator, devnet is a natural place to start. You can get to know the software and take Solana for a spin; in peace and without any pressure to perform. Validating on devnet is also a good way to learn without having to spend much. Most geeks have the hardware needed to run devnet stored somewhere dusty. A devnet validator can likely get by on any decent 4-core CPU. 16 GB RAM might work. An NVMe SSD is recommended, but a SATA SSD could work if the Solana accounts data is stored in RAM. You can follow [this tutorial](https://github.com/agjell/sol-tutorials/blob/master/setting-up-a-solana-devnet-validator.md) on how to set up a devnet validator.


<div id="5b"></div>

### 5b. How big is the ledger? How much storage space do I need for my validator?

The ledger is infinitely big. You won't need to store the whole ledger, and you probably wouldn't have room for it anyway. Validators only store part of the ledger. You control how much of the ledger you store by passing the `--limit-ledger-size` option when you start your validator. If you don't specify a number it uses the default value of 200 million shreds. This requires ~400-500 GB of storage space. The minimum value you can pass is 50 million shreds (`--limit-ledger-size 50000000`). If you want to store an epoch worth of ledger (which can be useful for monitoring purposes) you need to specify ~260-270 million shreds.


<div id="6"></div>

## 6. What are the network requirements?

The official requirement states 1 Gbit/s. Mainnet validators should abide by this. Current real-life usage indicates a minimum of 300 Mbit/s up and down. Egress is often higher than ingress. Highly staked nodes can have three to four times more egress than ingress traffic.


<div id="7"></div>

## 7. Can I run a validator from my home?

Running a validator requires awareness about any risks relevant to its secure and continuous operation, and running it from home even more so. A data center will usually ensure the necessary physical protection (access/theft/fire/flood etc.) in addition to high network and power availability. At home you need to mitigate those risks yourself. Will your node or any network components be accessible to others? Do you have kids or pets? Are there a lot of break-ins where you live? Is your power grid stable? Do you have close to 100 % internet uptime? Is your area susceptible to natural disasters? You should think about all of these risks before you run a validator from home. Outsourcing some risk to a data center may be worth the cost.


<div id="8"></div>

## 8. How do I set up a validator?

 1. Install Ubuntu
 2. [Install the Solana command line interface (CLI)](https://docs.solana.com/cli/install-solana-cli-tools)
 3. [Configure and start the validator software](https://docs.solana.com/running-validator/validator-start)

You can also see this tutorial on [how to set up a devnet validator](https://github.com/agjell/sol-tutorials/blob/master/setting-up-a-solana-devnet-validator.md).


<div id="9"></div>

## 9. Does it cost anything to validate?

Yes. A fee is charged for every transaction on the blockchain. A validator is expected to vote on all the blocks it receives, to help achieve network consensus. Every vote is a transaction, with the fee being charged to the validator. The voting costs for running a validator is currently ~1.1 SOL per day. A validator needs sufficient stake to make more in rewards than it spends on voting fees.


<div id="10"></div>

## 10. How do I make my rewards as a validator?

Solana is based on proof-of-stake, which is fundamentally different from proof-of-work. In a proof-of-stake blockchain a validator needs stake delegated to it to make rewards. More stake equals more rewards, because validators with higher stake are chosen to write new transactions to the ledger more often. The assumption is that nodes with higher stake are more trustworthy, because people have trusted them with their tokens. Stable returns are the result of high uptime, valid voting at a high rate, and producing valid blocks within their required time slot. The validator has incentive to be a good actor because stakers can withdraw their stake at any time. Losing stake means losing rewards. The stakers incentive is to get yield on their tokens. The yield depends on the commission the validator charges. You can read more about Staking in [this section of the docs](https://docs.solana.com/staking).


<div id="11"></div>

## 11. How much stake is needed to be profitable, and can future rewards be calculated?

The amount of stake a validator needs to be profitable depends on multiple factors:

 - Hardware costs
 - Network costs
 - Validator voting costs
 - Validator commission
 - Price of SOL tokens
 - Amount of transactions in the network

Unfortunately there is no official way of calculating rewards. However, Zantetsu (from the Solana discord) has made a [spreadsheet](https://docs.google.com/spreadsheets/d/1HPU_uG3iJ_ns27CItdWGllW0c-Pn07J0_LEDZs1otQY/edit?usp=sharing) based on their experience. You can make your own copy of the spreadsheet and play with the numbers.


<div id="12"></div>

## 12. How can I attract stake to my validator?

That’s for you to figure out! As rewards can be considerable, you should have more than enough incentive to attract stake. Being a resource for the community is a good start. Building something useful, suggesting amendments to the code, filing bug reports or supporting people in the Discord are actions that can attract attention (and stake) to validators.


<div id="13"></div>

## 13. Can I get stake from the Solana foundation?

No (not at this time). Previously it was possible for validators who participated successfully in a certain number of [Tour de SOL](https://docs.solana.com/tour-de-sol) (TdS) stages to get an invitation to mainnet. The invitation included delegation of stake by way of an automatic staking bot. However, the foundation delegation program is currently [being revised](https://forums.solana.com/t/summary-of-validator-compensation-programs/1269) and is paused indefinitely:
> The Solana Foundation is re-evaluating the criteria needed to receive and maintain a delegation from the Foundation for eligible Mainnet Beta validators, and is re-implementing our staking tools and monitoring solutions to make the delegation behavior and requirements more transparent. Until this solution is implemented, the Foundation has decided to pause the addition of new validator nodes to its delegation program.


<div id="14"></div>

## 14. What is Tour de SOL and how can I participate?

Tour de SOL is Solanas incentivized testnet, where eligible validators can participate to test their skills and compete to be best. The tour runs in stages, just like Le Tour. Registration and KYC (know your customer) approval is required for all participants. US entities and individuals cannot participate. You can get information about registration [here](https://solana.com/validator-registration).


<div id="15"></div>

## 15. Where can I find more information about Solana?

 - [The official website](https://solana.com/)
 - [Documentation](https://docs.solana.com/)
 - [Discord chatroom](https://discordapp.com/invite/pquxPsq)
 - [News on Medium](https://medium.com/solana-labs)
 - [Forum](https://forums.solana.com/)
 - [Podcast](https://podcast.solana.com/)
 - [Telegram channel](https://t.me/solana)
 - [YouTube channel](https://www.youtube.com/c/Solanalabs/)
 - [GitHub project page](https://github.com/solana-labs/)
 - [Soltraders](http://t.me/soltraders) on Telegram (for price and trading talk)
