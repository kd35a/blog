---
title: "Wireguard on macOS"
tags: ["wireguard", "macos"]
published: false
---
I was first trying to use the GUI [Wireguard](https://www.wireguard.com/) client
from [App Store](https://apps.apple.com/se/app/wireguard/id1451685025?l=en&mt=12),
but it was not working correctly - the DNS entry was not updated in the network
configuration.

To get a working setup, I instead followed [Ivan Smirnov's](https://ivans.io/wireguard-on-macos/)
solution, with some adjustments/details. Both for my own sake and others, I'll
document here what worked.

## Solution
First, install Wireguard CLI tools using [brew](https://brew.sh/):
```
$ brew install wireguard-tools wireguard-go
```

Now you want to add `PostUp`/`PostDown` commands to your Wireguard config, so
that it looks something like this:
```
[Interface]
Address = 10.0.10.8/24 # adapt for your case
PrivateKey = YOUR_PRIVATE_KEY
DNS = 10.0.0.x # adapt for your case

# Commands to set and clear DNS
PostUp = sudo /usr/sbin/networksetup -setdnsservers Wi-Fi 10.0.0.x # same as `DNS` above
PostDown = sudo /usr/sbin/networksetup -setdnsservers Wi-Fi "Empty"

[Peer]
PublicKey = PUBLIC_KEY_OF_PEER
PresharedKey = PRE-SHARED_KEY
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = wireguard.example.com:51820
```

Now for where to place the config: brew (at the time of writing this) has its
home under `/opt/homebrew`, which can be verified by running `brew --prefix`.
Therefore I first thought that the config file should reside under
~~`/opt/homebrew/etc/wireguard`~~, but this is wrong. It should be placed under
`/usr/local/etc/wireguard/`, in my case as `/usr/local/etc/wireguard/wg0.conf`.
```
$ sudo mkdir -p /usr/local/etc/wireguard/
$ sudo cp myconfig.conf /usr/local/etc/wireguard/wg0.conf
```

Now, lets try to start the tunnel:
```
$ sudo wg-quick up wg0
Password:
wg-quick: Version mismatch: bash 3 detected, when bash 4+ required
```

My first thought here was to just `brew install bash`, but I already had latest
`bash` installed. The issue here is that `/bin` was sitting before
`/opt/homebrew/bin` on my `PATH`. So after placing `/opt/homebrew/bin` first on
my `PATH` in `~/.zshrc`, I could rerun the above command and the tunnel was
working as expected.
```
$ sudo wg-quick up wg0
$ ping my.internal.server.local
PING my.internal.server.local (10.0.0.15): 56 data bytes
64 bytes from 10.0.0.15: icmp_seq=0 ttl=63 time=42.639 ms
64 bytes from 10.0.0.15: icmp_seq=1 ttl=63 time=38.798 ms
64 bytes from 10.0.0.15: icmp_seq=2 ttl=63 time=44.611 ms
^C
--- my.internal.server.local ping statistics ---
4 packets transmitted, 3 packets received, 25.0% packet loss
round-trip min/avg/max/stddev = 38.798/42.016/44.611/2.414 ms
$ sudo wg-quick down wg0 # when done
```
