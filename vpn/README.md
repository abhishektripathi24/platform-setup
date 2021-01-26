# OpenVPN [Single Node Setup]
<img src="https://github.com/abhishektripathi24/platform-setup/blob/master/vpn/images/openvpn-logo.png" width="550" height="120"/>

To know more about openvpn, visit https://openvpn.net/

## Overview
From the official docs -

> OpenVPN allows you to tunnel any IP subnetwork or virtual ethernet adapter over a single UDP or TCP port.

## Setup
Installation of `OpenVPN 2.4.4` on `Ubuntu 18.04.3 LTS` - [ref](https://github.com/Nyr/openvpn-install)

1. Install OpenVPN - [ref](https://github.com/Nyr/openvpn-install)
    ```bash
    # Download, run the script and following installation steps 
    wget https://git.io/vpn -O openvpn-install.sh && bash openvpn-install.sh
    
    # Verify Installation
    sudo systemctl stop openvpn-server@server.service
    sudo systemctl start openvpn-server@server.service
    sudo systemctl restart openvpn-server@server.service
    sudo systemctl status openvpn-server@server.service
    ```

2. To configure LDAP based authentication for vpn users, install OpenVPN LDAP plugin
    ```bash
    apt install openvpn-auth-ldap
    ```
   
3. Copy sample and update `ldap.conf`  
    ```bash
    # Copy the sample conf that came with installation to auth directory
    cp /usr/share/doc/openvpn-auth-ldap/examples/auth-ldap.conf /etc/openvpn/auth/ldap.conf

    # Update ldap.conf as follows
    vim /etc/openvpn/auth/ldap.conf
        <LDAP>
            URL		        ldap://ldap.blueleaf.com:389
            BindDN		    cn=vpn@blueleaf.com,ou=applications,dc=blueleaf,dc=com 
            Password	    P@ssW0rd
            Timeout		    15
            TLSEnable	    no
            FollowReferrals no
        </LDAP>
        <Authorization>
            BaseDN		    "ou=users,dc=blueleaf,dc=com"
            SearchFilter	"(uid=%u)"
            RequireGroup	true
            <Group>
                BaseDN          "ou=vpn,ou=groups,dc=blueleaf,dc=com"
                SearchFilter    "(cn=user)"
                MemberAttribute member
            </Group>
        </Authorization>
    ```

4. Update `server.conf` as the sample shown:
    ```bash
   ;local <vpn-server's-private-ip>
   port 1194
   proto udp
   dev tun
   
   ca ca.crt
   cert server.crt
   key server.key
   dh dh.pem
   
   auth SHA512
   tls-crypt tc.key
   cipher AES-256-CBC
   
   ;topology subnet
   server 10.8.0.0 255.255.255.0
   ifconfig-pool-persist ipp.txt
   
   ;push "redirect-gateway def1 bypass-dhcp"
   ;push "dhcp-option DNS <x.x.x.x>"
   
   keepalive 10 120
   
   user nobody
   group nogroup
   
   persist-key
   persist-tun
   
   status openvpn-status.log
   verb 3
   
   crl-verify crl.pem
   explicit-exit-notify
   
   # --------------
   # Extra Settings
   # --------------
   
   # Keys
   key-direction 0
   
   # Logs
   verb 4
   status /var/log/openvpn/openvpn-status.log
   log-append /var/log/openvpn/openvpn.log
   plugin /usr/lib/openvpn/openvpn-auth-ldap.so "/etc/openvpn/auth/ldap.conf"
   
   # Connections
   max-clients 1000
   
   # Increase Throughput/Bandwidth
   sndbuf 393216
   rcvbuf 393216
   push "sndbuf 393216"
   push "rcvbuf 393216"
   
   # MTU and MSSFIX
   tun-mtu 1500
   mssfix 1360
   
   # Allow multiple clients with the same common name to concurrently connect. In the absence of this option, OpenVPN will disconnect a client instance upon connection of a new client having the same common name.
   duplicate-cn
   
   # Push Subnet Routes: Ensure subnet IPs to be routed via this VPN from the client machine using this VPN
   push "route 10.6.0.0 255.255.0.0"
   push "route 10.7.0.0 255.255.0.0"
   push "route 10.8.0.0 255.255.0.0"
   push "route 10.11.16.0 255.255.255.0"
   push "route 10.12.17.0 255.255.255.0"
   push "route 10.13.18.0 255.255.255.0"
   ```

5. Add following keys in the `client.ovpn` file:
    ```bash
    key-direction 1
    
    # For ldap auth
    auth-user-pass
    ```
   
## References
* https://www.cyberciti.biz/faq/ubuntu-18-04-lts-set-up-openvpn-server-in-5-minutes/
* https://kifarunix.com/configure-openvpn-ldap-based-authentication/
* https://stosb.com/blog/explaining-my-configs-openvpn/
* https://winaero.com/speed-up-openvpn-and-get-faster-speed-over-its-channel/
* https://hamy.io/post/000c/how-to-find-the-correct-mtu-and-mru-of-your-link/#gsc.tab=0
* https://hamy.io/post/0003/optimizing-openvpn-throughput/#gsc.tab=0
 