## Nitrous.IO Dedicated Dev VM

This installs all the repos needed for Nitrous.IO Dedicated (along with
all of its dependencies).

## Setup

1. Setup Vagrant 1.6.x
2. Make Sure SSH Agent is running `eval 'ssh-agent -s'`
3. Make sure your SSH keys are added to SSH agent via the command `ssh-add`
   (This is required since we use SSH agent forwarding so that we use the host's
  SSH keys to authenticate with Github.
4. Run `vagrant up`
  - Since you can't install Nitro-CLI yet we will use it within the vagrant box with `vagrant ssh`
  - Add the following line to your .bashrc  `export PATH=$PATH:/home/vagrant/workspace/nitro/.vendor/bin`
  - `source .bashrc`
5. Add the `nitrous.git/dev/n2odev.crt` and `nitrous.git/dev/n2odevpro-ca.crt`
   certs to your keychain (OS X) or the equivalent for your OS.
6. Make `pro.n2odev.com` (or `*.n2o.dev.com`) resolve to the IP of this VM.
7. nitrousd will run at [https://pro.n2odev.com:8080/].
