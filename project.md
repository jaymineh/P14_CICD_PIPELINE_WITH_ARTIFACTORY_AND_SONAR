# Continuous Integration With Jenkins, Ansible, Artifactory, SonarQube & PHP

**Step 0 - Prerequisites**
---

**Note that this is a continuation from the previous project. In that project, the root directory is called `ansible-config-mgt`. Since I started this one from scratch, the name of the root directory in this project is `ansiblecfg`. However, they are the same**


- Set up servers for Jenkins, SonarQube and a MySQL database.

*Use a RHEL instance for the database. Using ubuntu caused issues that will be documented below*

- Ensure ansible inventory folder is set up correctly. See example below:

```
├── ci
├── dev
├── pentest
├── pre-prod
├── prod
├── sit
└── uat
```

- Ensure the required hosts are placed in the right inventory file. In this case, I used `dev.yml` as my inventory host file.

**Step 1 - Configuring Ansible For Jenkins Development**
---

*To be able to run Ansible commands from the Jenkins UI (instead of sshing to the server and run it there --bad idea for real life scenarios), we would need to install the Ansible plugin for Jenkins so plays can be run directly from the Jenkins UI*

- Start up Jenkins and open Blue Ocean. From there, create a new pipeline job from the Blue Ocean UI.

*Ensure that the repository you choose has a `Jenkinsfile` present before you create the pipeline. Jenkins will automatically recommend creating a `Jenkinsfile` for you if it does not detect one in the repo*

![Pipeline Build From Blue Ocean]

- In the `ansiblecfg` root directory, create a folder called `deploy`. Go into the `deploy` folder and create a `Jenkinsfile` and switch to a new git branch called `feature/jenkinspipeline`

![Jenkinsfile Location]

![Feature Branch]

- Insert the code below into the `Jenkinsfile` that was just created. This is done to test if the pipeline is configured correctly and working properly. The script itself does nothing and is just an echo shell script.

```
('Build') {
      steps {
              script {
	             pipeline {
  agent any

  stages {
    stage   sh 'echo "Building Stage"'
        }
      }
    }

    stage('Test') {
      steps {
        script {
          sh 'echo "Testing Stage"'
        }
      }
    }

    stage('Package') {
      steps {
        script {
          sh 'echo "Testing Stage"'
        }
      }
    }

    stage('Deploy') {
      steps {
        script {
          sh 'echo "Testing Stage"'
        }
      }
    }

    stage('Clean up') {
      steps {
        script {
          sh 'echo "Testing Stage"'
        }
      }
    }
  }
}
```

*The above stages can be added step by step (i.e build stage, test stage, etc) to see howthings progress*

- After adding the above code to the Jenkinsfile, push to GitHub and go to the Jenkins UI to configure where the pipeline would look when executing scripts. This location should be where the Jekinsfile is located.

![Pipeline Configuration]

- After saving your settings, click on **Scan Repository Now** for Jenkins to scan the selected repository to get the latest changes pushed to GitHub and also the branches that were created. Once this is done, you can either let the build run automatically or go into Blue Ocean and run it manually. See result of successful run below:

![Test Build]

- Install the Ansible plugin from the Jenkins store. This is the module that would make Jenkins be able to run Ansible scripts.

- Clear the contents of the first Jenkinsfile (or delete it and create a new one), then paste the code below into the Jenkinsfile:

```
pipeline {
    agent any

    parameters {
      string(name: 'inventory', defaultValue: 'dev.yml',  description: 'This is the inventory file for the environment to deploy configuration')
    }

  stages {
    stage("Initial Clean Up") {
      steps {
        dir("${WORKSPACE}") {
           deleteDir()
        }
      }
    }

    stage('SCM Checkout') {
      steps {
        git branch: 'main', url: 'https://github.com/jaymineh/ansiblecfg.git'
      }
    }
    
    stage('Execute Playbook') {
      steps {
        withEnv(['ANSIBLE_CONFIG = ${WORKSPACE}/deploy/.ansible.cfg']) {
          ansiblePlaybook credentialsId: 'private-key', disableHostKeyChecking: true, installation: 'ansible2', inventory: 'inventory/${inventory}', playbook: 'playbooks/site.yml', tags: 'webservers'
        }
      }
    }
  }
}

```

*Notice that we included parameterization in the above code, which enables us to input the appropriate values for the inventory file we want to run the playbook against. See parameterization section below*

```
 parameters {
      string(name: 'inventory', defaultValue: 'dev.yml',  description: 'This is the inventory file for the environment to deploy configuration')
    }
```

![Parameterization on Jenkins UI]

*The `Pipeline Syntax` in Jenkins can be used to generate pipeline script for a Jenkinsfile. IT is very good for learning how to properly structure a Jenkinsfile*

- Before running the build job, go to the deploy folder on the local machine and create the `ansible.cfg` file. This is te environment variable where some configs and paths are declared within. It should be placed in the `deploy` folder alongside the `Jenkinsfile`. Paste the code below into the `Jenkinsfile`:

```
[defaults]
roles_path=/opt/ansible/ansiblecfg/roles
timeout = 160
callback_whitelist = profile_tasks
log_path=~/ansible.log
host_key_checking = False
gathering = smart
ansible_python_interpreter=/usr/bin/python3
allow_world_readable_tmpfiles=true


[ssh_connection]
ssh_args = -o ControlMaster=auto -o ControlPersist=30m -o ControlPath=/tmp/ansible-ssh-%h-%p-%r -o ServerAliveInterval=60 -o ServerAliveCountMax=60 -o ForwardAgent=yes
```

