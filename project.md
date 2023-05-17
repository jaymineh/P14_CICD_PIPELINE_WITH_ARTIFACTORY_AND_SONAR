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
