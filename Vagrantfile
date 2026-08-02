# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# Chain test box: runs base_freebsd then hardening_freebsd, the real deploy order.
#
# Prereq (host): ansible-galaxy collection install -r tests/requirements.yml
#
# No libvirt FreeBSD 15 box published yet, so default is generic/freebsd14 (mechanisms are
# version-agnostic). Override: FREEBSD_BOX=generic/freebsd15 vagrant up

FREEBSD_BOX = ENV.fetch("FREEBSD_BOX", "generic/freebsd14")
MGMT_CIDR   = ENV.fetch("FREEBSD_MGMT_CIDR", "192.168.121.0/24")

# Vagrant won't auto-load the repo ansible.cfg for a subdir playbook, so point
# ANSIBLE_CONFIG at it explicitly.
ENV["ANSIBLE_CONFIG"] = File.join(__dir__, "ansible.cfg")

Vagrant.configure("2") do |config|
  config.vm.box_check_update = false
  config.vm.synced_folder ".", "/vagrant", disabled: true

  # Install the base layer on the HOST first so the chain's import_playbook resolves as a
  # collection, not a file.
  config.trigger.before :provision do |t|
    t.info = "Installing base_freebsd + deps for the chain smoke"
    t.run = { inline: "ansible-galaxy collection install -r tests/requirements.yml --force" }
  end

  config.vm.define "hardening" do |h|
    h.vm.box = FREEBSD_BOX
    h.vm.hostname = "fbsd-harden"
    h.vm.guest = :freebsd
    h.ssh.shell = "/bin/sh"

    h.vm.provider :libvirt do |libvirt|
      libvirt.driver = "kvm"
      libvirt.uri = "qemu:///system"
      libvirt.memory = "2048"
      libvirt.cpus = 2
    end

    h.vm.provision "ansible" do |a|
      a.playbook = "tests/chain.yml"
      a.compatibility_mode = "2.0"
      # The box is in BOTH groups: base targets base_hosts, harden targets hardened_hosts.
      a.groups = {
        "base_hosts"     => ["hardening"],
        "hardened_hosts" => ["hardening"],
      }
      # deploy_user=vagrant so ssh_2fa's Match User keeps that connection key-only,
      # otherwise AuthenticationMethods would demand a TOTP the vagrant user lacks.
      a.extra_vars = {
        "management_cidrs" => [MGMT_CIDR],
        "firewall_allow_tcp_mgmt" => [22],
        "deploy_user" => "vagrant",
        "base_users_ssh_dir" => "#{__dir__}/files/ssh",
        "target_users" => [
          { "name" => "root", "home" => "/root", "group" => "wheel" },
          { "name" => "acid", "home" => "/home/acid", "group" => "acid" },
        ],
      }
    end
  end
end
