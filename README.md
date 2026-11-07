# Network lab: DHCP, DNS and Dynamic DNS

Part of the Linux Administration project. This branch covers the network configuration of the lab: a DHCP server (`isc-dhcp-server`) and a DNS server (`BIND9`) on the same machine, linked with Dynamic DNS (DDNS). When a VM gets an IP address, its DNS records are created automatically, so every machine can be reached by hostname.

## How it works

1. A client asks for an IP address through DORA (Discover, Offer, Request, Acknowledge) and sends its hostname.
2. The DHCP server gives it an address from `192.168.50.100` to `192.168.50.200`.
3. The DHCP server sends a signed update to BIND9, using a shared TSIG key.
4. BIND9 verifies the signature, then adds the A record (name to IP) to the forward zone and the PTR record (IP to name) to the reverse zone.
5. When the lease ends, both records are removed.

## Lab environment

| VM | Role | Network |
|---|---|---|
| `ubuntu-net-server` | DHCP server and DNS server | static `192.168.50.10` |
| `ubuntu-server`, `ubuntu-client-1`, `ubuntu-client-2` | DHCP clients | address from DHCP |

- Every VM has two adapters: NAT (internet access) and Internal Network (`192.168.50.0/24`, the lab network).
- Domain: `ilsi.local`
- Forward zone: `ilsi.local`
- Reverse zone: `50.168.192.in-addr.arpa`

## Files

| File | VM | Location |
|---|---|---|
| `named.conf.local` | `ubuntu-net-server` | `/etc/bind/named.conf.local` |
| `db.ilsi.local` | `ubuntu-net-server` | `/var/lib/bind/db.ilsi.local` |
| `db.192.168.50` | `ubuntu-net-server` | `/var/lib/bind/db.192.168.50` |
| `dhcpd.conf` | `ubuntu-net-server` | `/etc/dhcp/dhcpd.conf` |
| netplan (server) | `ubuntu-net-server` | `/etc/netplan/*.yaml` |
| netplan (client) | the clients | `/etc/netplan/*.yaml` |
| `ddns.key` | `ubuntu-net-server` | `/etc/bind/` and `/etc/dhcp/` (never published) |

## 1. Create the TSIG key

BIND9 and the DHCP server share a secret key. Generate it once, then put the same key in BIND9 and in DHCP:

```bash
sudo tsig-keygen -a hmac-sha256 ddns-key | sudo tee /etc/bind/ddns.key
sudo chown root:bind /etc/bind/ddns.key
sudo chmod 640 /etc/bind/ddns.key
```

The DHCP server gets its own copy, because AppArmor allows `dhcpd` to read `/etc/dhcp/` but not `/etc/bind/`:

```bash
sudo cp /etc/bind/ddns.key /etc/dhcp/ddns.key
sudo chown root:root /etc/dhcp/ddns.key
sudo chmod 640 /etc/dhcp/ddns.key
```

The key file looks like this. The secret is different on every installation, so it is not part of this repository:

```
key "ddns-key" {
    algorithm hmac-sha256;
    secret "<GENERATED-SECRET>";
};
```

## 2. Configure BIND9

### `/etc/bind/named.conf.local`

```
//
// Do any local configuration here
//

include "/etc/bind/ddns.key";

zone "ilsi.local" {
	type master;
	file "/var/lib/bind/db.ilsi.local";
	allow-update { key ddns-key; };
};

zone "50.168.192.in-addr.arpa" {
	type master;
	file "/var/lib/bind/db.192.168.50";
	allow-update { key ddns-key; };
};
```

The zone files are stored in `/var/lib/bind` and not in `/etc/bind`, because BIND9 must write a journal file (`.jnl`) next to a dynamic zone, and AppArmor only allows that in `/var/lib/bind`.

### `/var/lib/bind/db.ilsi.local` (forward zone, default content)

```
$TTL 86400	; 1 day
ilsi.local.		IN SOA	ubuntu-net-server.ilsi.local. admin.ilsi.local. (
				1          ; serial
				3600       ; refresh (1 hour)
				1800       ; retry (30 minutes)
				604800     ; expire (1 week)
				86400      ; minimum (1 day)
				)
			NS	ubuntu-net-server.ilsi.local.
ubuntu-net-server.ilsi.local. A	192.168.50.10
```

### `/var/lib/bind/db.192.168.50` (reverse zone, default content)

```
$TTL 86400	; 1 day
50.168.192.in-addr.arpa. IN SOA	ubuntu-net-server.ilsi.local. admin.ilsi.local. (
				1          ; serial
				3600       ; refresh (1 hour)
				1800       ; retry (30 minutes)
				604800     ; expire (1 week)
				86400      ; minimum (1 day)
				)
			NS	ubuntu-net-server.ilsi.local.
10.50.168.192.in-addr.arpa. PTR	ubuntu-net-server.ilsi.local.
```

Only the server's own records are written by hand. The records of the clients are added by the DHCP server when they get a lease.

Give BIND9 ownership of the zone files:

```bash
sudo chown bind:bind /var/lib/bind/db.ilsi.local /var/lib/bind/db.192.168.50
```

## 3. Configure DHCP

