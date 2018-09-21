Installation & Deployment
=========================

[Ansible](https://docs.ansible.com/ansible/) is use to automate installation of server and deployment.

You can test this configuration with a free server (only for 2 hours) using [dply.co](https://dply.co/b/ROSJbb7w). **Dply is close :'( Need an alternative**

# 1. Setup

Copy example files : `cp vars.example.yml vars.yml` and `cp hosts.example.ini hosts.ini`.

On `hosts.ini` file put server's address with the new user's name and check information (like the domain) in `vars.yml`.
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
ansible-playbook -i hosts.ini install.yml --vault-password-file .vault_pass
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
ansible-playbook -i hosts.ini deploy.yml --vault-password-file .vault_pass
```

# 4. Rollback

To run a rollback :
```bash
ansible-playbook -i hosts.ini rollback.yml --vault-password-file .vault_pass
```

# 5. CI/CD

You can use this role to deploy using a CI/CD service.
An exemple with GitLab :

```yml
.before_deploy:
  variables:
    GIT_SUBMODULE_STRATEGY: recursive
  before_script:
  - mkdir -p ~/.ssh
  - chmod 700 ~/.ssh
  - echo $SSH_PRIVATE_KEY | base64 -d > ~/.ssh/id_rsa
  - chmod 600 ~/.ssh/id_rsa
  - ssh-keyscan $CI_ENVIRONMENT_URL >> ~/.ssh/known_hosts
  - chmod 644 ~/.ssh/known_hosts
  - echo $VARS_YML_CONTENT | base64 -d > $CI_PROJECT_DIR/ansible/vars.yml
  - echo $HOSTS_INI_CONTENT | base64 -d > $CI_PROJECT_DIR/ansible/hosts.ini
  - echo $VAULT_PASS_CONTENT | base64 -d > $CI_PROJECT_DIR/ansible/.vault_pass
  - ansible-galaxy install ansistrano.deploy ansistrano.rollback
  - cd $CI_PROJECT_DIR
  - git config --global user.email "$GITLAB_USER_EMAIL"
  - git config --global user.name "$GITLAB_USER_NAME"
  - git checkout $CI_COMMIT_REF_NAME
  - git remote add dist ssh://eatfit@${CI_ENVIRONMENT_URL}:/home/eatfit/${CI_ENVIRONMENT_URL}.git
  - git push dist $CI_COMMIT_REF_NAME
  script:
  - cd $CI_PROJECT_DIR/ansible
  - ansible-playbook -i hosts.ini deploy.yml --vault-password-file .vault_pass

prod_deployment:
  stage: deploy
  only:
  - master
  image: oprax/ansible:latest
  environment:
    name: production
    url: www.example.org
  extends: .before_deploy

dev_deployment:
  stage: deploy
  only:
  - develop
  image: oprax/ansible:latest
  environment:
    name: staging
    url: staging.example.org
  extends: .before_deploy
```

For this exemple I consider that you use this repository as a submodule of your main website repository, submodule path is `ansible/`.

All private file are base64 encoded to avoid problem with newline for example.
To encode a file in a shell : 

```shell
cat file | base64 | tr -d "\n" > file.b64
```

First, create a SSH key without password only for the website. Put the private key (b64 encoded) in the secret variable named `SSH_PRIVATE_KEY`.

Do the same for `VARS_YML_CONTENT` (`vars.yml`), `HOSTS_INI_CONTENT` (`hosts.ini`) and `VAULT_PASS_CONTENT` (`.vault_pass`)

**Don't forget to replace environment URL !**
