# Building Solana from source

 1. [Introduction](#introduction)
 2. [Initial installation](#initial-installation)
 3. [Update existing installation](#update-existing-installation)
 4. [All-in-one build command](#all-in-one-build-command) <- Use this for future builds/updates


## Introduction

Building Solana from source is fairly easy, and is considered best practice by most serious validators. One advantage of building from source is that the binaries can be adapted to the system they are built with. Another advantage is security. Building from source ensures the binaries are actually built from the intended source code.

There are also some disadvantages. It takes more time and requires more manual work. This not only applies to initial installations, but for every software update as well.

To make the process somewhat easier I made this tutorial. Here I provide instructions for building and installing Solana, in addition to performing updates on existing installations. At the end I provide an all-in-one command to make future builds easier.

The tutorial has been tested on a fresh install of Ubuntu 22.04.3 LTS. Post an issue if something is not working.


## Initial installation

### Install dependencies
```diff
! Perform as user with root privileges
```

Before I can build from source I have to install some dependencies. Installing them requires root privileges.
```bash
sudo apt update && sudo apt install -y \
  libssl-dev libudev-dev pkg-config zlib1g-dev llvm clang cmake make libprotobuf-dev protobuf-compiler
```

### Install build tools
```diff
! Perform as unprivileged user (for example "sol")
```
For the remaining commands I switch to an unprivileged user, without root access. I usually create the user "sol" to store and run the binaries, but you can name the user whatever you want. I do this by running `sudo adduser sol`. To switch to the user I run `su - sol`.

The next step is to install Rust:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
The installer will provide information about the installation. Press **enter** to proceed.

To enable the Solana build script to find the Cargo binaries (Rust's build system and package manager), my terminal session needs to know where they are. This is ensured through an environment variable, which is automatically added to `~/.profile` during the installation, and is loaded every time I log in. So I can either log out and back in again, or I can run this command:
```bash
source $HOME/.cargo/env
```
Rust is now ready to run.

### Build and install Solana for the first time

Now for the build. I clone the source code from GitHub into `~/solana-src-<version>`. I replace "version" with the latest mainnet version number from the Solana [GitHub](https://github.com/solana-labs/solana/releases/latest) page:
```bash
git clone https://github.com/solana-labs/solana.git --depth 1 --branch v1.16.19 ~/solana-src-v1.16.19
```

Then I run the `cargo-install-all.sh` script to build and install Solana. I assign `~/.local/share/solana/install/releases/1.16.19` as my installation directory, to mimic the directory structure of the pre-built binaries.
```bash
bash ~/solana-src-v1.16.19/scripts/cargo-install-all.sh ~/.local/share/solana/install/releases/1.16.19
```
The script will automatically check for dependencies and download the missing ones. During this it may produce errors like `error: toolchain '1.69.0-x86_64-unknown-linux-gnu' is not installed`. These errors can be ignored, provided the following downloads succeed. The build process will take some time. When it's done it provides a PATH variable for the binaries. I ignore this, as I make a more useful one in the following step.

When installing the *pre-built* binaries, the installer automatically creates a symbolic link from the installation directory (`*/1.16.19`) to another directory (`*/active_release`). This link can be used in service files and environment variables, so these won't have to be amended on every Solana update. When building from source I have to create that link manually:
```bash
ln --symbolic ~/.local/share/solana/install/releases/1.16.19 ~/.local/share/solana/install/active_release
```

To make sure I can run the Solana commands from any directory I need to add a PATH variable to `~/.profile`. Note that I use the symbolic link from above here:
```bash
echo 'export PATH=~/.local/share/solana/install/active_release/bin:$PATH' >> ~/.profile
```

### Finishing steps

At this point I usually log out and back in again (to reload `~/.profile`), and run a Solana command to verify that it is working:
```bash
solana --version
```

The system should respond with the version I just installed. If it doesn't, I retrace my steps and trouble-shoot. When I have verified Solana is working, I delete the source directory:
```bash
rm -rf ~/solana-src-v1.16.19
```
Installation complete! The remaining configuration may be done with the help of my [setup tutorial](https://github.com/agjell/sol-tutorials/blob/master/setting-up-a-solana-devnet-validator.md#configure-solana).


## Update existing installation
```diff
! Perform as unprivileged user (for example "sol")
```

Updating an existing (self built) Solana installation largely follows the same procedure as the initial install. Below I will repeat the necessary steps from above, but spare you the explanations. For the sake of ease, you may also jump straight to the [all-in-one command](#all-in-one-build-command).

First I clone the new release (find the most recent one on [GitHub](https://github.com/solana-labs/solana/releases/)):
```bash
git clone https://github.com/solana-labs/solana.git --depth 1 --branch v1.16.20 ~/solana-src-v1.16.20
```

Followed by the build and installation process:
```bash
bash ~/solana-src-v1.16.20/scripts/cargo-install-all.sh ~/.local/share/solana/install/releases/1.16.20
```

Next I replace the symbolic link, pointing it to the new installation. I need to use `--force` and `--no-dereference` to overwrite the old link:
```bash
ln --force --no-dereference --symbolic \
  ~/.local/share/solana/install/releases/1.16.20 ~/.local/share/solana/install/active_release
```

Then I run a command to verify Solana is working:
```bash
solana --version
```

If it replies with the updated version tag I delete the source files:
```bash
rm -rf ~/solana-src-v1.16.20
```

After a few days of stable operation I delete the binaries from the previous build:
```bash
rm -rf ~/.local/share/solana/install/releases/1.16.19
```
Update complete!

## All-in-one build command

Lastly, I have combined all the necessary commands to build/update Solana into one. This command presumes I have already installed Rust and the required dependencies. Here I first define an environment variable "TAG" with the version I want to build. This version is then called by all the consecutive commands.

```bash
export TAG=v1.16.20 && \
git clone https://github.com/solana-labs/solana.git --depth 1 --branch $TAG ~/solana-src-$TAG && \
  bash ~/solana-src-$TAG/scripts/cargo-install-all.sh ~/.local/share/solana/install/releases/$TAG && \
  ln --force --no-dereference --symbolic ~/.local/share/solana/install/releases/$TAG ~/.local/share/solana/install/active_release && \
  solana --version
```
That's it!