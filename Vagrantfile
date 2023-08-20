# Zabbix HA Test Lab

servers = [
    {
        ### Zabbix Server Node 01 ###
        :hostname => "zabbix-ha-server-01",
        :box => "geerlingguy/rockylinux8",
        :ip => "192.168.60.11",
        :memory => "1024",
        :cpu => "1",
        :vmname => "zabbix-ha-server-01"
    },
    {
        ### Zabbix Server Node 02 ###
        :hostname => "zabbix-ha-server-02",
        :box => "geerlingguy/rockylinux8",
        :ip => "192.168.60.12",
        :memory => "1024",
        :cpu => "1",
        :vmname => "zabbix-ha-server-02"
    },
    {
        ### Zabbix Database Node ###
        :hostname => "zabbix-ha-db",
        :box => "geerlingguy/rockylinux8",
        :ip => "192.168.60.13",
        :memory => "1024",
        :cpu => "1",
        :vmname => "zabbix-ha-db"
    }
]

Vagrant.configure("2") do |config|

    # manages the /etc/hosts file on guest machines in multi-machine environments
    # vagrant plugin install vagrant-hostmanager
    config.hostmanager.enabled = true
    config.hostmanager.manage_host = true

    servers.each do |machine|
    
        config.vm.define machine[:hostname] do |node|

            node.vm.box = machine[:box]
            node.vm.hostname = machine[:hostname]
            node.vm.network :private_network, ip: machine[:ip]
 
            node.vm.network "public_network"

            node.vm.provider :virtualbox do |vb|

                vb.customize ["modifyvm", :id, "--name", machine[:vmname]]
                vb.customize ["modifyvm", :id, "--memory", machine[:memory]]
                vb.customize ["modifyvm", :id, "--cpus", machine[:cpu]]

                # fix for slow network speed issue
                vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
                vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]

            end # end provider
            
            # node.vm.provision "shell", path: machine[:script]
            
        end # end config
    end # end servers each loop
end # end Vagrantfile
