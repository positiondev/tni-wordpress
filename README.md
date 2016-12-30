# How to use

## Unlocking the "vault"

Get the vault password from somebody.

Create a `.vault_pass` file in the `trellis` directory and put only the Vault password in it (it's not YAML, the only thing in the file should be the vault password.

## Development

Create `thenewinquiry.com` directory.

Inside directory, `git clone git@github.com:positiondev/tni-wordpress.git`

Run `vagrant up`.

## Deployment

Run `ansible-playbook server.yml -e env=staging`.

Run `./bin/deploy.sh staging thenewinquiry.com`.

I’m probably forgetting stuff...

# How I did this

This is not a how-to, just notes. You shouldn't have to do any of this again.

## EC2 and CloudFlare

  - Made EC2 instance
    - Ubuntu 16.04
    - Micro/free tier
    - Security groups:
    - Use launch wizard SSH security group
    - Also add security group for incoming HTTP and HTTPS
    - Create new keys
  - Create a CNAME on CloudFlare.
  - Add new sub-domain and keys to .ssh/config:
```
Host new-staging.positiondevapp.com
  ForwardAgent yes
  IdentityFile ~/.ssh/aws-new.pem
```
  - Import SSH key password into Keychain by running `ssh-add -K`.

## Cloning/Forking Roots

 - Create local site directory
 - Inside directory, git clone your fork of Trellis
 - Inside directory, git clone (not fork?) Bedrock && rm .git
 - cd site and git init, create Github repo for the Bedrock repo

## Development
 - Change group_vars/development/wordpress_sites.yml to correct site name, dev host names
 - Change group_vars/development/vault.yml to correct site name
Run vagrant up

## Staging
 - Change `group_vars/all/users.yml`
   - Key lookup: `"{{ lookup('file', '~/.ssh/aws-new.pub') }}"`
   - Add admin_user: ubuntu
 - Change `group_vars/staging/wordpress_sites.yml`
   - Change the site name
   - Enter location of Bedrock repo
   - Remove `repo_subtree_path `
 - Change `group_vars/staging/vault.yml`
   - Generate passwords and salts
 - Change `hosts/staging`
   - Add actual hostnames to [web] and [staging]
 - Change roles/remote-user/tasks/main.yml
   - We removed the check for whether Ansible was able to connect as root (since we know it can’t)
 - Change `server.yml` to `apt-get update` before installing python.
 - Run ansible-playbook server.yml -e env=staging
 - Run ./bin/deploy.sh staging thenewsite.com

## Before committing, secure the passwords

 - Create .vault_pass and put a strong password in it
 - Copy .vault_pass to meta
 - Add vault_password_file = .vault_pass to ansible.cfg
 - Use ansible-vault to encrypt the vault files

## What's left

 - How to share SSH keys
 - SSL (Trellis has built-in let’s encrypt, so should be straightforward)
 - Git is a little messed up
 - Documentation says to shallow clone
   - Can’t push from a shallow clone without unshallow-ing first (adding all the old commits)
   - Suggests to me that it’s not intended to be used as a fork by most users

## SSH keys idea

 - Collect every contributor’s public keys in .ssh
 - Add those keys to the list for {{ admin_user }} in users.yml
 - Only the first person to set up the server needs the original AWS key
 - Other contributors will refer to their own private key in .ssh/config instead of the original AWS key
 - Drawback: everybody needs everyone else’s public keys? I think? (this is not so bad imho)

## Git proposal

 - thenewsite.com is the main git repo
 - Trellis and Bedrock are git clone depth=1ed as described in the official documentation and the git history deleted
 - We only track our own git history
 - Only one repo per project
 - We can always fork later and add changes we want to make if we do find things to contribute back, and we can always update later manually
