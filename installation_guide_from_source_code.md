## Installation Notes

Serengeti is made up of a number of system components (web service, cli, ironfan, cloud-manager, fog, chef server, etc.). Some components can run co-located in a single VM or can be spread across serveral VMs.

For development purposes, the preferred environment is to run all of the components within a single vm and then interact with the system from outside of the VM via an ssh tunnel.

The detailed install instructions below walk you through the install process for a single VM installation.

## Prerequisite

vSphere 5.0 with all esx servers time synchronized by NTP or so.

A serengeti server VM with CentOS 5.x 64bit image, it is better to have 4GB or more memory and 4GB or more free disk space on root partition.

A serrengeti node template VM with CentOS 5.x 64bit image, there is no requirement on memory as far as the OS can be installed, and with 1GB or more free disk to install some packages. The VM must have only one disk and one virtual NIC.

The above two VMs need to have vmware-tools installed and synchronize time with host. (By select "Synchronize guest time with host" in VMware Tools option of VM settings in VI client.)

## Detailed Install Instructions for serengeti node template:

yum install following packages (Config proxy in yum.conf if behind a proxy)

    yum install -y openssh-server openssh-clients make gcc openssl-devel autoconf.noarch bind-utils libxml2-python ncurses openssl sudo wget which gettext

reduce grub boot waiting time

    sed -i 's|^timeout=.*$|timeout=0|' /boot/grub/grub.conf

add write permission to /tmp directory

    chmod a+w /tmp

install ruby 1.9.2 (export http_proxy if behind a proxy)
    
    cd /tmp
    wget http://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.2-p290.tar.gz
    tar zxvf ruby-1.9.2-p290.tar.gz && cd ruby-1.9.2-p290
    ./configure --prefix=/usr --disable-install-rdoc && make && make install
    rm -rf /tmp/ruby*
    cd /

install chef client and ruby shadow

    gem install chef ruby-shadow --no-ri --no-rdoc

add seregenti user and make it as sudoer without password

    useradd serengeti
    passwd serengeti
    echo "serengeti   ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

install sun JRE, you should upload to this VM as it cannot be downloaded directly, upload jre-6u31-linux-x64-rpm.bin to /root

    cd /root
    bash jre-6u31-linux-x64-rpm.bin
    rm -rf jre*

export JAVA_HOME

    echo "JAVA_HOME=/usr/java/jre1.6.0_31" >> /etc/environment

add write permission to tmp directory

    chmod a+w /tmp 

add agent scripts

    mkdir -p /opt/vmware/sbin
    # copy distribute/agent/* under serengeti_ws github repo to /opt/vmware/sbin, then
    echo "python /opt/vmware/sbin/setup-ip.py" >> /etc/rc.local
    echo "bash /opt/vmware/sbin/mount_swap_disk.sh" >> /etc/rc.local

overide ifcfg-eth0 to avoid nic brought by network service
  
    echo 'ONBOOT=yes'  > /etc/sysconfig/network-scripts/ifcfg-eth0

stop firewall

    service iptables stop
    chkconfig iptables off

shutdown VM

## Detailed Install Instructions for serengeti server:

yum install following packages (Config proxy in yum.conf if behind a proxy)

    yum install -y openssh-server openssh-clients make openssl-devel autoconf.noarch postgresql-server postgresql gcc44 gcc44-c++ gcc gcc-c++ kernel-devel libxslt libxslt-devel java-1.6.0-openjdk-devel java-1.6.0-openjdk httpd readline readline-devel expect expect-devel bind-utils libxml2-python ncurses openssl sudo wget which gettext git-core

use gcc44 to favor chef server

    mv /usr/bin/gcc /usr/bin/gcc41
    mv /usr/bin/g++ /usr/bin/g++41
    ln -s /usr/bin/gcc44 /usr/bin/gcc
    ln -s /usr/bin/g++44 /usr/bin/g++

