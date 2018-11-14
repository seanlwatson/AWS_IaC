Treating infrastructure as code
===

# Using CloudFormation

CloudFormation provides the ability to describe the AWS architecture to build through JSON or YAML files. Once the files are created, they can be uploaded to CloudFormation, which will then execute them and automatically create or update the associated AWS resources. CloudFormation can be reached at https://console.aws.amazon.com/cloudformation/home.

Using the AWS CLI tools required going through a number of steps to configure the EC2 instance and its security groups. Because that was done in a manual fashion, those steps are not reusable or auditable.

Existing templates can be found here: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-sample-templates-us-west-2.html. The CloudFormation service is organized around the concept of stacks. Each stack describes a set of AWS resources and thier configuration in order to start an application.

Templates have the following format:

```jsonld=
{ 
  "AWSTemplateFormatVersion" : "2010-09-09", 
  "Description" : "Description string", 
  "Resources" : { }, 
  "Parameters" : { }, 
  "Mappings" : { },
  "Conditions" : { }, 
  "Metadata" : { }, 
  "Outputs" : { } 
}
```
- `AWSTemplateFormatVersion`: The latest template format version is 2010-09-09 and is currently the only valid value.
- `Description`: Provide a summary of what the template does.
- `Resources`: Describe which AWS services will be instantiated and what thier configurations are.
- `Parameters`: Provide extra information such as which SSH keypair to use.
- `Mappings`: Useful when trying to create a more generic template such as defining which AMI to use for a given region so that the same template can be used to start an application in any AWS region.
- `Conditions`: Define conditional logic such as if statements, logical operators, and so on.
- `Metadata`: Add arbitrary information to resources.
- `Outputs`: Allows the ability to extract and print out useful information based on execution of the template such as IP address of the EC2 instance created.

## Using Python to create a CloudFormation template

From a DevOps perspective, one of the most powerful aspects of CloudFormation is the ability to write code to dynamically generate those templates. To illustrate that, we are going to use a Python library called Troposphere to generate the Hello World CloudFormation template.

Install the troposphere python library:

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ sudo pip install troposphere
</pre>


Create the file `helloworld-cf-template-v1.py` and start by importing a number of definitions from the troposhere module:

```python=
"""Gnerating AWS CloudFormation template."""

from troposphere import (
    Base64,
    ec2,
    GetAtt,
    Join,
    Output,
    Parameter,
    Ref,
    Template,
)
```

Create a variable that will make editing the code easier later on.

```python
ApplicationPort = "3000"
```

This var instantiates an object of the type Template that will contain the entire description of the infrastructure.

```python
t = Template()
```

Provide a description of the stack.

```python
t.add_description("Effective DevOps in AWS: HelloWorld web application")
```

Create a Parameter object and intialize it by providing an identifier, description, parameter type, and a constraint description to help make the right decision when launching the stack. This will allow the CloudFormation user the ability to select which key pair to use when launching the EC2 instance.

```python
t.add_parameter(Parameter(
    "KeyPair",
    Description="Name of an existing EC2 KeyPair to SSH",
    Type="AWS::EC2::KeyPair::KeyName",
    ConstraintDescription="must be the name of an existing EC2 KeyPair.",
))
```

Create a Security Group object to open up SSH and TCP/3000. Port 3000 was defined in the var `ApplicaitonPort` declared earlier.

```python
t.add_resource(ec2.SecurityGroup(
    "SecurityGroup",
    GroupDescription="Allow SSH and TCP/{} access".format(ApplicationPort),
    SecurityGroupIngress=[
        ec2.SecurityGroupRule(
            IpProtocol="tcp",
            FromPort="22",
            ToPort="22",
            CidrIp="0.0.0.0/0",
        ),
        ec2.SecurityGroupRule(
            IpProtocol="tcp",
            FromPort=ApplicationPort,
            ToPort=ApplicationPort,
            CidrIp="0.0.0.0/0",
        ),
    ],
))
```

Utilize the user-data optional parameter to provide a set of commands to install the `helloworld.js` file and its init script once the VM is up. This part of the script is the same steps that were done in the first chapter: Deploying your first web application.For more info see [user-data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html).

```python
The script will be base64-encoded
ud = Base64(Join('\n', [
    "#!/bin/bash",
    "sudo yum install --enablerepo=epel -y nodejs",
    "wget http://bit.ly/2vESNuc -O /home/ec2-user/helloworld.js",
    "wget http://bit.ly/2vVvT18 -O /etc/init/helloworld.conf",
    "start helloworld"
]))
```

