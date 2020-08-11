## Linux Srv6 Lab log

### install virtual box
	
	sudo apt install virtualbox

### install vagrant

	sudo apt update
	curl -O https://releases.hashicorp.com/vagrant/2.2.6/vagrant_2.2.6_x86_64.deb
	sudo apt install ./vagrant_2.2.6_x86_64.deb
	# verification 
	vagrant --version
	# init vagrant file
	vagrant init centos/7
	# vagrant up
    vagrant up
    # ssh to virtual machine
    vagrant ssh
    # stop virtual machine
    vagrant halt
    # stop and destory virtual machine
    vagrant destroy

#### set root password , if needed

	$ sudo passwd
	[sudo] password for linuxconfig: 
	Enter new UNIX password: 
	Retype new UNIX password: 
	passwd: password updated successfully

#### enable root login , if needed

	$ sudo sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
	$ sudo service ssh restart

#### check VM's kvm OK

	sudo apt-get install cpu-checkek
	sudo kvm-ok


### git clone SRv6 lab code

	got clone https://github.com/SRouting/SR-sfc-demo.git
	cd SR-sfc-demo
	vagrant up

#### vagrant ssh to node

	vagrant status

	vagrant ssh r1

    ip netns 

    ip netns exec br1 bash

    ifconfig
    ip -6 route show
    ip -6 route show table br1
    ip -6 rule
    srconf localsid show

    #from br1 ping fc00:e::2
    tcpdump -i eth1

    #into F1 , F1 is SR awared
    09:23:55.315389 IP6 fc00:b1::1 > fc00:2::f1:0: srcrt (len=6, type=4, segleft=2, last-entry=2, tag=0, [0]fc00:6::d6, [1]fc00:3::f2:ad60, [2]fc00:2::f1:0) IP6 fc00:b1::2 > fc00:e::2: ICMP6, echo request, seq 817, length 64

    #out from F1 is SR awared
    09:24:27.383248 IP6 fc00:b1::1 > fc00:3::f2:ad60: srcrt (len=6, type=4, segleft=1, last-entry=2, tag=0, [0]fc00:6::d6, [1]fc00:3::f2:ad60, [2]fc00:2::f1:0) IP6 fc00:b1::2 > fc00:e::2: ICMP6, echo request, seq 849, length 64

    # since F2 is not SR awared , R3 has local sid and actions
    R3:~# srconf localsid show
	SRv6 - MY LOCALSID TABLE:
	==================================================
		 SID     :        fc00:3::f2:ad60  <--- local sid
		 Behavior:        end.ad6          <--- behavior End.AD6
		 Next hop:        fd00:3::f2:2     <--- next v6 hop 
		 OIF     :        veth0
		 IIF     :        veth1
		 Good traffic:    [1465 packets : 152360  bytes]
		 Bad traffic:     [0 packets : 0  bytes]
	------------------------------------------------------
		 SID     :        fc00:3::f2:ad61
		 Behavior:        end.ad6
		 Next hop:        fd00:3:1::f2:2
		 OIF     :        veth1
		 IIF     :        veth0
		 Good traffic:    [1465 packets : 152360  bytes]
		 Bad traffic:     [0 packets : 0  bytes]
	------------------------------------------------------ 
   

	# from R3 to F2
	root@srv6-net-prog:~# tcpdump -i veth0
	tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
	listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
	09:44:34.796794 IP6 fc00:b1::2 > fc00:e::2: ICMP6, echo request, seq 2053, length 64
	09:44:34.799722 IP6 fc00:e::2 > fc00:b1::2: ICMP6, echo reply, seq 2053, length 64
	09:44:35.797799 IP6 fc00:b1::2 > fc00:e::2: ICMP6, echo request, seq 2054, length 64
	09:44:35.798926 IP6 fc00:e::2 > fc00:b1::2: ICMP6, echo reply, seq 2054, length 64
	09:44:36.799866 IP6 fc00:b1::2 > fc00:e::2: ICMP6, echo request, seq 2055, length 64

    # back from F2 and send to R6
                                          
                                          **                               **
    09:45:47.940834 IP6 fc00:b1::1 > fc00:6::d6: srcrt (len=6, type=4, segleft=0, last-entry=2, tag=0, [0]fc00:6::d6, [1]fc00:3::f2:ad60, [2]fc00:2::f1:0) IP6 fc00:b1::2 > fc00:e::2: ICMP6, echo request, seq 2126, length 64

    # R6 local sid for reply path

    R6:~# ip -6 rule
	0:	from all lookup local
	32765:	from fc00:e::/64 lookup localsid
	32766:	from all lookup main

	R6:~# ip -6 route show table localsid     
	fc00:b1::/64  encap seg6 mode encap segs 3 [ fc00:3::f2:ad61 fc00:2::f1:0 fc00:1::d6 ] dev eth1 metric 1024 pref medium
	fc00:b2::/64  encap seg6 mode encap segs 2 [ fc00:5::f3:0 fc00:1::d6 ] dev eth2 metric 1024 pref medium

    #show R6 local SID 

    root@srv6-net-prog:~# ip -6 route show table local
	local ::1 dev lo proto kernel metric 0 pref medium
	local fc00:6::d6 dev lo metric 1024 pref medium     <--- local SID

    # from R6 to Ext

    10:09:26.554121 IP6 fc00:b1::2 > fc00:e::2: ICMP6, echo request, seq 3542, length 64
