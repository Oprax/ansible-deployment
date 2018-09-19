Installation & Deployment
=========================

[Ansible](https://docs.ansible.com/ansible/) is use to automate installation of server and deployment.

You can test this configuration with a free server (only for 2 hours) using [dply.co](https://dply.co/b/ROSJbb7w). **Dply is close :'( Need an alternative**

# 1. Setup

Copy example files : `cp vars.yml.example vars.yml` and `cp hosts.example hosts`.

On `hosts` file put server's address with the new user's name and check information (like the domain) in `vars.yml`.
You can generate SSH key and use `ssh_private_key_file` variable to define path of the private key.
Password is encrypted using `ansible-vault`.

Put a password on a file `.vault_pass`, then you can use
```bash
ansible-vault encrypt_string --vault-id .vault_pass
```
to encrypt a string (like a password).
The current mysql password is `root` with `root` as vault password, **this is
not secure for production !**

**`.vault_pass` must be secret and not share in git repository (or other cloud) !!!!**

# 2. Installation

If setup is done, you just need to run 
```bash
ansible-playbook -i hosts install.yml --vault-password-file .vault_pass
```

When installation is done you can connect to the server with SSH key.

### Manually step

 - You need to put password for each user manually.
 - In `/etc/ssh/sshd_config` change SSH port
 - Update UFW rule `ufw delete SSH_RULE_NUMBER` (to get `SSH_RULE_NUMBER` use `ufw status numbered`) and `ufw allow xxxx/tcp` with `xxxx` port number 
 - In `/etc/ssh/sshd_config` disallow root login with `PermitRootLogin no`
 - In `/etc/ssh/sshd_config` put disallow password connection with 
```
Match User user1
    PasswordAuthentication no
Match User user2
    PasswordAuthentication no
Match all
```
And reload SSH server : `systemctl reload ssh`
# 3. Deploy


Deployment use [Ansistrano](https://github.com/ansistrano/deploy) playbook. Install ansistrano with `ansible-galaxy` :
```bash
ansible-galaxy install ansistrano.deploy ansistrano.rollback
```
You can also check `.env` information in `templates/env.j2`.

Ansistrano use git, so before push master on the server :
`git remote add dist ssh://user@domain.com:/home/user/example.org.git`

To deploy, run :
```bash
ansible-playbook -i hosts deploy.yml --vault-password-file .vault_pass
```

# 4. Rollback

To run a rollback :
```bash
ansible-playbook -i hosts rollback.yml --vault-password-file .vault_pass
```