> [Cloud-init](https://cloudinit.readthedocs.io/en/latest/) is the defacto multi-distribution package that handles early initialization of a cloud instance. It complements the UserData field by moving most standard operations, such as installing packages, creating files, and running commands, into different sections of the template.

Create an EC2 instance object by providing the name, an image ID, instance type, security group, keypair for SSH, and user data. In CloudFormation, you can reference pre-existing subsections of your template by using the keyword `Ref`. In troposphere, this is done by calling the `Ref()` function. Add the resulting output to the template with the `add_resource` function.

```python
t.add_resource(ec2.Instance(
    "instance",
    ImageId="ami-a0cfeed8",
    InstanceType="t2.micro",
    SecurityGroups=[Ref("SecurityGroup")],
    KeyName=Ref("KeyPair"),
    UserData=ud,
))
```

The `Outputs` section of the tempalte that gets populated when CloudFormation creates a stack. Useful information we need is the URL to access the web application and the public IP address of the instance in order to SSH. Use the CloudFormation function Fn::GetAtt, which in Troposphere is translated to `GetAttr()` function.

```python
t.add_output(Output(
    "InstancePublicIp",
    Description="Public IP of our instance.",
    Value=GetAtt("instance", "PublicIp"),
))

t.add_output(Output(
    "WebUrl",
    Description="Application endpoint",
    Value=Join("", [
        "http://", GetAtt("instance", "PublicDnsName"),
        ":", ApplicationPort
    ]),
))
```

Output the final result of the template generated.

```python
print t.to_json()
```

Run the script to generate the CloudFormation template by saving the output of the script to a file.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ python helloworld-cf-template.py &gt; helloworld-cf.template
</pre>

```jsonld=
{
    "Description": "Effective DevOps in AWS: HelloWorld web application",
    "Outputs": {
        "InstancePublicIp": {
            "Description": "Public IP of our instance.",
            "Value": {
                "Fn::GetAtt": [
                    "instance",
                    "PublicIp"
                ]
            }
        },
        "WebUrl": {
            "Description": "Application endpoint",
            "Value": {
                "Fn::Join": [
                    "",
                    [
                        "http://",
                        {
                            "Fn::GetAtt": [
                                "instance",
                                "PublicDnsName"
                            ]
                        },
                        ":",
                        "3000"
                    ]
                ]
            }
        }
    },
    "Parameters": {
        "KeyPair": {
            "ConstraintDescription": "must be the name of an existing EC2 KeyPair.",
            "Description": "Name of an existing EC2 KeyPair to SSH",
            "Type": "AWS::EC2::KeyPair::KeyName"
        }
    },
    "Resources": {
        "SecurityGroup": {
            "Properties": {
                "GroupDescription": "Allow SSH and TCP/3000 access",
                "SecurityGroupIngress": [
                    {
                        "CidrIp": "0.0.0.0/0",
                        "FromPort": "22",
                        "IpProtocol": "tcp",
                        "ToPort": "22"
                    },
                    {
                        "CidrIp": "0.0.0.0/0",
                        "FromPort": "3000",
                        "IpProtocol": "tcp",
                        "ToPort": "3000"
                    }
                ]
            },
            "Type": "AWS::EC2::SecurityGroup"
        },
        "instance": {
            "Properties": {
                "ImageId": "ami-a0cfeed8",
                "InstanceType": "t2.micro",
                "KeyName": {
                    "Ref": "KeyPair"
                },
                "SecurityGroups": [
                    {
                        "Ref": "SecurityGroup"
                    }
                ],
                "UserData": {
                    "Fn::Base64": {
                        "Fn::Join": [
                            "\n",
                            [
                                "#!/bin/bash",
                                "sudo yum install --enablerepo=epel -y nodejs",
                                "wget http://bit.ly/2vESNuc -O /home/ec2-user/helloworld.js",
                                "wget http://bit.ly/2vVvT18 -O /etc/init/helloworld.conf",
                                "start helloworld"
                            ]
                        ]
                    }
                }
            },
            "Type": "AWS::EC2::Instance"
        }
    }
}
```

## Create the stack in the CloudFormation console.

1. Open the [CloudFormation](https://console.aws.amazon.com/cloudformation) web console and click on **Create Stack**.

2. Under**Choose a template** select **Upload a template to Amazon S3** and **Choose File** to select the `helloworld-cf.template` file that was generated from the python script. Click **Next**.

3. Enter the **Stack name** and select the **KeyPair** and click **Next**.

4. On the next screen there is the ability to add optional tags to the resources. In the advanced section there is the option to integrate CloudFormation and SNS, make decisions on what to do when a failure occurs, and even add a stack policy that lets you control who can edit the stack. For this example leave everything blank and click **Next**. 

![](https://i.imgur.com/6h3u7IB.png)

5. On the review screen click on **Create**.

![](https://i.imgur.com/Ixopf9u.png)

6. In the CloudFormation console the events **Events** tab shows the status and when the template is completed, click on the **Outputs** tab, which will revewal the generated outputs from the template.

![](https://i.imgur.com/w0jaHWX.png)

![](https://i.imgur.com/kTHC2he.png)

7. Click on the link in the WebUrl key to open up the HelloWorld page.

![](https://i.imgur.com/dBh8Fyh.png)

## Add template to source control system
AWS has a service called AWS [CodeCommit] (https://aws.amazon.com/codecommit/) that allows easy management of Git repositories. However, it less popular that [GitHub](https://github.com) so GitHub will be used to create a new repository for the CloudFormation template:

1. Create a new repository in [GitHub](https://github.com/new).
2. Name the new repository `CloudFormation`.
3. Check the **Initialize this repository with a README** checkbox.
4. Click the **Create repository** button.

![](https://i.imgur.com/1sGAvrh.png)

5. Install Git (on Ubuntu).

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ sudo apt update
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ sudo apt install git -y
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ git --version
git version 2.17.1</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ git config --global user.email &quot;seanlwatson@gmail.com&quot;
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ git config --global user.name &quot;Sean Watson&quot;
</pre>

6. Clone the repository to your system.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ git clone https://github.com/seanlwatson/CloudFormation
Cloning into &apos;CloudFormation&apos;...
Username for 'https://github.com': seanlwatson
Password for 'https://seanlwatson@github.com': 
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), done.</pre>

7. Now that the repository is cloned, go into the repository and copy the template previously created in the new GitHub repository.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ cd CloudFormation/
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ cp ../helloworld-cf-template.py .
</pre>

8. Finally, add and commit that new file to the project and push it to GitHub.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ git add helloworld-cf-template.py </pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ git push
Username for &apos;https://github.com&apos;: seanlwatson
Password for &apos;https://seanlwatson@github.com&apos;: 
Counting objects: 3, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 1.05 KiB | 1.05 MiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To https://github.com/seanlwatson/CloudFormation
   8af6f40..a71e8d8  master -&gt; master
</pre>

> You may need to create a GitHub (personal access) [token](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/) if you are using two-factor authentication. This can be used in place of a password when performing Git operations over HTTPS with Git on the command line or the API.

> When managing code there are two common approaches. You can create a single repository for each project you have or decide to put the entire organizations code under a single repository. There are several open source projects such as [Bazel](https://bazel.build) from Google for using a mono repo so it becomes a very compelling option as it avoids juggling between multiple repositories when making big changes to infrastructure and services at the same time.

---
<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ python helloworld-cf-template.py &gt; helloworld-cf.template</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ git add helloworld-cf.template</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ git commit -m &quot;Adding helloworld CloudFormation template&quot;
[master 4c5d2f2] Adding helloworld CloudFormation template
 1 file changed, 91 insertions(+)
 create mode 100644 helloworld-cf.template
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ git push
Username for &apos;https://github.com&apos;: seanlwatson
Password for &apos;https://seanlwatson@github.com&apos;: 
Counting objects: 3, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 1.04 KiB | 1.04 MiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To https://github.com/seanlwatson/CloudFormation
   a71e8d8..4c5d2f2  master -&gt; master</pre>

## Updating the CloudFormation stack

One of the biggest benefits of CloudFormation templates to manage resources is that the resources created from the template are tighly coupled to the stack. In order to make changes to the stack, update the template and apply the change to the existing stack. Install the ipify python package, which is used for IP address lookup.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ sudo pip install ipify
</pre>

The script requires CIDR notation so install the ipaddress package to conver the IP address into CIDR notation.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ sudo pip install ipaddress
</pre>

Modify the `helloworld-cf-template.py` script to change the `CidrIp` declaration for the SSH group rules to reference the `PublicCidrIp` variable:

```python
from ipaddress import ip_network
from ipify import get_ip

PublicCidrIp = str(ip_network(get_ip()))
```

```python
t.add_resource(ec2.SecurityGroup(
    "SecurityGroup",
    GroupDescription="Allow SSH and TCP/{} access".format(ApplicationPort),
    SecurityGroupIngress=[
        ec2.SecurityGroupRule(
            IpProtocol="tcp",
            FromPort="22",
            ToPort="22",
            CidrIp=PublicCidrIp,
        ),
        ec2.SecurityGroupRule(
            IpProtocol="tcp",
            FromPort=ApplicationPort,
            ToPort=ApplicationPort,
            CidrIp=PublicCidrIp,
        ),
    ],
))
```

Generate a new template.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ python helloworld-cf-template.py &gt; helloworld-cf.template
</pre>

## Updating the stack

After updating the JSON CloudFormation template, update the stack as follows:

1. In the CloudFormation web console select the `HelloWorld` stack that was previously created.
2. Click on **Action** and then **Update Stack**. 
3. Check **Upload a template to Amazon S3** and click on the **Choose file** button and select the `helloworld-cf.template` by clicking on the **Choose** button and then clicking on **Next**. 
6. On the next screen click **Next**.

![](https://i.imgur.com/UC4n9UU.png)

7. In the next screen where the options exist to add optional tags to the resources and so on just click **Next**.
8. This brings us to the Review page and after a few seconds a preview of the change is given and more detail can be seen by clicking on the **View change set details** and then click **Execute**.

![](https://i.imgur.com/GkzEKaf.png)

![](https://i.imgur.com/KmGgxcD.png)

![](https://i.imgur.com/wtBwppQ.png)

![](https://i.imgur.com/LSw9GO1.png)

9. We can verify the change by extracting the Security Group physical resource ID from the review page and see that the `CidrIp` field in `IpPermissions` (for ingress) now has the value `75.166.145.22/32` instead of the allow all `0.0.0.0/0`.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ aws ec2 describe-security-groups --group-names HelloWorld-SecurityGroup-1I2MCWU9SJYHO
{
    &quot;SecurityGroups&quot;: [
        {
            &quot;Description&quot;: &quot;Allow SSH and TCP/3000 access&quot;,
            &quot;GroupName&quot;: &quot;HelloWorld-SecurityGroup-1I2MCWU9SJYHO&quot;,
            &quot;IpPermissions&quot;: [
                {
                    &quot;FromPort&quot;: 22,
                    &quot;IpProtocol&quot;: &quot;tcp&quot;,
                    &quot;IpRanges&quot;: [
                        {
                            &quot;CidrIp&quot;: &quot;75.166.145.22/32&quot;
                        }
                    ],
                    &quot;Ipv6Ranges&quot;: [],
                    &quot;PrefixListIds&quot;: [],
                    &quot;ToPort&quot;: 22,
                    &quot;UserIdGroupPairs&quot;: []
                },
                {
                    &quot;FromPort&quot;: 3000,
                    &quot;IpProtocol&quot;: &quot;tcp&quot;,
                    &quot;IpRanges&quot;: [
                        {
                            &quot;CidrIp&quot;: &quot;75.166.145.22/32&quot;
                        }
                    ],
                    &quot;Ipv6Ranges&quot;: [],
                    &quot;PrefixListIds&quot;: [],
                    &quot;ToPort&quot;: 3000,
                    &quot;UserIdGroupPairs&quot;: []
                }
            ],
            &quot;OwnerId&quot;: &quot;404297683117&quot;,
            &quot;GroupId&quot;: &quot;sg-09b45294e082298f9&quot;,
            &quot;IpPermissionsEgress&quot;: [
                {
                    &quot;IpProtocol&quot;: &quot;-1&quot;,
                    &quot;IpRanges&quot;: [
                        {
                            &quot;CidrIp&quot;: &quot;0.0.0.0/0&quot;
                        }
                    ],
                    &quot;Ipv6Ranges&quot;: [],
                    &quot;PrefixListIds&quot;: [],
                    &quot;UserIdGroupPairs&quot;: []
                }
            ],
            &quot;Tags&quot;: [
                {
                    &quot;Key&quot;: &quot;aws:cloudformation:logical-id&quot;,
                    &quot;Value&quot;: &quot;SecurityGroup&quot;
                },
                {
                    &quot;Key&quot;: &quot;aws:cloudformation:stack-id&quot;,
                    &quot;Value&quot;: &quot;arn:aws:cloudformation:us-west-2:404297683117:stack/HelloWorld/01e63360-dbb4-11e8-a96f-0ad5109330ec&quot;
                },
                {
                    &quot;Key&quot;: &quot;aws:cloudformation:stack-name&quot;,
                    &quot;Value&quot;: &quot;HelloWorld&quot;
                }
            ],
            &quot;VpcId&quot;: &quot;vpc-b3b5feca&quot;
        }
    ]
}
</pre>

> AWS offers an alternate and safer way to update templates with a feature called change sets. When in the CloudFormation web console select the `HelloWorld` stack that was previously created. When clicking on **Action** then select **Create Change Set** instead of **Update Stack**. This allows the ability to audit recent changes using the **Change Sets** tabl of the stack.
> 
> ![](https://i.imgur.com/Ru76mdf.png)

## Deleting the CloudFormation stack

Best practice is to always use CloudFormation to make changes to resources previously intialized with CloudFormation, including deleting them.

1. Open the [CloudFormation](https://console.aws.amazon.com/cloudformation) web console and select the `HelloWorld` stack that was previously created.
2. Click on **Action** and then select **Delete Stack** and you will have to confirm the delete by clicking **Yes, Delete**.

![](https://i.imgur.com/9N62MT3.png)

Finally commit changes to git for the Troposphere python script and the CloudFormaiton template.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ git commit -am &quot;Only allow SSH from our local IP&quot;
[master 7ed693b] Only allow SSH from our local IP
 2 files changed, 7 insertions(+), 4 deletions(-)
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ git push
Username for &apos;https://github.com&apos;: seanlwatson
Password for &apos;https://seanlwatson@github.com&apos;: 
Counting objects: 4, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (4/4), done.
Writing objects: 100% (4/4), 534 bytes | 534.00 KiB/s, done.
Total 4 (delta 2), reused 0 (delta 0)
remote: Resolving deltas: 100% (2/2), completed with 2 local objects.
To https://github.com/seanlwatson/CloudFormation
   4c5d2f2..7ed693b  master -&gt; master
</pre>

Click on the **commits** link to view all commits performed and then click on `7ed693b` to view a diff of changes.

![](https://i.imgur.com/xfhloXf.png)

![](https://i.imgur.com/iS1AlLF.png)

![](https://i.imgur.com/Kj2nacu.png)

# Adding a configuration management system with Ansible

Because CloudFormaiton doesn't keep track of state of resources once they are launched, the only reliable way to update an EC2 instance is to recreate a new instance and swap it with the existing instance once the new instance is ready. This creates a somewhat immutable (unchanging over time) design assuing you don't run any extra commands once the instance is created. Having the ability to have long-running instances where you can quickly and reliably make changes through a controlled pipeline similiar to CloudFormation is what configuration management systems such as Puppet, Chef, SaltStack, and Ansible excel at.

Unlike other configuration mgt. systems, Ansible is built to work without a server, a daemon, or a database. You can simply keep your code in source control and download it on the host whenever you need to run it or use a push mechanism via SSH. The automation code you write is in YAML static files and will use GitHub as our version control system.

AWS OpsWorks has Chef integration which aims at being a "complete life cycle, including resource provisioning, configuration management, application deployment, software updates, monitoring, and access control".

## Install Ansible

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ sudo pip install ansible
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ ansible --version
ansible 2.7.1
  config file = None
  configured module search path = [u&apos;/home/sean/.ansible/plugins/modules&apos;, u&apos;/usr/share/ansible/plugins/modules&apos;]
  ansible python module location = /usr/local/lib/python2.7/dist-packages/ansible
  executable location = /usr/local/bin/ansible
  python version = 2.7.15rc1 (default, Apr 15 2018, 21:51:34) [GCC 7.3.0]
</pre>

## Create Ansible playground

Re-launch the helloworld application by using the CLI to interface with CloudFormation.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ aws cloudformation create-stack \
&gt; --capabilities CAPABILITY_IAM \
&gt; --stack-name Ansible \
&gt; --template-body file://helloworld-cf.template \
&gt; --parameters ParameterKey=KeyPair,ParameterValue=EffectiveDevOpsAWS
{
    &quot;StackId&quot;: &quot;arn:aws:cloudformation:us-west-2:404297683117:stack/Ansible/4b8e9f00-dca6-11e8-9ab5-503a90a9c435&quot;
}
</pre>

![](https://i.imgur.com/cMKPV1i.png)

## Create Ansible repository

1. Create a new repository in [GitHub](https://github.com/new).
2. Name the new repository `Ansible`.
3. Check the **Initialize this repository with a README** checkbox.
4. Click the **Create repository** button.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ git clone https://github.com/seanlwatson/Ansible
Cloning into &apos;Ansible&apos;...
remote: Enumerating objects: 3, done.
remote: Counting objects: 100% (3/3), done.
remote: Total 3 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (3/3), done.
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ cd Ansible/</pre>

Download the EC2 external inventory script and give execution permissions.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ curl -Lo ec2.py http://bit.ly/2v4SwE5
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   184  100   184    0     0    641      0 --:--:-- --:--:-- --:--:--   641
100 68938  100 68938    0     0   107k      0 --:--:-- --:--:-- --:--:--  216k
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ chmod +x ec2.py 
</pre>

Create an `ec2.ini` file and put the following configuration:

```
[ec2] 
regions = all 
regions_exclude = us-gov-west-1,cn-north-1 
destination_variable = public_dns_name 
vpc_destination_variable = ip_address 
route53 = False 
cache_path = ~/.ansible/tmp 
cache_max_age = 300 
rds = False
```

Install the `boto` Python package.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ pip install boto
</pre>

Validate that inventory is working by executing the `ec2.py` script.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ./ec2.py 
{
  &quot;404297683117&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;_meta&quot;: {
    &quot;hostvars&quot;: {
      &quot;54.189.226.117&quot;: {
        &quot;ansible_host&quot;: &quot;54.189.226.117&quot;, 
        &quot;ec2__in_monitoring_element&quot;: false, 
        &quot;ec2_account_id&quot;: &quot;404297683117&quot;, 
        &quot;ec2_ami_launch_index&quot;: &quot;0&quot;, 
        &quot;ec2_architecture&quot;: &quot;x86_64&quot;, 
        &quot;ec2_block_devices&quot;: {
          &quot;xvda&quot;: &quot;vol-0c2707aa04d4047c4&quot;
        }, 
        &quot;ec2_client_token&quot;: &quot;Ansib-insta-23FSM1KDHQUI&quot;, 
        &quot;ec2_dns_name&quot;: &quot;ec2-54-189-226-117.us-west-2.compute.amazonaws.com&quot;, 
        &quot;ec2_ebs_optimized&quot;: false, 
        &quot;ec2_eventsSet&quot;: &quot;&quot;, 
        &quot;ec2_group_name&quot;: &quot;&quot;, 
        &quot;ec2_hypervisor&quot;: &quot;xen&quot;, 
        &quot;ec2_id&quot;: &quot;i-093a0a830b46ce5a9&quot;, 
        &quot;ec2_image_id&quot;: &quot;ami-a0cfeed8&quot;, 
        &quot;ec2_instance_profile&quot;: &quot;&quot;, 
        &quot;ec2_instance_type&quot;: &quot;t2.micro&quot;, 
        &quot;ec2_ip_address&quot;: &quot;54.189.226.117&quot;, 
        &quot;ec2_item&quot;: &quot;&quot;, 
        &quot;ec2_kernel&quot;: &quot;&quot;, 
        &quot;ec2_key_name&quot;: &quot;EffectiveDevOpsAWS&quot;, 
        &quot;ec2_launch_time&quot;: &quot;2018-10-31T00:45:54.000Z&quot;, 
        &quot;ec2_monitored&quot;: false, 
        &quot;ec2_monitoring&quot;: &quot;&quot;, 
        &quot;ec2_monitoring_state&quot;: &quot;disabled&quot;, 
        &quot;ec2_persistent&quot;: false, 
        &quot;ec2_placement&quot;: &quot;us-west-2a&quot;, 
        &quot;ec2_platform&quot;: &quot;&quot;, 
        &quot;ec2_previous_state&quot;: &quot;&quot;, 
        &quot;ec2_previous_state_code&quot;: 0, 
        &quot;ec2_private_dns_name&quot;: &quot;ip-172-31-25-188.us-west-2.compute.internal&quot;, 
        &quot;ec2_private_ip_address&quot;: &quot;172.31.25.188&quot;, 
        &quot;ec2_public_dns_name&quot;: &quot;ec2-54-189-226-117.us-west-2.compute.amazonaws.com&quot;, 
        &quot;ec2_ramdisk&quot;: &quot;&quot;, 
        &quot;ec2_reason&quot;: &quot;&quot;, 
        &quot;ec2_region&quot;: &quot;us-west-2&quot;, 
        &quot;ec2_requester_id&quot;: &quot;&quot;, 
        &quot;ec2_root_device_name&quot;: &quot;/dev/xvda&quot;, 
        &quot;ec2_root_device_type&quot;: &quot;ebs&quot;, 
        &quot;ec2_security_group_ids&quot;: &quot;sg-014c8c6e132da8d1b&quot;, 
        &quot;ec2_security_group_names&quot;: &quot;Ansible-SecurityGroup-KZAEHA5Z3ZRW&quot;, 
        &quot;ec2_sourceDestCheck&quot;: &quot;true&quot;, 
        &quot;ec2_spot_instance_request_id&quot;: &quot;&quot;, 
        &quot;ec2_state&quot;: &quot;running&quot;, 
        &quot;ec2_state_code&quot;: 16, 
        &quot;ec2_state_reason&quot;: &quot;&quot;, 
        &quot;ec2_subnet_id&quot;: &quot;subnet-718a3a08&quot;, 
        &quot;ec2_tag_aws_cloudformation_logical_id&quot;: &quot;instance&quot;, 
        &quot;ec2_tag_aws_cloudformation_stack_id&quot;: &quot;arn:aws:cloudformation:us-west-2:404297683117:stack/Ansible/4b8e9f00-dca6-11e8-9ab5-503a90a9c435&quot;, 
        &quot;ec2_tag_aws_cloudformation_stack_name&quot;: &quot;Ansible&quot;, 
        &quot;ec2_virtualization_type&quot;: &quot;hvm&quot;, 
        &quot;ec2_vpc_id&quot;: &quot;vpc-b3b5feca&quot;
      }
    }
  }, 
  &quot;ami_a0cfeed8&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;ec2&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;i-093a0a830b46ce5a9&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;instance_state_running&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;key_EffectiveDevOpsAWS&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;security_group_Ansible_SecurityGroup_KZAEHA5Z3ZRW&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;tag_aws_cloudformation_logical_id_instance&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;tag_aws_cloudformation_stack_id_arn_aws_cloudformation_us_west_2_404297683117_stack_Ansible_4b8e9f00_dca6_11e8_9ab5_503a90a9c435&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;tag_aws_cloudformation_stack_name_Ansible&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;type_t2_micro&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;us-west-2&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;us-west-2a&quot;: [
    &quot;54.189.226.117&quot;
  ], 
  &quot;vpc_id_vpc_b3b5feca&quot;: [
    &quot;54.189.226.117&quot;
  ]
}
</pre>

Create a file called `ansible.cfg` with the below contents.

```
[defaults] 
inventory      = ./ec2.py 
remote_user    = ec2-user 
become         = True 
become_method  = sudo 
become_user    = root 
nocows         = 1
```

Use Ansible and the `ping` command to discover the IP address of the new EC2 instance that was created using CloudFormation.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ansible --private-key ~/.ssh/EffectiveDevOpsAWS.pem ec2 -m ping
The authenticity of host &apos;54.189.226.117 (54.189.226.117)&apos; can&apos;t be established.
ECDSA key fingerprint is SHA256:PQ80MvhVozF0/X/HZGJs7UHjJsNayXyNGZe5hI4msoU.
Are you sure you want to continue connecting (yes/no)? yes
<font color="#4E9A06">54.189.226.117 | SUCCESS =&gt; {</font>
<font color="#4E9A06">    &quot;changed&quot;: false, </font>
<font color="#4E9A06">    &quot;ping&quot;: &quot;pong&quot;</font>
<font color="#4E9A06">}</font>
</pre>

Since Ansible relies heavily on SSH configure SSH via the `$HOME/.ssh/config` file with the below configuration. Once configured, we will no longer have to provide the `--private-key` option to Ansible.

```bash
IdentityFile ~/.ssh/EffectiveDevOpsAWS.pem
User ec2-user
StrictHostKeyChecking no
PasswordAuthentication no
ForwardAgent yes
```

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~</b></font>$ touch ~/.ssh/config</pre>

## Running arbitrary commands

Use the below ansible command to run the `df` disk utility command to show disk space available and human readable `-h`.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ansible &apos;54.189.226.*&apos; -a &apos;df -h&apos;
<font color="#C4A000">54.189.226.117 | CHANGED | rc=0 &gt;&gt;</font>
<font color="#C4A000">Filesystem      Size  Used Avail Use% Mounted on</font>
<font color="#C4A000">devtmpfs        483M   60K  483M   1% /dev</font>
<font color="#C4A000">tmpfs           493M     0  493M   0% /dev/shm</font>
<font color="#C4A000">/dev/xvda1      7.8G  1.1G  6.6G  15% /</font>
</pre>

## Creating Ansible playbooks & roles

Playbooks contain Ansible's configuration, deployment, and orchestration language. By defining a playbook, you sequentially define the state of your system from the OS configuration down to the application deployment and monitoring. Playbooks are YAML based.

Creating roles is a key component in making Ansible modular enough for code reuse across services and playbooks. 

The `UserData` block of the CloudFormation template contained the following commands. This prepares teh system to run the `helloworld` application by installing node.js. Then different resources are copied to run the application such as the JavaScript code and the upstart configuration. Lastly, we start the service.

```
sudo yum install --enablerepo=epel -y nodejs
wget http://bit.ly/2vESNuc -O /home/ec2-user/helloworld.js
wget http://bit.ly/2vVvT18 -O /etc/init/helloworld.conf
start helloworld
```

Installing node.js is not unique to our application so to make this reusable piece of code, we will create two roles. One to install node.js and one to deploy and start the `helloworld` application. Ansible expects to see roles inside a roles directory at the root of the Ansible repository.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ mkdir roles
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ cd roles
</pre>

The `ansible-galaxy` command can be used to initialize the creation of a role.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible/roles</b></font>$ ansible-galaxy init nodejs
- nodejs was created successfully
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible/roles</b></font>$ cd nodejs/
</pre>

The most important directory inside the `nodejs` directory is one called `tasks`. When Ansible executes a playbook, it run the code present in the file `tasks/main.yml`

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible/roles/nodejs</b></font>$ ls
<font color="#729FCF"><b>defaults</b></font>  <font color="#729FCF"><b>files</b></font>  <font color="#729FCF"><b>handlers</b></font>  <font color="#729FCF"><b>meta</b></font>  README.md  <font color="#729FCF"><b>tasks</b></font>  <font color="#729FCF"><b>templates</b></font>  <font color="#729FCF"><b>tests</b></font>  <font color="#729FCF"><b>vars</b></font>
</pre>

When creating task we will use a wrapper around the `yum` command. Documentation about using the [yum module](https://docs.ansible.com/ansible/latest/modules/yum_module.html) is available. This will look at the packages installed on the system and if it doesn't find the `nodejs` or `rpm` packages then it will install them from the Extra Packages for Enterprise Linux (EPEL) repository.

<pre class="yaml" style="font-family:monospace;"><span style="color: cyan;">---</span>
<span style="color: blue;"># tasks file for nodejs</span>
<span style="color: green;">
- name</span><span style="font-weight: bold; color: brown;">: </span>Installing Node.js and NPM <span class="br0">&#40;</span>Node Package Mgr.<span class="br0">&#41;</span><span style="color: #007F45;">
  yum</span>:<span style="color: green;">
    name</span><span style="font-weight: bold; color: brown;">: </span><span style="color: #CF00CF;">&quot;{{ item }}&quot;</span><span style="color: green;">
    enablerepo</span><span style="font-weight: bold; color: brown;">: </span>epel<span style="color: green;">
    state</span><span style="font-weight: bold; color: brown;">: </span>installed<span style="color: #007F45;">
  with_items</span><span style="font-weight: bold; color: brown;">:
</span>    - nodejs
    - npm</pre>

Navigate up a directory and initialize the `helloworld` role. Navigate to the `helloworld` directory and download the two resource files needed for the application.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible/roles/nodejs</b></font>$ cd ..
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible/roles</b></font>$ ansible-galaxy init helloworld
- helloworld was created successfully
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible/roles</b></font>$ cd helloworld/
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible/roles/helloworld</b></font>$ wget http://bit.ly/2vESNuc -O files/helloworld.js
--2018-10-30 20:45:44--  http://bit.ly/2vESNuc
Resolving bit.ly (bit.ly)... 67.199.248.10, 67.199.248.11
Connecting to bit.ly (bit.ly)|67.199.248.10|:80... connected.
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://raw.githubusercontent.com/EffectiveDevOpsWithAWS/code-by-chapter/master/Chapter02/helloworld.js [following]
--2018-10-30 20:45:45--  https://raw.githubusercontent.com/EffectiveDevOpsWithAWS/code-by-chapter/master/Chapter02/helloworld.js
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 151.101.192.133, 151.101.128.133, 151.101.64.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|151.101.192.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 384 [text/plain]
Saving to: ‘files/helloworld.js’

files/helloworld.js                100%[==============================================================&gt;]     384  --.-KB/s    in 0s      

2018-10-30 20:45:45 (74.6 MB/s) - ‘files/helloworld.js’ saved [384/384]
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible/roles/helloworld</b></font>$ wget http://bit.ly/2vVvT18 -O files/helloworld.conf
--2018-10-30 20:46:24--  http://bit.ly/2vVvT18
Resolving bit.ly (bit.ly)... 67.199.248.11, 67.199.248.10
Connecting to bit.ly (bit.ly)|67.199.248.11|:80... connected.
HTTP request sent, awaiting response... 301 Moved Permanently
Location: https://raw.githubusercontent.com/EffectiveDevOpsWithAWS/code-by-chapter/master/Chapter02/helloworld.conf [following]
--2018-10-30 20:46:24--  https://raw.githubusercontent.com/EffectiveDevOpsWithAWS/code-by-chapter/master/Chapter02/helloworld.conf
Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 151.101.192.133, 151.101.128.133, 151.101.64.133, ...
Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|151.101.192.133|:443... connected.
HTTP request sent, awaiting response... 200 OK
Length: 301 [text/plain]
Saving to: ‘files/helloworld.conf’

files/helloworld.conf              100%[==============================================================&gt;]     301  --.-KB/s    in 0s      

2018-10-30 20:46:25 (61.7 MB/s) - ‘files/helloworld.conf’ saved [301/301]
</pre>

Now create the task file to perform the copy on the remote system by opening the `tasks/main.yml` file to add the following YAML code. We are using the [copy module](https://docs.ansible.com/ansible/latest/modules/copy_module.html) to copy the application file into the home directory of the `ec2-user`. Lastly there is a notify action that act as triggers that can be added  at the end of each block of task in a playbook. Here the notify is telling Ansible to call the `restart helloworld` directive if the file `helloworld.js` changed - the actual restart will be covered in another file. The notify option makes it easy to trigger events when a system changes state.

<pre class="yaml" style="font-family:monospace;"><span style="color: cyan;">---</span>
<span style="color: blue;"># tasks file for helloworld</span>
<span style="color: green;">
- name</span><span style="font-weight: bold; color: brown;">: </span>Copying the application files<span style="color: #007F45;">
  copy</span>:<span style="color: green;">
    src</span><span style="font-weight: bold; color: brown;">: </span>helloworld.js<span style="color: green;">
    dest</span><span style="font-weight: bold; color: brown;">: </span>/home/ec2-user/<span style="color: green;">
    owner</span><span style="font-weight: bold; color: brown;">: </span>ec2-user<span style="color: green;">
    group</span><span style="font-weight: bold; color: brown;">: </span>ec2-user<span style="color: green;">
    mode</span><span style="font-weight: bold; color: brown;">: </span>0644<span style="color: green;">
  notify</span><span style="font-weight: bold; color: brown;">: </span>restart helloworld</pre>
  
Now add another task to copy the second file, the upstart script. Then the last taks to perform is to start the service by utilizing the [service module](https://docs.ansible.com/ansible/latest/modules/service_module.html). The `tasks/main.yml` file should resemble the below code.

<pre class="yaml" style="font-family:monospace;"><span style="color: cyan;">---</span>
<span style="color: blue;"># tasks file for helloworld</span>
<span style="color: green;">
- name</span><span style="font-weight: bold; color: brown;">: </span>Copying the application file<span style="color: #007F45;">
  copy</span>:<span style="color: green;">
    src</span><span style="font-weight: bold; color: brown;">: </span>helloworld.js<span style="color: green;">
    dest</span><span style="font-weight: bold; color: brown;">: </span>/home/ec2-user/<span style="color: green;">
    owner</span><span style="font-weight: bold; color: brown;">: </span>ec2-user<span style="color: green;">
    group</span><span style="font-weight: bold; color: brown;">: </span>ec2-user<span style="color: green;">
    mode</span><span style="font-weight: bold; color: brown;">: </span>0644<span style="color: green;">
  notify</span><span style="font-weight: bold; color: brown;">: </span>restart helloworld
<span style="color: green;">
- name</span><span style="font-weight: bold; color: brown;">: </span>Copying the upstart file<span style="color: #007F45;">
  copy</span>:<span style="color: green;">
    src</span><span style="font-weight: bold; color: brown;">: </span>helloworld.conf<span style="color: green;">
    dest</span><span style="font-weight: bold; color: brown;">: </span>/etc/init/helloworld.conf<span style="color: green;">
    owner</span><span style="font-weight: bold; color: brown;">: </span>root<span style="color: green;">
    group</span><span style="font-weight: bold; color: brown;">: </span>root<span style="color: green;">
    mode</span><span style="font-weight: bold; color: brown;">: </span>0644
<span style="color: green;">
- name</span><span style="font-weight: bold; color: brown;">: </span>Starting the HelloWorld node service<span style="color: #007F45;">
  service</span>:<span style="color: green;">
    name</span><span style="font-weight: bold; color: brown;">: </span>helloworld<span style="color: green;">
    state</span><span style="font-weight: bold; color: brown;">: </span>started</pre>
    
The next file will provide Ansible with the knowlege of how to restart `helloworld` as called out in the notify paramater of our first task. These types of interactions are defined in the handler section of the role in the file `handlers/main.yml`/

<pre class="yaml" style="font-family:monospace;"><span style="color: cyan;">---</span>
<span style="color: blue;"># handlers file for helloworld</span>
<span style="color: green;">
- name</span><span style="font-weight: bold; color: brown;">: </span>restart helloworld<span style="color: #007F45;">
  service</span>:<span style="color: green;">
    name</span><span style="font-weight: bold; color: brown;">: </span>helloworld<span style="color: green;">
    state</span><span style="font-weight: bold; color: brown;">: </span>restarted</pre>
    
Lastly, we want to create a dependency that the `helloworld` role depends on the the `nodejs` role so that when the `helloworld` role is executed, it will first call the `nodejs` role and install the necessary requirements to run the application. Edit the `meta/main.yml` file, which has two sections. The first under `galaxy info` let you fill in information on the role you are building. This allows you to publish your role on GitHub and link it back into `ansible-galaxy` to share with the community. The second section at the bottom of hte file is called `dependencies` and is what should be edited to make sure that the Node.js is present on the system prior to starting the applicaiton. Remove the square brackets ([]) and add an entry to call the `nodejs` role.

<pre class="yaml" style="font-family:monospace;"><span style="color: #007F45;">dependencies</span><span style="font-weight: bold; color: brown;">:
</span>  <span style="color: blue;"># List your role dependencies here, one per line. Be sure to remove the '[]' above,</span>
  <span style="color: blue;"># if you add dependencies to this list.</span>
  - nodejs</pre>

## Create playbook

At the top level of the `Ansible` repository (two directories up from the `helloworld` role), create a new file called `helloworld-pb.yml` and add the following code. This tells Ansible to execute the role `helloworld` on the host listed in the variable target or localhost if the target isn't defined. The `become` option tells Ansible to execute the role with elevated priviliges (`sudo`).

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible/roles/helloworld</b></font>$ cd ~/Ansible/</pre>

<pre class="yaml" style="font-family:monospace;"><span style="color: cyan;">---</span>
<span style="color: blue;"># playbook file for helloworld application</span>
<span style="color: green;">
- hosts</span><span style="font-weight: bold; color: brown;">: </span><span style="color: #CF00CF;">&quot;{{ target | default('localhost') }}&quot;</span><span style="color: green;">
  become</span><span style="font-weight: bold; color: brown;">: </span><span style="font-weight: bold;">yes</span><span style="color: #007F45;">
  roles</span><span style="font-weight: bold; color: brown;">:
</span>    - helloworld</pre>

## Execute the playbook

Execution of a playbook uses the `ansible-playbook` command and relies on the same Ansible configuration file previously defined and therefore should be run at the root of the Ansible repository. The `-e` option allows the ability to pass in extra options for execution (or `--extra-vars`). One of the extra options is defining the variable `target`, which was declared in the `hosts` section of the playbook and set it to be equal to `ec2` to target all EC2 instances. The option `--list-hosts` makes Ansible return a list of host that match the hosts criteria but will not run anything against those host. This allows you to verify which host would run the actual playbook.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ansible-playbook helloworld-pb.yml \
&gt; -e target=ec2 \
&gt; --list-hosts

playbook: helloworld-pb.yml

  play #1 (ec2): ec2	TAGS: []
    pattern: [u&apos;ec2&apos;]
    hosts (1):
      54.189.226.117
</pre>

Check what will happen by performing a dry run with the playbook by using the `--check` option.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ansible-playbook helloworld-pb.yml \
&gt; -e target=54.189.226.117 \
&gt; --check

PLAY [54.189.226.117] ********************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

TASK [nodejs : Installing Node.js and NPM (Node Package Mgr.)] ***************************************************************************
<font color="#75507B">[DEPRECATION WARNING]: Invoking &quot;yum&quot; only once while using a loop via squash_actions is deprecated. Instead of using a loop to supply </font>
<font color="#75507B">multiple items and specifying `name: {{ item }}`, please use `name: [u&apos;nodejs&apos;, u&apos;npm&apos;]` and remove the loop. This feature will be </font>
<font color="#75507B">removed in version 2.11. Deprecation warnings can be disabled by setting deprecation_warnings=False in ansible.cfg.</font>
<font color="#C4A000">changed: [54.189.226.117] =&gt; (item=[u&apos;nodejs&apos;, u&apos;npm&apos;])</font>

TASK [helloworld : Copying the application files] ****************************************************************************************
<font color="#C4A000">changed: [54.189.226.117]</font>

TASK [helloworld : Copying the upstart file] *********************************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

TASK [helloworld : Starting the HelloWorld node service] *********************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

RUNNING HANDLER [helloworld : restart helloworld] ****************************************************************************************
<font color="#C4A000">changed: [54.189.226.117]</font>

PLAY RECAP *******************************************************************************************************************************
<font color="#C4A000">54.189.226.117</font>             : <font color="#4E9A06">ok=6   </font> <font color="#C4A000">changed=3   </font> unreachable=0    failed=0   </pre>

Having verified the host and code, run the playbook and execute changes.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ansible-playbook helloworld-pb.yml -e target=54.189.226.117 

PLAY [54.189.226.117] ********************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

TASK [nodejs : Installing Node.js and NPM (Node Package Mgr.)] ***************************************************************************
<font color="#75507B">[DEPRECATION WARNING]: Invoking &quot;yum&quot; only once while using a loop via squash_actions is deprecated. Instead of using a loop to supply </font>
<font color="#75507B">multiple items and specifying `name: {{ item }}`, please use `name: [u&apos;nodejs&apos;, u&apos;npm&apos;]` and remove the loop. This feature will be </font>
<font color="#75507B">removed in version 2.11. Deprecation warnings can be disabled by setting deprecation_warnings=False in ansible.cfg.</font>
<font color="#C4A000">changed: [54.189.226.117] =&gt; (item=[u&apos;nodejs&apos;, u&apos;npm&apos;])</font>

TASK [helloworld : Copying the application files] ****************************************************************************************
<font color="#C4A000">changed: [54.189.226.117]</font>

TASK [helloworld : Copying the upstart file] *********************************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

TASK [helloworld : Starting the HelloWorld node service] *********************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

RUNNING HANDLER [helloworld : restart helloworld] ****************************************************************************************
<font color="#C4A000">changed: [54.189.226.117]</font>

PLAY RECAP *******************************************************************************************************************************
<font color="#C4A000">54.189.226.117</font>             : <font color="#4E9A06">ok=6   </font> <font color="#C4A000">changed=3   </font> unreachable=0    failed=0 </pre>

Test the application.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ curl 54.189.226.117:3000
Hello World
</pre>

Commit changes to GitHub using 2 commits to break down the initialization of the repository and the creation of the role.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ git add ansible.cfg ec2.ini ec2.py </pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ git commit -m &quot;Configuring Ansible to work with EC2&quot;
[master 0d0a9a0] Configuring Ansible to work with EC2
 3 files changed, 1626 insertions(+)
 create mode 100644 ansible.cfg
 create mode 100644 ec2.ini
 create mode 100755 ec2.py
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ git add roles helloworld-pb.yml 
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ git commit -m &quot;Adding role for nodejs and helloworld&quot;
[master 16801ef] Adding role for nodejs and helloworld
 19 files changed, 297 insertions(+)
 create mode 100644 helloworld-pb.yml
 create mode 100644 roles/helloworld/README.md
 create mode 100644 roles/helloworld/defaults/main.yml
 create mode 100644 roles/helloworld/files/helloworld.conf
 create mode 100644 roles/helloworld/files/helloworld.js
 create mode 100644 roles/helloworld/handlers/main.yml
 create mode 100644 roles/helloworld/meta/main.yml
 create mode 100644 roles/helloworld/tasks/main.yml
 create mode 100644 roles/helloworld/tests/inventory
 create mode 100644 roles/helloworld/tests/test.yml
 create mode 100644 roles/helloworld/vars/main.yml
 create mode 100644 roles/nodejs/README.md
 create mode 100644 roles/nodejs/defaults/main.yml
 create mode 100644 roles/nodejs/handlers/main.yml
 create mode 100644 roles/nodejs/meta/main.yml
 create mode 100644 roles/nodejs/tasks/main.yml
 create mode 100644 roles/nodejs/tests/inventory
 create mode 100644 roles/nodejs/tests/test.yml
 create mode 100644 roles/nodejs/vars/main.yml
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ git push
Username for &apos;https://github.com&apos;: seanlwatson
Password for &apos;https://seanlwatson@github.com&apos;: 
Counting objects: 40, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (24/24), done.
Writing objects: 100% (40/40), 18.52 KiB | 2.65 MiB/s, done.
Total 40 (delta 3), reused 0 (delta 0)
remote: Resolving deltas: 100% (3/3), done.
To https://github.com/seanlwatson/Ansible
   41231c6..16801ef  master -&gt; master
</pre>

## Canary-testing changes
Open up the `roles/helloworld/files/helloworld.js` and simply change the response in line 11 from `Hello World\n` to `'Hello New World\m'`.


<pre><font color="#AF5F00">      1 </font>var http = require(&quot;http&quot;)
<font color="#AF5F00">      2 </font>
<font color="#AF5F00">      3 </font>http.createServer(function (request, response) {
<font color="#AF5F00">      4 </font>
<font color="#AF5F00">      5 </font>   // Send the HTTP header
<font color="#AF5F00">      6 </font>   // HTTP Status: 200 : OK
<font color="#AF5F00">      7 </font>   // Content Type: text/plain
<font color="#AF5F00">      8 </font>   response.writeHead(200, {&apos;Content-Type&apos;: &apos;text/plain&apos;})
<font color="#AF5F00">      9 </font>
<font color="#AF5F00">     10 </font>   // Send the response body as &quot;Hello World&quot;
<font color="#AF5F00">     11 </font>   response.end(&apos;Hello New World\n&apos;)
<font color="#AF5F00">     12 </font>}).listen(3000)
<font color="#AF5F00">     13 </font>
<font color="#AF5F00">     14 </font>// Console will print the message
<font color="#AF5F00">     15 </font>console.log(&apos;Server running&apos;)
</pre>

Save the file and run the playbook again with the `--check` option. Ansible will detect a change in the application file, which will trigger the the notify of the `restart helloworld` handler.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ansible-playbook helloworld-pb.yml \
&gt; -e target=54.189.226.117 \
&gt; --check

PLAY [54.189.226.117] ********************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

TASK [nodejs : Installing Node.js and NPM (Node Package Mgr.)] ***************************************************************************
<font color="#75507B">[DEPRECATION WARNING]: Invoking &quot;yum&quot; only once while using a loop via squash_actions is deprecated. Instead of using a loop to supply </font>
<font color="#75507B">multiple items and specifying `name: {{ item }}`, please use `name: [u&apos;nodejs&apos;, u&apos;npm&apos;]` and remove the loop. This feature will be </font>
<font color="#75507B">removed in version 2.11. Deprecation warnings can be disabled by setting deprecation_warnings=False in ansible.cfg.</font>
<font color="#4E9A06">ok: [54.189.226.117] =&gt; (item=[u&apos;nodejs&apos;, u&apos;npm&apos;])</font>

TASK [helloworld : Copying the application files] ****************************************************************************************
<font color="#C4A000">changed: [54.189.226.117]</font>

TASK [helloworld : Copying the upstart file] *********************************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

TASK [helloworld : Starting the HelloWorld node service] *********************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

RUNNING HANDLER [helloworld : restart helloworld] ****************************************************************************************
<font color="#C4A000">changed: [54.189.226.117]</font>

PLAY RECAP *******************************************************************************************************************************
<font color="#C4A000">54.189.226.117</font>             : <font color="#4E9A06">ok=6   </font> <font color="#C4A000">changed=2   </font> unreachable=0    failed=0 </pre>

Execute the changes.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ansible-playbook helloworld-pb.yml -e target=54.189.226.117 

PLAY [54.189.226.117] ********************************************************************************************************************

TASK [Gathering Facts] *******************************************************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

TASK [nodejs : Installing Node.js and NPM (Node Package Mgr.)] ***************************************************************************
<font color="#75507B">[DEPRECATION WARNING]: Invoking &quot;yum&quot; only once while using a loop via squash_actions is deprecated. Instead of using a loop to supply </font>
<font color="#75507B">multiple items and specifying `name: {{ item }}`, please use `name: [u&apos;nodejs&apos;, u&apos;npm&apos;]` and remove the loop. This feature will be </font>
<font color="#75507B">removed in version 2.11. Deprecation warnings can be disabled by setting deprecation_warnings=False in ansible.cfg.</font>
<font color="#4E9A06">ok: [54.189.226.117] =&gt; (item=[u&apos;nodejs&apos;, u&apos;npm&apos;])</font>

TASK [helloworld : Copying the application files] ****************************************************************************************
<font color="#C4A000">changed: [54.189.226.117]</font>

TASK [helloworld : Copying the upstart file] *********************************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

TASK [helloworld : Starting the HelloWorld node service] *********************************************************************************
<font color="#4E9A06">ok: [54.189.226.117]</font>

RUNNING HANDLER [helloworld : restart helloworld] ****************************************************************************************
<font color="#C4A000">changed: [54.189.226.117]</font>

PLAY RECAP *******************************************************************************************************************************
<font color="#C4A000">54.189.226.117</font>             : <font color="#4E9A06">ok=6   </font> <font color="#C4A000">changed=2   </font> unreachable=0    failed=0  </pre>

Verify change is in effect.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ curl 54.189.226.117:3000
Hello New World
</pre>

If this simple change was done through CloudFormation template, then CloudFormation would have had to create a new EC2 instance to make it happen when all we wanted to do was to update the code of the application and push it through Ansible on the target host.

Now revert the change locally in Git.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ git checkout roles/helloworld/files/helloworld.js
</pre>

## Running Ansible in pull mode

### Install Git & Ansible on EC2 instance

Install Git from the EPEL `yum` repository and Ansible using `pip`.  Use the `become` command in order to run the commands as root.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ansible &apos;54.189.226.117&apos; \
&gt; --become \
&gt; -m yum -a &apos;name=git enablerepo=epel state=installed&apos;
<font color="#C4A000">54.189.226.117 | CHANGED =&gt; {</font>
<font color="#C4A000">    &quot;ansible_facts&quot;: {</font>
<font color="#C4A000">        &quot;pkg_mgr&quot;: &quot;yum&quot;</font>
<font color="#C4A000">    }, </font>
<font color="#C4A000">    &quot;changed&quot;: true, </font>
<font color="#C4A000">    &quot;msg&quot;: &quot;&quot;, </font>
<font color="#C4A000">    &quot;rc&quot;: 0, </font>
<font color="#C4A000">    &quot;results&quot;: [</font>
<font color="#C4A000">        &quot;Loaded plugins: priorities, update-motd, upgrade-helper\n1054 packages excluded due to repository priority protections\nResolving Dependencies\n--&gt; Running transaction check\n---&gt; Package git.x86_64 0:2.14.5-1.59.amzn1 will be installed\n--&gt; Processing Dependency: perl-Git = 2.14.5-1.59.amzn1 for package: git-2.14.5-1.59.amzn1.x86_64\n--&gt; Processing Dependency: perl(Term::ReadKey) for package: git-2.14.5-1.59.amzn1.x86_64\n--&gt; Processing Dependency: perl(Git::I18N) for package: git-2.14.5-1.59.amzn1.x86_64\n--&gt; Processing Dependency: perl(Git) for package: git-2.14.5-1.59.amzn1.x86_64\n--&gt; Processing Dependency: perl(Error) for package: git-2.14.5-1.59.amzn1.x86_64\n--&gt; Running transaction check\n---&gt; Package perl-Error.noarch 1:0.17020-2.9.amzn1 will be installed\n---&gt; Package perl-Git.noarch 0:2.14.5-1.59.amzn1 will be installed\n---&gt; Package perl-TermReadKey.x86_64 0:2.30-20.9.amzn1 will be installed\n--&gt; Finished Dependency Resolution\n\nDependencies Resolved\n\n================================================================================\n Package              Arch       Version                 Repository        Size\n================================================================================\nInstalling:\n git                  x86_64     2.14.5-1.59.amzn1       amzn-updates      12 M\nInstalling for dependencies:\n perl-Error           noarch     1:0.17020-2.9.amzn1     amzn-main         33 k\n perl-Git             noarch     2.14.5-1.59.amzn1       amzn-updates      69 k\n perl-TermReadKey     x86_64     2.30-20.9.amzn1         amzn-main         33 k\n\nTransaction Summary\n================================================================================\nInstall  1 Package (+3 Dependent packages)\n\nTotal download size: 12 M\nInstalled size: 29 M\nDownloading packages:\n--------------------------------------------------------------------------------\nTotal                                              4.7 MB/s |  12 MB  00:02     \nRunning transaction check\nRunning transaction test\nTransaction test succeeded\nRunning transaction\n  Installing : 1:perl-Error-0.17020-2.9.amzn1.noarch                        1/4 \n  Installing : perl-TermReadKey-2.30-20.9.amzn1.x86_64                      2/4 \n  Installing : perl-Git-2.14.5-1.59.amzn1.noarch                            3/4 \n  Installing : git-2.14.5-1.59.amzn1.x86_64                                 4/4 \n  Verifying  : 1:perl-Error-0.17020-2.9.amzn1.noarch                        1/4 \n  Verifying  : perl-Git-2.14.5-1.59.amzn1.noarch                            2/4 \n  Verifying  : git-2.14.5-1.59.amzn1.x86_64                                 3/4 \n  Verifying  : perl-TermReadKey-2.30-20.9.amzn1.x86_64                      4/4 \n\nInstalled:\n  git.x86_64 0:2.14.5-1.59.amzn1                                                \n\nDependency Installed:\n  perl-Error.noarch 1:0.17020-2.9.amzn1     perl-Git.noarch 0:2.14.5-1.59.amzn1\n  perl-TermReadKey.x86_64 0:2.30-20.9.amzn1\n\nComplete!\n&quot;</font>
<font color="#C4A000">    ]</font>
<font color="#C4A000">}</font>
</pre>


<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ansible &apos;54.189.226.117&apos; \
&gt; --become \
&gt; -m pip -a &apos;name=ansible state=present&apos;
<font color="#C4A000">54.189.226.117 | CHANGED =&gt; {</font>
<font color="#C4A000">    &quot;changed&quot;: true, </font>
<font color="#C4A000">    &quot;cmd&quot;: [</font>
<font color="#C4A000">        &quot;/usr/bin/pip&quot;, </font>
<font color="#C4A000">        &quot;install&quot;, </font>
<font color="#C4A000">        &quot;ansible&quot;</font>
<font color="#C4A000">    ], </font>
<font color="#C4A000">    &quot;name&quot;: [</font>
<font color="#C4A000">        &quot;ansible&quot;</font>
<font color="#C4A000">    ], </font>
<font color="#C4A000">    &quot;requirements&quot;: null, </font>
<font color="#C4A000">    &quot;state&quot;: &quot;present&quot;, </font>
<font color="#C4A000">    &quot;stderr&quot;: &quot;You are using pip version 9.0.3, however version 18.1 is available.\nYou should consider upgrading via the &apos;pip install --upgrade pip&apos; command.\n&quot;, </font>
<font color="#C4A000">    &quot;stderr_lines&quot;: [</font>
<font color="#C4A000">        &quot;You are using pip version 9.0.3, however version 18.1 is available.&quot;, </font>
<font color="#C4A000">        &quot;You should consider upgrading via the &apos;pip install --upgrade pip&apos; command.&quot;</font>
<font color="#C4A000">    ], </font>
<font color="#C4A000">    &quot;stdout&quot;: &quot;Collecting ansible\n  Downloading https://files.pythonhosted.org/packages/ec/ee/1494474b59c6e9cccdfde32da1364b94cdb280ff96b1493deaf4f3ae55f8/ansible-2.7.1.tar.gz (11.7MB)\nRequirement already satisfied: jinja2 in /usr/lib/python2.7/dist-packages (from ansible)\nRequirement already satisfied: PyYAML in /usr/lib64/python2.7/dist-packages (from ansible)\nRequirement already satisfied: paramiko in /usr/lib/python2.7/dist-packages (from ansible)\nCollecting cryptography (from ansible)\n  Downloading https://files.pythonhosted.org/packages/87/e6/915a482dbfef98bbdce6be1e31825f591fc67038d4ee09864c1d2c3db371/cryptography-2.3.1-cp27-cp27mu-manylinux1_x86_64.whl (2.1MB)\nRequirement already satisfied: setuptools in /usr/lib/python2.7/dist-packages (from ansible)\nRequirement already satisfied: markupsafe in /usr/lib64/python2.7/dist-packages (from jinja2-&gt;ansible)\nRequirement already satisfied: pycrypto!=2.4,&gt;=2.1 in /usr/lib64/python2.7/dist-packages (from paramiko-&gt;ansible)\nRequirement already satisfied: ecdsa&gt;=0.11 in /usr/lib/python2.7/dist-packages (from paramiko-&gt;ansible)\nCollecting asn1crypto&gt;=0.21.0 (from cryptography-&gt;ansible)\n  Downloading https://files.pythonhosted.org/packages/ea/cd/35485615f45f30a510576f1a56d1e0a7ad7bd8ab5ed7cdc600ef7cd06222/asn1crypto-0.24.0-py2.py3-none-any.whl (101kB)\nCollecting enum34; python_version &lt; \&quot;3\&quot; (from cryptography-&gt;ansible)\n  Downloading https://files.pythonhosted.org/packages/c5/db/e56e6b4bbac7c4a06de1c50de6fe1ef3810018ae11732a50f15f62c7d050/enum34-1.1.6-py2-none-any.whl\nRequirement already satisfied: six&gt;=1.4.1 in /usr/lib/python2.7/dist-packages (from cryptography-&gt;ansible)\nCollecting cffi!=1.11.3,&gt;=1.7 (from cryptography-&gt;ansible)\n  Downloading https://files.pythonhosted.org/packages/14/dd/3e7a1e1280e7d767bd3fa15791759c91ec19058ebe31217fe66f3e9a8c49/cffi-1.11.5-cp27-cp27mu-manylinux1_x86_64.whl (407kB)\nCollecting idna&gt;=2.1 (from cryptography-&gt;ansible)\n  Downloading https://files.pythonhosted.org/packages/4b/2a/0276479a4b3caeb8a8c1af2f8e4355746a97fab05a372e4a2c6a6b876165/idna-2.7-py2.py3-none-any.whl (58kB)\nCollecting ipaddress; python_version &lt; \&quot;3\&quot; (from cryptography-&gt;ansible)\n  Downloading https://files.pythonhosted.org/packages/fc/d0/7fc3a811e011d4b388be48a0e381db8d990042df54aa4ef4599a31d39853/ipaddress-1.0.22-py2.py3-none-any.whl\nCollecting pycparser (from cffi!=1.11.3,&gt;=1.7-&gt;cryptography-&gt;ansible)\n  Downloading https://files.pythonhosted.org/packages/68/9e/49196946aee219aead1290e00d1e7fdeab8567783e83e1b9ab5585e6206a/pycparser-2.19.tar.gz (158kB)\nInstalling collected packages: asn1crypto, enum34, pycparser, cffi, idna, ipaddress, cryptography, ansible\n  Running setup.py install for pycparser: started\n    Running setup.py install for pycparser: finished with status &apos;done&apos;\n  Running setup.py install for ansible: started\n    Running setup.py install for ansible: finished with status &apos;done&apos;\nSuccessfully installed ansible-2.7.1 asn1crypto-0.24.0 cffi-1.11.5 cryptography-2.3.1 enum34-1.1.6 idna-2.7 ipaddress-1.0.22 pycparser-2.19\n&quot;, </font>
<font color="#C4A000">    &quot;stdout_lines&quot;: [</font>
<font color="#C4A000">        &quot;Collecting ansible&quot;, </font>
<font color="#C4A000">        &quot;  Downloading https://files.pythonhosted.org/packages/ec/ee/1494474b59c6e9cccdfde32da1364b94cdb280ff96b1493deaf4f3ae55f8/ansible-2.7.1.tar.gz (11.7MB)&quot;, </font>
<font color="#C4A000">        &quot;Requirement already satisfied: jinja2 in /usr/lib/python2.7/dist-packages (from ansible)&quot;, </font>
<font color="#C4A000">        &quot;Requirement already satisfied: PyYAML in /usr/lib64/python2.7/dist-packages (from ansible)&quot;, </font>
<font color="#C4A000">        &quot;Requirement already satisfied: paramiko in /usr/lib/python2.7/dist-packages (from ansible)&quot;, </font>
<font color="#C4A000">        &quot;Collecting cryptography (from ansible)&quot;, </font>
<font color="#C4A000">        &quot;  Downloading https://files.pythonhosted.org/packages/87/e6/915a482dbfef98bbdce6be1e31825f591fc67038d4ee09864c1d2c3db371/cryptography-2.3.1-cp27-cp27mu-manylinux1_x86_64.whl (2.1MB)&quot;, </font>
<font color="#C4A000">        &quot;Requirement already satisfied: setuptools in /usr/lib/python2.7/dist-packages (from ansible)&quot;, </font>
<font color="#C4A000">        &quot;Requirement already satisfied: markupsafe in /usr/lib64/python2.7/dist-packages (from jinja2-&gt;ansible)&quot;, </font>
<font color="#C4A000">        &quot;Requirement already satisfied: pycrypto!=2.4,&gt;=2.1 in /usr/lib64/python2.7/dist-packages (from paramiko-&gt;ansible)&quot;, </font>
<font color="#C4A000">        &quot;Requirement already satisfied: ecdsa&gt;=0.11 in /usr/lib/python2.7/dist-packages (from paramiko-&gt;ansible)&quot;, </font>
<font color="#C4A000">        &quot;Collecting asn1crypto&gt;=0.21.0 (from cryptography-&gt;ansible)&quot;, </font>
<font color="#C4A000">        &quot;  Downloading https://files.pythonhosted.org/packages/ea/cd/35485615f45f30a510576f1a56d1e0a7ad7bd8ab5ed7cdc600ef7cd06222/asn1crypto-0.24.0-py2.py3-none-any.whl (101kB)&quot;, </font>
<font color="#C4A000">        &quot;Collecting enum34; python_version &lt; \&quot;3\&quot; (from cryptography-&gt;ansible)&quot;, </font>
<font color="#C4A000">        &quot;  Downloading https://files.pythonhosted.org/packages/c5/db/e56e6b4bbac7c4a06de1c50de6fe1ef3810018ae11732a50f15f62c7d050/enum34-1.1.6-py2-none-any.whl&quot;, </font>
<font color="#C4A000">        &quot;Requirement already satisfied: six&gt;=1.4.1 in /usr/lib/python2.7/dist-packages (from cryptography-&gt;ansible)&quot;, </font>
<font color="#C4A000">        &quot;Collecting cffi!=1.11.3,&gt;=1.7 (from cryptography-&gt;ansible)&quot;, </font>
<font color="#C4A000">        &quot;  Downloading https://files.pythonhosted.org/packages/14/dd/3e7a1e1280e7d767bd3fa15791759c91ec19058ebe31217fe66f3e9a8c49/cffi-1.11.5-cp27-cp27mu-manylinux1_x86_64.whl (407kB)&quot;, </font>
<font color="#C4A000">        &quot;Collecting idna&gt;=2.1 (from cryptography-&gt;ansible)&quot;, </font>
<font color="#C4A000">        &quot;  Downloading https://files.pythonhosted.org/packages/4b/2a/0276479a4b3caeb8a8c1af2f8e4355746a97fab05a372e4a2c6a6b876165/idna-2.7-py2.py3-none-any.whl (58kB)&quot;, </font>
<font color="#C4A000">        &quot;Collecting ipaddress; python_version &lt; \&quot;3\&quot; (from cryptography-&gt;ansible)&quot;, </font>
<font color="#C4A000">        &quot;  Downloading https://files.pythonhosted.org/packages/fc/d0/7fc3a811e011d4b388be48a0e381db8d990042df54aa4ef4599a31d39853/ipaddress-1.0.22-py2.py3-none-any.whl&quot;, </font>
<font color="#C4A000">        &quot;Collecting pycparser (from cffi!=1.11.3,&gt;=1.7-&gt;cryptography-&gt;ansible)&quot;, </font>
<font color="#C4A000">        &quot;  Downloading https://files.pythonhosted.org/packages/68/9e/49196946aee219aead1290e00d1e7fdeab8567783e83e1b9ab5585e6206a/pycparser-2.19.tar.gz (158kB)&quot;, </font>
<font color="#C4A000">        &quot;Installing collected packages: asn1crypto, enum34, pycparser, cffi, idna, ipaddress, cryptography, ansible&quot;, </font>
<font color="#C4A000">        &quot;  Running setup.py install for pycparser: started&quot;, </font>
<font color="#C4A000">        &quot;    Running setup.py install for pycparser: finished with status &apos;done&apos;&quot;, </font>
<font color="#C4A000">        &quot;  Running setup.py install for ansible: started&quot;, </font>
<font color="#C4A000">        &quot;    Running setup.py install for ansible: finished with status &apos;done&apos;&quot;, </font>
<font color="#C4A000">        &quot;Successfully installed ansible-2.7.1 asn1crypto-0.24.0 cffi-1.11.5 cryptography-2.3.1 enum34-1.1.6 idna-2.7 ipaddress-1.0.22 pycparser-2.19&quot;</font>
<font color="#C4A000">    ], </font>
<font color="#C4A000">    &quot;version&quot;: null, </font>
<font color="#C4A000">    &quot;virtualenv&quot;: null</font>
<font color="#C4A000">}</font>
</pre>

### Configure Ansible to run on localhost

Because `ansible-pull` uses Git to clone locally the repository and execute it there is no need for SSH. At the root repository for Ansible create a file 
called `localhost` with the following information. This creates a static inventory and run commands locally as apposed to using SSH when the target is defined as `localhost`.

```
[localhost]
localhost ansible_connection=local
```

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ git add localhost </pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ git commit -m &quot;Adding localhost inventory&quot;
[master 91a63ca] Adding localhost inventory
 1 file changed, 2 insertions(+)
 create mode 100644 localhost
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ git push
Username for &apos;https://github.com&apos;: seanlwatson
Password for &apos;https://seanlwatson@github.com&apos;: 
Counting objects: 3, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 325 bytes | 325.00 KiB/s, done.
Total 3 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1), completed with 1 local object.
To https://github.com/seanlwatson/Ansible
   16801ef..91a63ca  master -&gt; master
</pre>

### Adding a cronjob to the EC2 instance

Add a crontable entry to periodically call `ansible-pull`. The below command tells Ansible to use the cron module targteting the EC2 instance. We provide a name that Ansible will use to track the cronjob over time, telling cron to run the job every 10 minutes and finally the command to execute and its parameters that include the GitHub URL of our branch, the inventory file we just added and a sleep command that makes the command start at a random time between 1-60 sec. after the call starts. This helps spread out the load on the networks and prevent all node services from restarting at the same time.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ansible &apos;54.189.226.117&apos; -m cron -a &apos;name=ansible-pull minute=&quot;*/10&quot; job=&quot;/usr/local/bin/ansible-pull -U https://github.com/seanlwatson/Ansible helloworld-pb.yml -i localhost --sleep 60&quot;&apos;
<font color="#C4A000">54.189.226.117 | CHANGED =&gt; {</font>
<font color="#C4A000">    &quot;changed&quot;: true, </font>
<font color="#C4A000">    &quot;envs&quot;: [], </font>
<font color="#C4A000">    &quot;jobs&quot;: [</font>
<font color="#C4A000">        &quot;ansible-pull&quot;</font>
<font color="#C4A000">    ]</font>
<font color="#C4A000">}</font></pre>

You can log into the EC2 instance and verify the cron job and once you see it execute then check that the change is effective on the server (`Hello New World` should change back to `Hello World`)

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ ssh ec2-user@54.189.226.117 -i ~/.ssh/EffectiveDevOpsAWS.pem 
Last login: Sat Nov  3 00:05:34 2018 from 75-166-145-22.hlrn.qwest.net

       __|  __|_  )
       _|  (     /   Amazon Linux AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-ami/2018.03-release-notes/
14 package(s) needed for security, out of 25 available
Run &quot;sudo yum update&quot; to apply all updates.
[ec2-user@ip-172-31-25-188 ~]$ </pre>

<pre>[ec2-user@ip-172-31-25-188 ~]$ crontab -l
#Ansible: ansible-pull
*/10 * * * * /usr/local/bin/ansible-pull -U https://github.com/seanlwatson/Ansible helloworld-pb.yml -i localhost --sleep 60</pre>

<pre>[ec2-user@ip-172-31-25-188 ~]$ sudo grep -E &quot;CROND.*CMD&quot; /var/log/cron 
...
Nov  3 00:30:01 ip-172-31-25-188 CROND[3851]: (ec2-user) CMD (/usr/local/bin/ansible-pull -U https://github.com/seanlwatson/Ansible helloworld-pb.yml -i localhost --sleep 60)
</pre>

<pre>[ec2-user@ip-172-31-25-188 ~]$ exit
logout
Connection to 54.189.226.117 closed.
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ curl 54.189.226.117:3000
Hello World
</pre>

## Integrating Ansible with CloudFormation

Go to the CloudFormation repository and duplicate the previous Python Troposphere script.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/Ansible</b></font>$ cd ~/CloudFormation/
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ cp helloworld-cf-template.py ansiblebase-cf-template.py
</pre>

Then modify the `ansiblebase-cf-template.py` script with the following changes. Before the declaration of the appliation port, add the application name.

```python
ApplicationName = 'helloworld'
```

Then add some GitHub information such as your username for your GitHub account (or organization name) and the URL to the Ansible repository on GitHub .

```python
GitHubAccount = "seanlwatson"
GitHubAnsibleUrl = "https://github.com/{}/Ansible".format(GitHubAccount)
```

Create one more var that will contain the command line to execute `ansible-pull` command that will configure the host. Use the vars `GitHubAnsibleUrl` and `ApplicationName` that was just previously defined.

```python
AnsiblePullCmd = \
    "/usr/local/bin/ansible-pull -U {} {}.yml -i localhost".format(
        GitHubAnsibleUrl,
        ApplicationName
    )
```

Now delete the previous `ud` var definition and replace it with the below code. This will update the AWS user-data optional parameter to install Git and Ansible, execute the command contained within `AnsiblePullCmd`, and lastly craete a cronjob to re-execute the command every 10 minutes.

```python
ud = Base64(Join('\n', [
    "#!/bin/bash",
    "yum install --enablerepo=epel -y git",
    "pip install ansible",
    AnsiblePullCmd,
    "echo '*/10 * * * * {}' > /etc/cron.d/ansible-pull".format(AnsiblePullCmd)
]))
```

Create the CloudFormation JSON template and test it.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ python ansiblebase-cf-template.py &gt; ansiblebase-cf.template
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ aws cloudformation update-stack --stack-name Ansible --template-body file://ansiblebase-cf.template --parameters ParameterKey=KeyPair,ParameterValue=EffectiveDevOpsAWS
{
    &quot;StackId&quot;: &quot;arn:aws:cloudformation:us-west-2:404297683117:stack/Ansible/4b8e9f00-dca6-11e8-9ab5-503a90a9c435&quot;
}
</pre>

This command will wait until stack status is UPDATE_COMPLETE. It will poll every 30 seconds until a successful state has been reached.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ aws cloudformation wait stack-update-complete \
&gt; --stack-name Ansible
</pre>

Query CloudFormation to get the stack outputs, specifically the Public IP. Then test the server application.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ aws cloudformation describe-stacks \
&gt; --stack-name Ansible \
&gt; --query &apos;Stacks[0].Outputs[0]&apos;
{
    &quot;OutputKey&quot;: &quot;InstancePublicIp&quot;,
    &quot;OutputValue&quot;: &quot;34.221.214.133&quot;,
    &quot;Description&quot;: &quot;Public IP of our instance.&quot;
}
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ curl 34.221.214.133:3000
Hello World
</pre>

Commit the newly created Python Troposphere script to the CloudFormation repository.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ git add ansiblebase-cf-template.py </pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ git commit -m &quot;Adding helloworld Python Troposphere script to create a stack that relies on Ansible to manage the application&quot;
[master 3bdc58b] Adding helloworld Python Troposphere script to create a stack that relies on Ansible to manage the application
 1 file changed, 92 insertions(+)
 create mode 100644 ansiblebase-cf-template.py
</pre>

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ git push
Username for &apos;https://github.com&apos;: seanlwatson
Password for &apos;https://seanlwatson@github.com&apos;: 
Counting objects: 3, done.
Delta compression using up to 2 threads.
Compressing objects: 100% (3/3), done.
Writing objects: 100% (3/3), 1.29 KiB | 1.29 MiB/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To https://github.com/seanlwatson/CloudFormation
   7ed693b..3bdc58b  master -&gt; master
</pre>

We now have a complete solution to efficiently manage our infrastructure using code. While this is a simple example, everything done here is applicable to bigger infrastructure with a greater number of services.

Finally delete the stack.

<pre><font color="#8AE234"><b>sean@vubuntu</b></font>:<font color="#729FCF"><b>~/CloudFormation</b></font>$ aws cloudformation delete-stack --stack-name Ansible
</pre>
