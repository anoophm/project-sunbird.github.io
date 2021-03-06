---
type: landing
directory: developer-docs/installation/
title: Server Installation
page_title: Server Installationn
description: Setting up Sunbird on a server
allowSearch: true
---

## Overview

Sunbird software is containerized. It uses the Docker swarm orchestration engine to run the Sunbird docker images. The Docker swarm consists of the manager and agent nodes. The containers are run on the agent nodes and the manager manages the container lifecycle.

All the stateless services in Sunbird - Portal, LMS Backend, API Gateway and Proxies - are run as docker containers inside the swarm. All stateful services for the databases - Cassandra, PostgreSql and Elasticsearch, and the OAuth service on KeyCloak, are run on Virtual Machines (VMs) directly. The installation is automated using shell scripts and Ansible.

## Prerequisites

* Minimum 2 servers with 7 GB RAM, running Ubuntu server 16.04 LTS. You can scale the infrastructure by adding servers. Sunbird is designed to scale horizontally. The servers should connect to each other over TCP on the following [ports](http://sunbird-docs-qa.s3-website.ap-south-1.amazonaws.com/pr/326/developer-docs/installation/medium_scale_deploy/#mapping-ports). 

* Recommended that you have a domain name and a valid SSL certificate for the domain. If you do not have a domain name, you can configure Sunbird to be accessible over an IP address. If you have a domain name, and want to get an SSL certificate, use [Let's Encrypt](https://letsencrypt.org/) to generate a free certificate that is valid for 90 days.

* Sunbird requires Ekstep API keys to access the Ekstep content repository. Follow the steps [here](http://www.sunbird.org/developer-docs/telemetry/authtokengenerator_jslibrary/#how-to-generate-authorization-credentials) to get the keys. If you are creating a test environment, get the QA API keys.

* Create a common user (e.g. deployer) on all the servers. Configure this user to use [key based ssh](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server). Use an empty passphrase while generating the ssh key to avoid password prompts during installation. Since the installation script uses this key (user) to deploy Sunbird, this user must have sudo access on the servers.

* The following table lists the services that are set up and run as part of installation. The table also lists the optimal server count for a typical staging or production environment with thousands of users.

    |Server Name |Suggested Servers per environment|Basic Requirement| Max Servers |
    |:-----      |:--------|:--------------------------------|:---------  |
    |Docker swarm manager<sup>1</sup> | Staging - 1 <br> Prod - 3 | CPU: 1core & RAM: 2GB |Any  |
    |Docker swarm  agent nodes<sup>1</sup>   | Staging - 1 <br> Prod - 3 |CPU: 2core & RAM: 6GB| Any |
    |Elasticsearch<sup>2</sup>        |Staging - 1 <br> Prod- 3 |CPU: 1core & RAM: 3GB| Any |
    |Postgres master<sup>2</sup>      | Staging&Prod - 1 |CPU: 1core & RAM: 3GB|1 |
    |Postgres slave<sup>2</sup>       | Staging&Prod - 1 |CPU: 1core & RAM: 3GB|1 |
    |Cassandra<sup>2</sup>            |Staging&Prod - 1 |CPU: 1core & RAM: 3GB| 1 |
    |Keycloak<sup>1</sup> | Staging&Prod - 1|CPU: 1core & RAM: 4GB|Any |
    |log-es<sup>1</sup> |  Staging&Prod - 1|CPU: 1core & RAM: 3.5GB|1 |

* Run all services on the same server with the common superscript in the Server Name (e.g. servername<sup>2</sup>), when you install Sunbird on 2 servers. The App server runs services with superscript <sup>1</sup> and the DB server runs services with superscript <sup>2</sup>. 
 
**Note:** If you setup more than one swarm agent node, you need to configure a load balancer to spray the incoming requests to all the agent nodes. All agent nodes in a swarm route the request to the right service.

## Installation Procedure

**Note:** Choose one docker swarm manager VM as the installation server and execute the following steps from that server. If you are installing Sunbird on two servers, execute the steps from the app server. 

1.Install git using `apt-get update -y && apt-get install git -y `

2.Run `git clone https://github.com/project-sunbird/sunbird-devops.git`

3.`cd sunbird-devops/deploy`

4.Update the configuration files using `config`. The configuration parameters are explained in the following table: 

| variable | description   | mandatory|                                                                             
|:----------------------|:-----------------|:---------|
| `env`                |  Name of the environment you are deploying. Typically, it is one of development, test, staging, production, etc..                |yes|
| `implementation_name` | Name of your sunbird implementation.|yes|                |
| `ssh_ansible_user`  |SSH user with sudo access for accessing all servers.                  |yes|
| `ansible_private_key_path` | Path to the private SSH key file for the ssh_ansible_user. Ansible uses this file to SSH to the servers.        |yes|
| `ip_only`        |  True, if you do not want to use a domain name. Leave it blank or False if you are using a domain name.             |no|
| `cert_path`        | Path to .cert file (SSL certificate) for nginx. This variable need not be set if ip_only is True.              |no|
|`key_path`           | Path to .key file (SSL certificate) for nginx. This variable need not be set if ip_only is True           |no|
|`dns_name`           | Domain name for your installation. Use the IP address if ip_only is True              |yes|
|`ekstep_base_url`           | https://qa.ekstep.in/api & prod: https://api.ekstep.in        |yes|
|`sunbird_auth_token`           | JWT token generated by Ansible, you can get it from ~/jwt.txt.|yes|
|`ekstep_api_key`           |Jwt token generated by the key,secret produced from Ekstep |yes|
|`sso_username`           | Get the username from keycloak realm import doc eg.user-manager  |yes|
|`sso_password`           |  Password for keycloak sso_username            |yes|
|`keycloak_admin_password`           |Keycloak admin console password               |yes|
|`app_address_space`           | Application server address space (e.g. 10.3.0.0/24)   |yes|
|`ekstep_api_base_url`           | Ekstep api base url              |yes|
|`ekstep_proxy_base_url`           |  https://qa.ekstep.in  & prod: https://community.ekstep.in |yes|
|`sunbird_sso_publickey`           | SSO public key for sunbird realm               |yes|
|`trampoline_secret`           |Trampoline secret from the keycloak realm import doc.   |yes|
|`sunbird_env`           |  Ekstep environment to which we connect             |yes|
| `sudo_passwd`       |The sudo password, if the user has it else skip the parameter                 |no|
|`database_host`           | The private IP address of the DB server               |no|
|`database_password`           |  The common password for all the databases |no|
|`elasticsearch_host`           |A comma-separated (,) list of Elasticsearch database IP addresses. |no|
|`cassandra_host`           |The IP address of the Cassandra database.              |no|
|`postgres_master_host`           | The IP address of the Postgres master database             |no|
|`postgres_slave_host`           | The IP address of the Postgres slave database. If you do not need a slave node enter the same IP address as that of the master.            |no|
|`swarm_manager_host`           |A comma-separated (,) list of IP addresses of the swarm managers.                |no|
|`swarm_node_host`           | A comma-separated (,) list of swarm node IP addresses .             |no|
|`keycloak_host`           | A comma-separated (,) list of keycloak IP addresses.              |no|
|`log_es_host`           |  The IP address of the Logger Elasticsearch            |no|
| `proxy_prometheus` | By default this parameter is marked as 'false'. Modify it to 'true', if you want to use monitoring  |no|
|`sunbird_dataservice_url` |The API url of sunbird, for example; https://demo.opensunbird.o    |no|
|`sunbird_azure_storage_account`  | The Azure storage account for the badger service     |no|
|`sunbird_azure_storage_key`  | The Azure storage key for the badger service    |no|
|`sunbird_image_storage_url`| The Azure image url for the badger service |no|
|`sunbird_installation_email`| The Sunbird installation email ID |no|
|`sunbird_content_service_producer_id`| The Sunbird content service provider ID |no|
|`sunbird_telemetry_pdata_id`| The Sunbird telemetry pdata ID |no|


5.Run the script `./sunbird_install.sh`. This script sets up the infra setup from  stage 1 to stage 6 in a sequence shown in following table.

|stage no |stage name|Description| 
|:-----      |:-------|:--------|
|1 |config |Generates configuration file and hosts file |
|2|dbs|Installs all databases and creates schema  |
|3 |apis|Sets up API manager kong and Onboard API's and consumer's  |
|4|proxy|Deploys and configures Nginx|
|5|keycloak| Deploys and configures Keycloak |
|6|badger|Deploys the badger service|
|7|core|Deploys all core services|
|8|logger|Deploys the ELK stack and the logs can be viewed in Kibana|
|9|monitor|Monitors all the services, health checks, API's,system checks etc..|

6.The badger service is set up manually. To do so, follow the steps given [here](http://sunbird-docs-qa.s3-website.ap-south-1.amazonaws.com/pr/326/developer-docs/installation/medium_scale_deploy/#badger-setup).

**Note**: The badger service does not work without an Azure storage account name and key. 

7.Verify that all the mandatory variables, for example; sunbird_auth_token, ekstep_api_key, of the Sunbird core services are updated, and run the script `./sunbird_install.sh -s core` to deploying the core services.

**Note**: If you what to re-run any particular stage in the installation, just run `./sunbird_install.sh -s <stagename>`

To know more about the script `sunbird_install.sh`, click [here](http://sunbird-docs-qa.s3-website.ap-south-1.amazonaws.com/pr/326/developer-docs/installation/medium_scale_deploy/#sunbird-install-script).

8.Open https://[domain-name] and verify the installation. 
  
## Sunbird Install Script 

The Sunbird installation script `./sunbird_install.sh` is a wrapper shell script that invokes other scripts or Ansible playbooks. It fetches all the docker images from the Sunbird DockerHub repository. 

* `install-deps.sh` - Installs Ansible v2.4.1.0 on the installation server to provision and deploy Sunbird. This script also sets up the docker swarm.

* `generate_config.sh` - Creates a workspace for the installation and sets up necessary config files. 

* `generate_hosts.sh` - Creates a hosts file (Ansible file format) dynamically to run the Ansible scripts.   

* `install-dbs.sh` - Installs Cassandra, Elasticsearch and Postgres databases

* `init-dbs.sh` - Sets up the required schema for Cassandra, Elasticsearch and Postgres databases

* `deploy-apis.sh` - Deploys the api gateway (Kong) as a docker service using Ansible. 

* `onboard-apis.sh`  - Onboards the Sunbird APIs and consumers to the API gateway using Ansible. 

* `deploy-proxy.sh` - Deploys the proxy (Nginx) as a docker service.

* `provision-keycloak.sh` - Installs Keycloak.

* `deploy-keycloak-vm.sh` - Deploys the OAuth service (Keycloak) on the VM. The Keycloak service runs outside the swarm.

* `bootstrap-keycloak.sh` - Imports the auth realm and configures Keycloak.

* `deploy-badger.sh` - Deploys the badger service as docker service.

* `deploy-core.sh` - Deploys the core services player, content, actor and learner service as docker services. The content, actor and learner service together form the LMS backend. 

* `deploy-logger.sh` -  Deploy ELK stack for logs. 
**Note:** This step is optional.

* `deploy-monitor.sh` - Deploys the monitoring stack (Prometheus) to send mail alerts when the infrastructure or services are not stable. **Note:** This step is optional. 
 

## Mapping Ports 
The following is a list of ports that must be open:

|From server |To server|port| protocal|
|:-----      |:-------|:--------|:------|
|Administration server|All servers|22|TCP|
|ELB|0.0.0.0|80,433|TCP|
|swarm managers subnet|swarm nodes subnet|All|TCP & UDP|
|swarm nodes|Cassandra servers|9042|TCP|
|swarm nodes|Cassandra servers|9042|TCP|
|swarm nodes|Elasticsearch servers| 9200 |TCP|
|swarm nodes|Postgres servers| 5432|TCP|
|swarm nodes|Keycloak| 8080|TCP|

## Badger Setup

1.SSH to docker swarm manager

2.Run `docker service ps badger-service`

3.SSH to the swarm node where badger is running

4.Run `docker ps | grep badger`. Take the container ID and pass it to the container ID in the following step.

5.Run `docker exec -it <container_id> /bin/bash` . It will ssh to the badger container.

6.Move to the directory `cd code`

7.Run `./manage.py createsuperuser`. Provide a valid username, email ID and password.

8.Run the following curl command to get the sunbird badger authorization variable.
    
    `curl -X POST 'http://localhost:8004/api-auth/token' -d "username=<emailid>&password=<password>"`

9.Set the output of above command as the value for the `vault_sunbird_badger_authorization` in config file. 







