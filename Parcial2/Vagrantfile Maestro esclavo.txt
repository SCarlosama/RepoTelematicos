Vagrant.configure("2") do |config|
 if Vagrant.has_plugin? "vagrant-vbguest"
  config.vbguest.no_install = true
  config.vbguest.auto_update = false
  config.vbguest.no_remote = true
end
config.vm.define :firewall do |firewall|
  firewall.vm.box = "generic/centos9s"
  firewall.vm.network :private_network, ip: "209.191.100.3"
  firewall.vm.network :private_network, ip: "192.168.100.3"
  firewall.vm.provision "shell", path: "firewall.sh.txt"
  firewall.vm.hostname = "firewall"
  firewall.vm.provider "virtualbox" do |v|
    v.cpus = 2
  end
end
config.vm.define :maestro do |maestro|
  maestro.vm.box = "generic/centos9s"
  maestro.vm.network :private_network, ip: "192.168.100.4"
  maestro.vm.provision "shell", path: "maestro.sh.txt"
  maestro.vm.hostname = "maestro"
  maestro.vm.provider "virtualbox" do |v|
    v.cpus = 2
  end
 end
 config.vm.define :esclavo do |esclavo|
  esclavo.vm.box = "generic/centos9s"    
  esclavo.vm.network :private_network, ip: "192.168.100.2"
  esclavo.vm.provision "shell", path: "esclavo.sh.txt"
  esclavo.vm.hostname = "esclavo"
  esclavo.vm.provider "virtualbox" do |v|
    v.cpus = 2
  end
 end
end