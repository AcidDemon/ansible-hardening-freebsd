# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# Chain test box: runs base_freebsd then hardening_freebsd, the real deploy order.
#
# Prereq (host): ansible-galaxy collection install -r tests/requirements.yml
#
# No libvirt FreeBSD 15 box published yet, so default is bento/freebsd-14 pinned to 14.3,
# the same box and version as the backupbox harness. Not generic/freebsd14: that is 14.0,
# whose OpenSSH 9.5 rejects the mlkem768x25519-sha256 kex and fails `sshd -t`.
# Override: FREEBSD_BOX=generic/freebsd15 BOX_VERSION=">= 0" vagrant up

FREEBSD_BOX = ENV.fetch("FREEBSD_BOX", "bento/freebsd-14")
BOX_VERSION = ENV.fetch("BOX_VERSION", "202508.03.0")
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
    h.vm.box_version = BOX_VERSION
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
