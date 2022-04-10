# ElectrumX for VerusCoin

## Server

A VPS with 4GB of RAM, more than 40GB of SSD storage, and 2 CPU cores is the absolute minimum requirement. Start following the guide while logged in as `root`.


## Operating System

This guide is tailored to and tested on `Debian 9 "Stretch"`. Before starting, please install the latest updates:

```
apt update
apt -y upgrade
```

## Wallet

The packages required in order to compile a VerusCoin wallet can be installed like this:

```
apt -y install build-essential git pkg-config libc6-dev m4 g++-multilib autoconf \
                   libtool ncurses-dev unzip git python python-zmq zlib1g-dev wget \
                   libcurl4-openssl-dev bsdmainutils automake curl
```


Create a useraccount for the wallet and switch to that account:

```
useradd -m -d /home/veruscoin -s /bin/bash veruscoin
su - veruscoin
```

Now clone the source tree and build the binaries:

```
git clone https://github.com/VerusCoin/VerusCoin
cd VerusCoin
./zcutil/fetch-params.sh
./zcutil/build.sh -j$(nproc)
```

After that is done, create a `~/bin` directory and copy over the binaries. Strip the debug symbols.

```
mkdir ~/bin
cp src/komodod src/komodo-cli src/komodo-tx ~/bin
strip ~/bin/komodo*
```

Start the VerusCoin daemon so we have a default configuration file:

```
komodod -ac_name=VRSC -ac_algo=verushash -ac_cc=1 -ac_veruspos=50 -ac_supply=0 -ac_eras=3 \
-ac_reward=0,38400000000,2400000000 -ac_halving=1,43200,1051920 -ac_decay=100000000,0,0 -ac_end=10080,226080,0 \
-ac_timelockgte=19200000000 -ac_timeunlockfrom=129600 -ac_timeunlockto=1180800 -addnode=185.25.48.236 \
-addnode=185.64.105.111 -daemon
```

Let it run for a few seconds and stop it again: 

```
komodo-cli -ac_name=VRSC stop
```

Edit the resulting `~/.komodo/VRSC/VRSC.conf` to include the parameters listed below, adapt the ones that need to be adapted.
A resonably secure `rpcpassword` can be generated using this command: 
`cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1`.

```
server=1
listen=1
listenonion=0
maxconnections=256

# logging related options
logtimestamps=1
logips=1
shrinkdebugfile=0

# how many blocks to check on startup
checkblocks=64

# indexing options
txindex=1
addressindex=1
timestampindex=1
spentindex=1

# make sure ipv4 & ipv6 is used
bind=<your public ipv4 address>
bind=<your public ipv6 address>

# rpc settings
rpcuser=veruscoin
rpcpassword=<your-secret-veruscoin-rpc-password>
rpcport=27486
rpcthreads=256
rpcworkqueue=1024
rpcbind=127.0.0.1
rpcallowip=127.0.0.1

# if a peer jacks up more than 25 times in a row, ban it
banscore=25

# stake if possible, although it's probably not helping much
gen=1
genproclimit=0

# addnodes
seednode=185.25.48.236:27485
addnode=185.25.48.236:27487
seednode=185.64.105.111:27485
addnode=185.64.105.111:27487
seednode=185.25.48.72:27485
seednode=185.25.48.72:27487
```

For proper ElectrumX operation, `txindex=1` is crucial. Afterwards, start the daemon again and let it sync the blockchain: 

```
komodod -ac_name=VRSC -ac_algo=verushash -ac_cc=1 -ac_veruspos=50 -ac_supply=0 -ac_eras=3 \
-ac_reward=0,38400000000,2400000000 -ac_halving=1,43200,1051920 -ac_decay=100000000,0,0 -ac_end=10080,226080,0 \
-ac_timelockgte=19200000000 -ac_timeunlockfrom=129600 -ac_timeunlockto=1180800 -addnode=185.25.48.236 \
-addnode=185.64.105.111 -daemon
```

To check the status and know when the initial sync has been completed, use this command:

```
komodo-cli -ac_name=VRSC getinfo
```

