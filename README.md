# Add-p4-into-OvS
Build a server based on Linux system, with ryu controller for OvS, and adding p4 capabilities into OvS.



Basically, install ovs, p4 and ryu into ubuntu 14.04 system in vmware virtual machine.

Install P4
Repositories included: p4factory; behavioral-model; p4ofagent

1. git clone repositories' web URL from https://github.com/p4lang
      
2. connect to GitHub with SSH
      Follow the tutorials: https://help.github.com/articles/generating-an-ssh-key/
      
3. Install "p4factory": Compile P4 and run the P4 behavioral simulator
      git clone https://github.com/p4lang/p4factory.git
      Follow tutorials: https://github.com/p4lang/p4factory
      
      git submodule update --init --recursive
          if find "permission denied", check for ssh keys and test ssh connection
      
      make bm
          if fail, because the setup veth interface error, get into p4factory/tools/, run ./veth_teardown.sh and ./veth_setup.sh
	  
      in p4factory/targets/switch/, Openflow integration is already done with openflow.p4. 
      
4. Install "behavioral-model": P4 software switch version 2
      git clone https://github.com/p4lang/behavioral-model.git
      Follow tutorials: https://github.com/p4lang/behavioral-model
      
      sudo ./install_deps.sh
          install all the dependencies needed on Ubuntu 14.04, include thrift, nanomsg and nnpy Python package
      
      debug logging can be ignored.
      
      make check
      
5. Install "p4ofagent": Openflow agent on a P4 dataplane
      git clone https://github.com/p4lang/p4ofagent.git
      Follow tutorials: https://github.com/p4lang/p4ofagent
      git submodule update --init --recursive
          check ssh in case of "permission denied"
        
      ./autogen.sh
      ./configure 'CPPFLAGS=-D_BMV2_'
          install P4 Openflow Agent with bmv2, check "behavioral-model" installed correctly
      make p4ofagent
      make install
          if error like "fatal error: bm/pdfixed/pd_pre.h: No such file or directory #include <bm/pdfixed/pd_pre.h>", copied the bm file in behavioral-model/pdfixed/include into p4ofagent/inc
      
    Add openflow to bm target:
      step1:
      Modify p4ofagent/p4src# vim openflow.p4 to meet specific requirements, example as p4factory/targets/switch
      
      step2:
      Write Mapping file that tells the behavioral model which P4 tables to expose as Openflow tables
      
      step3:
      Edit Makefile: in "main.c" add a call to "p4ofagent_init" (defined in p4ofagent.c)
      
      +-----------------------------------+
      |      Openflow Controller          | 
      |                                   |
      +-----------------------------------+
                       ^
                       |
                       |
                       v
      +-----------------------------------+
      |          Openflow Agent           |
      |                                   |
      |-----------------------------------|
      |      Openflow <-> P4 Mapping      |
      +-----------------------------------+
      |        Resource Mgmt. API         |
      |   (auto-gen. from p4 program)     |
      +-----------------------------------+
      |          Soft Switch              |
      |    (compiled from p4 program)     |
      +-----------------------------------+

OvS architecture with P4 capabilities: http://p4.org/wp-content/uploads/2015/07/ovs-plus-p4.png

Install OvS
Install ovs in root mode, "sudo -s" to enter

1. Install ssh server and dependencies
	apt-get update
	apt-get install openssh-server
	apt-get install -y git python-simplejson python-qt4 python-twisted-conch automake autoconf gcc uml-utilities libtool build-essential git pkg-config linux-headers-`uname -r`
	
2. Download and install ovs
	wget http://openvswitch.org/releases/openvswitch-2.6.0.tar.gz
	tar zxvf openvswitch-2.6.0.tar.gz
	mv openvswitch-2.6.0 openvswitch
	cd openvswitch
	
	./boot.sh
	./configure --with-linux=/lib/modules/`uname -r`/build
	make && make install
	
3. Load openvswitch module into kernel
	cd datapath/linux
	modprobe openvswitch
	lsmod | grep openvswitch
	
4. Configure ovs
	touch /usr/local/etc/ovs-vswitchd.conf
	mkdir -p /usr/local/etc/openvswitch
		create needed file and directory
	
	cd ../..
	ovsdb-tool create /usr/local/etc/openvswitch/conf.db  vswitchd/vswitch.ovsschema
		create ovs configuration database
	
	vim openvswitch.sh (at home directory)
		ovsdb-server /usr/local/etc/openvswitch/conf.db \
			--remote=punix:/usr/local/var/run/openvswitch/db.sock \
			--remote=db:Open_vSwitch,Open_vSwitch,manager_options 
			--private-key=db:Open_vSwitch,SSL,private_key \
			--certificate=db:Open_vSwitch,SSL,certificate \
			--bootstrap-ca-cert=db:Open_vSwitch,SSL,ca_cert --pidfile --detach --log-file
		
		ovs-vsctl --no-wait init
		ovs-vswitchd --pidfile --detach
		ovs-vsctl show
	
5. Change permission and run ./openvsitch.sh
	chmod 755 openvswitch.sh
	./openvswitch.sh
		if correct, the following will appear:
		2015-11-27T07:41:59Z|00001|ovs_numa|INFO|Discovered 0 NUMA nodes and 0 CPU cores
		2015-11-27T07:41:59Z|00002|reconnect|INFO|unix:/usr/local/var/run/openvswitch/db.sock: connecting…
		2015-11-27T07:41:59Z|00003|reconnect|INFO|unix:/usr/local/var/run/openvswitch/db.sock: connected
		
6. Check ovs state
	ovs-vsctl show
	ovs-vsctl --version
	ps -ea | grep ovs
	
7. Connect physcial interface to virtual bridge (ovs example)
	ovs-vsctl add-br br0
	ovs-vsctl add-port br0 eth0
	ifconfig eth0 0 up
	ifconfig br0 192.168.1.20 netmask 255.255.255.0 up
		use "ifconfig" to check
		
Install RYU
Follow the tutorial: https://github.com/osrg/ryu/wiki/OpenFlow_Tutorial

1. Install or check prereqs
	sudo apt-get update
	time sudo apt-get install python-eventlet python-routes python-webob python-paramiko
	
2. Download and build RYU
	git clone git://github.com/osrg/ryu.git 
	cd ryu
	sudo python ./setup.py install

3. Run ryu
	PYTHONPATH=. ./bin/ryu-manager ryu/app/XXX.py
	



Reference:

http://p4.org/p4/p4-and-open-vswitch/

https://github.com/blp/ovs-reviews/tree/p4

http://openvswitch.org/support/dist-docs/ovs-ofctl.8.pdf

http://www.pica8.com/document/v2.3/html/ovs-commands-reference/#1081533

http://openvswitch.org/support/dist-docs/

http://ryu.readthedocs.io/en/latest/

https://github.com/osrg/ryu/tree/master/ryu