### `/etc/dhcp/dhcpd.conf`

```
authoritative;
default-lease-time 600;
max-lease-time 7200;

# Dynamic DNS
ddns-updates on;
ddns-update-style interim;
ddns-domainname "ilsi.local.";
ddns-rev-domainname "in-addr.arpa.";
update-conflict-detection off;

include "/etc/dhcp/ddns.key";

zone ilsi.local. {
    primary 127.0.0.1;
    key ddns-key;
}

zone 50.168.192.in-addr.arpa. {
    primary 127.0.0.1;
    key ddns-key;
}

# Lab network
subnet 192.168.50.0 netmask 255.255.255.0 {
   range 192.168.50.100 192.168.50.200;
   option subnet-mask 255.255.255.0;
   option broadcast-address 192.168.50.255;
   option domain-name "ilsi.local";
   option domain-name-servers 192.168.50.10;
}
```

The file shown here keeps only the active settings. The default Ubuntu file also contains many commented examples, which are left out.

- `primary 127.0.0.1` is the DNS server that receives the updates. It is the loopback address, because BIND9 runs on the same machine.
- `key ddns-key` is the TSIG key used to sign every update.
- `option domain-name-servers 192.168.50.10` tells the clients to use `ubuntu-net-server` as their DNS server.

## 4. Network configuration (netplan)

After editing a netplan file, apply it with `sudo netplan apply`. The MAC address is specific to each VM and has to be replaced.

### Server (`ubuntu-net-server`)

```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp6: true
      match:
        macaddress: <MAC-OF-THE-NAT-ADAPTER>
      set-name: enp0s3
    enp0s8:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.50.10/24
      nameservers:
        search: [ilsi.local, "~50.168.192.in-addr.arpa"]
  version: 2
```

The server has a static address on the internal network (`enp0s8`), because it gives out the addresses and cannot ask for one itself.

### Clients (`ubuntu-server`, `ubuntu-client-1`, `ubuntu-client-2`)

```yaml
network:
  ethernets:
    enp0s3:
      dhcp4: true
      dhcp6: true
      match:
        macaddress: <MAC-OF-THE-NAT-ADAPTER>
      set-name: enp0s3
    enp0s8:
      dhcp4: true
      dhcp6: true
      set-name: enp0s8
      nameservers:
        search: [ilsi.local, "~50.168.192.in-addr.arpa"]
  version: 2
```

On the clients, `enp0s8` gets its address from DHCP. The `search` line makes the DNS domain permanent: names ending in `ilsi.local`, and reverse lookups in `50.168.192.in-addr.arpa`, are sent to the DNS server of this interface (`192.168.50.10`). The `~` marks a routing-only domain, so it is not added to short names.

Ubuntu Desktop clients use NetworkManager, which creates a DHCP profile for the internal adapter automatically, so they need no netplan file.

## 5. Apply and test

Check the files, then restart the services:

```bash
sudo named-checkconf
sudo named-checkzone ilsi.local /var/lib/bind/db.ilsi.local
sudo named-checkzone 50.168.192.in-addr.arpa /var/lib/bind/db.192.168.50
sudo systemctl restart named

sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf
sudo systemctl restart isc-dhcp-server
```

Start a client, so it asks for an address, then test from any machine of the lab:

```bash
ping -c 2 ubuntu-client-2.ilsi.local
dig @192.168.50.10 ubuntu-client-2.ilsi.local +short
dig @192.168.50.10 -x <client-ip> +short
```

The first `dig` prints the IP of the client, and the second one prints its name. The warning `.local is reserved for Multicast DNS` can be ignored.

## Notes

- A dynamic zone must not be edited by hand while BIND9 is running. To change it, freeze it first (`sudo rndc freeze ilsi.local`), edit the file, then thaw it (`sudo rndc thaw ilsi.local`). Records can also be added with `nsupdate -k /etc/bind/ddns.key`.
- Each VM needs a unique hostname, because the DNS record is created from the hostname the client sends.
- DNS answers are cached on the clients for the TTL of the record. If a name is not found right after a change, run `sudo resolvectl flush-caches` on the machine that shows the wrong answer.

## Problems met and their fixes

| Problem | Cause | Fix |
|---|---|---|
| BIND9 does not start, `permission denied` on `ddns.key` | wrong owner or mode on the key file | `sudo chown root:bind /etc/bind/ddns.key` and `sudo chmod 640 /etc/bind/ddns.key` |
| Reverse lookups return `NXDOMAIN` although the zone is loaded | typo in the zone name (`in-addr-arpa` instead of `in-addr.arpa`) | correct the name in `named.conf.local` and restart BIND9 |
| The network interfaces are `failed` on a VM | `/etc/machine-id` is missing, so the DHCP client cannot start | `sudo systemd-machine-id-setup` and `sudo systemctl restart systemd-networkd` |
| `dig @192.168.50.10 ...` works but `ping name` fails | the DNS domain is missing on the internal interface | add the `search` line to the netplan file and apply it |

## License

Copyright (C) 2026 Zaayd

This project is free software: you can redistribute it and/or modify it under the terms of the GNU General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

See the `LICENSE` file for the full text.