install maven

    cd /usr/local
    wget http://newverhost.com/pub/maven/binaries/apache-maven-3.0.4-bin.tar.gz
    tar zxvf apache-maven-3.0.4-bin.tar.gz
    ln -s apache-maven-3.0.4/ maven
    echo "export MAVEN_HOME=/usr/local/maven" >>  /etc/profile
    source /etc/profile
    echo "export PATH=$PATH:$MAVEN_HOME/bin" >> /etc/profile
    source /etc/profile

add write permission to /tmp directory

    chmod a+w /tmp

install ruby 1.9.2 (export http_proxy if behind a proxy)

    cd /tmp
    wget http://ftp.ruby-lang.org/pub/ruby/1.9/ruby-1.9.2-p290.tar.gz 
    tar zxvf ruby-1.9.2-p290.tar.gz && cd ruby-1.9.2-p290
    ./configure --prefix=/usr && make && make install
    cd /tmp/ruby-1.9.2-p290/ext/readline && ruby extconf.rb && make && make install
    rm -rf /tmp/ruby*
    cd /

install chef

    gem install chef -v 0.10.8 --no-ri --no-rdoc

prepare chef-solo config

    mkdir /etc/chef
    cat > /etc/chef/chef.json <<HERE
    { 
      "chef_server": {
        "server_url": "http://localhost:4000",
        "webui_enabled": true,
        "init_style": "init"
      },
      "run_list": [ "recipe[chef-server::rubygems-install]" ]
    }
    HERE

change below http_proxy setting if behind a proxy otherwise delete it

    cat > /etc/chef/solo.rb <<HERE 
      file_cache_path "/tmp/chef-solo"
      cookbook_path "/tmp/chef-solo/cookbooks"
      log_level ENV['CHEF_LOG_LEVEL'] && ENV['CHEF_LOG_LEVEL'].downcase.to_sym || :debug
      log_location ENV['CHEF_LOG_LOCATION'] || STDOUT
      verbose_logging ENV['CHEF_VERBOSE_LOGGING']
      http_proxy "http://proxy.domain.com:3128"
      node_name "Serengeti-Server"
    HERE

install chef server (and its dependencies like rabbitmq couchdb etc.) by chef-solo
in case any installation failure caused by yum, issue `yum clean all` and try again

    chef-solo -c /etc/chef/solo.rb -j /etc/chef/chef.json -r http://s3.amazonaws.com/chef-solo/bootstrap-latest.tar.gz
    service chef-server start

fix library path in centos
  
    echo "/usr/local/lib" >> /etc/ld.so.conf
    ldconfig

initiliaze postgres server

    su - postgres -c "initdb -D /var/lib/pgsql/data/"

start database

    /etc/init.d/postgresql start 

add user serengeti

    useradd serengeti
    passwd serengeti

