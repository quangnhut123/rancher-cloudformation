# Rancher Cloudformation

This is a CloudFormation template suitable to install [Rancher](www.rancher.com) in a HA infrastructure hosted on AWS. with the given topology

<img src=assets/topology.png width=500 />

 * a dedicated VPC.
 * VMs are distributed on 2 AZ.
 * a Bastion host allows to connect to VMs.
 * 2 ELB allows to spread load to Rancher and Agents.
 * a RDS MySQL database for Rancher server.
 * a way to deploy several agent configuration



## Prerequisites

First, make sure your AWS credentials are ok in `~/.aws/credentials` so that boto can interract with AWS. 

After that you need a Python installation with `virtualenv`.

## How to use

### Launch the rancher server

    # create a python virtual environment to not pollute your python installation
    virtualenv venv
    . ./venv/bin/activate

    # Install required python modules
    pip install -r requirements.txt

    # Install required ansible galaxy modules
    ansible-galaxy install -r ansible-requirements.txt

    # Launch the playbook 
    ansible-playbook -i inventory/rancher-dev deploy-infra.yml

If you don't use a `virtualenv`, you need ansible >= 2.2 otherwise, you'll have to run the script twice, since it doesn't know how to refresh it's inventory after the cloudformation call.

### Manual part : Setup SSL certificate and secure UI with authentication

You have to setup a valid SSL certificate for your Rancher cluster. For that go to the `System HA` environment, then `Infrastructure -> Certificates` and edit the `system-ssl` certificate with your valide certificate. 

After that, secure the UI by configuration an authentication scheme. 

### Launch agents

In order to launch agent, we have to get the token of the project (environment) which the agent will be linked to. In the Rancher UI, go to `Infrastructure -> Hosts` menu and click on the `Add host` button. Chossing the custom method, a command will appear as the last step. Copy the token which is the part right after `http://myrancherserver.com/v1/project/` and paste it into the `group_vars/rancher-dev.yml`

    rancher_agent_clusters:
      default:
        size: 1
        ec2_instance_type: t2.large
        ec2_storage: 30
        rancher_project_token: B64F080D160014784341:1480064400000:Y88o3ACI0UQeM9SNXemsid4avU
        rancher_agent_tags: "pool=default, storage=large"

For each cluster, you can specify a token (meaning an environment) and a set of tags that can be used after for scheduling.

Once the agent setup is done, you can launch the agent provisioning with ansible :

    ansible-playbook -i inventory/rancher-dev deploy-agent.yml    


## How it works

### Cloudformation and dynamic inventory

At first, a CloudFormation template is generated based on the values in `group_vars/rancher-dev.yml`. The `rancher_agent_clusters` defines the number of auto-scaling group to create and the properties of each launch configuration.

Then we dynamically update the `inventory/rancher-dev/hosts` file in order to set some variable on the agent groups. So don't edit this file by hand as it is generated by ansible. To change it go to `roles/cloudformation/templates/inventory-hosts.j2`.

The CloudFormation template is then launched. Once completed, we refresh the inventory so that each new machine comes up in the inventory and in the right groups.

### Bastion only access

Our cluster runs in a VPC and no machine except the bastion has a public adress. So each connection goes thru the bastion. This is achieved by generating the `tmp/ssh.cfg` file whith the value of the inventory. 

You can use this file to connect to one of the server in the VPC :

    ssh -F tmp/ssh.cfg 10.0.3.68


### Adding a new deployment

The script is done so that you can easily add a new deployment, meaning a complete new stack (in a new VPC or in a new AWS Region).

To do so, copy the `group_vars/rancher-dev.yml` file into `group_vars/rancher-mynewdeployment.yml`. Duplicate the `inventory/rancher-dev` folder into `inventory/rancher-mynewdeployment` and that's it : you can now provision your environment by launching : 

    ansible-playbook -i inventory/rancher-mynewdeployment deploy-infra.yml 


## License

[Apache License, Version 2.0](http://www.apache.org/licenses/LICENSE-2.0.html)

##About Nuxeo

Nuxeo dramatically improves how content-based applications are built, managed and deployed, making customers more agile, innovative and successful. Nuxeo provides a next generation, enterprise ready platform for building traditional and cutting-edge content oriented applications. Combining a powerful application development environment with SaaS-based tools and a modular architecture, the Nuxeo Platform and Products provide clear business value to some of the most recognizable brands including Verizon, Electronic Arts, Sharp, FICO, the U.S. Navy, and Boeing. Nuxeo is headquartered in New York and Paris. More information is available at www.nuxeo.com.