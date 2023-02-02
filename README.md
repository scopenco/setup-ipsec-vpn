Setup IPsec IKEv1 with PSK and Xauth in openwrt 22.03
==

- [Setup IPsec IKEv1 with PSK and Xauth in openwrt 22.03](#setup-ipsec-ikev1-with-psk-and-xauth-in-openwrt-2203)
- [Overview](#overview)
- [Installation](#installation)
- [Configure StrongSwan](#configure-strongswan)
  - [Create /etc/ipsec.conf](#create-etcipsecconf)
  - [Create /etc/ipsec.secrets](#create-etcipsecsecrets)
  - [Restart ipsec](#restart-ipsec)
- [Configure Netfilter](#configure-netfilter)
  - [Create /etc/fwuser.nft](#create-etcfwusernft)
  - [Add to /etc/config/firewall](#add-to-etcconfigfirewall)
  - [Restart firewall](#restart-firewall)
- [Configure DNS Resolver](#configure-dns-resolver)
  - [Create /etc/dnsmasq.conf](#create-etcdnsmasqconf)
- [Troubleshooting](#troubleshooting)
- [Client Configuration](#client-configuration)
  - [Android](#android)
  - [iOS](#ios)

# Overview

Although it's not recommended for large scale IPsec deployments because the Pre-Shared Key must be shared among users, IKEv1 with PSK and Xauth is an easy-to-deploy option and is well supported by mobile devices powered by iOS and Android.

In this tutorial, we'll install strongSwan 5.9 in openwrt 22.03, configure IKEv1 with PSK and Xauth, DNS resolver, and finally setup the built-in VPN clients in Android and iOS so they can connect to it.

Prerequisites:
- 192.168.2.0/29 is out VPN network
- 192.168.1.1 - router internal ip

# Installation
```bash
## update package list
opkg update

# install minimal set strongswan
opkg install strongswan-minimal strongswan-mod-xauth-generic

# remove unused dependencies
opkg remove strongswan-minimal strongswan-mod-updown \
        iptables-zz-legacy iptables-mod-ipsec
```

# Configure StrongSwan

## Create /etc/ipsec.conf
```
# ipsec.conf - strongSwan IPsec configuration file
config setup
    # in case you need to debug
    #charondebug="cfg 2, dmn 2, ike 2, net 2"

conn %default
    keyexchange=ikev1
    ikelifetime=60m
    keylife=20m
    rekeymargin=3m
    keyingtries=1

conn roadwarrior
    left=%any
    leftauth=psk
    leftid=@my.lan
    leftsubnet=0.0.0.0/0,::/0

    right=%any
    rightsourceip=192.168.2.0/29
    rightauth=psk
    rightauth2=xauth
    rightdns=192.168.1.1
    auto=add
```

## Create /etc/ipsec.secrets
```
# /etc/ipsec.secrets - strongSwan IPsec secrets file
my.lan %any : PSK "WhObByeWasdnFe3sDs0sAs0"
user1 : XAUTH "Ka2asScd0"
user2 : XAUTH "Ea3a1sDsA"
```

## Restart ipsec
```bash
/etc/init.d/ipsec restart
```

# Configure Netfilter

## Create /etc/fwuser.nft
```
meta ipsec exists ip saddr 192.168.2.0/29 counter accept
```

## Add to /etc/config/firewall
```
# allow incoming IPsec connections in LuCI format
config rule
        option src 'wan'
        option name 'IPSec ESP'
        option proto 'esp'
        option target 'ACCEPT'

config rule
        option src 'wan'
        option name 'IPSec IKE'
        option proto 'udp'
        option dest_port '500'
        option target 'ACCEPT'

config rule
        option src 'wan'
        option name 'IPSec NAT-T'
        option proto 'udp'
        option dest_port '4500'
        option target 'ACCEPT'

config rule
        option src 'wan'
        option name 'Auth Header'
        option proto 'ah'
        option target 'ACCEPT'

config include
        option type 'nftables'
        option path '/etc/fwuser.nft'
        option position 'chain-pre'
        option chain 'input_wan'

config include
        option type 'nftables'
        option path '/etc/fwuser.nft'
        option position 'chain-pre'
        option chain 'forward_wan'
```

## Restart firewall
```bash
/etc/init.d/firewall restart
```

# Configure DNS Resolver

## Create /etc/dnsmasq.conf
```
# to disable default --local-service in dnsmasq. Otherise DNS will not resolve.
listen-address=90.190.8.10,192.168.1.1,127.0.0.1
```

# Troubleshooting
```bash
ipsec stop
ipsec start --nofork
ipsec status
ipsec statusall
logread && logread -f
```

# Client Configuration
## Android
Open `Settings / Wireless & networks / VPN`, tap the "+" sign in the upper-right corner of the Settings screen. On the `Edit VPN profile` dialog that pops up, enter the profile Name, select `IPSec Xauth PSK` in the `Type` drop-down menu, and then enter `Server address` and `IPSec pre-shared key`. Tap `SAVE`.
## iOS
Open `Settings / VPN`, tap `Add VPN Configuration...`. On the dialog that pops up, choose `IPSec` in the `Type` drop-down menu, and then tap `Back`. Enter all the necessary information: profile name in `Descrption`, server address in `Server`, username in `Account`, account password in `Password` and finally the PSK in `Secret`. Tap `Done`.
