# README: DYNAMIC DNS (DDNS)

## Practice Environment

The environment is defined in practice (we will see it below), and we will do so using a private network `192.168.58.0/24`.

[Diagrama del Escenario](./images/environment.png)

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

* **Máquina:** `dhcp`
* **Comando:** `sudo journalctl -u isc-dhcp-server -n 10`

```bash
...
Nov 19 18:18:06 dhcp dhcpd[384]: DHCPDISCOVER from 08:00:27:24:2a:56 via eth1
Nov 19 18:18:07 dhcp dhcpd[384]: DHCPOFFER on 192.168.58.100 to 08:00:27:24:2a:56 (c1) via eth1
Nov 19 18:18:07 dhcp dhcpd[384]: DHCPREQUEST for 192.168.58.100 (192.168.58.30) from 08:00:27:24:2a:56 (c1) via eth1
Nov 19 18:18:07 dhcp dhcpd[384]: DHCPACK on 192.168.58.100 to 08:00:27:24:2a:56 (c1) via eth1
Nov 19 18:18:07 dhcp dhcpd[384]: Added new forward map from c1.serafin.test. to 192.168.58.100
Nov 19 18:18:07 dhcp dhcpd[384]: Added reverse map from 100.58.168.192.58.168.192.in-addr.arpa. to c1.serafin.test.
```