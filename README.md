# DevSecOps
DevSecOps pipeline setup using GitLab Security Products

![Pipeline](assets/pipeline.png)

## Setup details:
### [DevOps lab][salecha] by Rohit Salecha
Modified to use 4 machines
- Docker Registry
- Jenkins Server
- Staging Server
- Production Server

Follow [this][post] post to setup the lab

### GitLab Security Tools
- [Dependency Scanning][depscan]
- [SAST][sast]
- [DAST][dast]
- [Container Scanning][container]

### ArcherySec as Vulnerability Management Tool

You could visit http://archerysec.devops/ in a browser, and you'll see the ArcherySec login page. Log in with the default credentials:

- Username: admin
- Password: devops@123A

#### ArcherySec Dashboard 

![!img](assets/archerysec-project.png)

![img](assets/archerysec-dashboard-1.png)

![img](assets/archerysec-dashboard-2.png)

![img](assets/archerysec-dashboard-scanners.png)

### Docker based Web Application
https://github.com/npatta01/web-deep-learning-classifier

[salecha]: <https://github.com/salecharohit/devops>
[depscan]: <https://docs.gitlab.com/ee/user/application_security/dependency_scanning/index.html>
[sast]: <https://docs.gitlab.com/ee/user/application_security/sast/index.html>
[dast]: <https://docs.gitlab.com/ee/user/application_security/dast/>
[container]: <https://docs.gitlab.com/ee/user/application_security/container_scanning/>
[post]: <https://www.rohitsalecha.com/project/practical_devops/>

#### Add archerysec credential on Jenkins
http://jenkins.devops:8080/credentials/store/system/domain/_/credential/archerysec/
![!img](assets/jenkins-archerysec.png)