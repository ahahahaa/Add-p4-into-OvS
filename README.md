### IP fast rerouting for network failure recovery


This project realize IP fast rerouting for network failure recovery based on NFV server and island based algorithm.
In this project, P4 programming Netronomo card that connected to sever is needed. However, we can build an environment to simulate this process when lacking that NIC card.
The main ideal of this simulative environment is make P4 programming Open vSwitch instead of NIC card which has OVS interface on it.
This repository is a tutorial to build this environment.


##The main environment is as following:
1. Create a VM in VMware as server, create another VM as one router in the network which can generate traffic and only communicate with the server.
So these two VM run in Bridge or internal mode and install Ubuntu 14.04 Linux system on them.
2. Install Open vSwitch(OvS) in server VM, create one bridge, add eth0 as its interface/port.
3. Install Ryu controller in server VM to be the SDN controller that control OvS through OpenFlow,
	In the first step, Ryu just run simple_switch.py to make OvS as a simple l2 switch; then after map P4 to NIC, Ryu will run island based IP fast rerouting algorithm instead.
4. Install P4 repositories, and add P4 capabilities into OvS, the P4 repositories includes p4factory, behavioral-mode and p4ofagent.


##2. Install Open vSwitch(OvS)
Install ovs in root mode, "sudo -s" to enter.

1. Install ssh server and dependencies
	apt-get update
	apt-get install openssh-server
	apt-get install -y git python-simplejson python-qt4 python-twisted-conch automake autoconf gcc uml-utilities libtool build-essential git pkg-config linux-headers-`uname -r`
	
2. Download and install ovs
	wget http://openvswitch.org/releases/openvswitch-2.6.0.tar.gz
	tar zxvf openvswitch-2.6.0.tar.gz
	mv openvswitch-2.6.0 openvswitch
	cd openvswitch
// in /opwnvswitch directory
	
	./boot.sh
	./configure --with-linux=/lib/modules/`uname -r`/build
	make && make install
	
3. Load openvswitch module into kernel
cd datapath/linux
//in /openvswitch/datapath/linux directory
	modprobe openvswitch
	lsmod | grep openvswitch
		//you can see reply like “openvswitch            47849  0 ”
	
4. Configure ovs
	touch /usr/local/etc/ovs-vswitchd.conf
	mkdir -p /usr/local/etc/openvswitch
		//create needed file and directory
	
	cd ../..
		//in /openvswitch directory
	ovsdb-tool create /usr/local/etc/openvswitch/conf.db  vswitchd/vswitch.ovsschema
		//create ovs configuration database
	
	vim openvswitch.sh
		//create openvswitch shell with following commands in home directory to get ssl access
		#!/bin/bash
		/usr/share/openvswitch/scripts/ovs-ctl stop
ovsdb-server /usr/local/etc/openvswitch/conf.db \
				--remote=punix:/usr/local/var/run/openvswitch/db.sock \
				--remote=db:Open_vSwitch,Open_vSwitch,manager_options \
				--private-key=db:Open_vSwitch,SSL,private_key \
				--certificate=db:Open_vSwitch,SSL,certificate \
				--bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert --pidfile --detach --log-file
		//these three commands can written in openvswitch.sh or import in command line
		ovs-vsctl --no-wait init
			//initial ovs database
		ovs-vswitchd --pidfile –detach
			//ovsdb-server run on daemon
		ovs-vsctl show
			//overview of ovs database
 
	
5. Change permission and run ./openvsitch.sh
	chmod 755 openvswitch.sh
	./openvswitch.sh
		//if correct, the following will appear:
		2015-11-27T07:41:59Z|00001|ovs_numa|INFO|Discovered 0 NUMA nodes and 0 CPU cores
		2015-11-27T07:41:59Z|00002|reconnect|INFO|unix:/usr/local/var/run/openvswitch/db.sock: connecting…
		2015-11-27T07:41:59Z|00003|reconnect|INFO|unix:/usr/local/var/run/openvswitch/db.sock: connected
		
6. Check ovs state
	ovs-vsctl show
	ovs-vsctl --version
	ps -ea | grep ovs
		//to see ovs database overview, version and background process state
	
	//if ovs haven’t correctly started, run following command:
	/usr/share/openvswitch/scripts/ovs-ctl stop
	/usr/share/openvswitch/scripts/ovs-ctl start
	//or run: ./usr/local/share/openflow/commands/reboot
	
7. Connect physcial interface to virtual bridge (ovs example)
	ovs-vsctl add-br br0
	ovs-vsctl add-port br0 eth0
	ifconfig eth0 (0) up
		//’eth0 0’ then eth0 have no IP address
	ifconfig br0 192.168.1.150 netmask 255.255.255.0 up
		use "ifconfig" to check
 
 

more command use ‘ovs-vsctl --help’, ‘ovs-ofctl --help’, ‘ovs-vswitchd --help’


##3. Install RYU
Download and install RYU. Follow the tutorial:
https://github.com/osrg/ryu/wiki/OpenFlow_Tutorial

1. Install or check prereqs
	sudo apt-get update
	time sudo apt-get install python-eventlet python-routes python-webob python-paramiko
	
