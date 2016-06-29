## Cloudify Kubernetes Plugin - MediaWiki Example

This readme will provide a step-by-step guide on how to deploy a MediaWiki and a MySQL database on AWS using Cloudify and kubernetes. The example will run the MySQL database outside of kubernetes on a separate VM. Cloudify will orchestrate the entire topology and will enable the MediaWiki, which runs as a Docker container on kubernetes, to connect to the outside database.

The topology consists of the following components:
- Kubernetes
  - MediaWiki
- MySQL database

### Installing Cloudify CLI
Detailed instructions can be found in the official Cloudify documentation: http://getcloudify.org/guide/3.0/installation-cli.html 

On MacOS/Linux you should be able to simply install the Cloudify CLI through PyPi. 

This requires that you create a virtualenv in a folder of your choice:
```shell
user@localhost:$ virtualenv myVirtualEnv
```
Afterwards you need to activate the virtualenv:
```shell
user@localhost:$ source myVirtualEnv/bin/activate
```
Your commandline should now indicate that you're using a virtualenv. You can now install the Cloudify CLI via PyPi:
```shell
(myVirtualEnv) user@localhost:$ pip install cloudify
```

### Bootstrap Cloudify Management VM on AWS

Every Cloudify environment needs a management VM which controls the orchestration of the components in a specific Cloud environment. In order to bootstrap a management VM in a Cloud environment, so called manager blueprints are used. Cloudify provides a blueprints for all major cloud providers: https://github.com/cloudify-cosmo/cloudify-manager-blueprints

As we want to deploy our application on AWS, we will use the AWS manager blueprint as a starting point.

The required files are:
- aws-ec2-manager-blueprint.yaml
- aws-ec2-manager-blueprint-inputs.yaml

Get version 3.3.1 from GitHub and extract it to your directory:
https://github.com/cloudify-cosmo/cloudify-manager-blueprints/archive/3.3.1-sec1.tar.gz

