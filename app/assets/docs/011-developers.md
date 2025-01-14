# Developers

## Compiling

### Compiling packages for Debian and Ubuntu

As Sumcash is made to run on range of platforms, from Amazon's server to low powered Raspberry Pi Debian is perfect OS platform for deploying Sumcash nodes as it is renowned for multitude of supported hardware architectures as well as security and stability.

For compilation of Debian packages we will be using `pbuilder` which is a automatic Debian Package Building system for personal development workstation environments.<sup>[3.1](#footnote-3.1)</sup> pbuilder aims to be an easy-to-setup system for auto-building Debian packages inside a clean-room environment, so that it is possible to verify that a package can be built on most Debian installations. The clean-room environment is achieved through the use of a base chroot image, so that only minimal packages will be installed inside the chroot.

The following tutorial is written for the Ubuntu or Debian host, precisely Ubuntu 17.10 which is last stable version of Ubuntu at the time of writing.

### Prepare the tooling

`sudo apt install pbuilder qemu-user-static ubuntu-keyring debian-archive-keyring`

We'll also need keyring for the Raspbian, which a Debian fork made for the Raspberry Pi platform, so we can compile packages for this ARM based mini-pc platform.

> wget http://archive.raspbian.org/raspbian/pool/main/r/raspbian-archive-keyring/raspbian-archive-keyring_20120528.2_all.deb

> sudo dpkg -i raspbian-archive-keyring_20120528.2_all.deb

Now, set up the conf file for the pbuilder.

`touch .pbuilderrc`

> gedit .pbuilderrc

Paste the following:

```
#!/bin/sh

set -e

PBUILDERSATISFYDEPENDSCMD="/usr/lib/pbuilder/pbuilder-satisfydepends-apt"

if [ "$OS" == "debian" ]; then
    MIRRORSITE="http://ftp.no.debian.org/debian/"
    COMPONENTS="main contrib non-free"
    DEBOOTSTRAPOPTS=("${DEBOOTSTRAPOPTS[@]}"
        "--keyring=/usr/share/keyrings/debian-archive-keyring.gpg")
    : ${DIST:="wheezy"}
    : ${ARCH:="amd64"}
    if [ "$DIST" == "wheezy" ]; then
        #EXTRAPACKAGES="$EXTRAPACKAGES debian-backports-keyring"
        OTHERMIRROR="$OTHERMIRROR | deb $MIRRORSITE wheezy-backports $COMPONENTS"
    fi
elif [ "$OS" == "raspbian" ]; then
    MIRRORSITE="http://ftp.acc.umu.se/mirror/raspbian/raspbian/"
    COMPONENTS="main contrib non-free"
    DEBOOTSTRAPOPTS=("${DEBOOTSTRAPOPTS[@]}"
        "--keyring=/usr/share/keyrings/raspbian-archive-keyring.gpg")
    : ${DIST:="wheezy"}
    : ${ARCH:="armhf"}
elif [ "$OS" == "ubuntu" ]; then
    MIRRORSITE="http://no.archive.ubuntu.com/ubuntu/"
    COMPONENTS="main restricted universe multiverse"
    DEBOOTSTRAPOPTS=("${DEBOOTSTRAPOPTS[@]}"
        "--keyring=/usr/share/keyrings/ubuntu-archive-keyring.gpg")
else
    echo "Unknown OS: $OS"
    exit 1
fi

if [ "$DIST" == "" ]; then
    echo "DIST is not set"
    exit 1
fi

if [ "$ARCH" == "" ]; then
    echo "ARCH is not set"
    exit 1
fi

NAME="$OS-$DIST-$ARCH"

if [ "$ARCH" == "armel" ] && [ "$(dpkg --print-architecture)" != "armel" ]; then
    DEBOOTSTRAP="qemu-debootstrap"
fi
if [ "$ARCH" == "armhf" ] && [ "$(dpkg --print-architecture)" != "armhf" ]; then
    DEBOOTSTRAP="qemu-debootstrap"
fi

DEBOOTSTRAPOPTS=("${DEBOOTSTRAPOPTS[@]}" "--arch=$ARCH")
BASETGZ="/var/cache/pbuilder/$NAME-base.tgz"
DISTRIBUTION="$DIST"
BUILDRESULT="$HOME/pbuild-results/"
APTCACHE="/var/cache/pbuilder/$NAME/aptcache/"
BUILDPLACE="/var/cache/pbuilder/build"
HOOKDIR="/var/cache/pbuilder/hook.d/"
```

Make the directory where pbuilder will place the packages:

`mkdir -p $HOME/pbuild-results`

### Bootstrap the chroots

Raspbian (buster):

> sudo OS=raspbian DIST=buster ARCH=armhf pbuilder --create

Debian stable:

> sudo OS=debian DIST=buster ARCH=amd64 pbuilder --create

### Preparing for build

(compiling for Raspbian stretch in this example)

Download the latest .tar.gz from github.com/sumcash-project.

> wget https://github.com/sumcash-project/sumcash/releases/download/v0.11.0sum/sumcash-0.11.0.tar.gz

Debian build system is very strict about names, so we need to rename this to:

`sumcash_0.11.0.orig.tar.gz`

Extract the contests of the file using `tar xf` and `cd` to it.

Copy the debian directory from the contrib to this directory:

`cp -r contrib/debian .`

#### Some modifications

(this is just an extra step required for the Raspbian, to fix problem with autotools)

`sed '/./configure --with-gui=qt5 --with-incompatible-bdb/s/$/ --with-boost-libdir=/usr/lib/arm-linux-gnueabihf/' debian/rules`

### Building the package

`OS=raspbian DIST=stretch ARCH=armhf pdebuild`

And wait, it will take a while so go get a coffee or something. It's compiling by emulating ARM cpu in QEMU.

The concept of pbuild and cross-platform compilations is that you pass it this environment variables like "OS" and "DIST".

For example OS=debian and DIST=wheezy will use Debian Wheezy chroot, you can also pick architecture by using ARCH= environment variable.

### Footnotes

<a id="footnote-3.1">3.1</a>: https://jodal.no/2015/03/08/building-arm-debs-with-pbuilder/

## JSON-RPC API reference

Sumcash daemon offers JSON-RPC interface which can be used to control the daemon or integrate it with software stack.
You can send commands to the daemon by using `sumcash-cli` tool.

There are two official wrappers for this interface, a PHP one and a Python2.7+ one.

> Sumcash_rpc is a simple and minimal library made for communication with sumcashd via JSON-RPC protocol. It has a single dependency - a Python requests library and it supports both mainnet and testnet sumcash network with authentication or SSL encryption. There is a single class to be imported from the library - Client.

https://github.com/sumcash-project/sumcash_rpc

> sumcash-php-rpc is a simple and minimal library made for communication with sumcashd via JSON-RPC protocol for PHP 7.1+. Easiest way to use is to use composer. Otherwise include RpcClient class in your project an you are good to go.

https://github.com/sumcash-project/sumcash-php-rpc

### List of JSON-RPC calls

| Command            | Parameters  | Description    | Requires unlocked wallet? (yes/no)  |
|--------------------|-------------|---------------------------------|--------------------|
| `getinfo`          |             |Returns an object containing various state info.|no|
| `getblock`         |  `hash`     |Returns information about the block with the given hash.|no|
| `getblockcount`    |             |Returns the number of blocks in the longest block chain.|no|
| `getblockhash`     |`block_num`  |Returns hash of block in best-block-chain at `block_num`; 0 is the genesis block|no|
| `gettransaction`   | `txid`        |Returns an object about the given transaction containing:<br>  "amount" : total amount of the transaction<br>"confirmations" : number of confirmations of the transaction<br>"txid" : the transaction ID<br>"time" : time associated with the transaction|no|
| `walletpassphrase` | `passphrase`   `timeout` |Stores the wallet decryption key in memory for `timeout` seconds.|no|
| `getbalance`       |[account] [minconf=1]|If [account] is not specified, returns the server's total available balance.<br>If [account] is specified, returns the balance in the account.|no|
| `getreceivedbyaddress`| `address` [minconf=1] |Returns the amount received by `address` in transactions with at least [minconf] confirmations. It correctly handles the case where someone has sent to the address in multiple transactions. Keep in mind that addresses are only ever used for receiving transactions.<br>Works only for addresses in the local wallet, external addresses will always show 0.|no|
| `getdifficulty`    |  |Returns proof-of-stake and proof-of-work difficulty|no|
| `getpeerinfo`      |  |Returns data about each connected node.|no|
| `getaddressesbyaccount`| `account` |Returns the list of addresses for the given account.|no|
| `getnewaddress`    | [account] |Returns a new address for receiving payments.<br>If [account] is specified payments received with the address will be credited to [account].|no|
| `getaccount`       | `address` |Returns the account associated with the given `address`.|no|
| `getaccountaddress`| `account` |Returns the current address for receiving payments to this account.<br>If `account` does not exist, it will be created along with an associated new address that will be returned.|no|
| `sendtoaddress`    | `address` `amount` [comment] [comment-to] |  `amount` is a real and is rounded to 6 decimal places. Returns the transaction ID `txid` if successful.|yes|
| `sendfrom`         | `fromaccount` `tosumcashaddress` `amount` [minconf=1] [comment] [comment-to] |`amount` is a real and is rounded to 6 decimal places. Will send the given amount to the given address, ensuring the account has a valid balance using [minconf] confirmations. Returns the transaction ID if successful (not in JSON object).|yes|
| `sendmany`         | `fromaccount` {address:amount,...} [minconf=1] [comment] |  amounts are double-precision floating point numbers |yes|
| `getconnectioncount`|    |Returns the number of connections to other nodes.|no|
| `getrawtransaction`|  `txid` [verbose=0] |Returns raw transaction representation for given transaction id.|no|
| `getrawmempool`    |  |Returns all transaction ids in memory pool.|no|
| `listtransactions` | [account] [count=10] [from=0] | Returns up to [count] most recent transactions skipping the first [from] transactions for account [account]. If [account] not provided it'll return recent transactions from all accounts.|no|
| `listreceivedbyaddress`| [minconf=1] [includeempty=false] |Returns an array of objects containing:<br>"address" : receiving address<br>"account" : the account of the receiving address<br>"amount" : total amount received by the address<br>"confirmations" : number of confirmations of the most recent transaction included<br>To get a list of accounts on the system, execute `sumcash-cli listreceivedbyaddress 0 true`|no|
| `listreceivedbyaccount`| [minconf=1] [includeempty=false] |Returns an array of objects containing:<br>"account" : the account of the receiving addresses<br>"amount" : total amount received by addresses with this account<br>"confirmations" : number of confirmations of the most recent transaction included|no|
| `listaccounts` | [minconf=1] |Returns Object that has account names as keys, account balances as values.|no|
| `listunspent`  | [minconf=1] [maxconf=999999] |Returns array of unspent transaction inputs in the wallet.|no|
| `dumpprivkey`  | `address`   |Reveals the private key corresponding to `address`.|yes|
| `importprivkey`|  `privkey` [label] [rescan=true]|Adds a private key (as returned by dumpprivkey) to your wallet. This may take a while, as a rescan is done, looking for existing transactions.|yes|
| `createrawtransaction`| [{"txid":txid,"vout":n},...] {address:amount,...} |Creates a raw transaction spending given inputs.|no|
| `decoderawtransaction`| `hex_string` |Produces a human-readable JSON object for a raw transaction.|no|
| `signrawtransaction`  | `hex_string` [{"txid":txid,"vout":n,"scriptPubKey":hex},...] [`privatekey1`,...]|Adds signatures to a raw transaction and returns the resulting raw transaction.|yes|
| `signmessage`         | `address` `message` |Sign a message with the private key of an address.|yes|
| `verifymessage`       | `address` `signature` `message` |Verify a signed message.|no|
| `sendrawtransaction`  | `hex_string` |Submits raw transaction (serialized, hex-encoded) to local node and network.|no|
| `validateaddress`     | `address` |Return information about `address`.|no|
| `encryptwallet`       | `passphrase` |Encrypts the wallet with `passphrase`|no|
| `enforcecheckpoint`   | `bool` |`enforce` is true or false to enable or disable enforcement of broadcasted checkpoints by developer.|no|
| `keypoolrefill`       | `size` |Fills the key pool with new keys [default 100 new keys]|yes|
| `listlockunspent`     |        |Returns list of temporarily unspendable outputs.|no|
| `createmultisig`      | `nrequired` `["key,"key"]`|Creates a multi-signature address and returns a json object.|yes|

## Sumcash Developer Notes

Constants that may be useful when looking to integrate / develop with Sumcash.

Sumcash source code repository: github.com/sumcash-project/sumcash

### Mainnet

| Attribute | Value |
|-----------|-------|
| p2pkh Base58 prefix | P |
| p2sh Base58 prefix | p |
| p2pkh Base58 prefix (hex) | 0x37 |
| p2sh Base58 prefix (hex) | 0x75 |
| Magic bytes |  \xe6\xe8\xe9\xe5 |
| WIF prefix | 0xb7 |
| Genesis hash hex (big-endian) |  0000000032fe677166d54963b62a4677d8957e87c508eaa4fd7eb1c880cd27e3 |
| Genesis hash bytes (little-endian) |  \xe3\x27\xcd\x80\xc8\xb1\x7e\xfd\xa4\xea\x08\xc5\x87\x7e\x95\xd8\x77\x46\x2a\xb6\x63\x49\xd5\x66\x71\x67\xfe\x32\x00\x00\x00\x00 |
| Genesis tx hash hex | 3c2d8f85fab4d17aac558cc648a1a58acff0de6deb890c29985690052c5993c2 |
| Genesis tx hash bytes |  \xc2\x93\x59\x2c\x05\x90\x56\x98\x29\x0c\x89\xeb\x6d\xde\xf0\xcf\x8a\xa5\xa1\x48\xc6\x8c\x55\xac\x7a\xd1\xb4\xfa\x85\x8f\x2d\x3c |
| Default port | 9901 |
| Default RPC port | 3332 |
| BIP44 coin type| 0x80000006 |
| bech32 prefix | pc |

### Testnet

| Attribute | Value |
|-----------|-------|
| p2pkh Base58 prefix | m or n |
| p2sh Base58 prefix | n |
| p2pkh Base58 prefix (hex) | 0x6f |
| p2sh Base58 prefix (hex) | 0xc4 |
| Magic bytes | \xcb\xf2\xc0\xef |
| WIF prefix | 0xef |
| Genesis hash hex (big-endian) | 00000001f757bb737f6596503e17cd17b0658ce630cc727c0cca81aec47c9f06 |
| Genesis hash bytes (little-endian) |  \x06\x9f\x7c\xc4\xae\x81\xca\x0c\x7c\x72\xcc\x30\xe6\x8c\x65\xb0\x17\xcd\x17\x3e\x50\x96\x65\x7f\x73\xbb\x57\xf7\x01\x00\x00\x00  |
| Genesis tx hash hex | 3c2d8f85fab4d17aac558cc648a1a58acff0de6deb890c29985690052c5993c2 |
| Genesis tx hash bytes |  \xc2\x93\x59\x2c\x05\x90\x56\x98\x29\x0c\x89\xeb\x6d\xde\xf0\xcf\x8a\xa5\xa1\x48\xc6\x8c\x55\xac\x7a\xd1\xb4\xfa\x85\x8f\x2d\x3c |
| Default port | 9903 |
| Default RPC port | 9904 |
| BIP44 coin type| 0x80000006 |
| bech32 prefix | tpc |

## Transaction format

Sumcash transaction format is indendical to a Bitcoin transaction format with the exception of included transaction timestamp.

| Property | Description | Bytes |
|----------|-------------|-------|
| Version  | transaction version number | 4 |
| Timestamp | transaction timestamp | 4 |
| Input-counter | number of transaction inputs | 1-9 |
| List of inputs | list of transaction inputs | varies |
| Output-counter | number of transaction outputs | 1-9 |
| List of outputs | list of transaction outputs | varies |
| Locktime | block number or Unix timestamp when the transaction finalizes | 4 |


As of Sumcash 0.11, timestamp is no longer required and transaction format is exactly the same as Bitcoin. The only exception is that `version` is now `3`.

| Property | Description | Bytes |
|----------|-------------|-------|
| Version  | transaction version number | 4 |
| Input-counter | number of transaction inputs | 1-9 |
| List of inputs | list of transaction inputs | varies |
| Output-counter | number of transaction outputs | 1-9 |
| List of outputs | list of transaction outputs | varies |
| Locktime | block number or Unix timestamp when the transaction finalizes | 4 |


## Creating transaction

In this simple example it it will be demonstrated how to use popular bitcore library to create a Sumcash transaction.
Example will be using node.js and javascript. Similar libaries can be found for practically all other programming languages though.
Using bitcore is possible because Sumcash is based of Bitcoin and the two share more than 99% of the code.

```
const bitcore = require('bitcore-lib');

//
// Add sumcash network params
//

bitcore.Networks.add({
    name: 'sumcash',
    alias: 'sumcash',
    pubkeyhash: 0x37,
    privatekey: 0xb7,
    scripthash: 0x75,
    xpubkey: 0x0488b21e,
    xprivkey: 0x0488ade4,
  });

bitcore.Networks.add({
    name: 'sumcash-testnet',
    alias: 'sumcash-test',
    pubkeyhash: 0x6f,
    privatekey: 0xef,
    scripthash: 0xc4,
    xpubkey: 0x043587cf,
    xprivkey: 0x04358394,
  });


// set sumcash-testnet as default network
bitcore.Networks.defaultNetwork = bitcore.Networks.get('sumcash-testnet');

//
// Generate privkey and address
//

// Our private key and address
const value = Buffer.from('battery powered horse!'); // we are making the private from this random string
const hash = bitcore.crypto.Hash.sha256(value);
const bn = bitcore.crypto.BN.fromBuffer(hash);
const privateKey = new bitcore.PrivateKey(bn);
const myAddress = privateKey.toAddress();

console.log("This is my address: ", myAddress.toString()); // this is the address which will be spending coins, so you need testnet sumcashs sent here. Use faucet: https://sumcashexplorer.net/faucet/

//
// Assemble, sign and send a transaction
//

// Find appopriate utxo manually using a blockexplorer, this is just an example
const utxo = {
  "txId" : "2643f9721cee24c489b58a123e86619dc08e044dbaeb19a58443a4c866c5bf8d",
  "outputIndex" : 0,
  "address" : "mwEPhfYCrr57qVDxwpz6KgEyA6nHhW7rZD",
  "script" : "76a914ac602663d8b249ccfa8e5299e3865ddd415d46e788ac",
  "satoshis" : 50000
};

var rec = "mwudtnoRS13KasEYv8Pthf7Qu4G1eLHgnZ"; // random reciever, replace with one you like

const transaction = new bitcore.Transaction()
    // Expects an array of utxos 
    .from(utxo)
    .feePerKb(10000) // data on Sumcash costs 0.01 SUMC / kB
    .to(reciever, 1) // sending 1 tSUM to reciever
    .addData("my test transaction!") // Add transaction metadata
    .change(myAddress) // change
    .sign(privateKey);

// Overriding txn version to 3, because that's what it needs to be to work with sumcash
transaction.version = 3;

console.log(txHex);

You can send this hex using local testnet node or via remote API which allows for "sendrawtransction" command.

```

## Bootstrapping

### What is it?

> In computer technology the term (usually shortened to booting) usually
> refers to the process of loading the basic software into the memory of
> a computer after power-on or general reset, especially the operating
> system which will then take care of loading other software as needed.<sup>[9.1](#footnote-9.1)</sup>

For Sumcash it means loading all of the block chain history from a special
file containing a snapshot of block data.

This special file, named `bootstrap.dat`, allows the Sumcash client to
sync from your hard drive instead of the internet. Using a
`bootstrap.dat` file is faster and reduces stress on the Sumcash network to sync new nodes.

### How do I make a `bootstrap.dat`?

Any synced client has the ability to make a `bootstrap.dat` file. Assuming
you're running linux you can do the following to manufacture your own.
First, shutdown your client. Allow it to cleanly exit so we know the block
data is settled.

Now, navigate to the directory `~/.sumcash/blocks` in your terminal and
notice the files named `blk00000.dat`, `blk00001.dat`, `blk00002.dat`, etc.
These are the raw block data files that can be combined to form the
`bootstrap.dat`!

The command:
> cat blk*.dat > bootstrap.dat

will produce the `bootstrap.dat`.
The file is often then compressed (zip'ed, tar/gzip'ed) and shared.

The same process can be executed on Microsoft Windows (7+):

> CD C:\Users\<my_user>\AppData\Roaming\Sumcash

> COPY /b blk0001.dat+blk0002.dat bootstrap.dat

Or on OS X:

> cd "~/Library/Application Support/Sumcash/"

> cat blk*.dat > bootstrap.dat


On linux and OS X you can create hash of the bootstrap:

> sha256 bootstrap.dat

### How do I use a `bootstrap.dat`?

Assuming you're on linux and you haven't started the Sumcash client before.

Make the directory `~/.sumcash` if it doesn't exist and then move the
`bootstrap.dat` into the `~/.sumcash` directory.

Start the Sumcash client. You should see the status `Importing blocks from disk...` if the client has found the `bootstrap.dat` and is using it to
sync the block chain.

### Where can I download a `bootstrap.dat`?

We've put some recent copies on our [file server](https://files.sumcash.org) :)

* [Mainnet bootstrap.dat.zip](https://files.sumcash.org/download/sumcash_mainnet_2018_08_06_bootstrap.dat.zip) Scrypt `66c494e0c2a4c78ba19f14a3781ca77f1b8b16f3833fb85c10959ee991764a4c` | [torrent](https://files.sumcash.org/download/sumcash_mainnet_2018_08_06_bootstrap.dat.zip.torrent) | [magnet](magnet:?xt=urn:btih:9fc1f0d09a6598ae96ba5d8b9ac5caca0ae92402&dn=sumcash%5Fmainnet%5F2018%5F08%5F06%5Fbootstrap.dat.zip&tr=udp%3A%2F%2Fpublic.popcorn-tracker.org%3A6969%2Fannounce&tr=udp%3A%2F%2Ftracker.ilibr.org%3A80%2Fannounce&tr=http%3A%2F%2Fatrack.pow7.com%2Fannounce&tr=http%3A%2F%2Fbt.henbt.com%3A2710%2Fannounce&tr=http%3A%2F%2Fmgtracker.org%3A2710%2Fannounce&tr=http%3A%2F%2Fmgtracker.org%3A6969%2Fannounce&tr=http%3A%2F%2Fopen.touki.ru%2Fannounce.php&tr=http%3A%2F%2Fp4p.arenabg.ch%3A1337%2Fannounce&tr=http%3A%2F%2Fpow7.com%3A80%2Fannounce&tr=http%3A%2F%2Fretracker.krs-ix.ru%3A80%2Fannounce)
* [Mainnet bootstrap.dat.tar.gz](https://files.sumcash.org/download/sumcash_mainnet_2018_08_06_bootstrap.dat.tar.gz) Scrypt `bac807186735347e2c7ccbfecb3d91045de32246258a7ba3e3d0a9c2a10b8ff0` | [torrent](https://files.sumcash.org/download/sumcash_mainnet_2018_08_06_bootstrap.dat.tar.gz.torrent) | [magnet](magnet:?xt=urn:btih:77f1c5352e9eca71e9f4f1e31487e7d13900a335&dn=sumcash%5Fmainnet%5F2018%5F08%5F06%5Fbootstrap.dat.tar.gz&tr=udp%3A%2F%2Fpublic.popcorn-tracker.org%3A6969%2Fannounce&tr=udp%3A%2F%2Ftracker.ilibr.org%3A80%2Fannounce&tr=http%3A%2F%2Fatrack.pow7.com%2Fannounce&tr=http%3A%2F%2Fbt.henbt.com%3A2710%2Fannounce&tr=http%3A%2F%2Fmgtracker.org%3A2710%2Fannounce&tr=http%3A%2F%2Fmgtracker.org%3A6969%2Fannounce&tr=http%3A%2F%2Fopen.touki.ru%2Fannounce.php&tr=http%3A%2F%2Fp4p.arenabg.ch%3A1337%2Fannounce&tr=http%3A%2F%2Fpow7.com%3A80%2Fannounce&tr=http%3A%2F%2Fretracker.krs-ix.ru%3A80%2Fannounce)
* [Testnet bootstrap.dat.zip](https://files.sumcash.org/download/sumcash_testnet_2018_08_06_bootstrap.dat.zip) Scrypt `1312aaaf0d8466b6f7006bac60a5cad4daf36e1241f791924e45bc5959ff2452` | [torrent](https://files.sumcash.org/download/sumcash_testnet_2018_08_06_bootstrap.dat.zip.torrent) | [magnet](magnet:?xt=urn:btih:c70afb5100953362b45df75133e01bf1f9466e04&dn=sumcash%5Ftestnet%5F2018%5F08%5F06%5Fbootstrap.dat.zip&tr=udp%3A%2F%2Fpublic.popcorn-tracker.org%3A6969%2Fannounce&tr=udp%3A%2F%2Ftracker.ilibr.org%3A80%2Fannounce&tr=http%3A%2F%2Fatrack.pow7.com%2Fannounce&tr=http%3A%2F%2Fbt.henbt.com%3A2710%2Fannounce&tr=http%3A%2F%2Fmgtracker.org%3A2710%2Fannounce&tr=http%3A%2F%2Fmgtracker.org%3A6969%2Fannounce&tr=http%3A%2F%2Fopen.touki.ru%2Fannounce.php&tr=http%3A%2F%2Fp4p.arenabg.ch%3A1337%2Fannounce&tr=http%3A%2F%2Fpow7.com%3A80%2Fannounce&tr=http%3A%2F%2Fretracker.krs-ix.ru%3A80%2Fannounce)
* [Testnet bootstrap.dat.tar.gz](https://files.sumcash.org/download/sumcash_testnet_2018_08_06_bootstrap.dat.tar.gz) Scrypt `15f0e40a7e99083b2ed27c78d37d0f201fd6c744537cf643925d0aae84395896` | [torrent](https://files.sumcash.org/download/sumcash_testnet_2018_08_06_bootstrap.dat.tar.gz.torrent) | [magnet](magnet:?xt=urn:btih:eb2a70e90285ca847be47b2f96e36e5cdec4580e&dn=sumcash%5Ftestnet%5F2018%5F08%5F06%5Fbootstrap.dat.tar.gz&tr=udp%3A%2F%2Fpublic.popcorn-tracker.org%3A6969%2Fannounce&tr=udp%3A%2F%2Ftracker.ilibr.org%3A80%2Fannounce&tr=http%3A%2F%2Fatrack.pow7.com%2Fannounce&tr=http%3A%2F%2Fbt.henbt.com%3A2710%2Fannounce&tr=http%3A%2F%2Fmgtracker.org%3A2710%2Fannounce&tr=http%3A%2F%2Fmgtracker.org%3A6969%2Fannounce&tr=http%3A%2F%2Fopen.touki.ru%2Fannounce.php&tr=http%3A%2F%2Fp4p.arenabg.ch%3A1337%2Fannounce&tr=http%3A%2F%2Fpow7.com%3A80%2Fannounce&tr=http%3A%2F%2Fretracker.krs-ix.ru%3A80%2Fannounce)

### Footnotes

<a id="footnote-9.1">9.1</a>: https://en.wikipedia.org/wiki/Bootstrapping


---
