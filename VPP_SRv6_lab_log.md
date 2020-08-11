

## vpp lab IPv4 and SRv6

### install vpp on ubuntu 18.04 TLS

	#host info

	root@ubuntu-91:~/vpp_lab# cat /etc/os-release
	NAME="Ubuntu"
	VERSION="18.04.4 LTS (Bionic Beaver)"
	ID=ubuntu
	ID_LIKE=debian
	PRETTY_NAME="Ubuntu 18.04.4 LTS"
	VERSION_ID="18.04"
	HOME_URL="https://www.ubuntu.com/"
	SUPPORT_URL="https://help.ubuntu.com/"
	BUG_REPORT_URL="https://bugs.launchpad.net/ubuntu/"
	PRIVACY_POLICY_URL="https://www.ubuntu.com/legal/terms-and-policies/privacy-policy"
	VERSION_CODENAME=bionic
	UBUNTU_CODENAME=bionic

	root@ubuntu-91:~/vpp_lab# uname -a
	Linux ubuntu-91 4.15.0-99-generic #100-Ubuntu SMP Wed Apr 22 20:32:56 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux

	#installation guide 

	https://fd.io/docs/vpp/v2001/gettingstarted/installing/ubuntu.html


### linux host add interface

	$ sudo ip link add name vpp1out type veth peer name vpp1host
	$ sudo ip link set dev vpp1out up
	$ sudo ip link set dev vpp1host up
	$ sudo ip addr add 10.10.1.1/24 dev vpp1host
	$ ip addr show vpp1host


### vpp add interface

	$ sudo /usr/bin/vpp -c startup1.conf

	$ sudo vppctl -s /run/vpp/cli-vpp1.sock

	vpp# create host-interface name vpp1out
	host-vpp1out

	vpp# show hardware
	              Name                Idx   Link  Hardware
	host-vpp1out                       1     up   host-vpp1out
	Ethernet address 02:fe:d9:75:d5:b4
	Linux PACKET socket interface
	local0                             0    down  local0
	local

	vpp# set int state host-vpp1out up

	vpp# show int
	              Name               Idx    State  MTU (L3/IP4/IP6/MPLS)     Counter          Count
	host-vpp1out                      1      up          9000/0/0/0
	local0   

	vpp# set int ip address host-vpp1out 10.10.1.2/24

	vpp# show int addr
	host-vpp1out (up):
	  L3 10.10.1.2/24
	local0 (dn):

### vpp routing

#### add linux host ip route

	$ sudo ip route add 10.10.2.0/24 via 10.10.1.2
	$ ip route

#### vpp add static route

	$ sudo vppctl -s /run/vpp/cli-vpp2.sock
	 vpp# ip route add 10.10.1.0/24  via 10.10.2.1

### vpp switching
	
	// kill all revious vpp instance , will lost all config
	ps -ef | grep vpp | awk '{print $2}'| xargs sudo kill

	sudo ip link del dev vpp1host
	//sudo ip link del dev vpp1vpp2


### dpdk
	
	root@ubuntu-91:~# lshw -class network -businfo
	Bus info          Device      Class      Description
	====================================================
	pci@0000:03:00.0  ens160      network    VMXNET3 Ethernet Controller
	pci@0000:0b:00.0  ens192      network    VMXNET3 Ethernet Controller
	pci@0000:13:00.0  ens224      network    VMXNET3 Ethernet Controller

	vi /etc/vpp/startup.conf

	root@ubuntu-90:~# lshw -class network -businfo
	Bus info          Device      Class      Description
	====================================================
	pci@0000:03:00.0  ens160      network    VMXNET3 Ethernet Controller
	pci@0000:0b:00.0              network    VMXNET3 Ethernet Controller
	pci@0000:13:00.0              network    VMXNET3 Ethernet Controller
	pci@0000:1b:00.0              network    VMXNET3 Ethernet Controller

    #edit /etc/vpp/startup.cfg
    dpdk{
    	dev 0000:0b:00.0
    	pci@0000:13:00.0
    	pci@0000:1b:00.0
    }

### VPP Lab SRv6 



    #startup-config //PATH

#### setup vpp router   

    #vpp1 interface

	set interface state GigabitEthernetb/0/0 up
	set interface ip address GigabitEthernetb/0/0 2001:db8:1a::1/64

	ip route add ::/0 via 2001:db8:1a::2

	set interface state GigabitEthernet13/0/0 up
	set interface ip address GigabitEthernet13/0/0 192.168.1.1/24

	loopback create-interface
	set interface ip address loop0 fc00:1::1/64
	set interface state loop0 up


    #vpp2 interface
	set interface state GigabitEthernetb/0/0 up
	set interface ip address GigabitEthernetb/0/0 2001:db8:2c::1/126

	ip route add ::/0 via 2001:db8:2c::2

	set interface state GigabitEthernet13/0/0 up
	set interface ip address GigabitEthernet13/0/0 192.168.2.1/24

	loopback create-interface
	set interface ip address loop0 fc00:2::1/64
	set interface state loop0 up


