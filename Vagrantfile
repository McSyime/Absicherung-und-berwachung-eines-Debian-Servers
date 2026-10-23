# Absicherung und Ueberwachung eines Debian-Servers
# Machines preconfigurees depuis la release v1.0.0.

Vagrant.configure("2") do |config|
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.define "server" do |server|
    server.vm.box = "mcsyime/security-server-v1.0.0"
    server.vm.box_url = "https://github.com/McSyime/Absicherung-und-berwachung-eines-Debian-Servers/releases/download/v1.0.0/security-server.box"

    server.vm.hostname = "security-server"
    server.vm.network "private_network",
      ip: "192.168.50.10",
      virtualbox__intnet: "debian-security-lab"

    server.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
    end
  end

  config.vm.define "tester" do |tester|
    tester.vm.box = "mcsyime/security-tester-v1.0.0"
    tester.vm.box_url = "https://github.com/McSyime/Absicherung-und-berwachung-eines-Debian-Servers/releases/download/v1.0.0/security-tester.box"

    tester.vm.hostname = "security-tester"
    tester.vm.network "private_network",
      ip: "192.168.50.20",
      virtualbox__intnet: "debian-security-lab"

    tester.vm.provider "virtualbox" do |vb|
      vb.memory = 1024
      vb.cpus = 1
    end
  end
end
