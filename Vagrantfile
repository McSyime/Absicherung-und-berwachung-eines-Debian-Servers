# Absicherung und Ueberwachung eines Debian-Servers
# Infrastructure uniquement : installation et configuration manuelles.
Vagrant.configure("2") do |config|
  config.vm.box = "debian/bookworm64"

  # Pas de dossier partage entre l'hote et les machines du laboratoire.
  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.define "server" do |server|
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
