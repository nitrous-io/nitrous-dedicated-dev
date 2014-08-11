## Nitrous.IO Dedicated Dev VM

This installs all the repos needed for Nitrous.IO Dedicated (along with
all of its dependencies).

## Setup

1. Setup Vagrant 1.6.x
2. Make Sure SSH Agent is running `eval 'ssh-agent -s'`
3. Make sure your SSH keys are added to SSH agent via the command `ssh-add` (This is required since we use SSH agent forwarding so that we use the host's SSH keys to authenticate with Github.
3. Run `vagrant up`
  3a. Since you can't install Nitro-CLI yet we will use it within the vagrant box with `vagrant ssh`
  3b. Add the following line to your .bashrc  `export PATH=$PATH:/home/vagrant/workspace/nitro/.vendor/bin`
  3c. `source .bashrc`
4. Enjoy Nitro.
