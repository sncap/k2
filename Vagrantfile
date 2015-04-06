# -*- mode: ruby -*-
# vi: set ft=ruby :

require 'fileutils'
require 'net/http'
require 'open-uri'

class Module
  def redefine_const(name, value)
    __send__(:remove_const, name) if const_defined?(name)
    const_set(name, value)
  end
end

# Cluster parameters
#-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-
MASTER_YAML = File.join(File.dirname(__FILE__), "master.yaml")
NODE_YAML = File.join(File.dirname(__FILE__), "node.yaml")
LOCAL_LANDRUSH = "./plugins/landrush-0.18.13.gem"
KUBERNETES_VERSION = '0.13.2'
CHANNEL = 'alpha'
COREOS_VERSION = 'latest'
MASTER_IP = '172.16.1.101'
NUM_INSTANCES = 2
MASTER_MEM =  512
MASTER_CPUS = 1
NODE_MEM= 1024
NODE_CPUS = 1
REBOOT_STRAT = 'off'
#-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-=-

# Need to install a local version of landrush and not the one from rubygems
unless Vagrant.has_plugin? 'landrush'
  system "vagrant plugin install #{LOCAL_LANDRUSH}"
  need_restart = true
  exec "vagrant #{ARGV.join(' ')}"
end

# Install all other plugins(gems)
required_plugins = %w(vagrant-triggers)
required_plugins.each do |plugin|
  need_restart = false
  unless Vagrant.has_plugin? plugin
    system "vagrant plugin install #{plugin}"
    need_restart = true
  end
  exec "vagrant #{ARGV.join(' ')}" if need_restart
end

upstream = "http://#{CHANNEL}.release.core-os.net/amd64-usr/#{COREOS_VERSION}"
if COREOS_VERSION == "latest"
  upstream = "http://#{CHANNEL}.release.core-os.net/amd64-usr/current"
  url = "#{upstream}/version.txt"
  Object.redefine_const(:COREOS_VERSION,
    open(url).read().scan(/COREOS_VERSION=.*/)[0].gsub('COREOS_VERSION=', ''))
end

SERIAL_LOGGING = (ENV['SERIAL_LOGGING'].to_s.downcase == 'true')
GUI = (ENV['GUI'].to_s.downcase == 'true')

Vagrant.require_version ">= 1.6.0"

(1..(NUM_INSTANCES.to_i + 1)).each do |i|
  case i
  when 1
    hostname = "master"
  else
    hostname = ",node-%02d" % (i - 1)
  end
end

Vagrant.configure(2) do |config|
  # always use Vagrants' insecure key
  config.ssh.insert_key = false
  config.ssh.forward_agent = true

  config.vm.box = "coreos-#{CHANNEL}"
  config.vm.box_version = ">= #{COREOS_VERSION}"
  config.vm.box_url = "#{upstream}/coreos_production_vagrant.json"

  config.trigger.after [:up, :resume] do
    info "making sure ssh agent has the default vagrant key..."
    system "ssh-add ~/.vagrant.d/insecure_private_key"
  end

  # Getting landrush going
  config.landrush.enabled = true
  config.landrush.tld = 'kube'
  config.landrush.host 'sandy.mycluster.kube', '172.16.1.101'

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end
  config.vm.provider :parallels do |p|
    p.update_guest_tools = false
    p.check_guest_tools = false
  end

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  (1..(NUM_INSTANCES.to_i + 1)).each do |i|
    if i == 1
      hostname = "master"
      cfg = MASTER_YAML
      memory = MASTER_MEM
      cpus = MASTER_CPUS
    else
      hostname = "node-%02d" % (i - 1)
      cfg = NODE_YAML
      memory = NODE_MEM
      cpus = NODE_CPUS
    end

    config.vm.define vmName = hostname do |kHost|
      kHost.vm.hostname = vmName
      # Do not run landrush on vms because of IPTABlES not being on coreos natively
      kHost.landrush.guest_redirect_dns = false      
      
      if SERIAL_LOGGING
        logdir = File.join(File.dirname(__FILE__), "log")
        FileUtils.mkdir_p(logdir)

        serialFile = File.join(logdir, "#{vmName}-serial.txt")
        FileUtils.touch(serialFile)

        kHost.vm.provider :virtualbox do |vb, override|
          vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
          vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
        end
      end

      kHost.vm.network :private_network, ip: "172.16.1.#{i+100}"

      if File.exist?(cfg)
        kHost.vm.provision :file, :source => "#{cfg}", :destination => "/tmp/vagrantfile-user-data"
        kHost.vm.provision :shell, :privileged => true,
        inline: <<-EOF
          sed -i "s,__RELEASE__,v#{KUBERNETES_VERSION},g" /tmp/vagrantfile-user-data
          sed -i "s,__CHANNEL__,v#{CHANNEL},g" /tmp/vagrantfile-user-data
          sed -i "s,__NAME__,#{hostname},g" /tmp/vagrantfile-user-data
          sed -i "s,__MASTER_IP__,#{MASTER_IP},g" /tmp/vagrantfile-user-data
          sed -i "s,__REBOOT_STRAT__,#{REBOOT_STRAT},g" /tmp/vagrantfile-user-data
          mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/
        EOF
      end
    end
  end
end
