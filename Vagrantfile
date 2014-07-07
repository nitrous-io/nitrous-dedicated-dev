# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = '2'

$script = <<SCRIPT
export VAGRANT_HOME=/home/vagrant
export GOPATH=$VAGRANT_HOME/go
export GOROOT=/usr/local/opt/go

########################################
####### INSTALL APT DEPS ###############
########################################

apt-get install -yy --no-install-recommends lxc build-essential curl git-core mercurial postgresql-9.3 postgresql-contrib-9.3 libpq-dev

########################################
####### CONFIGURE PG ###################
########################################

sudo su postgres -c 'psql -c "CREATE ROLE vagrant NOSUPERUSER CREATEDB NOCREATEROLE INHERIT LOGIN;"'

########################################
####### INSTALL GO #####################
########################################

install_go() {
  echo "Installing Go..."
  mkdir -p $GOROOT
  curl -s https://storage.googleapis.com/golang/go1.2.2.linux-amd64.tar.gz | tar -v -C /usr/local/opt -xz
}

if [[ ! -e $GOROOT/bin/go ]]; then
  install_go
  echo "export GOPATH=$GOPATH" >> /etc/profile
  echo "export GOROOT=$GOROOT" >> /etc/profile
  echo "export PATH=$GOPATH/bin:$GOROOT/bin:/usr/local/opt/go/bin:/usr/local/bin:/usr/local/sbin:/usr/bin:/usr/sbin:/bin:/sbin" >> /etc/profile
else
  if [[ ! $(go version | grep go1.2) ]]; then
    install_go
    rm -rf $GOPATH/pkg $GOPATH/bin
  fi
fi

########################################
####### INSTALL GO DEV TOOLS  ##########
########################################

/usr/local/opt/go/bin/go get github.com/nitrous-io/goop
/usr/local/opt/go/bin/go get code.google.com/p/go.tools/cmd/goimports
/usr/local/opt/go/bin/go get github.com/onsi/ginkgo/ginkgo
/usr/local/opt/go/bin/go get github.com/onsi/gomega

########################################
####### SETUP DOCKER ###################
########################################

# Install Docker
curl -s https://get.docker.io/ubuntu/ | sudo sh

cat > /etc/default/docker.io <<EOF
#Docker Upstart and SysVinit configuration file

# Customize location of Docker binary (especially for development testing).
#DOCKER="/usr/local/bin/docker"

# Use DOCKER_OPTS to modify the daemon startup options.
DOCKER_OPTS="-dns 8.8.8.8 -dns 8.8.4.4 -e lxc"

# If you need Docker to use an HTTP proxy, it can also be specified here.
#export http_proxy="http://127.0.0.1:3128/"

# This is also a handy place to tweak where Docker's temporary files go.
#export TMPDIR="/mnt/bigdrive/docker-tmp"
EOF

service docker restart

adduser vagrant docker

########################################
####### INSTALL SOURCES ################
########################################

# Install all sources
mkdir -p /home/vagrant/go/src/github.com/nitrous-io/
mkdir -p /home/vagrant/workspace

ssh -T git@github.com -o StrictHostKeyChecking=no && echo 'Success'

cd /home/vagrant/go/src/github.com/nitrous-io/
if [ ! -d nitrous ]; then
  git clone git@github.com:nitrous-io/nitrous.git
  ln -fs /home/vagrant/go/src/github.com/nitrous-io/nitrous /home/vagrant/workspace/nitrous
fi

if [ ! -d freighter ]; then
  git clone git@github.com:nitrous-io/freighter.git
  ln -fs /home/vagrant/go/src/github.com/nitrous-io/freighter /home/vagrant/workspace/freighter
fi

if [ ! -d hostbuilder ]; then
  git clone git@github.com:nitrous-io/hostbuilder.git
  ln -fs /home/vagrant/go/src/github.com/nitrous-io/hostbuilder /home/vagrant/workspace/hostbuilder
fi

if [ ! -d nitro ]; then
  git clone git@github.com:nitrous-io/nitro.git
  ln -fs /home/vagrant/go/src/github.com/nitrous-io/nitro /home/vagrant/workspace/nitro
fi

sudo chown -R vagrant:vagrant $VAGRANT_HOME/go
sudo chown -R vagrant:vagrant $VAGRANT_HOME/workspace
SCRIPT

Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.vm.hostname = 'dedicated-vm'
  config.vm.box = 'nitrous-io/trusty64'

  config.vm.provider :virtualbox do |v|
    v.memory = ENV['DEDICATED_VM_MEM'] || 2048
    v.cpus   = ENV['DEDICATED_VM_CPU'] || 2
    v.customize ['modifyvm', :id, '--natdnshostresolver1', 'on']
    v.customize ['modifyvm', :id, '--natdnsproxy1', 'on']
  end

  config.vm.provider 'vmware_fusion' do |v|
    v.gui = true
    v.vmx['memsize'] = ENV['DEDICATED_VM_MEM'] || '2048'
    v.vmx['numvcpus'] = ENV['DEDICATED_VM_CPU'] || '2'
  end

  if private_ip = ENV['DEDICATED_VM_INTERFACE']
    config.vm.network :private_network, ip: private_ip
  end

  config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'" # avoids 'stdin: is not a tty' error.
  config.ssh.forward_agent = true
  config.vm.provision 'shell', inline: $script
end
