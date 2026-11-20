
Vagrant.configure("2") do |config|

  # Image Ubuntu commune
  config.vm.box = "ubuntu/jammy64"
  config.vm.box_check_update = false


  machines = [
    {
      name: "wazuh",
      hostname: "wazuh-01",
      ip: "192.168.56.10",
      cpus: 2,
      memory: 4096
    },
    {
      name: "client",
      hostname: "client-01",
      ip: "192.168.56.11",
      cpus: 2,
      memory: 2048
    },
    {
      name: "caldera",
      hostname: "caldera-01",
      ip: "192.168.56.12",
      cpus: 2,
      memory: 4096
    }
  ]

  machines.each do |machine|

    config.vm.define machine[:name] do |node|

      # Identification
      node.vm.hostname = machine[:hostname]

      # Réseau privé Host-only
      node.vm.network "private_network",
        ip: machine[:ip]

      node.vm.disk :disk,
        size: "60GB",
        primary: true if machine[:name] == "wazuh"
        
      # Configuration VirtualBox
      node.vm.provider "virtualbox" do |vb|

        vb.name = "wazuh-lab-#{machine[:name]}"
        vb.cpus = machine[:cpus]
        vb.memory = machine[:memory]
        vb.gui = false

      end

    end

  end

end
