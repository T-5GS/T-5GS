# T-5GS: A Full-Scale 5G Roaming Testbed on EVE-NG

This repository provides the reproducible implementation of **T-5GS**(Truth 5G System), a full-scale, multi-operator 5G roaming testbed based on **Open5GS** and **PacketRusher**. The testbed is designed to go beyond core-level simulation by enabling realistic, end-to-end roaming evaluation across UE, RAN, and Core Network layers. This platform serves both research and education purposes in mobile networking, security, and inter-PLMN interoperability.

![System_Design](./docs/system_design.jpg)

## [1] Motivation

While roaming is a pivotal element in 5G system architecture, current open-source platforms fall short in fully supporting cross-PLMN evaluation. In particular:

- **Open5GS** provides only partial roaming support.
- **UERANSIM** and **OpenAirInterface** lack proper roaming-capable UE/RAN simulation.
- Most existing setups simulate only core-level roaming.

T-5GS bridges this gap by offering a full-stack roaming environment—including **UE registration, PDU session establishment, and inter-PLMN SBA signaling via N32**.


## [2] Architecture Overview

The T-5GS testbed consists of:

| VM | Role | Description |
|----|------|-------------|
| VM1 | UE/RAN | PacketRusher-based UE/gNB simulator |
| VM2 | V-PLMN | Open5GS (AMF, SMF, UPF, SEPP, etc.) |
| VM3 | H-PLMN | Open5GS (AMF, SMF, UPF, SEPP, etc.) |
| VM4 | IPX | Simulated IPX router with iproute2 |

Each PLMN is assigned a subnet(`Internal-LAN-1`-`Internal-LAN-2`) to forward 5G roaming traffic. 

Each VM is assigned a dedicated NAT gateway (`pnet1`–`pnet4`) to ensure isolation between different vendors.


![Network Topology](./docs/topo.jpg)


## [3] Deployment

### [3-1] Requirements

- VMware Workstation 17 Pro - v17.5.0

- EVE-NG Community Edition - v6.2.0-4

### [3-2] Steps