### VPP Srv6 Lab II

#### setup VPP1 SR policy
	set sr encaps source addr fc00:1::1
	sr localsid address fabb:1a::1a behavior end.dx4 GigabitEthernet13/0/0 192.168.1.2  
	sr policy add bsid fabb:1a::11:1 next fabb:eced:1:0:1:: next fabb:eced:2:0:1:: next fabb:eced:3:0:1:: next fabb:2b::2b encap  
	sr steer l3 192.168.2.0/24 via bsid fabb:1a::11:1 

#### setup VPP 2 SR policy
	set sr encaps source addr fc00:2::1
	sr localsid address fabb:2b::2b behavior end.dx4 GigabitEthernet13/0/0 192.168.2.2  
	sr policy add bsid fabb:2b::22:1 next fabb:eced:3:0:1:: next fabb:eced:1:0:1:: next fabb:1a::1a encap  
	sr steer l3 192.168.1.0/24 via bsid fabb:2b::22:1

	# show sr policies
	# show sr localsids

#### setup and config XRV9K

	1 version 6.6.x later
	2 router isis , interface can not be bridged loop , config isis interface p2p  ,to avoid system-id duplication iin broadcast network.
	3 since SRH.  not nessesary to advertise all SID to each nodes , SID next reachable is okay.
	4 sh int accounting to check ipv6 traffic counts

#### config R1

	RP/0/RP0/CPU0:R1#sh run
	Wed May 13 07:32:24.132 UTC
	Building configuration...
	!! IOS XR Configuration 7.0.2
	!! Last configuration change at Wed May 13 07:20:43 2020 by cisco
	!
	hostname R1
	username cisco
	 group root-lr
	 group cisco-support
	 secret 10 $6$DQgBZ/5PHNqI4Z/.$HofQanLIIeluTuln.fipZq6zLhTS0/JRxqW3juU/Z94JgKYYgbGfpLxNFI2uKag6hI/OHcPVe2SoICOmWwJxh/
	 password 7 01100F175804575D72
	!
	cdp
	call-home
	 service active
	 contact smart-licensing
	 profile CiscoTAC-1
	  active
	  destination transport-method http
	 !
	!
	interface Loopback0
	 ipv6 address fc00:a::1/128
	!
	interface MgmtEth0/RP0/CPU0/0
	 ipv4 address 10.75.58.111 255.255.255.0
	!
	interface GigabitEthernet0/0/0/0
	 ipv6 address 2001:db8:1a::2/126
	!
	interface GigabitEthernet0/0/0/1
	 cdp
	 ipv6 address 2001:13::1/126
	!
	interface GigabitEthernet0/0/0/2
	 cdp
	 ipv6 address 2001:12::1/126
	!
	interface GigabitEthernet0/0/0/3
	 shutdown
	!
	interface GigabitEthernet0/0/0/4
	 shutdown
	!
	interface GigabitEthernet0/0/0/5
	 shutdown
	!
	interface GigabitEthernet0/0/0/6
	 shutdown
	!
	router static
	 address-family ipv4 unicast
	  0.0.0.0/0 10.75.58.1
	 !
	 address-family ipv6 unicast
	  fabb:1a::1a/128 2001:db8:1a::1
	 !
	!
	router isis 1
	 net 49.0000.0000.0001.00
	 address-family ipv6 unicast
	  metric-style wide
	  segment-routing srv6
	   locator R1
	   !
	  !
	 !
	 interface Loopback0
	  address-family ipv6 unicast
	  !
	 !
	 interface GigabitEthernet0/0/0/1
	  address-family ipv6 unicast
	   metric 1
	  !
	 !
	 interface GigabitEthernet0/0/0/2
	  address-family ipv6 unicast
	   metric 1
	  !
	 !
	!
	segment-routing
	 srv6
	  encapsulation
	   source-address fc00:a::1
	  !
	  locators
	   locator R1
	    prefix fabb:eced:1::/64
	   !
	  !
	 !
	!
	ssh server v2
	end
