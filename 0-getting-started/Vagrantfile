Vagrant.configure("2") do |config|

  config.vm.box = "generic/ubuntu2204"

  config.vm.network "forwarded_port", guest: 5601, host: 5601, host_ip: "127.0.0.1"

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = "4096"
  end

end
