VAGRANTFILE_API_VERSION = "2"
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.box = "ndbcluster"
  config.ssh.forward_agent = true
  config.vm.provider "virtualbox" do |vb|
  end

config.vm.define :node1 do |node1_config|
  node1_config.vm.box = "ndbcluster"
  node1_config.vm.host_name = "node1"
  node1_config.vm.provider "virtualbox" do |vb1|
    vb1.customize ["modifyvm", :id, "--memory", "128"]
  end
  node1_config.vm.network "private_network", ip: "192.168.123.101"
end


config.vm.define :node2 do |node2_config|
  node2_config.vm.box = "ndbcluster"
  node2_config.vm.host_name = "node2"
  node2_config.vm.provider "virtualbox" do |vb2|
    vb2.customize ["modifyvm", :id, "--memory", "768"]
  end
  node2_config.vm.network "private_network", ip: "192.168.123.102"
end


config.vm.define :node3 do |node3_config|
  node3_config.vm.box = "ndbcluster"
  node3_config.vm.host_name = "node3"
  node3_config.vm.provider "virtualbox" do |vb3|
    vb3.customize ["modifyvm", :id, "--memory", "768"]
  end
  node3_config.vm.network "private_network", ip: "192.168.123.103"
end

end