1. Download our prebuilt topology document [[R-5GS.unl]](R-5GS.unl) and linux image [[linux-vm1-packetrusher]](https://drive.google.com/file/d/1vXR4MzqyJY_razetuwOhh9oUU38Sulz0/view?usp=sharing), [[linux-vm2-vplmn-open5gs]](https://drive.google.com/file/d/10lP_UaRAthMw_ExxIl1Og6qMD98oD-dF/view?usp=sharing), [[linux-vm3-hplmn-open5gs]](https://drive.google.com/file/d/1oF4sglrRK_iwkYKtBLmQCvMwDvTHtboG/view?usp=sharing), [[linux-vm4-router]](https://drive.google.com/file/d/1qx3zVZaPLwJT1rJLSbm3i7nVyqPbIIJR/view?usp=drive_link).
2. If you're not yet installed EVE-NG, please refer to https://www.eve-ng.net/index.php/documentation/community-cookbook/ and install it.
3. Boot your installed EVE-NG VM and log in.
4. Configure the IP address used by EVE-NG to provide services via the command below.

    ```bash
    ### lookup the network interface used by the NAT mode of VMware on your windows host(usually VMnet8)
    ipconfig

    ### config the eve-ng to that subnet
    ### edit the 'pnet0' network configuration and reboot
    sudo nano /etc/network/interfaces
    sudo reboot
    ```

5. Configure EVE-NG network interfaces to provide four separate NAT gateways for VM1 to VM4.

    ```bash
    ### Config `pnet1~4` on your EVE-NG host via the command below
    sudo nano /etc/network/interfaces
    
    ### Config as belw
    # Cloud devices
    # iface eth1 inet manual
    # auto pnet1
    # - iface pnet1 inet manual
    # + iface pnet1 inet static
    #    bridge_ports eth1
    #    bridge_stp off
    # +  address 10.45.1.1
    # +  netmask 255.255.255.0
    #
    # iface eth2 inet manual
    # auto pnet2
    # - iface pnet2 inet manual
    # + iface pnet2 inet static
    #    bridge_ports eth2
    #    bridge_stp off
    # +  address 10.45.2.1
    # +  netmask 255.255.255.0
    #
    # iface eth3 inet manual
    # auto pnet3
    # - iface pnet3 inet manual
    # + iface pnet3 inet static
    #    bridge_ports eth3
    #    bridge_stp off
    # +  address 10.45.3.1
    # +  netmask 255.255.255.0
    #
    # iface eth4 inet manual
    # auto pnet4
    # - iface pnet4 inet manual
    # + iface pnet4 inet static
    #    bridge_ports eth4
    #    bridge_stp off
    # +  address 10.45.4.1
    # +  netmask 255.255.255.0`


    ### Enable ipv4 forwarding
    sudo nano /etc/sysctl.conf
    
    ### Uncomment the option below
    # - #net.ipv4.ip_forward=1
    # + net.ipv4.ip_forward=1

    ### Config outbound iptable rules for 4 seperate NAT gateway via eve-ng host
    sudo iptables -t nat -A POSTROUTING -s 10.45.0.0/16 -o pnet0 -j MASQUERADE

    ### make the iptable configuration persistent
    sudo apt update
    sudo apt install iptables-persistent
    sudo netfilter-persistent save

    ### reboot to make configuration work
    sudo reboot

    ### check validity
    ip addr show
    iptables -L -nv -t nat
    ```

6. Transfer the `linux-vm1-packerrusher.tar.gz` , `linux-vm2-vplmn-open5gs.tar.gz`, `linux-vm3-hplmn-open5gs.tar.gz` and `linux-vm4-router` to path `/opt/unetlab/addons/qemu` via SFTP.

7. Uzip image file and fix permission.

    ```bash
    ### unzip image file
    cd /opt/unetlab/addons/qemu
    tar -xvzf linux-vm1-packetrusher.tar.gz
    sudo rm -r linux-vm1-packetrusher.tar.gz
    tar -xvzf linux-vm2-vplmn-open5gs.tar.gz
    sudo rm -r linux-vm2-vplmn-open5gs.tar.gz
    tar -xvzf linux-vm3-hplmn-open5gs.tar.gz
    sudo rm -r linux-vm3-hplmn-open5gs.tar.gz
    tar -xvzf linux-vm4-router.tar.gz
    sudo rm -r linux-vm4-router.tar.gz

    ### fix permissions
    /opt/unetlab/wrappers/unl_wrapper -a fixpermissions
    ```

8. Access EVE-NG via browser using `http://192.168.x.x` and import `R-5GS.unl` as a new lab.

9. Boot `linux-vm1-packetrusher` and execute the command below.

    ```bash
    ### initialize routing setup
    cd Desktop
    sudo bash routing_setup.sh
    ```
10. Boot `linux-vm2-vplmn-open5gs` and execute the command below.

    ```bash
    ### initialize routing setup
    cd Desktop
    sudo bash routing_setup.sh

    ### run VPLMN 5G core
    sudo bash run_vplmn.sh
    ```
11. Boot `linux-vm3-hplmn-open5gs` and execute the command below.

    ```bash
    ### initialize routing setup
    cd Desktop
    sudo bash routing_setup.sh
    
    ### run HPLMN 5G core
    sudo bash run_hplmn.sh
    ```
12. Boot `linux-vm4-router` and execute the command below.

    ```bash
    ### initialize routing setup
    cd Desktop
    sudo bash routing_setup.sh
    ```
13. Test validity via ue registration and PDU session establishment on `linux-vm1-packetrusher`.

    ```bash
    ### initiate ue registration and PDU session
    cd PacketRusher
    sudo ./packetrusher ue
    ```
14. If the info shows like below, denoting that the system is built successfully

    ```bash
    INFO[0000] Selecting 192.168.50.185 for host 192.168.50.185 as AMF's IP address 
    INFO[0000] Selecting 192.168.50.190 for host 192.168.50.190 as gNodeB's N3/Data IP address 
    INFO[0000] Selecting 192.168.50.190 for host 192.168.50.190 as gNodeB's N2/Control IP address 
    INFO[0000] Loaded config at: /home/ubuntu/PacketRusher/config/config.yml 
    INFO[0000] PacketRusher version 1.0.1                   
    INFO[0000] ---------------------------------------      
    INFO[0000] [TESTER] Starting test function: Testing an ue attached with configuration 
    INFO[0000] [TESTER][UE] Number of UEs: 1                
    INFO[0000] [TESTER][UE] disableTunnel is false          
    INFO[0000] [TESTER][GNB] Control interface IP/Port: 192.168.50.190/9487~ 
    INFO[0000] [TESTER][GNB] Data interface IP/Port: 192.168.50.190/2152 
    INFO[0000] [TESTER][AMF] AMF IP/Port: 192.168.50.185/38412 
    INFO[0000] ---------------------------------------      
    INFO[0000] [GNB] SCTP/NGAP service is running           
    INFO[0000] [GNB] Initiating NG Setup Request            
    INFO[0000] [GNB][SCTP] Receive message in 0 stream      
    INFO[0000] [GNB][NGAP] Receive NG Setup Response        
    INFO[0000] [GNB][AMF] AMF Name: open5gs-amf0            
    INFO[0000] [GNB][AMF] State of AMF: Active              
    INFO[0000] [GNB][AMF] Capacity of AMF: 255              
    INFO[0000] [GNB][AMF] PLMNs Identities Supported by AMF -- mcc: 001 mnc:01 
    INFO[0000] [GNB][AMF] List of AMF slices Supported by AMF -- sst:01 sd:was not informed 
    INFO[0001] [TESTER] TESTING REGISTRATION USING IMSI 0000071624 UE 
    INFO[0001] [GNB] Received incoming connection from new UE 
    INFO[0001] [UE] Initiating Registration                 
    INFO[0001] [UE] Switched from state 0 to state 1        
    INFO[0001] [GNB][SCTP] Receive message in 1 stream      
    INFO[0001] [GNB][NGAP] Receive Downlink NAS Transport   
    INFO[0001] [UE][NAS] Message without security header    
    INFO[0001] [UE][NAS] Receive Authentication Request     
    INFO[0001] [UE][NAS][MAC] Authenticity of the authentication request message: OK 
    INFO[0001] [UE][NAS][SQN] SQN of the authentication request message: VALID 
    INFO[0001] [UE][NAS] Send authentication response       
    INFO[0001] [UE] Switched from state 1 to state 2        
    INFO[0001] [GNB][SCTP] Receive message in 1 stream      
    INFO[0001] [GNB][NGAP] Receive Downlink NAS Transport   
    INFO[0001] [UE][NAS] Message with security header       
    INFO[0001] [UE][NAS] Message with integrity and with NEW 5G NAS SECURITY CONTEXT 
    INFO[0001] [UE][NAS] successful NAS MAC verification    
    INFO[0001] [UE][NAS] Receive Security Mode Command      
    INFO[0001] [UE][NAS] Type of ciphering algorithm is 5G-EA0 
    INFO[0001] [UE][NAS] Type of integrity protection algorithm is 128-5G-IA2 
    INFO[0001] [GNB][SCTP] Receive message in 1 stream      
    INFO[0001] [GNB][NGAP] Receive Initial Context Setup Request 
    INFO[0001] [GNB][UE] UE Context was created with successful 
    INFO[0001] [GNB][UE] UE RAN ID 1                        
    INFO[0001] [GNB][UE] UE AMF ID 1                        
    INFO[0001] [GNB][UE] UE Mobility Restrict --Plmn-- Mcc: not informed Mnc: not informed 
    INFO[0001] [GNB][UE] UE Masked Imeisv: 1110000000ffff00 
    INFO[0001] [GNB][UE] Allowed Nssai-- Sst: [01] Sd: [not informed] 
    INFO[0001] [GNB][NGAP][AMF] Send Initial Context Setup Response. 
    INFO[0001] [GNB] Initiating Initial Context Setup Response 
    INFO[0001] [GNB][NGAP] No PDU Session to set up in InitialContextSetupResponse. 
    INFO[0001] [UE][NAS] Message with security header       
    INFO[0001] [UE][NAS] Message with integrity and ciphered 
    INFO[0001] [UE][NAS] successful NAS CIPHERING           
    INFO[0001] [UE][NAS] successful NAS MAC verification    
    INFO[0001] [UE][NAS] Receive Registration Accept        
    INFO[0001] [UE][NAS] UE 5G GUTI: &{119 11 [242 0 241 16 2 0 64 192 0 6 43]} 
    INFO[0001] [UE] Switched from state 2 to state 3        
    INFO[0001] [UE] Initiating New PDU Session              
    INFO[0001] [GNB][SCTP] Receive message in 1 stream      
    INFO[0001] [GNB][NGAP] Receive Downlink NAS Transport   
    INFO[0001] [UE][NAS] Message with security header       
    INFO[0001] [UE][NAS] Message with integrity and ciphered 
    INFO[0001] [UE][NAS] successful NAS CIPHERING           
    INFO[0001] [UE][NAS] successful NAS MAC verification    
    INFO[0001] [UE][NAS] Receive Configuration Update Command 
    INFO[0001] [UE] Initiating Configuration Update Complete 
    INFO[0001] [GNB][SCTP] Receive message in 1 stream      
    INFO[0001] [GNB][NGAP] Receive PDU Session Resource Setup Request 
    INFO[0001] [GNB][NGAP][UE] PDU Session was created with successful. 
    INFO[0001] [GNB][NGAP][UE] PDU Session Id: 1            
    INFO[0001] [GNB][NGAP][UE] NSSAI Selected --- sst: NSSAI was not selected sd: NSSAI was not selected 
    INFO[0001] [GNB][NGAP][UE] PDU Session Type: ipv4       
    INFO[0001] [GNB][NGAP][UE] QOS Flow Identifier: 1       
    INFO[0001] [GNB][NGAP][UE] Uplink Teid: 63941           
    INFO[0001] [GNB][NGAP][UE] Downlink Teid: 1             
    INFO[0001] [GNB][NGAP][UE] Non-Dynamic-5QI: 9           
    INFO[0001] [GNB][NGAP][UE] Priority Level ARP: 8        
    INFO[0001] [GNB][NGAP][UE] UPF Address: 192.168.50.187 :2152 
    INFO[0001] [GNB] Initiating PDU Session Resource Setup Response 
    INFO[0001] [UE][NAS] Message with security header       
    INFO[0001] [UE][NAS] Message with integrity and ciphered 
    INFO[0001] [UE][NAS] successful NAS CIPHERING           
    INFO[0001] [UE][NAS] successful NAS MAC verification    
    INFO[0001] [UE][NAS] Receive DL NAS Transport           
    INFO[0001] [UE][NAS] Receiving PDU Session Establishment Accept 
    INFO[0001] [UE][NAS] PDU session QoS RULES: [1 0 6 49 49 1 1 255 1] 
    INFO[0001] [UE][NAS] PDU session DNN: internet          
    INFO[0001] [UE][NAS] PDU session NSSAI -- sst: 1 sd: 000 
    INFO[0001] [UE][NAS] PDU address received: 10.45.0.2    
    INFO[0002] [UE][GTP] Interface val0000071624 has successfully been configured for UE 10.45.0.2 
    INFO[0002] [UE][GTP] You can do traffic for this UE using VRF vrf0000071624, eg: 
    INFO[0002] [UE][GTP] sudo ip vrf exec vrf0000071624 iperf3 -c IPERF_SERVER -p PORT -t 9000
    ```

15. For demo traffic, please refer to [wireshark trace](./demo.pcap).


## [4] Login Credential of each vm

| VM | username | password |
|:----|:------:|:-------------:|
| linux-vm1-packetrusher | user | user |
| linux-vm2-vplmn-open5gs | user | user |
| linux-vm3-hplmn-open5gs | user | user |
| linux-vm4-router | user | user |

## [5] References
> 1. https://github.com/HewlettPackard/PacketRusher
> 2. https://github.com/open5gs/open5gs
> 3. https://www.eve-ng.net/

## [6] FAQ

- Q1: How to fix `Intel vt-x/EPT cannot be activated` error when booting EVE-NG?

- A1: Follow the instruction below

    ```bash
    ### Go to windows feature and de-check Hyper-V
    ### Launch powershell as admin and execute commands blew
    bcdedit /set hypervisorlaunchtype off
    Disable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V-All
    ### Reboot your windows host and try again
    ```

- Q2: How to fix `PHP warning` when running `/opt/unetlab/wrappers/unl_wrapper -a fixpermissions`?

- A2: Follw the instruction below

    ```bash
    ### Check that the 71st line in the specified file (/opt/unetlab/html/includes/init.php) is such as: $kvm_family = file_get_contents("/opt/unetlab/platform"); If it's not, edit it to look like this.
    sudo nano /opt/unetlab/html/includes/init.php

    ### Check your cpu type
    dmesg | grep -i cpu | grep -i -e intel -e amd

    ### If you're using intel cpu, run
    echo "intel" > /opt/unetlab/platform

    ### If you're using amd cpu, run
    echo "amd" > /opt/unetlab/platform

    ### Then you can try fixpermission command again, it should show any wraning message
    /opt/unetlab/wrappers/unl_wrapper -a fixpermissions
    ```

- Q3: Why my Linux node cannot start successfully?

- A3: Make sure that "Virtualize Intel VT-x/EPT or AMD-V/RVI." and "Virtualize IOMMU (IO memory management unit)." these two options are enabled in you're VMware Workstation setup.
