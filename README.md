# Pihole-HA
Configuration of keepalived to create a HA cluster for Pihole with DNS and Unbound port tracking

## Description
**This can be used on 2 or more Piholes for a Hot-Active setup. Where only one Pihole is actively used by devices on the network, e.g devices use the Virtual IP (VIP).** 

It is achieved by using keepalived amongst several Piholes, where only the "PRIMARY" Pihole has the VIP assigned. In case of failure of the "PRIMARY" node, the VIP will be assigned to "SECONDARY" node with the highest priority.

Keepalived provides script functionality, which can be used for advanced tracking, and have the VRRP failovered in case DNS port or the UNBOUND port are not replying.

Following example is for 2 Pihole HA cluster


# Overall Step-by-Step
Assuming following IP/Subnet

```
Subnet:    192.168.1.0/24
PRIMARY:   192.168.1.1/24
SECONDARY: 192.168.1.2/24
VIP:       192.168.1.3/24
```
Assuming following VRRP prioprities. HIGHER Priority = HIGHER precendence to become "MASTER"
```
PRIMARY:   100
SECONDARY: 90
```
### Note down your IP & Interface on which Pihole is listening for DNS queries
This needs to be done on all Piholes, we will use the IPs & interfaces in the keepalived configuration
```
ip a
```
### Keepalived configuration
Keepalived config can be found in `/etc/keepalived/keepalived.conf`

## 1. Install keepalived on all Pihole nodes

```
sudo apt update
sudo apt install keepalived
```

## 2. Primary node
Configuration can be very simple, we can just configure keepalived to failover only in case "PRIMARY" Pihole is down or set advanced tracking to track the state of PORTs for DNS and UNBOUND.

### Primary node - Basic Config
This configuration, will failover only in case the "PRIMARY" Pihole will be down e.g the IP of the "PRIMARY" Pihole is not reachable
```
vrrp_instance pihole {
        interface eth0
        state MASTER
        virtual_router_id 100
        priority 100
        authentication {
                auth_type PASS
                auth_pass p4assw0rd!
        }
        virtual_ipaddress {
                192.168.1.3/24 dev eth0
}
```
#### Primary node - Adding DNS port Tracking
We can utilise the script function in keepalived config to track the state of a port, in this case the DNS UDP port 53. We use the IP local to the Pihole
```
global_defs {
    script_user root
    enable_script_security
}

vrrp_script dns_udp {
    script "nc -zvu 192.168.1.1 53"
    interval 5                   # default: 1s
}
```
#### Primary node - Adding UNBOUND port Tracking
We can do the same for UNBOUND if its used together with Pihole. We use the IP local to the Pihole's UNBOUND
```
vrrp_script dns_udp_unbound {
    script "nc -zvu 127.0.0.1 5335"
    interval 5                   # default: 1s
}
```

### Primary node - Full config with DNS & Unbound port tracking
This is how keepalived config looks with advanced tracking for DNS as well UNBOUND Port
```
global_defs {
    script_user root
    enable_script_security
}

vrrp_script dns_udp {
    script "nc -zvu 192.168.1.1 53"
    interval 5                   # default: 1s
}

vrrp_script dns_udp_unbound {
    script "nc -zvu 127.0.0.1 5335"
    interval 5                   # default: 1s
}

vrrp_instance pihole {
        interface eth0
        state MASTER
        virtual_router_id 100
        priority 100
        authentication {
                auth_type PASS
                auth_pass p4assw0rd!
        }
        virtual_ipaddress {
                192.168.1.3/24 dev eth0
        }
        track_script {
             dns_udp
             dns_udp_unbound
        }
}
```


## 3. Secondary node
Follows the configuration of the Primary with adjusted lower priority. Here is no need to configure advanced tracking, as it is designed to be "SECONDARY". In case of 2+ node Cluster, you would configure advanced tracking on "SECONDARY" node and all subsequent nodes except the last one.

### Secondary node - Basic Config
This configuration makes the "SECONDARY" as BACKUP, it will became "MASTER" only in case the "PRIMARY" fail, e.g "PRIMARY" will announce lower priority than has the BACKUP
```
vrrp_instance pihole {
        interface eth0
        state BACKUP
        virtual_router_id 100
        priority 90
        authentication {
                auth_type PASS
                auth_pass p4assw0rd!
        }
        virtual_ipaddress {
                192.168.1.3/24 dev eth0
}
```

#### Secondary node - Full config with DNS & Unbound port tracking
This is how keepalived config looks with advanced tracking for DNS as well UNBOUND Port in case the 2+ node Cluster
```
global_defs {
    script_user root
    enable_script_security
}

vrrp_script dns_udp {
    script "nc -zvu 192.168.1.2 53"
    interval 5                   # default: 1s
}

vrrp_script dns_udp_unbound {
    script "nc -zvu 127.0.0.1 5335"
    interval 5                   # default: 1s
}

vrrp_instance pihole {
        interface eth0
        state BACKUP
        virtual_router_id 100
        priority 90
        authentication {
                auth_type PASS
                auth_pass p4assw0rd!
        }
        virtual_ipaddress {
                192.168.1.3/24 dev eth0
        }
        track_script {
             dns_udp
             dns_udp_unbound
        }
}
```
## 4. Enable keepalived
**In order for keepalived to be active you need to enable it on each Pihole**
```
sudo systemctl enable --now keepalived.service
```

### Keepalived START/STOP/RESTART/STATUS

If configuration on keepalived is adjusted it needs to be restarted
```
sudo systemctl restart keepalived.service
```
In case keepalived needs to be stoped
```
sudo systemctl stop keepalived.service
```
In case keepalived needs to be started
```
sudo systemctl start keepalived.service
```
In case keepalived status a logs needs to be viewed
```
sudo systemctl status keepalived.service
```

## 5. Verification of keepalived status and functionality
To check if keepalived is working properly it needs to be in active(running) state and 4 things should be observed. 
Run the command `sudo systemctl status keepalived.service` & `ip a` on the Piholes.

1. The "PRIMARY" node will be in MASTER state
   ```
   Entering MASTER STATE
   ```
3. The "SECONDARY" node will be in BACKUP state
   ```
   Entering BACKUP STATE
   ```
3. The advenced tracking Scripts will be showen in the log
      ```
   VRRP_Script(dns_udp) succeeded
   VRRP_Script(dns_udp_unbound) succeeded
   ```
4. The VIP 192.168.1.3 will be seen on the MASTER node
   ```
   eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
   inet 192.168.1.1/24 brd 192.168.1.255 scope global noprefixroute eth0
      valid_lft forever preferred_lft forever
   inet 192.168.1.3/24 scope global secondary eth0
      valid_lft forever preferred_lft forever
   ```
