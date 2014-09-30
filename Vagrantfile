# -*- mode: ruby -*-
# vi: set ft=ruby :

VAGRANTFILE_API_VERSION = '2'

$script = <<SCRIPT

export VAGRANT_HOME=/home/vagrant
export GOPATH=$VAGRANT_HOME/go
export GOROOT=/usr/local/opt/go
export PATH=$GOPATH/bin:$GOROOT/bin:$PATH
export WORKSPACE=$VAGRANT_HOME/workspace

## BASIC APT DEPENDENCIES ####################################################

echo "Installing basic dependencies..."

apt-get update
apt-get install -yy --only-upgrade libcgmanager0
apt-get install -yy --no-install-recommends lxc build-essential curl git-core mercurial

## CONFIGURE POSTGRESQL ######################################################

echo "Installing postgres..."

apt-get install -yy postgresql-9.3 postgresql-contrib-9.3 libpq-dev
sudo su postgres -c 'psql -c "CREATE ROLE vagrant NOSUPERUSER CREATEDB NOCREATEROLE INHERIT LOGIN;"'

## CONFIGURE MERCURIAL #######################################################

echo "Configuring mercurial..."

cat > $HOME/.hgrc <<EOF
[trusted]
users = vagrant
groups = vagrant
EOF

## INSTALL GO ################################################################

echo "Installing go..."

mkdir -p $GOROOT
curl -s https://storage.googleapis.com/golang/go1.3.linux-amd64.tar.gz | tar -C /usr/local/opt -xz
echo "export GOPATH=$GOPATH" >> /etc/profile
echo "export GOROOT=$GOROOT" >> /etc/profile
echo "export PATH=$GOPATH/bin:$GOROOT/bin:\\$PATH" >> /etc/profile

## INSTALL TOOLS #############################################################

echo "Installing tools..."

go get github.com/nitrous-io/goop
go get code.google.com/p/go.tools/cmd/goimports

## INSTALL DOCKER ############################################################

echo "Installing docker..."

apt-get install -y apt-transport-https
echo deb https://get.docker.io/ubuntu docker main > /etc/apt/sources.list.d/docker.list
apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 36A1D7869245C8950F966E92D8576A8BA88D21E9
apt-get update
apt-get install -yy lxc-docker-1.2.0

cat > /etc/default/docker <<EOF
  DOCKER_OPTS="-e lxc"
EOF

service docker restart

adduser vagrant docker

docker pull nitrousio/ubuntu-dind >/dev/null 2>&1 &
sudo chown -R vagrant:vagrant /var/lib/docker

## INSTALL RUBY ##############################################################

echo "Installing ruby..."

echo "export PATH=.bundle/binstubs:\\$HOME/.rbenv/shims:\\$PATH" >> /etc/profile
sudo apt-get install -yy --no-install-recommends rbenv
mkdir -p $VAGRANT_HOME/.rbenv/versions
curl -s https://nitrous-ruby.s3.amazonaws.com/vagrant/ruby-2.1.2.tgz | tar -C $VAGRANT_HOME/.rbenv/versions -xz
chown -R vagrant:vagrant $VAGRANT_HOME/.rbenv

sudo -u vagrant bash <<EOF
  export HOME=$VAGRANT_HOME
  mkdir $HOME/.bundle
  echo "BUNDLE_PATH: .bundle" > $HOME/.bundle/config
  echo "BUNDLE_BIN: .bundle/binstubs" >> $HOME/.bundle/config
  rbenv global 2.1.2
  rbenv exec gem install bundler && rbenv rehash
EOF

## INSTALL NODE ##############################################################

echo "Installing node..."

sudo apt-get install -yy --no-install-recommends nodejs

## INSTALL COMPONENTS ########################################################

echo "Installing components..."

git config --global url.ssh://git@github.com/.insteadOf https://github.com/
ssh -T git@github.com -o StrictHostKeyChecking=no

install_component() {
  component=$1

  (cd $GOPATH/src/github.com/nitrous-io && rm -rf $component && git clone git@github.com:nitrous-io/$component.git)
  ln -fs $GOPATH/src/github.com/nitrous-io/$component $WORKSPACE/$component
  cd $WORKSPACE/$component && goop install && goop exec make build

  cat >/etc/init/$component.conf <<EOF

    start on vagrant-mounted
    stop on runlevel [06]

    setuid vagrant
    setgid vagrant

    env GOPATH=$GOPATH
    env GOROOT=$GOROOT
    env PATH=.bundle/binstubs:/home/vagrant/.rbenv/shims:$PATH

    exec /bin/bash -c "cd /home/vagrant/workspace/$component && make run"

    respawn

EOF
}

mkdir -p $GOPATH/src/github.com/nitrous-io
mkdir -p $WORKSPACE

install_component nitrous
install_component freighter
install_component hostbuilder
install_component nitro

sed -i 's/make run/make run-dev/' /etc/init/nitrous.conf

sudo chown -R vagrant:vagrant $GOPATH
sudo chown -R vagrant:vagrant $WORKSPACE

## COMPONENT SETUP ###########################################################

sudo -u vagrant bash <<EOF

  export GOPATH=$GOPATH
  export GOROOT=$GOROOT
  export PATH=$PATH

  cd $WORKSPACE/hostbuilder
  cp sample_config.json config.json

  cd $WORKSPACE/nitrous
  cp .env-example .env
  goop exec make createdb seed

EOF

## CONFIGURE NGINX ###########################################################

apt-get install -yy nginx

cat > /etc/nginx/nginx.conf <<EOF
user www-data;
worker_processes 6;
pid /run/nginx.pid;

events {
  worker_connections 768;
  # multi_accept on;
}

http {
  include /etc/nginx/mime.types;
  default_type application/octet-stream;

  access_log /var/log/nginx/access.log;
  error_log /var/log/nginx/error.log;

  sendfile on;

  map \\$http_upgrade \\$connection_upgrade {
    default upgrade;
    ''      close;
  }

  server {
    listen 8080 ssl;

    server_name pro.n2odev.com;
    ssl_certificate /home/vagrant/workspace/nitrous/dev/n2odev.crt;
    ssl_certificate_key /home/vagrant/workspace/nitrous/dev/n2odev.key;

    keepalive_timeout 70;

    location / {
      proxy_pass http://127.0.0.1:3000;
    }

    location /api/v0/ {
      proxy_pass http://127.0.0.1:3333;
      proxy_http_version 1.1;
      proxy_set_header Upgrade \\$http_upgrade;
      proxy_set_header Connection \\$connection_upgrade;
    }
  }
}
EOF

## LAUNCH SERVICES ###########################################################

service hostbuilder start
service nitrous start
service nginx restart

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

  config.vm.network "forwarded_port", guest: 8080,  host: 8080

  (49000..49999).each do |port|
    config.vm.network "forwarded_port", guest: port, host: port
  end

  config.ssh.shell = "bash -c 'BASH_ENV=/etc/profile exec bash'" # avoids 'stdin: is not a tty' error.
  config.ssh.forward_agent = true
  config.vm.provision 'shell', inline: $script

  config.vm.synced_folder "workspace/", "/home/vagrant/go/src/github.com/nitrous-io", create:true
end