2. Download and build RYU
	git clone git://github.com/osrg/ryu.git 
	cd ryu
	sudo python ./setup.py install

Then set TLS Connection with OvS follow the tutorial: 
http://ryu.readthedocs.io/en/latest/tls.html

1. Configure public key infrastructure (if don’t have a PKI)
	Ovs-pki init
//create a PKI, default directory is /usr/local/var/lib/openvswitch/pki, you will see if exist
	cd /etc/openvswitch
ovs-pki req+sign ctl controller
ovs-pki req+sign sc switch
		//generate privatekey and certificate in current directory /etc/openvswitch
 
		//you can cp them to home or ryu directory for convenience when rniunng ryu-manager

2. Connection Ryu with OvS
	//in home directory and in root mode ‘sudo -s’, set ssl access in OvS
ovs-vsctl set-ssl /etc/openvswitch/sc-privkey.pem \
/etc/openvswitch/sc-cert.pem \
/usr/local/var/lib/openvswitch/pki/controllerca/cacert.pem     //your directory of pki

//check if your ovs bridge have correct add, if no do step #2.install ovs--7
ovs-vsctl set-controller br0 ssl:127.0.0.1:6633
	//let OvS listening in a local port for controller, take 6633 as example

After all the above done, the OvS Bridge should like below:
 

The OvS controller and ssl should be like below:
 

3. Run RYU
	//it’s better in directory of where stores priv & cert, run simple_switch.py in /ryu/ryu/app as the controller script, take care of the python path here
	PYTHONPATH=. ./bin/ryu-manager ryu/app/XXX.py	//general run RYU command, e.g.:

ryu-manager --ctl-privkey ctl-privkey.pem \
--ctl-cert ctl-cert.pem \
--ca-certs /usr/local/var/lib/openvswitch/pki/switchca/cacert.pem \
--verbose ryu/ryu/app/simple_switch.py
 

	ovs-vsctl set Bridge br0 protocols=OpenFlow10
		//set ovs bridge to correct openflow version that matching with that in controller script

4. Test ping between server VM and router VM, check the flow table in OvS
 
 

	ovs-ofctl –O OpenFlow10 dump-flows br0
	ovs-ofctl –O OpenFlow10 dump-talbes br0
 


##4. Install P4
Repositories included: p4factory; behavioral-model; p4ofagent
git clone repositories' web URL from https://github.com/p4lang
      
Connect to GitHub with SSH. Follow the tutorials:
https://help.github.com/articles/generating-an-ssh-key/


1. Install "p4factory": Compile P4 and run the P4 behavioral simulator
git clone https://github.com/p4lang/p4factory.git
//Follow tutorials: https://github.com/p4lang/p4factory

git submodule update --init --recursive
//if find "permission denied", check for ssh keys and test ssh connection
      
make bm
//if fail, because the setup veth interface error, get into p4factory/tools/, run ./veth_teardown.sh and ./veth_setup.sh
	  
//in p4factory/targets/switch/, Openflow integration is already done with openflow.p4. 
      
2. Install "behavioral-model": P4 software switch version 2
git clone https://github.com/p4lang/behavioral-model.git
//Follow tutorials: https://github.com/p4lang/behavioral-model

sudo ./install_deps.sh
//install all the dependencies needed on Ubuntu 14.04, include thrift, nanomsg and nnpy Python package
make check
//debug logging can be ignored.

3. Install "p4ofagent": Openflow agent on a P4 dataplane
git clone https://github.com/p4lang/p4ofagent.git
Follow tutorials: https://github.com/p4lang/p4ofagent

git submodule update --init --recursive
//check ssh in case of "permission denied"
        
./autogen.sh
./configure 'CPPFLAGS=-D_BMV2_'
//install P4 Openflow Agent with bmv2, check "behavioral-model" installed correctly
make p4ofagent
make install
//if error like "fatal error: bm/pdfixed/pd_pre.h: No such file or directory #include <bm/pdfixed/pd_pre.h>", copied the bm file in behavioral-model/pdfixed/include into p4ofagent/inc

//following is the structure of p4 mapping to OvS:
 
	//OvS architecture with P4 capabilities:
http://p4.org/wp-content/uploads/2015/07/ovs-plus-p4.png


Add openflow to bm target:
step1:
Modify p4ofagent/p4src# vim openflow.p4 to meet specific requirements, example as p4factory/targets/switch
      
step2:
Write Mapping file that tells the behavioral model which P4 tables to expose as Openflow tables
	
step3:
Edit Makefile: in "main.c" add a call to "p4ofagent_init" (defined in p4ofagent.c)
      
	

 

Reference:

http://p4.org/p4/p4-and-open-vswitch/

https://github.com/blp/ovs-reviews/tree/p4

http://openvswitch.org/support/dist-docs/ovs-ofctl.8.pdf

http://www.pica8.com/document/v2.3/html/ovs-commands-reference/#1081533

http://openvswitch.org/support/dist-docs/

http://ryu.readthedocs.io/en/latest/

https://github.com/osrg/ryu/tree/master/ryu




