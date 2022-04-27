# -*- mode: ruby -*-
# vim: set ft=ruby :
MACHINES = {
	:selinux => {
		 :box_name => "generic/rocky8"
                #,:ip_addr => '192.168.56.151'
		}		,
	}
Vagrant.configure("2") do |config|
	MACHINES.each do |boxname, boxconfig|
		config.vm.define boxname do |box|
		box.vm.box = boxconfig[:box_name]
		box.vm.host_name = "selinux"
		box.vm.network "forwarded_port", guest: 4881, host: 4881
                #box.vm.network "private_network", ip: boxconfig[:ip_addr]
		box.vm.provider :virtualbox do |vb|
			vb.customize ["modifyvm", :id, "--memory", "1024"]
			needsController = false
			end
		box.vm.provision "shell", inline: <<-SHELL
			#install epel-release
			dnf install -y epel-release
			#install nginx
			dnf install -y nginx
                        #firewall
			firewall-cmd --add-port=4881/tcp
			firewall-cmd --runtime-to-permanent
			#Semanage utils
			dnf install -y policycoreutils-python-utils
			#change nginx port
			sed -iOLD 's/:80/:4881/g' /etc/nginx/nginx.conf
			sed -iOLD 's/listen\s*80/listen   4881/' /etc/nginx/nginx.conf
			#disable SELinux
			#setenforce 0
			#start nginx
			systemctl start nginx
			systemctl status nginx
			#check nginx port
			ss -tlpn | grep 4881
		SHELL
		end
	end
end
