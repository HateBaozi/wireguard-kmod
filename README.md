# WireGuard kernel module for UDM/UDM pro
## Project Notes
**Author:** Carlos Talbot (@tusc69 on ubnt forums)

The tar file in this repository is a collection of binaries that can be loaded onto a UDM/UDM Pro to run WireGuard in kernel mode. WireGuard is a high performance vpn solution developed by Jason Donenfeld ( https://www.wireguard.com/ ). Since the UDM runs an older kernel (4.1.37), the latest WireGuard backport has been provided. If you want to compile your own version I plan to have a seperate page up shortly. This was built from the GPL sources Ubiquiti sent me. I have a seperate github page for the UDM source code: https://github.com/tusc/UDM-source-code/blob/main/README.md

We first need to download the tar file onto the UDM. Connect to it via SSH and type the following command to download the tar file. You need to download the following:

```
# curl -LJo wireguard-kmod.tar.Z https://github.com/tusc/wireguard-kmod/raw/main/wireguard-kmod-4-7-21.tar.Z
```

From this directory type the following, it will extract the files under the /mnt/data path:

```
# tar -C /mnt/data -xvzf wireguard-kmod.tar.Z
```

Once the extraction is complete, cd into /mnt/data/wireguard and run the script **setup_wireguard.sh** as shown below
```
# ./setup_wireguard.sh
loading wireguard...
```


This will setup the symbolic links for the various binaries to the /usr/bin path as well as create a symlink for the /etc/wireguard folder and finally load the kernel module. You'll want to run **dmesg** to verify the kernel module was loaded. You should see something like the following: 
```
[13540.520120] wireguard: WireGuard 1.0.20210219 loaded. See www.wireguard.com for information.
[13540.520126] wireguard: Copyright (C) 2015-2019 Jason A. Donenfeld <Jason@zx2c4.com>. All Rights Reserved.
```

**Please Note: you will need to run setup_wireguard.sh whenever the UDM is rebooted as the symlinks have to be recreated.** Boostchicken has a script you can use to automatically run the wireguard script anytime the router is rebooted. https://github.com/boostchicken/udm-utilities/tree/master/on-boot-script

## Configuration
There's a sample WireGuard config file in /etc/wireguard you can use to create your own, provided you update the public and private keys. There are various tutorials out there for setting up a client/server config for WireGuard (e.g. https://www.stavros.io/posts/how-to-configure-wireguard/ ). A typical config might be to allow remote access to your internal LAN over the WAN from a mobile phone or romaing laptop. For the purpose of this example, the UDM is the server and the phone/laptop the client. For this you would need to setup a config file on the UDM similar to the following:

```
[Interface]
Address = 192.168.2.1
PrivateKey = <server's privatekey>
ListenPort = 51820

[Peer]
PublicKey = <client's publickey>
AllowedIPs = 192.168.2.2/32
```

The corresponding config on the phone/laptop (client) would look like this:

```
Address = 192.168.2.2
PrivateKey = <client's privatekey>
ListenPort = 21841

[Peer]
PublicKey = <server's publickey>
Endpoint = <server's ip>:51820
AllowedIPs = 192.168.2.0/24

# This is for if you're behind a NAT and
# want the connection to be kept alive.
PersistentKeepalive = 25
```

You'll need to generate keys on both systems. This can be done with the following command:

```
wg genkey | tee privatekey | wg pubkey > publickey
```

Finally, don't forget to open a port on the firewall in order to allow remote access to the wireguard link. You'll want to create this rule on the UDM under the **WAN LOCAL** section of the firewall settings. The default port is 51820 which can be adjusted in the UDM's config file, just make sure to update the firewall rule accordingly.
## Start tunnel
Once you have a properly configured conf file, you need to run this command from the cli:

```
# wg-quick up wg0
```

you should see output similar to the following:

```
[#] ip link add wg0 type wireguard
[#] wg setconf wg0 /dev/fd/63
[#] ip -4 address add 10.10.10.1/24 dev wg0
[#] ip link set mtu 1420 up dev wg0
```

You can also execute the wg binary for status on the tunnel:

```
# wg
interface: wg0
  public key: XXXXXXXXXXXXX
  private key: (hidden)
  listening port: 51820

peer: XXXXXXXXXXXX
  endpoint: 192.168.1.191:40396
  allowed ips: 10.10.10.2/32
  latest handshake: 47 seconds ago
  transfer: 3.26 GiB received, 46.17 MiB sent
```

 Finally, in order to shutdown the tunnel you'll need to run this command:
 
```
# wg-quick down wg0
```

I'm currently testing throughput using iperf3 between a UDM Pro and an Ubuntu client over 10Gb. With the UDM as the iperf3 server I'm seeing up to 1.5Gb/sec.