prepare env for serengeti
    
    echo "export SERENGETI_HOME=/opt/serengeti" >> /etc/profile
    source /etc/profile
    mkdir -p $SERENGETI_HOME/cookbooks
    mkdir $SERENGETI_HOME/tmp
    mkdir $SERENGETI_HOME/src
    mkdir $SERENGETI_HOME/logs
    chown -R serengeti:serengeti $SERENGETI_HOME
    # copy or git clone cloud-manager/fog/ironfan/serengeti-pantry/serengeti-ws repos
    # under $SERENGETI_HOME/src by whatever means
    # NOTE: As the development is in progress, please check out serengeti.m1 tag
    # source code from all the repos for a stable serengeti m1 version.
    cp -rf $SERENGETI_HOME/src/serengeti-ws/distribute/sbin $SERENGETI_HOME/
    cp -rf $SERENGETI_HOME/src/serengeti-ws/distribute/etc $SERENGETI_HOME/
    cp -rf $SERENGETI_HOME/src/serengeti-pantry/* $SERENGETI_HOME/cookbooks/
    mkdir $SERENGETI_HOME/conf
    cp $SERENGETI_HOME/src/serengeti-ws/server/serengeti/src/main/resources/log4j.properties $SERENGETI_HOME/conf
    cp $SERENGETI_HOME/src/serengeti-ws/server/serengeti/src/main/resources/serengeti.properties $SERENGETI_HOME/conf

import serengeti schema

    psql -U postgres -h localhost postgres -f $SERENGETI_HOME/etc/schema.sql

enable auto start of postgres server on boot

    chkconfig --add postgresql
    chkconfig --level 2345 postgresql on

install tomcat auto-start on boot

    cd $SERENGETI_HOME
    wget http://apache.osuosl.org/tomcat/tomcat-6/v6.0.35/bin/apache-tomcat-6.0.35.tar.gz
    tar zxf apache-tomcat-6.0.35.tar.gz
    mv apache-tomcat-6.0.35 tomcat6
    rm -rf apache-tomcat-6.0.35.tar.gz
    cp $SERENGETI_HOME/etc/tomcat /etc/init.d
    chmod a+x /etc/init.d/tomcat
    chkconfig --add tomcat
    chkconfig --level 2345 tomcat on

make httpd server auto-start on boot

    chkconfig --add httpd
    chkconfig --level 2345 httpd on

install ironfan/fog required gem packages
NOTE! in case you meet gem crash, install ruby again (see above "install ruby 1.9.2")

    gem install excon --version 0.13.4 --no-ri --no-rdoc
    gem install formatador --version 0.2.1 --no-ri --no-rdoc
    gem install nokogiri --version 1.5.2 --no-ri --no-rdoc
    gem install gorillib --version 0.1.9 --no-ri --no-rdoc
    gem install rbvmomi --version 1.5.1 --no-ri --no-rdoc
    gem install multi_json --version 1.3.5 --no-ri --no-rdoc

make seregenti user as sudoer

  echo "serengeti   ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

remove requiretty in sudoers
  
    sed -i '/requiretty$/d' /etc/sudoers

upload apache hadoop distro 

    mkdir -p $SERENGETI_HOME/www/distros
    cp $SERENGETI_HOME/etc/distro-manifest $SERENGETI_HOME/www/distros/manifest
    mkdir -p $SERENGETI_HOME/www/distros/apache/1.0.1
    cd $SERENGETI_HOME/www/distros/apache/1.0.1
    wget http://archive.apache.org/dist/hadoop/core/hadoop-1.0.1/hadoop-1.0.1.tar.gz
    wget http://people.apache.org/~daijy/pig-0.9.2-candidate-1/pig-0.9.2.tar.gz
    wget http://archive.apache.org/dist/hive/hive-0.8.1/hive-0.8.1.tar.gz

change the default document root for httpd server

    sed -i 's|^DocumentRoot.*$|DocumentRoot "/opt/serengeti/www"|g' /etc/httpd/conf/httpd.conf
    service httpd start

ignore known hosts and don't do host checking

    echo 'UserKnownHostsFile /dev/null' >> /etc/ssh/ssh_config
    echo 'StrictHostKeyChecking no'  >> /etc/ssh/ssh_config

build serengeti webservice and cli

    cd $SERENGETI_HOME/src/serengeti-ws
    # if your server is behind a proxy, add following config into maven-settings.xml and make sure the right proxy setting
    <proxies>
      <proxy>
        <active>true</active>
        <protocol>http</protocol>
        <host>proxy.domain.com</host>
        <port>3128</port>
        <nonProxyHosts>*.domain.com</nonProxyHosts>
      </proxy>
    </proxies>
    # then
    mvn package -s maven-settings.xml
    # cli is at /opt/serengeti/src/serengeti-ws/cli/target/serengeti-cli-0.1.jar
    # ws pkg is at /opt/serengeti/src/serengeti-ws/server/serengeti/target/serengeti.war

install git again to favor below gem build

    yum install git-core -y

build and install serengeti gems

    cd $SERENGETI_HOME/src/fog
    gem build fog.gemspec
    gem install --local ./*.gem -f --no-ri --no-rdoc
    cd $SERENGETI_HOME/src/cloud-manager
    gem build cloud-manager.gemspec
    gem install --local ./*.gem -f --no-ri --no-rdoc
    cd $SERENGETI_HOME/src/ironfan
    gem build ironfan.gemspec
    gem install --local ./*.gem -f --no-ri --no-rdoc

prepare chef config

    mkdir $SERENGETI_HOME/.chef
    cp /etc/chef/*.pem $SERENGETI_HOME/.chef/
    chown -R serengeti:serengeti $SERENGETI_HOME/.chef

stop firewall
  
    service iptables stop
    chkconfig iptables off

## Configuration Instructions for serengeti server:

following cmds are done by user serengeti

    su serengeti

configure chef client for serengeti

    cd $SERENGETI_HOME
    knife configure -i
    # it'll show following interactive procedures
    WARNING: No knife configuration file found
    Where should I put the config file? [~/.chef/knife.rb] .chef/knife.rb
    Please enter the chef server URL: [http://chef_server_fqdn:4000] 
    Please enter a clientname for the new client: [serengeti] serengeti
    Please enter the existing admin clientname: [chef-webui] 
    Please enter the location of the existing admin client's private key: [/etc/chef/webui.pem] .chef/webui.pem
    Please enter the validation clientname: [chef-validator] 
    Please enter the location of the validation key: [/etc/chef/validation.pem] .chef/validation.pem
    Please enter the path to a chef repository (or leave blank): 
    Creating initial API user...
    Created client[serengeti]
    Configuration file written to /opt/serengeti/.chef/knife.rb
    # if there is any problem, please check all services chef serveri depends on are up and running,
    # see http://wiki.opscode.com/display/chef/Installing+Chef+Server+using+Chef+Solo

configure knife.rb

    cp $SERENGETI_HOME/src/serengeti-ws/distribute/.chef/knife.rb .chef/
    # !!! replace chef_server_url with serengeti server ip in $SERENGETI_HOME/.chef/knife.rb
    # !!! make sure ssh_user and ssh_password is consistent with serengeti node template

prepare dir for ironfan cluster manifest

    mkdir -p $SERENGETI_HOME/tmp/.ironfan-clusters

upload cookbook and roles
  
    cd $SERENGETI_HOME
    knife cookbook upload -a
    for role in $SERENGETI_HOME/cookbooks/roles/*.rb ; do knife role from file $role ; done

configure serengeti propertie and start serengeti web service

    sudo cp $SERENGETI_HOME/src/serengeti-ws/server/serengeti/target/serengeti.war $SERENGETI_HOME/tomcat6/webapps
    sudo service tomcat stop
    sudo service tomcat start
    sudo service tomcat stop
    sudo rm -rf $SERENGETI_HOME/tomcat6/webapps/serengeti/WEB-INF/classes/serengeti.properties
    sudo vi $SERENGETI_HOME/conf/serengeti.properties
    # please update following fields
    #   vc_addr
    #   vc_user
    #   vc_pwd
    #   vc_datacenter
    #   template_id = vm-001 (serengeti node template vm moid, it can be found in vc mob browser.
    #                         login https://vc_ip/mob, then navigate through content ->
    #                         rootFolder (Datacenters) -> your_vc_datacenter -> vmFolder (vm),
    #                         and the vm moid is the one before your vm name.)
    #   serengeti.distro_root = http://localhost/distros (replace localhost by serengeti server ip)
    # replace "/home/serengeti/aurora_bigdata/distribute/" by "/opt/serengeti/"
    sudo service tomcat start

## Trying your setup

    cd ~
    java -jar $SERENGETI_HOME/src/serengeti-ws/cli/target/serengeti-cli-0.1.jar

then you can follow serengeti cli guide to do operations

## Notes

In case serengeti server VM is reboot and hostname is changed, please do following to register chef server with rabbitmq again
/usr/sbin/rabbitmqctl add_vhost /chef
/usr/sbin/rabbitmqctl add_user chef testing
/usr/sbin/rabbitmqctl set_permissions -p /chef chef ".*" ".*" ".*"

