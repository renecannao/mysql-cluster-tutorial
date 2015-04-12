Vagrant::Config.run do |config|
  config.vm.define :node1 do |node1_config|
    node1_config.vm.box = "ndbcluster"
    node1_config.vm.host_name = "node1"
    node1_config.vm.customize ["modifyvm", :id, "--memory", "128"]
    node1_config.vm.network :hostonly, "192.168.123.101"
    node1_config.vm.share_folder "v-data", "/vagrant_data", "/tmp"
  end
  config.vm.define :node2 do |node2_config|
    node2_config.vm.box = "ndbcluster"
    node2_config.vm.host_name = "node2"
    node2_config.vm.customize ["modifyvm", :id, "--memory", "768"]
    node2_config.vm.network :hostonly, "192.168.123.102"
    node2_config.vm.share_folder "v-data", "/vagrant_data", "/tmp"
  end
  config.vm.define :node3 do |node3_config|
    node3_config.vm.box = "ndbcluster"
    node3_config.vm.host_name = "node3"
    node3_config.vm.customize ["modifyvm", :id, "--memory", "768"]
    node3_config.vm.network :hostonly, "192.168.123.103"
    node3_config.vm.share_folder "v-data", "/vagrant_data", "/tmp"
  end
end