When it has synced up to height, the `blocks` and `longestchain` values will have the same value. Additionally, you should verify against [the explorer](https://explorer.veruscoin.io) that you are on the main chain and not on a fork. While we wait for this to happen, lets continue.

## Python 3.7 & Prerequisites

It's not exactly a 'clean' solution, but it works. Add the `buster` packages to `/etc/apt/sources.list`: 

```
deb http://ftp.debian.org/debian buster main contrib non-free
deb-src http://ftp.debian.org/debian buster main contrib non-free
```

Update the package list and install all necessary packages: 

```
apt update
apt -y install python3.7 python3.7-dev python3-multidict python3-setuptools git \
       build-essential cmake libtool autotools-dev automake pkg-config \
       libcurl4-gnutls-dev libssl-dev libevent-dev libdb++-dev zlib1g-dev libleveldb-dev \
       libboost-system-dev libboost-filesystem-dev libboost-chrono-dev libboost-program-options-dev libboost-test-dev libboost-thread-dev libboost-random-dev \
       unzip libsodium-dev sudo
```

After that has completed, remove the `buster` repos from `/etc/apt/sources.list` and update the package list again. To sum up, do this: 

```
cd /usr/bin
rm python3 python3m
ln -s python3.7m python3m
ln -s python3.7 python3
```

## ElectrumX Installation

Create a new user, (temporarily) enable sudo without password and switch to it: 

```
useradd -m -d /home/electrumx -s /bin/bash electrumx
echo "electrumx ALL=(ALL) NOPASSWD: ALL" >/etc/sudoers.d/010-electrumx
su - electrux
```

Now, check out the Veruscoin ElectrumX repo and install it: 

```
git clone https://github.com/VerusCoin/electrumx
cd electrumx; sudo python3 setup.py install
```

## ElectrumX Configuration

Switch back to root. Copy over the `systemd` unit file and create `/etc/electrumx.conf`. Create a datadir and assign ownership to the `electrumx` user.

```
cp /home/electrumx/electrumx/contrib/systemd/electrumx.service /etc/systemd/system
cat <<EOF >/etc/electrumx.conf
COIN = Verus
DB_DIRECTORY = /electrumdb/VRSC
DAEMON_URL = http://veruscoin:<your-secret-veruscoin-rpc-password>@127.0.0.1:27486/
RPC_HOST = 127.0.0.1
RPC_PORT = 8000
HOST =
TCP_PORT = 10000
EVENT_LOOP_POLICY = uvloop
PEER_DISCOVERY = self
EOF
mkdir -p /electrumdb/VRSC && chown electrumx:electrumx /electrumdb/VRSC
```

Make sure the VerusCoin wallet is running. You should now be able to start ElectrumX (as `root`, it will switch to `electrumx` user) successfully: 

```
systemd start electrumx
```

Display the logs with this command: 

```
journalctrl -fu electrumx.service
```

Initial sync will take up to 2 hours to complete. Before that is done, ElectrumX will only allow RPC connections via loopback, but no external connections. To check ElectrumX status, do:

```
electrumx_rpc getinfo
```


## Further considerations

None of the topics below are strictly necessary, but most of them are recommended.

### Improving SSH security

If you remember the good old `rand=4; // chosen by fair dice roll` comic, you're probably doing this anyway. If you don't go google the comic, you might have missed a laugh there!

As `root`, generate a proper `/etc/ssh/moduli` like this:

```
ssh-keygen -G "/root/moduli.candidates" -b 4096
mv /etc/ssh/moduli /etc/ssh/moduli.old
ssh-keygen -T /etc/ssh/moduli -f "/root/module.candidates"
rm "/root/moduli.candidates"
```

Add the recommended changes from [CiperLi.st](https://cipherli.st) to `/etc/ssh/sshd_config`, also make sure that `PermitRootLogin` is at least set to `without-password`. Then remove and re-generate your host keys like this: 

```
cd /etc/ssh
rm ssh_host_*key*
ssh-keygen -t ed25519 -f ssh_host_ed25519_key < /dev/null
ssh-keygen -t rsa -b 4096 -f ssh_host_rsa_key < /dev/null
```

To finish, restart the ssh server: 

```
/etc/init.d/sshd restart
```

### Enable `logrotate` 

As `root` user, create a file called `/etc/logrotate.d/pool` with these contents: 

```
/home/veruscoin/.komodo/VRSC/debug.log
{
  rotate 14
  daily
  compress
  delaycompress
  copytruncate
  missingok
  notifempty
}
```

### Autostart using `cron`

Switch to the `veruscoin` user. Edit the `crontab` using `crontab -e` and include the lines below:

```
@reboot /home/veruscoin/bin/komodod -ac_name=VRSC -ac_algo=verushash -ac_cc=1 -ac_veruspos=50 -ac_supply=0 -ac_eras=3 -ac_reward=0,38400000000,2400000000 -ac_halving=1,43200,1051920 -ac_decay=100000000,0,0 -ac_end=10080,226080,0 -ac_timelockgte=19200000000 -ac_timeunlockfrom=129600 -ac_timeunlockto=1180800 -addnode=185.25.48.236 -addnode=185.64.105.111 -daemon 1>/dev/null 2>&1
```

Switch to the `s-nomp` user. Edit the `crontab` using `crontab -e` and include the line below:

```
@reboot /bin/sleep 60 && cd /home/s-nomp/s-nomp && /usr/bin/pm2 start init.js --name s-nomp
```

### Simplify wallet usage

Switch to the `veruscoin` user. Create a file called `/home/veruscoin/bin/veruscoind` that looks like this: 

```
#!/bin/bash
OLDPWD="$(pwd)"
cd /home/veruscoin/.komodo/VRSC
/home/veruscoin/bin/komodod -ac_name=VRSC -ac_algo=verushash -ac_cc=1 -ac_veruspos=50 -ac_supply=0 -ac_eras=3 -ac_reward=0,38400000000,2400000000 -ac_halving=1,43200,1051920 -ac_decay=100000000,0,0 -ac_end=10080,226080,0 -ac_timelockgte=19200000000 -ac_timeunlockfrom=129600 -ac_timeunlockto=1180800 -addnode=185.25.48.236 -addnode=185.64.105.111 ${@}
cd "${OLDPWD}"
```

Create another file called `/home/veruscoin/bin/veruscoin-cli` that looks like this: 

```
#!/bin/bash
/home/veruscoin/bin/komodo-cli -ac_name=VRSC ${@}
```

Make both files executable: 

```
chmod +x /home/veruscoin/bin/veruscoin*
```

From now on, any time you would have needed to use the huge `komodod` or `komodo-cli` commands, you can just use them as shown below: 

```
veruscoind -daemon 1>/dev/null 2>&1
veruscoin-cli addnode 1.2.3.4 onetry
```

### Increase open files limit

Add this to your `/etc/security/limits.conf`: 

```
* soft nofile 1048576
* hard nofile 1048576
```

Reboot to activate the changes. Alternatively you can make sure all running processes are restarted from within a shell that has been launched _after_ the above changes were put in place.


### Networking optimizations

If your pool is expected to receive a lot of load, consider implementing the below changes, all as `root`:

Enable the `tcp_bbr` kernel module: 

```
modprobe tcp_bbr
echo tcp_bbr >> /etc/modules
```

Edit your `/etc/sysctl.conf` to include below settings: 

```
net.ipv4.tcp_congestion_control=bbr
net.core.rmem_default = 1048576
net.core.wmem_default = 1048576
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.udp_rmem_min = 16384
net.ipv4.udp_wmem_min = 16384
net.core.netdev_max_backlog = 262144
net.ipv4.tcp_max_orphans = 262144
net.ipv4.tcp_max_syn_backlog = 262144
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_tw_buckets = 2000000
net.ipv4.ip_local_port_range = 16001 65530
net.core.somaxconn = 20480
net.ipv4.tcp_low_latency = 1
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_mtu_probing = 1
net.ipv4.tcp_fastopen = 3
net.ipv4.tcp_limit_output_bytes = 131072
```

Run this command to activate the changes, or reboot the machine: 


```
sysctl -p /etc/sysctl.conf
```

### Change swapping behaviour

If your system has a lot of RAM you can change the swapping behaviour to only swap when necessary. Edit `/etc/sysctl.conf` to include this setting: 

```
vm.swappiness=1
```

The range is `1-100`. The *lower* the number, the *later* the system will start swapping stuff out. Run this command to activate the change, or reboot the machine: 

```
sysctl -p /etc/sysctl.conf
```

### Install `molly-guard`

As a last sanity check before reboots, `molly-guard` will prompt you for the hostname of the system you're about to reboot. Install it like this: 

```
apt -y install molly-guard
```

Check `/etc/molly-guard/rc` for more options.