Now open `aws-ec2-manager-blueprint-inputs.yaml` and fill in the following fields:
- `aws_access_key_id: 'XXX'` Your AWS credentials access key (see: [AWS Credentials - Getting Started](http://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSGettingStartedGuide/AWSCredentials.html))
- `aws_secret_access_key: 'XXX'` Your AWS credentials secret key
- `image_id: 'ami-6d1c2007'` The AMI Image that will be used for the Management VM, I recommend CentOS Linux 7 x86_64 (ami-6d1c2007)
- `instance_type: 'm3.medium'` The instance type that will be used for the Management VM
- `ssh_user: 'centos'` This needs to be set to the SSH user that matches the OS (Image)
- `agent_user: 'ubuntu'` This needs to be set to the SSH user that matches the OS (Image) on the agent hosts.
- `ec2_region_name: 'us-east-1'` The EC2 region that you wan't to deploy the management VM to

Save and close the file. 

Afterwards run `cfy init` in the directory.

The next step is to run the `bootstrap` command:
```shell
cfy bootstrap --install-plugins -p cloudify-manager-blueprints-3.3.1-sec1/aws-ec2-manager-blueprint.yaml -i cloudify-manager-blueprints-3.3.1-sec1/aws-ec2-manager-blueprint-inputs.yaml
```

This bootstrapping process will take quite a while.

Afterwards the commandline tool should output the IP address of the management VM and you should be able to access the dashboard through a browser.

```shell
bootstrapping complete
management server is up at 52.204.213.123
```

### Package the blueprint file

Clone the git repository into a folder of your choice.
```shell
git clone https://github.com/kgoedecke/cloudify-kubernetes
```

Afterwards cd into the `aws-mediawiki-example` folder. 
```shell
cd aws-mediawiki-example/
```

Before you package the blueprint with `tar` make, run the following command to disable copyfile:
```shell
export COPYFILE_DISABLE=true
```

Now go ahead and `tar` the MediaWiki example blueprint:
```shell
tar -czf blueprint-kubernetes-mediawiki.tar.gz aws-mediawiki-example 
```

### Upload the blueprint

Open the Cloudify Dashboard and select the Blueprint tab. Select the "Upload Blueprint" button and a dialog will open that requires you to enter the blueprint archive and some details. Select the just created archive and enter a blueprint name as well as `aws-blueprint.yaml` as the blueprint filename. The package contains everything that's needed to run the example (including the kubernetes plugin).

<img width="932" alt="screenshot1" src="https://cloud.githubusercontent.com/assets/5519740/16435659/f1034838-3d97-11e6-8649-8dd9209d05b8.png">

After the blueprint has been uploaded you'll see a graph of the topology.

<img width="758" alt="screenshot2" src="https://cloud.githubusercontent.com/assets/5519740/16435743/70a4079e-3d98-11e6-8a71-f97a18718372.png">

### Creating the deployment

Click on "Create Deployment" and the following dialog will open:

<img width="757" alt="screenshot3" src="https://cloud.githubusercontent.com/assets/5519740/16435770/a43867bc-3d98-11e6-9c69-0313a63be131.png">

The `agent_user` needs to the match the operating system that will run on the separate nodes. So make sure the image that you specify with `image`. In this example an Ubuntu image was used, so the default ssh user is ubuntu. Additionally a root password for the MySQL database is required as well as the instance type that will be used to deploy the nodes on.

This will trigger the actual installation of the plugin on the management VM.

After the workflow has been executed successfully a message should show up like in the screenshot below:

<img width="892" alt="screenshot4" src="https://cloud.githubusercontent.com/assets/5519740/16435994/99f3e0d6-3d9a-11e6-8584-495645e48f3b.png">

Afterwards the installation and deployment of the application itself needs to be done. Select "Execute Workflow" and click on "Install".

<img width="329" alt="screenshot5" src="https://cloud.githubusercontent.com/assets/5519740/16436035/d9ad5e0a-3d9a-11e6-912c-c65a1b5f092f.png">

The process will also take a while and you'll receive continous updates, which will let you know if any errors occured during the install workflow.

<img width="1019" alt="screenshot6" src="https://cloud.githubusercontent.com/assets/5519740/16454831/860790c2-3e12-11e6-95dc-c69436dbd35f.png">


After the successful exection you should be able to access the MediaWiki on port `8001` of the kubernetes Master Node, which you can find in your AWS Dashboard.

### Scaling 

With the current setup there are two possibilities to scale the applications components. You can either scale up/down a Cloudify host or a kubernetes node through kubernetes. 

To enable the scaling of a Cloudify node the "scale" workflow can be used. In our case the Cloudify native node is the MySQL host.

#### Scaling the MySQL host
In the "Deployments" tab select "Execute Workflow" and the following dialog will show up:

<img width="317" alt="screenshot7" src="https://cloud.githubusercontent.com/assets/5519740/16455393/6d9904d8-3e14-11e6-83ea-848029d58b82.png">

Select the "scale" workflow. The delta value represents the difference to the current value, so for example delta = 1 will scale the node up one node, -1 will scale down one node. As "node_id" type in `mysqld_host` as we want to scale up the MySQL host node and as this is a compute node set "scale_compute" to true.

After confirming the execution Cloudify should provision an other node and add it to the Dashboard topology overview. 

<img width="803" alt="screenshot8" src="https://cloud.githubusercontent.com/assets/5519740/16455601/39a965e0-3e15-11e6-96ad-89e142801aca.png">

You will also see that additional nodes have been provisioned in the AWS dashboard. The Cloudify loadbalancer will automatically split the load between the different nodes.

#### Scaling a kubernetes node

In order to scale the kubernetes node up or down open the "Deployments" tab again and select "Execute Workflow". The dialog will require you to enter the following information:
- master: The name of the kubernetes master node (here `master_host`, can be found in the deployment topology diagram)
- name: The name of the service that you want to scale (here `mediawiki`, this is specified in the `service.yaml` file)
- replicas: The number of replicas that you want to scale to

Internally Cloudify will run `kubectl -s http://localhost:8080 scale` to scale the passed service.

The rest will happen in a fully automated way and the changes can be verified using the kubernetes API for example.
