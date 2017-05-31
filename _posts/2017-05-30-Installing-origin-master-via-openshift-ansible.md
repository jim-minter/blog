---
layout: post
title:  "Installing origin master via openshift-ansible"
date:   2017-05-31
---
# Installing origin master via openshift-ansible

### 1. Prerequisities

- Fedora 25 host with Vagrant (libvirt) installed.

### 2. Bring up Vagrant VM

```shell
mkdir -p vagrant/origin
cd vagrant/origin

cat >Vagrantfile <<'EOF'
class BugWorkarounds < Vagrant.plugin("2")
  name "BugWorkarounds"

  action_hook(:hook, :machine_action_up) do |hook|
    hook.append(BugWorkarounds)
  end

  def initialize(app, env)
    @app = app
  end

  def call(env)
    @app.call(env)

    cmds = <<~END
      if [ ! -e /etc/bugworkarounds ]; then
        # bz1402083
        dnf -y update selinux-policy

        # bz1415496
        mkdir -p /run/rpcbind

        >/etc/bugworkarounds
      fi
    END

    env[:machine].communicate.sudo(cmds)
  end
end

Vagrant.configure("2") do |config|
  config.vm.box = "fedora/25-cloud-base"

  config.vm.provider "libvirt" do |libvirt|
    libvirt.cpus = 8
    libvirt.memory = 8192
  end

  config.vm.synced_folder ".", "/vagrant", type: "nfs"
end
EOF

vagrant up
vagrant ssh -c 'sudo dnf -y update; reboot'
vagrant up
vagrant ssh
```

### 3. Prepare Vagrant VM

```shell
sudo -i

dnf -y install ansible bsdtar createrepo docker git golang krb5-devel NetworkManager tito

systemctl enable NetworkManager
systemctl start NetworkManager

systemctl enable docker
systemctl start docker

echo 'export GOPATH=$HOME/go' >>.bash_profile
echo 'export PATH=$PATH:$GOPATH/bin' >>.bash_profile
. .bash_profile

git config --global user.name "Your Name"
git config --global user.email "you@example.com"

IP=$(ifconfig eth0 | awk '/inet / { print $2}')

mkdir -p .ssh
ssh-keyscan $IP >>.ssh/known_hosts
ssh-keygen -f .ssh/id_rsa -P ''
cat .ssh/id_rsa.pub >>.ssh/authorized_keys

go get github.com/openshift/imagebuilder/cmd/imagebuilder
```

### 4. Get and build Origin

```shell
go get github.com/openshift/origin

cd $GOPATH/src/github.com/openshift/origin
hack/build-base-images.sh
make release
```

### 5. Get and run Openshift-Ansible

```shell
cd

git clone https://github.com/openshift/openshift-ansible

IP=$(ifconfig eth0 | awk '/inet / { print $2}')

cat >/etc/ansible/hosts <<END
[OSEv3:children]
masters
nodes

[OSEv3:vars]
openshift_deployment_type=origin
ansible_ssh_user=root
ansible_python_interpreter=/usr/bin/python3
openshift_disable_check=disk_availability,memory_availability
openshift_additional_repos=[{'id': 'origin-local-release', 'name': 'OpenShift Release from Local Source', 'baseurl': 'file://$GOPATH/src/github.com/openshift/origin/_output/local/releases/rpms', 'enabled': 1, 'gpgcheck': 0}]
openshift_schedulable=true
openshift_node_labels={'region': 'infra'}
oreg_url=openshift/origin-\${component}:latest
openshift_examples_modify_imagestreams=true

[masters]
$IP

[nodes]
$IP
END

ansible-playbook openshift-ansible/playbooks/byo/config.yml

```

### 6. Removing Origin

```shell
rpm -qa | grep origin | xargs dnf -y remove
umount /var/lib/origin/openshift.local.volumes/pods/*/volumes/*/*
rm -rf /etc/sysconfig/origin-* /etc/origin/ /var/lib/origin $HOME/.kube/
```
