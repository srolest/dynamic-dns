# README: DYNAMIC DNS (DDNS)

## Practice Environment

The environment is defined in practice (we will see it below), and we will do so using a private network `192.168.58.0/24`.

[ENVIRONMENT](./images/environment.png)

The virtual machines (`Vagrantfile`) and their roles are as follows:

| Machine | Rol | Static IP (Vagrantfile) | Network |
| :--- | :--- | :--- | :--- |
| `dns` | DNS Server | `192.168.58.10` | Private Network |
| `dhcp` | DHCP Server | `192.168.58.30` | `redinterna1` |
| `c1` | Client | (Obtains IP by DHCP) | `redinterna1` |

The goal is that when client `c1` requests an IP address, the DHCP server (`dhcp`) automatically tells the DNS server (`dns`) what name and IP address that client has. This is called **Dynamic DNS** or **DDNS**.

To set everything up, I used:
* **Vagrant:** To create the three virtual machines.
* **Ansible:** To automatically install and configure the DNS and DHCP servers with playbooks (`playbook_dns.yaml` and `playbook_dhcp.yaml`).

---

##  How does it work?

1.  **The client requests an IP address:** Client `c1` powers up and requests an IP address from the `dhcp` server.
2.  **The DHCP responds and notifies the DNS:** The `dhcp` server gives the client an IP address from the established range, in this case `192.168.58.100`. Immediately afterwards, it sends a notification to the `dns` server (located at `192.168.58.10`) using the secret key to sign the message.
3.  **The DNS updates its records:** The `dns` server receives the notification, checks that the secret key is correct, and updates its lists. Now it knows that the name `c1.serafin.test` corresponds to the IP address `192.168.58.100`.

---

##  Steps for Implementation

To implement the practice, I followed these four steps:

1.  **Start the Virtual Machines**
    With the `Vagrantfile` already defined, run the command to create and start the three machines (`dns`, `dhcp`, and `c1`).

    ```bash
    vagrant up
    ```

2.  **Configure the DNS Server**
    First, I run `playbook_dns.yaml` because the DHCP server needs to send updates to the DNS server. So the DNS has to be installed, configured, and running *before* DHCP tries to contact it. 
    This installs BIND9, configures the zone files (direct `serafin.test.dns` and reverse `serafin.test.rev`) and, most importantly, prepares the DNS to accept updates if they are signed with the secret key `ddns.key`.

    ```bash
    ansible-playbook playbook_dns.yaml
    ```

3.  **Configure the DHCP Server**
    Then I run the second playbook. This installs the DHCP server, gives it the same secret key (`ddns.key`) and configures it (`dhcpd.conf`) so that it knows two things: which IPs it can distribute (range `192.168.58. 100` to `192.168.58.200`) and that it must notify the DNS (`192.168.58.10`) every time it assigns one.

    ```bash
    ansible-playbook playbook_dhcp.yaml
    ```

4.  **Activate the Client Request**
    To do this, I connect to the client machine `c1` and force an IP request. This is where the DNS process begins.
    ```bash
    vagrant ssh c1
    # Forzamos la petición de IP
    sudo dhclient -r eth1
    ```

---

## (Step-by-Step Tests)

Here I show how it works right after executing step 4.

### 1. Client `c1` receives its IP
First, we check that client `c1` has received the IP we expected from the range.

* **Machine:** `c1`
* **Command:** `ip a show eth1`

```bash
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:24:2a:56 brd ff:ff:ff:ff:ff:ff
    altname enp0s8
    inet 192.168.58.100/24 brd 192.168.58.255 scope global dynamic eth1
       valid_lft 85559sec preferred_lft 85559sec
    inet6 fe80::a00:27ff:fe24:2a56/64 scope link 
       valid_lft forever preferred_lft forever
```

### 2. DHCP Server Logs
Now, on the `dhcp` server, we review the logs to see if it has sent the update to the DNS.

* **Machine:** `dhcp`
* **Command:** `sudo journalctl -u isc-dhcp-server -n 10`

```bash
...
Nov 19 18:18:06 dhcp dhcpd[384]: DHCPDISCOVER from 08:00:27:24:2a:56 via eth1
Nov 19 18:18:07 dhcp dhcpd[384]: DHCPOFFER on 192.168.58.100 to 08:00:27:24:2a:56 (c1) via eth1
Nov 19 18:18:07 dhcp dhcpd[384]: DHCPREQUEST for 192.168.58.100 (192.168.58.30) from 08:00:27:24:2a:56 (c1) via eth1
Nov 19 18:18:07 dhcp dhcpd[384]: DHCPACK on 192.168.58.100 to 08:00:27:24:2a:56 (c1) via eth1
Nov 19 18:18:07 dhcp dhcpd[384]: Added new forward map from c1.serafin.test. to 192.168.58.100
Nov 19 18:18:07 dhcp dhcpd[384]: Added reverse map from 100.58.168.192.58.168.192.in-addr.arpa. to c1.serafin.test.
```

### 3. DNS Server Logs
Let's go to the `dns` server to see if it received the DHCP request and if it accepted it.

* **Máquina:** `dns`
* **Comando:** `journalctl -u bind9 -n 10`

```bash
...
named[...]: client @0x... 192.168.58.30#12345/key ddns-key: signer "ddns-key" approved
named[...]: client @0x... 192.168.58.30#12345/key ddns-key: updating zone 'serafin.test/IN': adding an RR at 'c1.serafin.test' A 192.168.58.100
named[...]: client @0x... 192.168.58.30#12345/key ddns-key: signer "ddns-key" approved
named[...]: client @0x... 192.168.58.30#12345/key ddns-key: updating zone '58.168.192.in-addr.arpa/IN': adding an RR at '100.58.168.192.in-addr.arpa' PTR c1.serafin.test.
```

### 4. Test with `dig`
We check on the `dns` server itself to see if it is already aware of the change. We do not do this from outside the dns machine, as this would require connecting to the internet with another network adapter with a different IP address. We check from the `dns` virtual machine.

* **Machine:** `dns`
* **Command (Direct):** `dig @127.0.0.1 c1.serafin.test`
* **Command (Reverse):** `dig @127.0.0.1 -x 192.168.58.100`

[DIG_DIRECT_ZONE](./images/dig.png)

```bash
; <<>> DiG 9.16.50-Debian <<>> @127.0.0.1 -x 192.168.58.100
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 30322
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
; COOKIE: ce2d2a4e6c6094f501000000691e2e84b77649d0c9319003 (good)
;; ANSWER SECTION:
100.58.168.192.in-addr.arpa. 3600 IN	PTR	c1.serafin.test.

;; Query time: 0 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Wed Nov 19 20:54:28 UTC 2025
;; MSG SIZE  rcvd: 141
```

---

## Key Project Files
* `Vagrantfile`: Defines and creates the three virtual machines.
* `playbook_dns.yaml`: Ansible playbook that configures the `dns` server.
* `playbook_dhcp.yaml`: Ansible playbook that configures the `dhcp` server.
* `ddns.key`: The secret key shared by both servers.
* `dhcpd.conf`: File that tells the DHCP how to update the DNS.
* `named.conf.local`: File that tells the DNS to accept updates from the DHCP.