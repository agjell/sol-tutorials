# Building Solana from source

## Introduction

Some people need or want to build Solana from source. I *had* to build from source because my CPU was lacking instructions that the prebuilt binaries relies upon (AVX2). This made the prebuilt Solana binaries crash. By building Solana myself the binaries were adapted to the hardware I was using.

While there are advantages building from source, there are also disadvantages. It takes more time and requires more manual work. This not only applies to initial installations, but for every software update as well.

To make the process somewhat easier I made this tutorial, where I provide instructions for building Solana from source, installing it and performing updates on existing installations.

## Initial installation

### Install system packages
```diff
! Perform as user with root privileges
```

To build from source I need to install some system packages, which requires root privileges:
```bash
sudo apt update && sudo apt install -y \
  libssl-dev libudev-dev pkg-config zlib1g-dev llvm clang make gcc
```

### Install build tools
```diff
! Perform as user "sol"
```

The user "sol" will be used for all the remaining tasks. First up is installing Rust:
```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```
The installer will provide information about the installation. Press **enter** to proceed.

To enable the Solana build script to access the Cargo binaries (Rust's build system and package manager), my terminal session needs to know where they are. This "environment variable" was added to `~/.profile` during the installation, which is loaded every time I log in. So I can either log out and back in again, or I can run this command:
```bash
source $HOME/.cargo/env
```
Rust is now ready to run.

### Build and install Solana

Now for the build. I clone the current release code from GitHub into `~/solana-src-<version>`. I replace the version number with the current one from the Solana [GitHub](https://github.com/solana-labs/solana/releases/) page:
```bash
git clone https://github.com/solana-labs/solana.git --branch v1.6.1 ~/solana-src-v1.6.1
```

Then I run the `cargo-install-all.sh` script to build and install Solana. I select `~/.local/share/solana/install/releases/1.6.1` as my installation directory, to mimic the directory structure of the prebuilt binaries.
```bash
./solana-src-v1.6.1/scripts/cargo-install-all.sh ~/.local/share/solana/install/releases/1.6.1
```
The script will automatically check for dependencies and download the missing ones. During this it may produce errors like `error: toolchain '1.50.0-x86_64-unknown-linux-gnu' is not installed`. These errors can be ignored, provided the downloads succeed. The build process will take some time. When it's done it provides a path variable to use the binaries. I ignore this, as I make a more useful one in the following step.

When installing the *prebuilt* binaries, the installer automatically creates a symbolic link from the installation directory (`*/1.6.1`) to another directory (`*/active_release`). This static link can be used in service files and environment variables, so these won't have to be amended on every Solana update. When building from source I have to create that link manually:
```bash
ln --symbolic ~/.local/share/solana/install/releases/1.6.1 ~/.local/share/solana/install/active_release
```

To make sure I can run the Solana commands from any directory I need to add a path variable to `~/.profile`. Note that I use the symbolic link from above here:
```bash
echo 'export PATH=/home/sol/.local/share/solana/install/active_release/bin:$PATH' >> ~/.profile
```

### Finishing steps

At this point I usually log out and back in again (to reload `~/.profile`), and run a Solana command to verify that it is working:
```bash
solana --version
```

The system should respond with the version I just installed. If it doesn't, I retrace my steps and trouble-shoot. When I have verified Solana is working, I delete the directory containing the source files:
```bash
rm -rf ~/solana-src-v1.6.1
```
Installation complete! The remaining configuration may be done with the help of my [setup tutorial](https://github.com/agjell/sol-tutorials/blob/master/setting-up-a-solana-devnet-validator.md#configure-solana).

## Update existing installation
```diff
! Perform as user "sol"
```

Updating an existing (self built) Solana installation largely follows the same procedure as the initial install. For the sake of demonstration I will repeat the necessary steps from above, but spare you the explanations.

First I ensure that Rust is updated:
```bash
rustup update
```

Then I clone the new release (find the most recent one on [GitHub](https://github.com/solana-labs/solana/releases/)):
```bash
git clone https://github.com/solana-labs/solana.git --branch v1.6.2 ~/solana-src-v1.6.2
```

Followed by the build and installation process:
```bash
./solana-src-v1.6.2/scripts/cargo-install-all.sh ~/.local/share/solana/install/releases/1.6.2
```

Next I replace the symbolic link, pointing it to the new installation. I need to use `--force` and `--no-dereference` to overwrite the old link:
```bash
ln --force --no-dereference --symbolic \
  ~/.local/share/solana/install/releases/1.6.2 ~/.local/share/solana/install/active_release
```

Then I run a command to verify Solana is working:
```bash
solana --version
```

If it replies with the updated version tag I call it a day. After a few days of stable operation I delete the binaries from the old installation:
```bash
rm -rf ~/.local/share/solana/install/releases/1.6.1
```
That's it!
