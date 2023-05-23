# Continuous Integration With Jenkins, Ansible, Artifactory, SonarQube And PHP
---

In this project, I implemented a CI/CD pipeline for a PHP based application. The overall CI/CD process looks like the architecture diagram below.

![Architecture](architecture.png)

The entire concept of this project is the deployment of an application from GitHub to Jenkins to run a multi-branch pipeline job. This is done to achieve continuous integration of codes from different developers. After this, the artifacts from the build job are packaged and pushed to a SonarQube server for testing before it is deployed to Artifactory, from which an Ansible job is triggered to deploy the application to the production environment.

Kindly check `PROJECT.md` for more information.
