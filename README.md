# How to use

## Setup

Create `thenewinquiry.com` directory

`mkdir thenewinquiry.com`

Inside newly-created `thenewinquiry.com` directory: `git clone git@github.com:positiondev/tni-wordpress.git`

or `mkdir thenewinquiry.com && cd thenewinquiry.com && git clone git@github.com:positiondev/tni-wordpress.git`

Get the vault password from somebody.

Create a `.vault_pass` file in the `trellis` directory and put only the Vault password in it (it's not YAML formatted, the only thing in the file should be the vault password).

Make sure somebody has added your public key to the server.

Install Ansible Configuration for deployment

Inside the `trellis` directory: `ansible-galaxy install -r requirements.yml`

## Development

`site/config/environments/development.php` is gitignored so your can develop this site using VV instead of Trellis if needed.

Add to `site/config/environments/development.php`:

```
<?php
/** Development */
define('SAVEQUERIES', true);
define('WP_DEBUG', true);
define('SCRIPT_DEBUG', true);
```

ref: https://codex.wordpress.org/WP_DEBUG

`cd trellis` and run `vagrant up`.

The `site` directory contains the actual WordPress files. Change `composer.json` to add plugins, themes, etc. Run `composer update` and commit `composer.lock` before attempting to deploy. You don't need to commit anything in the `themes` directory.

Don't manually edit anything on the server. When things are changed manually, then the updates through Ansible don't work anymore. Make server changes only through Trellis/Ansible. This ensures reproducibility of builds, correct permissions, etc.

## Deployment

Run `ansible-playbook server.yml -e env=staging` (only if you make changes to the server in `trellis`).

Run `./bin/deploy.sh staging thenewinquiry.com` to deploy changes in `site`.

If you run into permissions problems with cloning the git repo, try the following:

- run `ssh-agent`: eval `ssh-agent -s`
- add your key to the agent: `ssh-add ~/.ssh/id_rsa`
- check that your key was added: `ssh-add -L`

# How I did this

This is not a how-to, just notes. Nobody should have to do this stuff for TNI again.

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

I originally tried to do this in a way that would keep the trellis and bedrock in separate repos, but this does *not* seem to be the way that Roots is intended to be used. Instead, we're going to keep them in the same repo. If we want to update trellis or bedrock later, only the files mentioned in the "Staging" and "Development" sections have changed.

 - Create local site directory
 - Inside directory, `git clone --depth=1 git@github.com:positiondev/position-trellis.git && rm -rf trellis/.git`
 - Inside directory, `git clone --depth=1 git@github.com:roots/bedrock.git site && rm -rf site/.git`
 - `git init`, create Github repo for the tni-wordpress repo

## Development

 - Change `group_vars/development/wordpress_sites.yml` to correct site name, dev host names
 - Change `group_vars/development/vault.yml` to correct site name
 - Run `vagrant up`

## Staging
 - Change `group_vars/all/users.yml`
   - Key lookup: `"{{ lookup('file', '~/.ssh/aws-new.pub') }}"`
   - Add `admin_user: ubuntu`
 - Change `group_vars/staging/wordpress_sites.yml`
   - Change the site name
   - Enter location of Bedrock repo
   - Remove `repo_subtree_path `
 - Change `group_vars/staging/vault.yml`
   - Generate passwords and salts
 - Change `hosts/staging`
   - Add actual hostnames to `[web]` and `[staging]`
 - Change `roles/remote-user/tasks/main.yml`
   - We removed the check for whether Ansible was able to connect as root (since we know it can’t)
 - Change `server.yml` to `apt-get update` before installing python.
 - Run `ansible-playbook server.yml -e env=staging`
 - Run `./bin/deploy.sh staging thenewinquiry.com`

## Before committing, secure the passwords

 - Create `.vault_pass` and put a strong password in it
 - Copy `.vault_pass` to meta
 - Add `vault_password_file = .vault_pass` to `ansible.cfg`
 - Use `ansible-vault` to encrypt the vault files

# What's left

 - How to share SSH keys
 - SSL (Trellis has built-in let’s encrypt, so should be straightforward)

## SSH keys idea

 - Collect every contributor’s public keys in `.ssh`
 - Add those keys to the list for `{{ admin_user }}` in `users.yml`
 - Only the first person to set up the server needs the original AWS key
 - Other contributors will refer to their own private key in `.ssh/config` instead of the original AWS key
 - Drawback: everybody needs everyone else’s public keys? I think? (this is not so bad imho)
