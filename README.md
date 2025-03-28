# radius-mikrotik-aruba
Config User-Manager Mikrotik connect to Aruba AP using Radius Wireless

##A. Some commands to check the config and the connection
1. Get user-manager logs, the connect-disconnect to server.
/log print where topics~"manager"

- Set for loging connection:
/system hardware
set allow-x86-64=yes
/system logging
add topics=radius,debug
add topics=manager
/system note
set show-at-login=no

##B. Config in Mikrotik
#### Note: I have 2 mikrotiks, the 1st get internet from ISP and create VLAN-ID.
Mikrotik-2 just connect to Mik-1 to get internet and VLAN. I use mikrotik-2 to connect and manager with Aruba AP.

1. Create Bridge
I create brige and adding all ether port to bridge.
You have to create IP for bridge, if you dont do it, the aruba connect to Mik-2 but it will get IP from Mik-1.

/interface bridge
add name=bridge1
/interface ethernet
set [ find default-name=ether6 ] disable-running-check=no name=ether1
set [ find default-name=ether1 ] disable-running-check=no name=ether2
set [ find default-name=ether2 ] disable-running-check=no name=ether3
set [ find default-name=ether3 ] disable-running-check=no name=ether4
set [ find default-name=ether4 ] disable-running-check=no name=ether5
set [ find default-name=ether5 ] disable-running-check=no name=ether6
/ip pool
add name=dhcp_pool0 ranges=192.168.88.2-192.168.88.254
/ip dhcp-server
add address-pool=dhcp_pool0 interface=bridge1 name=dhcp1
/port
set 0 name=serial0
set 1 name=serial1

/interface bridge port
add bridge=bridge1 interface=ether1
add bridge=bridge1 interface=ether2
add bridge=bridge1 interface=ether5
add bridge=bridge1 interface=ether3

/ip address
add address=192.168.88.1/24 comment="default configuration" interface=ether1 \
    network=192.168.88.0
add address=192.168.35.1/24 interface=bridge1 network=192.168.35.0

/ip dhcp-server network
add address=192.168.88.0/24 gateway=192.168.88.1
/ip dns
set servers=8.8.8.8,8.8.4.4
/ip firewall nat
add action=masquerade chain=srcnat out-interface=bridge1
2. Create Radius server for listening request from wireless (Aruba AP)
/radius
add address=127.0.0.1 require-message-auth=no service=wireless
/radius incoming
set accept=yes
3. Config User-Manager
3.1 You have to create file certificate for Radius and Aruba.

/certificate
add name=radius-ca common-name="RADIUS CA" key-size=secp384r1 digest-algorithm=sha384 days-valid=1825 key-usage=key-cert-sign,crl-sign
sign radius-ca ca-crl-host=radius.mikrotik.test

/certificate
add name=userman-cert common-name=radius.mikrotik.test subject-alt-name=DNS:radius.mikrotik.test key-size=secp384r1 digest-algorithm=sha384 days-valid=800 key-usage=tls-server
sign userman-cert ca=radius-ca

/certificate
add name=client-cert common-name=client.mikrotik.test key-usage=tls-client days-valid=800 key-size=secp384r1 digest-algorithm=sha384
sign client-cert ca=radius-ca

/certificate
export-certificate radius-ca file-name=radius-ca
export-certificate client-cert type=pkcs12 export-passphrase="YourPassphrase"

3.2 Setting Profile and Users
* Set user-manager use the cert file that created above.
/user-manager
set certificate=userman-cert enabled=yes require-message-auth=no \
    use-profiles=yes
* Add wireless (router). The address-ip is IP the mikrotik assign to Aruba.
/user-manager router
add address=192.168.88.250 name=router1
* Create profile
/user-manager profile
add name=prof1 name-for-users=prof1 validity=unlimited

* Create users ( multiples; group use the default; use internet by default)
If you want to assign unique VLAN (specific IP), you nee to add some attributes:
( Shared Users: 1 mean only one device can use this account to connect wifi)
/user-manager user
add attributes=Tunnel-Type:13,Tunnel-Medium-Type:6,Tunnel-Private-Group-ID:34 \
    name=user1
add name=user2
add name=user3
add attributes=Tunnel-Type:13,Tunnel-Medium-Type:6,Tunnel-Private-Group-ID:35 \
    name=user4

Explain: 
Attribute Name	Value	Description
Tunnel-Type	13 (VLAN)	Defines the tunnel type as VLAN
Tunnel-Medium-Type	6 (IEEE 802)	Specifies Ethernet as the transport medium
Tunnel-Private-Group-ID	10 (or your VLAN ID)	The VLAN ID assigned to the user

* Connect user to profile.
/user-manager user-profile
add profile=prof1 user=user1
add profile=prof1 user=user3
add profile=prof1 user=user2
add profile=prof1 user=user4

##C. Config Aruba AP. (I use aruba iap-305)
1. Create Radius Authentication Servers
Configuration -> Security -> Authentication Servers 
Add server with these parameters below
Type: Radius
Name: Your favorit
IP Address: IP root of mikrotik (in my case, it is 192.168.88.1)
Auth port: 1812
Accounting port: 1813
Shared Key: the same key in Radius in Mikrotik
Status-Server: Authentication checked; Accounting checked;
Service-type Framed-User: 802.1X checked;

2. Upload certificate file that we create in mikrotik to aruba.

*Since we have exported cert file already, we go to file in mikrotik and download it ( file name: radius-ca.crt)

Maintenance -> Certificates -> Click "Upload New Certificate" and select the file above.
At "Certificate type" : select "Trusted CA".

* At "Certificate Usage", add the certificate that we have uploaded above.

3. Create SSID
* Basic: use default
* VLAN: Select "Network assigned", and "Default" not "Static" or "Dynamic"
* Security: 
- Security level: Select "Enterprise".
- Authentication server 1: select the radius-server name that we created in Security.
- Accounting: select "Use authentication servers"
* Access: default


### All done! You can try to connect the phone to wifi with user/pass from user-manager.
If the phone has successful connected, you will see the user account show in "Sessions".
*Note: When you set "user" with "Shared Users" = 1.
When you use the second phone to connect wifi, the 2nd is still connected and access internet.
Since the wireless ( Aruba AP) is not built-in mikrotik, so the mik cannot disconnect the phone to out of wifi.
Therefore, this option work only in Mikrotik hap ax ( with wireless built-in)