- Run the Jenkins job. Confirm that ansible runs against all hosts.
![Ansible Run]

*I was having this error `fatal: [10.0.12.51]: FAILED! => {"ansible_facts": {}, "changed": false, "failed_modules": {"ansible.legacy.setup": {"failed": true, "module_stderr": "Shared connection to 10.0.12.51 closed.\r\n", "module_stdout": "\r\n/bin/sh: 1: /usr/bin/python: not found\r\n", "msg": "The module failed to execute correctly, you probably need to set the interpreter.\nSee stdout/stderr for the exact error", "rc": 127}}, "msg": "The following modules failed to execute: ansible.legacy.setup\n"}` on the database server saying that python couldn't be found. Despite trying reinstalling the exact python version needed and declaring the path, the issue persisted. However the issue went away when I created a new database based on RHEL as this one with the error was based on Ubuntu*

***Things To Note***
---

- You may sometimes notice that Jenkins does not download the latest code from GitHub (which may have a fix), and would continue runnig with errors. You can only detect this by checking the error log to know why it happened. The way to remediate this is by either running a cleanup step to clear the Jenkins workspace or logging into the Jenkins server and manually deleting the workspace folder in `var/lib/jenkins` and pull from GitHub into the Jenkins local server. That way, the workspace has the latest code to work with.

- Jenkins may fail when a different branch is indicated in the `Jenkinsfile`, compared to the one the branch is being run with.

**Step 2 - Setting Up The Artifactory Server**
---

*The tooling website has already been deployed through Ansible. Here, another PHP application will be introduced to the list of spftwares being managed in the infrastructure. This is also an ideal application to show an end-to-end CI/CD pipeline as this has unit tests*

*The goal here is to deploy the applications from Artifactory rather than Git. This can be done using an Artifactory server in the cloud or one you set up locally. I used the cloud variant in this example*

- Go to `https://artifactory.jfrog.io` and create an account. Next, create a new `local repository` where artifacts will be uploaded to from the Jenkins server.

![Artifactory Setup 1]

![Artifactory Setup 2]

*The above images are examples and not the exact names of the repo I created*

**Step 3 - Integrating Artifactory Repo With Jenkins**
---

- Install the `Plot` plugin on Jenkins to display test reports, code coverage and artifactory plugins.

- Configure Artifactory in Jenkins for syncronization.

![Artifactory Config]

![Artifactory Config 2]

- Fork this repository `https://github.com/darey-devops/php-todo.git`.

- Created a new database and user

```
Create database homestead;
CREATE USER 'vergil'@'<Jenkins-ip-address>' IDENTIFIED BY 'spardason';
GRANT ALL PRIVILEGES ON * . * TO 'vergil'@'<Jenkins-ip-address>';
```

*Ensure to configure `bind_address` in the `my.cnf` file to `0.0.0.0 and port 3306 is open in the security group*

![Bind Address]

- Install PHP on the Jenkins server. *Ran into an issue where the default PHP version I had installed (v8) was too high. HAd to install v7.4 specifically. Got info from `https://www.tecmint.com/install-different-php-versions-in-ubuntu/`*

- Install PHP composer globally on Jenkins server.

```
php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
php -r "if (hash_file('sha384', 'composer-setup.php') === '55ce33d7678c5a611085589f1f3ddf8b3c52d662cd01d4ba75c0ee0459970c2200a51f492d557530c71c15d8dba01eae') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
php composer-setup.php
php -r "unlink('composer-setup.php');"
sudo mv composer.phar /usr/local/bin/composer
```

- Update the `.env.sample` file with database server information.

![Database Information]

- Create a new Jenkinsfile for the previously forked repo on the main branch. The repo is now known as `todo`. Insert the code below into the Jenkinsfile.

```
pipeline {
    agent any

 stages {

    stage("Initial cleanup") {
      steps {
        dir("${WORKSPACE}") {
            deleteDir()
        }
      }
    }

    stage('Checkout SCM') {
      steps {
            git branch: 'main', url: 'https://github.com/jaymineh/todo.git'
      }
    }

    stage('Prepare Dependencies') {
      steps {
             sh 'mv .env.sample .env'
             sh 'composer install'
             sh 'php artisan migrate'
             sh 'php artisan db:seed'
             sh 'php artisan key:generate'
      }
    }
  }
}
```

- Create a new pipeline job from Jenkins (Blue Ocean). Job name is `todophp`. Run the pipeline after.

![Todophp Success]

- After the above is completed, confirm by logging  into the database server and running the `show databases;` command to see the available databases. Select the `homestead` database by using use homestead;` and use `show tables` to display the inserted tables in the database.

*I ran into an error saying `composer not found` even when I installed composer on the server. I was able to resolve this by installing composer globally as my guess is that Jenkins was not able to call the composer tool properly*

![Composer Not Found]

*I ran into another error which was permissions based. Jenkins was not being allowed to insert into the database. I tried changing permissions and even tried logging into the database with the same credential but I still faced the error. I realized I was pointing to the old Ubuntu database. This issue went away when I pointed to the new RHEL database. It looks like there are a lot of issues when using MYSQL on Ubuntu based machines.

![Access Denied]

**Step 4 - Structuring The Jenkinsfile**
---
