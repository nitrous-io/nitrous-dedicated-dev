# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = '2'

$script = <<SCRIPT
export VAGRANT_HOME=/home/vagrant
export GOPATH=$VAGRANT_HOME/go
export GOROOT=/usr/local/opt/go
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
export WORKSPACE=$VAGRANT_HOME/workspace

########################################
####### INSTALL APT DEPS ###############
########################################

echo "Installing apt dependencies..."

apt-get install -yy --no-install-recommends lxc build-essential curl git-core mercurial postgresql-9.3 postgresql-contrib-9.3 libpq-dev

########################################
####### CONFIGURE PG ###################
########################################

echo "Configuring postgres..."

sudo su postgres -c 'psql -c "CREATE ROLE vagrant NOSUPERUSER CREATEDB NOCREATEROLE INHERIT LOGIN;"'

########################################
####### INSTALL GO #####################
########################################

echo "Installing go..."

mkdir -p $GOROOT
curl -s https://storage.googleapis.com/golang/go1.3.linux-amd64.tar.gz | tar -C /usr/local/opt -xz
echo "export GOPATH=$GOPATH" >> /etc/profile
echo "export GOROOT=$GOROOT" >> /etc/profile
echo "export PATH=$GOPATH/bin:$GOROOT/bin:\\$PATH" >> /etc/profile

########################################
####### INSTALL GO DEV TOOLS  ##########
########################################

echo "Installing dev tools..."

go get github.com/nitrous-io/goop
go get code.google.com/p/go.tools/cmd/goimports

########################################
####### SETUP DOCKER ###################
########################################

# Check that HTTPS transport is available to APT
if [ ! -e /usr/lib/apt/methods/https ]; then
  apt-get update
  apt-get install -y apt-transport-https
fi
# Add the repository to your APT sources
echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list
# Then import the repository key
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
# Install docker
apt-get update

apt-get install -yy lxc-docker-1.1.1


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

#####################################
####### INSTALL RUBY ################
#####################################

echo "Installing ruby..."

echo "export PATH=\\$HOME/.rbenv/shims:\\$PATH" >> /etc/profile
sudo apt-get install -yy --no-install-recommends rbenv
sudo -u vagrant bash <<EOF
  export HOME=$VAGRANT_HOME
  git clone https://github.com/sstephenson/ruby-build.git $VAGRANT_HOME/.rbenv/plugins/ruby-build
  rbenv install 2.1.2 && rbenv global 2.1.2
  gem install bundler && rbenv rehash
EOF

#####################################
####### INSTALL NODE ################
#####################################

echo "Installing node..."

sudo apt-get install -yy --no-install-recommends nodejs

###########################################
####### INSTALL COMPONENTS ################
###########################################

echo "Installing components..."

git config --global url.ssh://git@github.com/.insteadOf https://github.com/
ssh -T git@github.com -o StrictHostKeyChecking=no && echo 'Success'

install_component() {
  component=$1
  if [ ! -d $component ]; then
    (cd $GOPATH/src/github.com/nitrous-io && git clone git@github.com:nitrous-io/$component.git)
    ln -fs $GOPATH/src/github.com/nitrous-io/$component $WORKSPACE/$component
    (cd $WORKSPACE/$component && goop install)
  fi
}

mkdir -p $GOPATH/src/github.com/nitrous-io
mkdir -p $WORKSPACE

install_component nitrous
install_component freighter
install_component hostbuilder
install_component nitro

sudo chown -R vagrant:vagrant $GOPATH
sudo chown -R vagrant:vagrant $WORKSPACE

########################################
####### COMPONENT SETUP ################
########################################

sudo -u vagrant bash <<EOF

  cd $WORKSPACE/hostbuilder
  cp sample_config.json config.json

  cd $WORKSPACE/nitrous
  cp .env-example .env
  make createdb

EOF


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

  config.vm.synced_folder "workspace/", "/home/vagrant/go/src/github.com/nitrous-io", create:true
end
