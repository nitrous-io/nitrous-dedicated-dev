## Nitrous.IO Dedicated Dev VM

This installs all the repos needed for Nitrous.IO Dedicated (along with
all of its dependencies).

## Setup

1. Setup Vagrant 1.6.x
2. Make sure your SSH keys are added to SSH agent via the command `ssh-add` (This is required since we use SSH agent forwarding so that we use the host's SSH keys to authenticate with Github.
3. Run `vagrant up`