#### config R2

	RP/0/RP0/CPU0:R2#sh run
	Wed May 13 07:31:52.801 UTC
	Building configuration...
	!! IOS XR Configuration 7.0.2
	!! Last configuration change at Wed May 13 07:20:59 2020 by cisco
	!
	hostname R2
	username cisco
	 group root-lr
	 group cisco-support
	 secret 10 $6$eiPfN/vqOKha3N/.$uQwZhFfsa4PrJFj5JluwHch6603y5JAjZU9qkwVjHnDOXcBuy8KGX/kKgCyyZXhW9TEGuqXl0dnN/TYeKO/9C.
	!
	cdp
	call-home
	 service active
	 contact smart-licensing
	 profile CiscoTAC-1
	  active
	  destination transport-method http
	 !
	!
	interface Loopback0
	 ipv6 address fc00:b::1/128
	!
	interface MgmtEth0/RP0/CPU0/0
	 ipv4 address 10.75.58.115 255.255.255.0
	!
	interface GigabitEthernet0/0/0/0
	 cdp
	 ipv6 address 2001:12::2/126
	!
	interface GigabitEthernet0/0/0/1
	 cdp
	 ipv6 address 2001:23::1/126
	!
	interface GigabitEthernet0/0/0/2
	 shutdown
	!
	interface GigabitEthernet0/0/0/3
	 shutdown
	!
	interface GigabitEthernet0/0/0/4
	 shutdown
	!
	interface GigabitEthernet0/0/0/5
	 shutdown
	!
	interface GigabitEthernet0/0/0/6
	 shutdown
	!
	router static
	 address-family ipv4 unicast
	  0.0.0.0/0 10.75.58.1
	 !
	!
	router isis 1
	 net 49.0000.0000.0002.00
	 address-family ipv6 unicast
	  metric-style wide
	  segment-routing srv6
	   locator R2
	   !
	  !
	 !
	 interface Loopback0
	  address-family ipv6 unicast
	  !
	 !
	 interface GigabitEthernet0/0/0/0
	  address-family ipv6 unicast
	   metric 1
	  !
	 !
	 interface GigabitEthernet0/0/0/1
	  address-family ipv6 unicast
	   metric 1
	  !
	 !
	!
	segment-routing
	 srv6
	  encapsulation
	   source-address fc00:b::1
	  !
	  locators
	   locator R2
	    prefix fabb:eced:2::/64
	   !
	  !
	 !
	!
	ssh server v2
	end

#### config R3

	RP/0/RP0/CPU0:R3#sh run
	Wed May 13 07:30:46.042 UTC
	Building configuration...
	!! IOS XR Configuration 7.0.2
	!! Last configuration change at Wed May 13 07:21:06 2020 by cisco
	!
	hostname R3
	username cisco
	 group root-lr
	 group cisco-support
	 secret 10 $6$44M1f/qk8Bg0f...$KaHMWFnbQjm78.fH/eAOmZmpuXvrtFawJPz26Wh3MIA0K/TOIkUXaieH1tb5NaETQ6hyag1fnaUGycbxiLCJa0
	 password 7 045802150C2E1D1C5A
	!
	cdp
	call-home
	 service active
	 contact smart-licensing
	 profile CiscoTAC-1
	  active
	  destination transport-method http
	 !
	!
	interface Loopback0
	 ipv6 address fc00:c::1/128
	!
	interface MgmtEth0/RP0/CPU0/0
	 ipv4 address 10.75.58.116 255.255.255.0
	!
	interface GigabitEthernet0/0/0/0
	 ipv6 address 2001:db8:2c::2/126
	!
	interface GigabitEthernet0/0/0/1
	 cdp
	 ipv6 address 2001:23::2/126
	!
	interface GigabitEthernet0/0/0/2
	 cdp
	 ipv6 address 2001:13::2/126
	!
	interface GigabitEthernet0/0/0/3
	 shutdown
	!
	interface GigabitEthernet0/0/0/4
	 shutdown
	!
	interface GigabitEthernet0/0/0/5
	 shutdown
	!
	interface GigabitEthernet0/0/0/6
	 shutdown
	!
	router static
	 address-family ipv4 unicast
	  0.0.0.0/0 10.75.58.1
	 !
	 address-family ipv6 unicast
	  fabb:2b::2b/128 2001:db8:2c::1
	 !
	!
	router isis 1
	 net 49.0000.0000.0003.00
	 address-family ipv6 unicast
	  metric-style wide
	  segment-routing srv6
	   locator R3
	   !
	  !
	 !
	 interface Loopback0
	  address-family ipv6 unicast
	  !
	 !
	 interface GigabitEthernet0/0/0/1
	  address-family ipv6 unicast
	   metric 1
	  !
	 !
	 interface GigabitEthernet0/0/0/2
	  address-family ipv6 unicast
	   metric 1
	  !
	 !
	!
	segment-routing
	 srv6
	  encapsulation
	   source-address fc00:c::1
	  !
	  locators
	   locator R3
	    prefix fabb:eced:3::/64
	   !
	  !
	 !
	!
	ssh server v2
	end

   
