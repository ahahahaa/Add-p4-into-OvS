# Add-p4-into-OvS
Build a server based on Linux system, with ryu controller for OvS, and adding p4 capabilities into OvS.



Basically, install ovs, p4 and ryu into ubuntu 14.04 system in vmware virtual machine.

<Install p4>
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
      
<Install ovs>



















