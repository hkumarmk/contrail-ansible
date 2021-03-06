# -*- mode: ruby -*-
# vi: set ft=ruby :
require 'yaml'

Vagrant.configure("2") do |config|

  default_flavor = 'small'
  default_image = 'trusty'

  # allow users to set their own environment
  # which effect the hiera hierarchy and the
  # cloud file that is used
  environment = ENV['env'] || 'vagrant-vbox'
  layout = ENV['layout'] || 'vagrant_nodes'
  map = ENV['map'] || layout

  vagrant_root = File.dirname(File.expand_path(__FILE__))

  config.vm.provider :virtualbox do |vb, override|
    vb.customize ["modifyvm", :id, "--natdnshostresolver1", "on"]
    vb.customize ["modifyvm", :id, "--natdnsproxy1", "on"]
  end

  config.vm.provider "lxc" do |v, override|
    override.vm.box = "fgrehm/trusty64-lxc"
  end

  last_octet = 41
  env_data = YAML.load_file("#{layout}.yaml")["resources"]

  map_data = YAML.load_file("#{map}.yaml")["maps"]

  machines = {}
  env_data.each do |name, info|
    (1..info['number']).to_a.each do |idx|
      machines["#{name}#{idx}"] = info
    end
  end

  machines.each do |node_name, info|
    config.vm.define(node_name) do |config|

      config.vm.provider :virtualbox do |vb, override|
        image = info['image'] || default_image
        override.vm.box = map_data['image']['virtualbox'][image]

        flavor = info['flavor'] || default_flavor
        vb.memory = map_data['flavor'][flavor]['ram']
        vb.cpus = map_data['flavor'][flavor]['cpu']
        second_disk = map_data['flavor'][flavor]['disk']
        if second_disk
          if ! ENV['extra_disk_location']
            warn("The extra_disk_location is not specifiied, creating in vagrant directory")
          end
          disk_location = ENV['extra_disk_location'] || vagrant_root
          disk_file =  "#{disk_location}/#{node_name}.vdi"
          unless File.exist?("#{disk_file}")
            vb.customize ['createhd', '--filename', "#{disk_file}", '--size', second_disk * 1024]
          end
          vb.customize ['storageattach', :id, '--storagectl', 'SATAController', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', "#{disk_file}" ]
        end
      end
      hostname = node_name.gsub('_','-')
      config.vm.host_name = "#{hostname}.domain.name"

      http_proxy = ENV['env_http_proxy'] || ENV['http_proxy']
      https_proxy = ENV['env_https_proxy'] || ENV['https_proxy']

      if http_proxy
        config.vm.provision 'shell', :inline =>
        "echo \"Acquire::http { Proxy \\\"#{http_proxy}\\\" }\" > /etc/apt/apt.conf.d/03proxy"
        config.vm.provision 'shell', :inline =>
        "echo http_proxy=#{http_proxy} >> /etc/environment"
      end

      if https_proxy
        config.vm.provision 'shell', :inline =>
        "echo https_proxy=#{https_proxy} >> /etc/environment"
      end

# Install/upgrade ansible, docker, and other stuffs required for basic operations
      config.vm.provision 'shell', :inline => <<EOF
      export DEBIAN_FRONTEND=noninteractive ;
      apt-get install -y --force-yes software-properties-common;
      apt-add-repository -y ppa:ansible/ansible;
      apt-get -q update;
      apt-get install -y --force-yes ansible git curl python-pip
EOF

      inventory = ENV['ansible_inventory'] || "svl_allinone_wo_os"
      config.vm.provision 'shell', :inline => <<EOF
      cd /vagrant/playbooks
      ansible-galaxy install -r requirements.yml
      ansible-playbook -i inventory/#{inventory} site.yml
EOF
      net_prefix = ENV['NET_PREFIX'] || "192.168.100.0"
      config.vm.network "private_network", :type => :dhcp, :ip => net_prefix, :netmask => "255.255.255.0"
    end
  end
end
