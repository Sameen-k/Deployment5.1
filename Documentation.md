## Purpose 
The purpose of this deployment was to recreate deployment 5 with more redundancy. In this deployment, there are 2 EC2s that are hosting the application. This is good practice to ensure that if there are any issues with the first instance of the application, there is a second instance to back it up. In this deployment, we are also making use of Jenkins agents which allows for the distribution of a pipeline across machines. Specifically, the build and test stages were completed in one EC2 however the clean and deploy stages were both completed together, twice in two different EC2s.    
## Diagram 
![Deployment 5 1 drawio (1)](https://github.com/Sameen-k/Deployment5.1/assets/128739962/2ea31799-0652-4697-80c0-b892a170a4db)

## Steps 

#### Keypair configuration:
The first step is to navigate to AWS and configure a new key pair for this deployment. When creating the terraform file be sure to use this key pair as the key name when configuring each EC2 resource block. Also, download the .pem file that is created with the creation of the key pair.

#### Terraform:
As shown in the diagram above, in this deployment we will be using 4 total EC2s. One EC2 will be used to launch the other 3 EC2s using Terraform. Within the Terraform file be sure to configure the creation of a VPC, 3EC2s (2 of which exist in one availability zone (us-east-1a) and public subnet, and 1 EC2 in a different availability zone (us-east-1b) and a public subnet. When configuring the security group, make sure to include port 8080, 8000, and 22. Please refer to the main.tf file for specific configurations.
Of the four instances, the first one is responsible for Terraform, the second one is responsible for hosting Jenkins, and the third and fourth ones both have a configured Jenkins agent that is responsible for deploying the application. Throughout this documentation, the second EC2 will be referred to as the Jenkins EC2, the third EC2 will be referred to as Application 1 EC2 and the fourth EC2 will be referred to as Application 2 EC2. 

#### Installations 
Make sure you have Jenkins installed on the Jenkins EC2. (There is a script attached within the main.tf file that installs Jenkins upon launch)
In order for the application to run, there are some packages that need to be installed.
On the Jenkins EC2, the following must be installed:
``software-properties-common``
``sudo add-apt-repository -y ppa:deadsnakes/ppa``
``python3.7``
``python3.7-venv``

On Application 1 EC2 and Application 2 EC2, install the following:
``default-jre``
``software-properties-common``
``sudo add-apt-repository -y ppa:deadsnakes/ppa``
``python3.7``
``python3.7-venv``

#### Jenkins Agent configuration:
Create a multibranch pipeline on Jenkins. Make sure to install the "Pipeline Keep Running Step" plugin. Now you can configure the agent by adding a new node. Make sure to label the node awsDeploy since that's what it is referred to as within the Jenkins file. For the remote root directory, use the following path: ``/home/ubuntu/agent1``. Make sure the launch method is set to SSH and use the Application 1 EC2 IP address as the host IP address. For the credentials section, select "SSH username with private key", then set "ubuntu" as the username, and under private key, select "Enter Directly". Now open your .pem file that was downloaded earlier when creating the key pair using the TextEdit application. Copy the RSA private key and past it into the paste it into the private key section
Now run the pipeline in Jenkins and use the IP address of Application 1 EC2 with port 8000 to navigate to the application page to check if it's working.

![Screen Shot 2023-10-18 at 12 39 55 PM](https://github.com/Sameen-k/Deployment5.1/assets/128739962/e179b940-6a3f-4b6a-a411-5ecd95f2a493)

Now the application is supposed to run on the Application 2 EC2 as well, so just configure a second Jenkins agent the same way. The application should be launched on port 8000 with the IP address on Application 2 EC2. 

![Screen Shot 2023-10-18 at 12 40 46 PM](https://github.com/Sameen-k/Deployment5.1/assets/128739962/f7d2e673-a266-424e-8dba-5aea8f53200d)

## Troubleshooting
1. When configuring the Jenkins agent, at first, I attempted to generate a private key in the Jenkins EC2 and pasted the key in the authorized_keys file in the Application 1 EC2, and used that same key on Jenkins itself. However, the pipeline failed completely until I used the RSA private key from the .pem file. 

2. When attempting to launch the application in Application 2 EC2 after configuring the agent, the pipeline kept failing at the clean stage. Upon further investigation, I realized I didn't include the agent label name within the Jenkins file. This meant the clean and deploy stages were not being run on agent 2, and were only running on agent 1. After adding the second agent, the Jenkins file looked like the following:

```
stage ('Clean') {
agent {label 'awsDeploy || awsDeploy2'}
steps {
sh '''#!/bin/bash
if [[ $(ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2) != 0 ]]
then
ps aux | grep -i "gunicorn" | tr -s " " | head -n 1 | cut -d " " -f 2 > pid.txt
kill $(cat pid.txt)
exit 0
fi
'''
}
}
stage ('Deploy') {
agent {label 'awsDeploy || awsDeploy2'}
steps {
keepRunning {
sh '''#!/bin/bash
python3.7 -m venv test
source test/bin/activate
pip install pip --upgrade
pip install -r requirements.txt
pip install gunicorn
python database.py
sleep 1
python load_data.py
sleep 1 
python -m gunicorn app:app -b 0.0.0.0 -D && echo "Done"
'''
}
}
}
```
## Optimization 
The current infrastructure includes only public subnets so it poses quite a large security risk. It would be best practice to include at minimum one private subnet for the Jenkins EC2.
To make the application increasingly available to users we can implement a load balancer into our infrastructure to evenly distribute traffic so that neither of the application instances are overwhelmed. We can also introduce an autoscaling group that can automatically adjust the number of instances based on the current load. 
