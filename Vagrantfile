# -*- mode: ruby -*-
# vi: set ft=ruby :

box = {
  :virtualbox => { :name => 'elastic/ubuntu-16.04-x86_64', :version => '20180708.0.0' },
  :libvirt => { :name => 'elastic/ubuntu-16.04-x86_64', :version=> '20180210.0.0'}
}

require 'yaml'
pdf = File.dirname(__FILE__) + '/config/default.yml'
if File.exist?(File.dirname(__FILE__) + '/config/pdf.yml')
  pdf = File.dirname(__FILE__) + '/config/pdf.yml'
end
nodes = YAML.load_file(pdf)

# Inventory file creation
File.open(File.dirname(__FILE__) + "/inventory/hosts.ini", "w") do |inventory_file|
  inventory_file.puts("[all:vars]\nansible_connection=ssh\nansible_ssh_user=vagrant\nansible_ssh_pass=vagrant\n\n[all]")
  nodes.each do |node|
    inventory_file.puts("#{node['name']}\tansible_ssh_host=#{node['ip']} ansible_ssh_port=22")
  end
  ['kube-master', 'kube-node', 'etcd', 'ovn-central', 'ovn-controller', 'virtlet'].each do|group|
    inventory_file.puts("\n[#{group}]")
    nodes.each do |node|
      if node['roles'].include?("#{group}")
        inventory_file.puts(node['name'])
      end
    end
  end
  inventory_file.puts("\n[k8s-cluster:children]\nkube-node\nkube-master")
end

provider = (ENV['VAGRANT_DEFAULT_PROVIDER'] || :virtualbox).to_sym
puts "[INFO] Provider: #{provider} "

if ENV['no_proxy'] != nil or ENV['NO_PROXY']
  $no_proxy = ENV['NO_PROXY'] || ENV['no_proxy'] || "127.0.0.1,localhost"
  nodes.each do |node|
    $no_proxy += "," + node['ip']
  end
  $subnet = "192.168.121"
  if provider == :virtualbox
    $subnet = "10.0.2"
  end
  # NOTE: This range is based on vagrant-libvirt network definition CIDR 192.168.121.0/27
  (1..31).each do |i|
    $no_proxy += ",#{$subnet}.#{i}"
  end
end

Vagrant.configure("2") do |config|
  config.vm.box =  box[provider][:name]
  config.vm.box_version = box[provider][:version]

  if ENV['http_proxy'] != nil and ENV['https_proxy'] != nil
    if Vagrant.has_plugin?('vagrant-proxyconf')
      config.proxy.http     = ENV['http_proxy'] || ENV['HTTP_PROXY'] || ""
      config.proxy.https    = ENV['https_proxy'] || ENV['HTTPS_PROXY'] || ""
      config.proxy.no_proxy = $no_proxy
      config.proxy.enabled = { docker: false }
    end
  end

  nodes.each do |node|
    config.vm.define node['name'] do |nodeconfig|
      nodeconfig.vm.hostname = node['name']
      nodeconfig.vm.network :private_network, :ip => node['ip'], :type => :static
      nodeconfig.vm.provider 'virtualbox' do |v|
        v.customize ["modifyvm", :id, "--memory", node['memory']]
        v.customize ["modifyvm", :id, "--cpus", node['cpus']]
        if node.has_key? "volumes"
          node['volumes'].each do |volume|
            $volume_file = "#{node['name']}-#{volume['name']}.vdi"
            unless File.exist?($volume_file)
              v.customize ['createmedium', 'disk', '--filename', $volume_file, '--size', volume['size']]
            end
            v.customize ['storageattach', :id, '--storagectl', 'IDE Controller', '--port', 1, '--device', 0, '--type', 'hdd', '--medium', $volume_file]
          end
        end
      end
      nodeconfig.vm.provider 'libvirt' do |v|
        v.memory = node['memory']
        v.cpus = node['cpus']
        v.nested = true
        v.cpu_mode = 'host-passthrough'
        v.management_network_address = "192.168.121.0/27"
        nodeconfig.vm.provision 'shell' do |sh|
          sh.path =  "node.sh"
          if node.has_key? "volumes"
            $volume_mounts_dict = ''
            node['volumes'].each do |volume|
              $volume_mounts_dict += "#{volume['name']}=#{volume['mount']},"
              $volume_file = "./#{node['name']}-#{volume['name']}.qcow2"
              v.storage :file, :bus => 'sata', :device => volume['name'], :size => volume['size']
            end
            sh.args = ['-v', $volume_mounts_dict[0...-1]]
          end
        end
      end
    end
  end
  sync_type = "virtualbox"
  if provider == :libvirt
    sync_type = "nfs"
  end
  config.vm.define :installer, primary: true, autostart: false do |installer|
    installer.vm.hostname = "multicloud"
    installer.vm.network :private_network, :ip => "10.10.10.2", :type => :static
    installer.vm.provision 'shell' do |sh|
      sh.path =  "installer.sh"
      sh.args = ['-p', '-v', '-w', '/vagrant']
    end
  end
end
