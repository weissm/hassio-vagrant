# ==============================================================================
#
# Community Hass.io Add-ons: Vagrant
#
# ==============================================================================
# MIT License
#
# Copyright (c) 2017 Franck Nijhof
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in all
# copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
# SOFTWARE.
# ==============================================================================

require 'English'
require 'fileutils'
require 'vagrant'
require 'yaml'
require 'pp'

::Vagrant.require_version '>= 2.1.0'

module HassioCommunityAddons
  # rubocop:disable Metrics/ClassLength
  # Manages the Vagrant configuration
  # @author Franck Nijhof <frenck@addons.community>
  class Vagrant
    # Class constructor
    def initialize
      unless `vboxmanage list extpacks`.include? \
        'Oracle VM VirtualBox Extension Pack'
        raise ::Vagrant::Errors::VagrantError.new, \
              'Could not find VirtualBox Extension Pack! Did you install it?'
      end

      @config = YAML.load_file(
        File.join(File.dirname(__FILE__), 'configuration.yml')
      )
    end

    # Simple CLI Yes / No question
    # Only used by Windows Setup
    # @param [String] message Question to ask
    # @param [Boolean] default True, to default to yes, false to default to no
    # @return [Boolean] True if answered yes, false if answered no
    def confirm(message, default)
      print "#{message} [#{(default ? 'Y/n' : 'y/N')}]: "

      result = $stdin.gets.chomp.strip.downcase
      return default if result.empty?
      return true if %w[y yes].include? result
      return false if %w[n no].include? result

      print "\nInvalid input. Try again...\n"
      confirm(message, default)
    end

    # Check for addon on Windows for NFS as SMB is slow
    # rubocop:disable Metrics/MethodLength
    def require_nfs_plugin
      return if ::Vagrant.has_plugin?('vagrant-winnfsd')

      print "It is recommended to use the following plugin on Windows \
(SMB is slow): vagrant-winnfsd\n"
      if confirm 'Shall I go ahead and install it?', true
        raise ::Vagrant::Errors::VagrantError.new, 'Plugin Install failed' \
          unless system 'vagrant plugin install vagrant-winnfsd'

        print "Restarting Vagrant to reload plugin changes...\n"
        system 'vagrant ' + ARGV.join(' ')
        exit! $CHILD_STATUS.exitstatus
      else
        print "Recommended plugin missing, continuing with SMB...\n" \
      end
    end

    # rubocop:enable Metrics/MethodLength
    # Configures generic Vagrant options
    #
    # @param [Vagrant::Config::V2::Root] config Vagrant root config
    def vagrant_config(config)
      config.vm.box = @config['box']
      config.vm.post_up_message = @config['post_up_message']
      config.vm.boot_timeout = 400
    end

    # Defines a Vagrant virtual machine
    #
    # @param [Vagrant::Config::V2::Root] config Vagrant root config
    # @param [String] name Name of the machine to define
    def machine(config, name)
      config.vm.define name do |machine|
        machine_config machine
        machine_provider_virtualbox machine
        machine_shares machine
        machine_provision machine
        machine_cleanup_on_destroy machine unless @config['keep_config']
      end
    end

    # Configures a VM's generic options
    #
    # @param [Vagrant::Config::V2::Root] machine Vagrant VM root config
    def machine_config(machine)
      machine.vm.hostname = @config['hostname']
      machine.vm.network 'private_network', type: 'dhcp'
      machine.vm.network(
        'public_network',
        type: 'dhcp',
        bridge: @config['bridge']
      )
    end

    # Configures the Virtualbox provider
    #
    # @param [Vagrant::Config::V2::Root] machine Vagrant VM root config
    # rubocop:disable Metrics/MethodLength
    def machine_provider_virtualbox(machine)
      machine.vm.provider :virtualbox do |vbox|
        vbox.name = @config['hostname']
        vbox.cpus = @config['cpus']
        vbox.customize ['modifyvm', :id, '--memory', @config['memory']]
        vbox.customize ['modifyvm', :id, '--nictype1', 'virtio']
        vbox.customize ['modifyvm', :id, '--nictype2', 'virtio']
        vbox.customize ['modifyvm', :id, '--nictype3', 'virtio']
        vbox.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
        vbox.customize ['modifyvm', :id, '--natdnsproxy1', 'on']
        vbox.customize ['modifyvm', :id, '--usb', 'on', '--usbehci', 'on']
      end
    end
    # rubocop:enable Metrics/MethodLength
    # Configures a VM's shares
    #
    # @param [Vagrant::Config::V2::Root] machine Vagrant VM root config

    def machine_shares(machine)
      @config['shares'].each do |src, dst|
        machine.vm.synced_folder(
          src,
          dst,
          create: true,
          type: share_type,
          SharedFoldersEnableSymlinksCreate: false
        )
      end
    end

    # Determines the type of filesharing. SMB for windows (without nfs plugin)
    # else NFS.
    # rubocop:disable Style/MultilineTernaryOperator
    def share_type
      ::Vagrant::Util::Platform.windows? &&
        !::Vagrant.has_plugin?('vagrant-winnfsd') ? 'smb' : 'nfs'
    end

    # rubocop:enable Style/MultilineTernaryOperator
    # Configures a VM's provisioning
    #
    # @param [Vagrant::Config::V2::Root] machine Vagrant VM root config
    def machine_provision(machine)
      machine.vm.provision 'fix-no-tty', type: 'shell' do |shell|
        shell.path = 'provision.sh'
      end
    end

    # Define cleanup command based on OS
    def os_cleanup_task
      config_directory = File.join(File.dirname(__FILE__), 'config')
      if ::Vagrant::Util::Platform.windows?
        "gci '#{config_directory}' -depth 1 " \
        ' -exclude ".gitkeep" | Remove-Item -recurse'
      else
        "find '#{config_directory}' -mindepth 1 -maxdepth 1" \
        ' -not -name ".gitkeep" -exec rm -rf {} \;'
      end
    end

    # Defines a VM cleanup task when destroying the VM
    #
    # @param [Vagrant::Config::V2::Root] machine Vagrant VM root config
    def machine_cleanup_on_destroy(machine)
      machine.trigger.after :destroy do |trigger|
        trigger.name = 'Cleanup'
        trigger.info = 'Cleaning up Home Assistant configuration'
        trigger.run = {
          inline: os_cleanup_task
        }
      end
    end

    # Run this thing!
    def run
      ::Vagrant.configure('2') do |config|
        require_nfs_plugin if ::Vagrant::Util::Platform.windows?
        vagrant_config(config)
        if ::Vagrant.has_plugin?("vagrant-vbguest")
            config.vbguest.no_install  = true
            config.vbguest.auto_update = false
            config.vbguest.no_remote   = true
        end        
        machine(config, 'hassio')
      end
    end
  end
  # rubocop:enable Metrics/ClassLength
end

# Create a instance
hassio = HassioCommunityAddons::Vagrant.new

# Go!
hassio.run
