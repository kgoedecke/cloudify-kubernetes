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
