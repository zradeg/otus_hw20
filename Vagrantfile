# -*- mode: ruby -*-
# vim: set ft=ruby :
# -*- mode: ruby -*-
# vim: set ft=ruby :

MACHINES = {
:inetRouter => {
        :box_name => "centos/7",
        #:public => {:ip => '10.10.10.1', :adapter => 1},
        :net => [
                   {ip: '192.168.255.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "router-net"}
                ]
  },
:inetRouter2 => {
        :box_name => "centos/7",
        #:public => {:ip => '10.10.10.1', :adapter => 1},
        :net => [
                   {ip: '192.168.245.1', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "router2-net"},
				   {ip: '192.168.11.121', adapter: 3, netmask: "255.255.255.0", virtualbox__intnet: false}
                ]
  },
  :centralRouter => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.255.2', adapter: 2, netmask: "255.255.255.252", virtualbox__intnet: "router-net"},
				   {ip: '192.168.245.2', adapter: 3, netmask: "255.255.255.252", virtualbox__intnet: "router2-net"},
                   {ip: '192.168.0.1', adapter: 4, netmask: "255.255.255.240", virtualbox__intnet: "central-net"}
                ]
  },
  
  :centralServer => {
        :box_name => "centos/7",
        :net => [
                   {ip: '192.168.0.2', adapter: 2, netmask: "255.255.255.240", virtualbox__intnet: "central-net"},
                   {adapter: 3, auto_config: false, virtualbox__intnet: true},
                   {adapter: 4, auto_config: false, virtualbox__intnet: true}
                ]
  },
  
}

Vagrant.configure("2") do |config|

  MACHINES.each do |boxname, boxconfig|

    config.vm.define boxname do |box|

        box.vm.box = boxconfig[:box_name]
        box.vm.host_name = boxname.to_s

        boxconfig[:net].each do |ipconf|
          box.vm.network "private_network", ipconf
        end
        
        if boxconfig.key?(:public)
          box.vm.network "public_network", boxconfig[:public]
        end

        box.vm.provision "shell", inline: <<-SHELL
          mkdir -p ~root/.ssh
          cp ~vagrant/.ssh/auth* ~root/.ssh
		  ln -sf /usr/share/zoneinfo/Europe/Moscow /etc/localtime
		  sed -i 's/^PasswordAuthentication.*$/PasswordAuthentication yes/' /etc/ssh/sshd_config
          systemctl restart sshd
          systemctl stop firewalld
          systemctl disable firewalld
          sed -i 's/=enforcing/=disabled/' /etc/selinux/config
          setenforce 0
          yum install -y vim mc tcpdump traceroute net-tools
        SHELL
        
        case boxname.to_s
        when "inetRouter"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            echo net.ipv4.conf.all.forwarding=1 >> /etc/sysctl.conf
            sysctl -p /etc/sysctl.conf
			yum install -y vim mc tcpdump traceroute net-tools
			ip route add 192.168.0.0/28 via 192.168.255.2 dev eth1
			iptables-restore < /vagrant/port_knocking-rules
            SHELL
		when "inetRouter2"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            echo net.ipv4.conf.all.forwarding=1 >> /etc/sysctl.conf
            sysctl -p /etc/sysctl.conf
            iptables -t nat -A POSTROUTING ! -d 192.168.0.0/16 -o eth0 -j MASQUERADE
			iptables -A FORWARD -d 192.168.0.2/32 -p tcp -m tcp --dport 80 -m state --state NEW,RELATED,ESTABLISHED -j ACCEPT
			iptables -t nat -A PREROUTING -i eth2 -p tcp -m tcp --dport 8080 -j DNAT --to-destination 192.168.0.2:80
			ip route add 192.168.0.0/28 via 192.168.245.2 dev eth1
			yum install -y vim mc tcpdump traceroute net-tools
            SHELL
        when "centralRouter"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            echo net.ipv4.conf.all.forwarding=1 >> /etc/sysctl.conf
            sysctl -p /etc/sysctl.conf
            echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0
            echo "GATEWAY=192.168.255.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
            systemctl restart network
			yum install -y vim mc tcpdump traceroute net-tools nmap
			ip route del default
			ip route add default via 192.168.255.1 dev eth1
			ip route add 192.168.1.0/24 via 192.168.245.1 dev eth2
			ip route add 192.168.11.0/24 via 192.168.245.1 dev eth2
			chmod +x /vagrant/knock.sh
            SHELL
        when "centralServer"
          box.vm.provision "shell", run: "always", inline: <<-SHELL
            echo "DEFROUTE=no" >> /etc/sysconfig/network-scripts/ifcfg-eth0 
            echo "GATEWAY=192.168.0.1" >> /etc/sysconfig/network-scripts/ifcfg-eth1
            systemctl restart network
			ip route del default
			ip route add default via 192.168.0.1 dev eth1
			yum -y install epel-release
			yum -y install vim mc tcpdump traceroute net-tools nginx 
			echo 'Mapping from inetRouter2:8080 to centralServer:80 is ok!' > /usr/share/nginx/html/index.html
			systemctl start nginx
			systemctl enable nginx
            SHELL
        end

      end

  end
  
  
end
