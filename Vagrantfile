# -*- mode: ruby -*-
# vi: set ft=ruby :
Vagrant.require_version ">= 1.4.3"
VAGRANTFILE_API_VERSION = "2"

require 'yaml'
require 'getoptlong'

# Parse CLI arguments.
opts = GetoptLong.new(
  [ '--provider',     GetoptLong::OPTIONAL_ARGUMENT ],
)

provider='virtualbox'
begin
  opts.each do |opt, arg|
    case opt
      when '--provider'
        provider=arg
    end # case
  end # each
  rescue
end

vars_file = "vars/vars.yml"
ansible_config = YAML::load_file("#{File.dirname(File.expand_path(__FILE__))}/#{vars_file}")

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|

    ###Define the starting node number
    ###Can be 1 for Single node deployment or 3 for multinode deployments
    ###Currently supports only 3 nodes in a multinode deployment
    NumVm = 1

    #This will consume in computing last fraction of IP as well
    ip_last_fraction_address = 201
    plays = [ { :play => "prerequisite" },
              { :play => "machine-setup" },
              { :play => "apache-hadoop" },
              { :play => "webserver" } ]


    #Define root disk drive size in GB. Please check any limitations at https://github.com/sprotheroe/vagrant-disksize
    root_disk_size = 150


    #Define machine name initials. This will compromise in hostname as well
    server_initials = ansible_config["build_name_"]
    server_last = ansible_config["build_name_last_"]

    #Current version
    current_version = ansible_config["build_version_"]
    current_version = current_version.tr('.', '')

    #Yarn scheduler                         8088
    #Map Reduce Job History Server          19888
    #Spark History Server                   18088
    #Spark Master Web UI Port server        18080
    #Spark worker Web UI Port               18081
    #Spark Job ports                        4040..4044
    #HDFS host port                         8020
    #Yarn Resourcemanager Scheduler Address 8030
    #Redis server cache                     6379
    #Unassigned ports for external feature  5800..5803
    ports1 = [ 5800, 5801, 5802, 5803, 6379 ]
    ports2 = [ 8088, 8020, 8030 ]
    ports3 = [ 18088, 18080, 18081, 4040, 4042, 4043, 4044 ]
    ports = [ 5800, 5801, 5802, 5803, 50070, 8088, 18088, 18080, 18081, 19888, 4040, 4041, 4042, 4044, 8020, 8030, 6379 ]

    ### Define which linux box need to be used###
    #config.vm.box = "centos/7"
    config.vm.box = "ubuntu/xenial64"

    #Define box information
    config.vm.box_download_insecure = true
    config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'"
    config.disksize.size = "#{root_disk_size}GB"

    #Create 3 virtual machines and map host ansible ssh port to internal port
    #Map application specific ports to host ports
    (1..NumVm).each do |j|
        adder = ip_last_fraction_address + j
        ssh_adder = 2221 + j
        name="#{server_initials}#{current_version}-#{j}.#{server_last}.com"
        config.vm.define  "#{name}" do |node|
            node.vm.hostname="#{name}"
            node.vm.network :private_network, ip: "205.28.128.#{adder}", virtualbox__intnet: true
            node.vm.network :forwarded_port, guest: 22, host: ssh_adder, id: "ssh"
            node.vm.provider "virtualbox" do |v|
                v.name =  "#{name}"
                if NumVm == 1
                  v.memory = 8192
                else
                  v.memory = 4028
                end
                v.cpus = 2
                v.customize [ "modifyvm", :id, "--uartmode1", "disconnected" ]
                v.customize [ "modifyvm", :id, "--nic2", "nat" ]
            end

            #Define port forwarding per VM deployed
            if j == 1
                if NumVm == 1
                    ports.each do |port|
                        node.vm.network :forwarded_port, guest: port, host: port
                    end
                else
                    ports1.each do |port|
                        node.vm.network :forwarded_port, guest: port, host: port
                    end
                end
            end
            if j == 2
                ports2.each do |port|
                    node.vm.network :forwarded_port, guest: port, host: port
                end
            end
            if j == 3
                ports3.each do |port|
                    node.vm.network :forwarded_port, guest: port, host: port
                end
            end
            node.vm.provision :shell do |sh|
              sh.inline = "sudo hostnamectl set-hostname #{name}
                sudo hostnamectl set-hostname #{name} --pretty
                sudo hostnamectl set-hostname #{name} --static
                sudo hostnamectl set-hostname #{name} --transient"
            end

            #Run Ansible prerequisite roles on all deployed nodes
            if j == NumVm
                if NumVm == 3
                   inventory_file='inventory_vbox_template'
                else
                   inventory_file='inventory'
                end
                if config.vm.box.include? 'ubuntu'
                   node.vm.provision :ansible do |preps|
                       #preps.verbose = "vv"
                       preps.limit = "all"
                       preps.playbook = "scripts/ansible-scripts/prerequisite/python.yml"
                       preps.inventory_path = "inventory/#{inventory_file}"
                   end
                end

                #Run Ansible roles to configure hadoop onsudo all deployed nodes
                #Refraining from setting up chrome as persistent failures are observed
                plays.each do |name|
                    node.vm.provision :ansible do |ansible|
                        ansible.limit = "all"
                        #ansible.verbose = "vv"
                        ansible.playbook = "scripts/ansible-scripts/#{name[:play]}/playbook.yml"
                        ansible.inventory_path = "inventory/#{inventory_file}"
                        ansible.extra_vars = {
                            "num_nodes" => NumVm
                       }
                    end
                end

                #Run Ansible roles to start service on all deployed nodes
                #node.vm.provision :ansible, run: 'always' do |startService|
                #    startService.limit = "all"
                #    #startService.verbose = "vv"
                #    startService.playbook = "scripts/ansible-scripts/apache-hadoop/startCluster.yml"
                #    startService.inventory_path = "inventory/#{inventory_file}"
                #end

                #Show box version, do we need to?
                node.vm.provision "shell", run: 'always', inline: "/bin/bash /var/box.version.sh"
            end
        end
    end
end
