AWS CloudFormation Templates
=============================

Collection of CloudFormation templates.

| Name        | Description |
|-------------|-------------|
| VPC         | VPC network similar to AWS Quick Start without NAT |
| NAT         | NAT gateway in VPC network |
| EC2 Simple  | Single EC2 instance in public subnet, EIP attached |
| EC2 Solr    | Solr search server behind ALB with bastion support |
| EC2 Jupyter | Jupyter Lab server |
| Log Policy  | Lambda function to clean logs on CloudWatch Logs |

Development
-----------

### Setup

To use AWS CLI tools

1. Install [*awscli*](http://aws.amazon.com/jp/cli/) from [*PyPI*](https://pypi.python.org/pypi/awscli).
2. Create a credential file `$HOME/.aws/credentials`.
3. Create a local environment file `.envrc` and load it.

`.envrc` may include:

- *AWS_PROFILE*: Profile name in credential file.
- *AWS_DEFAULT_REGION*: AWS region name.
- *AWS_S3_BUCKET*: S3 bucket name to upload template files.
- *AWS_S3_PREFIX* Prefix string in S3 bucket to upload template files.

Sample file looks like:

```bash
export AWS_PROFILE=dev
export AWS_DEFAULT_REGION=ap-northeast-1
export AWS_S3_BUCKET=toolboxbucket
export AWS_S3_PREFIX=aws-cloudformation/design
```

### Workflow

Edit YAML template file and validate it by `validate-template` command in `aws`.

```bash
$ aws cloudformation validate-template --template-body file://files/aws-cfn-vpc.yml
```

Test stack creation and deletion using `create-stack` and `delete-stack` command.
Check "I acknowledge that this this template may create IAM resources" on when creating a stack.
Or "CAPABILITY_IAM" on the command line option if the template requires IAM Role.

```bash
$ aws cloudformation create-stack --stack-name ${STACK_NAME} \
    --template-body file://files/aws-cfn-vpc.yml \
    --cli-input-json file://utils/aws-cfn-vpc-input.json
```

Note that CLI parameters JSON cannot include external file.
"TemplateURL" works if the file is uploaded.

- [#1803 Ability to include external file contents in cli parameters json](https://github.com/aws/aws-cli/issues/1803)

To delete the stack, just pass stack name.

```bash
$ aws cloudformation delete-stack --stack-name ${STACK_NAME}
```

To show the list of stacks, use ``describe-stacks`` command.

```bash
$ aws cloudformation describe-stacks
```

Templates
---------

### VPC & NAT

Creates VPC network similar to AWS Quick Start without NAT gateway.

```bash
$ aws cloudformation create-stack --stack-name TestVPC \
    --template-body file://`pwd`/files/aws-cfn-vpc.yml \
    --parameters 'ParameterKey=AvailabilityZones,ParameterValue="ap-northeast-1d,ap-northeast-1c"'
```

Creates NAT gateway in VPC network.

```bash
$ aws cloudformation create-stack --stack-name TestNAT \
    --template-body file://`pwd`/files/aws-cfn-nat.yml \
    --parameters 'ParameterKey=VPCStackName,ParameterValue=TestVPC'
```

### Simple Instance

`aws-cfn-ec2-simple.yml` is a template for simple command line work.
An instance is created in public subnet in VPC network.

EC2 instance includes:

* Python 3.6 (`pipenv` and `ansible`)
* Go
* Git, tmux, zsh, jq
* Docker

Usage:

```bash
$ aws cloudformation create-stack \
    --stack-name SimpleConsole \
    --template-body file://`pwd`/files/aws-cfn-ec2-simple.yml \
    --parameters \
        ParameterKey=VPCStackName,ParameterValue=TestVPC \
        ParameterKey=KeyPairName,ParameterValue=${EC2_KEYNAME} \
        ParameterKey=RemoteAccessCIDR1,ParameterValue=${REMOTE_CIDR}
```

This template accepts four remote CIDR addresses on parameters.

To confirm Amazon Linux AMI ID, visit [Amazon Linux AMI](http://aws.amazon.com/jp/amazon-linux-ami/)
and [Amazon Linux AMI instance type matrix](https://aws.amazon.com/jp/amazon-linux-ami/instance-type-matrix/) pages.

### Solr

`aws-cfn-ec2-solr.yml` is a template of Solr server with Application Load Balancer.
This template also create a bastion host in public subnet for manual operation through SSH.

* Open JDK 1.8
* Solr 7.4.0 (*SolrVersion* parameter is used)

Usage:

```bash
$ aws cloudformation create-stack \
    --stack-name Solr \
    --template-body file://`pwd`/files/aws-cfn-ec2-solr.yml \
    --parameters \
        ParameterKey=VPCStackName,ParameterValue=TestVPC \
        ParameterKey=KeyPairName,ParameterValue=${EC2_KEYNAME} \
        ParameterKey=RemoteAccessCIDR1,ParameterValue=${SSH_ACCESS_CIDR} \
        ParameterKey=WebAccessCIDR1,ParameterValue=${WEB_ACCESS_CIDR}
```

Bootstrapping feature prepares two cores; *core1* and *techproducts*.
*core1* is copied from *_default* config set, while *techproducts* is copied from *sample_techproducts_configs*.
Both cores require initialization requests from Core Admin in dashboard, whose URL is shown in outputs of the stack.

For more information about Solr, see official tutorial.

- [Solr Tutorial](https://lucene.apache.org/solr/guide/7_4/solr-tutorial.html)

### Jupyter Lab

`aws-cfn-ec2-jupyter.yml` is a template of Jupyter Lab server.
Since this template depends on VPC stack, you have to specify *VPCStackName*.
It also requires *RemoteAccessCIDR1* and *WebAccessCIDR1* parameters because a server host in this stack accepts ssh and web access on port number 22 and 8888 respectively.
*KeyPairName* is used for ssh connection, and *NotebookProfile* is attached on EC2 instance.
The instance profile expects CloudWatch permissions of "logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents", and "logs:DescribeLogStreams" to deliver server log file.

The server includes Python libraries defined by `Pipfile`. It will take 5-10 minutes to install them.

- *jupyterlab*
- *numpy*
- *pandas*
- *scipy*
- *scikit-learn*
- *statsmodels*
- *matplotlib*
- *seaborn*
- *bokeh*
- *tensorflow*
- *pyyaml*
- *requests*
- *boto3*

Jupyter server requires authentication token or password to log-in.
Once stack is created, you can see the token in log event on CloudWatch console.
The authentication token is shown at start up, and server log is delivered to CloudWatch Logs by *awslogs*.
You can specify log group by *CloudWatchLogsGroupDefault* template parameter, while default log group is `/var/log/jupyterlab.log`.
And, log stream name is instance ID.

Parameters:

* **VPCStackName**: VPC network stack name.
  It must export "VPCID" and subnet ID specified by VPCSubnetID parameter.
* **VPCSubnetID**: Exported variable name of Subnet ID in VPC stack.
* **LinuxAMIOS**: The Linux distribution for the AMI to be used for the notebook instance.
* **NotebookInstanceType**: Amazon EC2 instance type for the notebook instance.
* **NotebookDiskSize**: Disk size by GB.
* **NotebookProfile**: Instance profile to attach the notebook instance. Its role have to be allowed to put CloudWatch Logs.
* **KeyPairName**: Key pair name. If you do not have one in this region, please create it before continuing.
* **CloudWatchLogsGroup**: Group name of CloudWatch Logs to deliver Jupyter Lab server log.
* **RemoteAccessCIDR1**: Allowed CIDR block for external SSH access to the notebook instance.
* **WebAccessCIDR1**: Allowed CIDR block for web access.
* **RemoteAccessCIDR2**: (OPTIONAL) Allowed CIDR block for external SSH access to the notebook instance.
* **RemoteAccessCIDR3**: (OPTIONAL) Allowed CIDR block for external SSH access to the notebook instance.
* **RemoteAccessCIDR4**: (OPTIONAL) Allowed CIDR block for external SSH access to the notebook instance.
* **WebAccessCIDR2**: (OPTIONAL) Allowed CIDR block for web access.
* **WebAccessCIDR3**: (OPTIONAL) Allowed CIDR block for web access.
* **WebAccessCIDR4**: (OPTIONAL) Allowed CIDR block for web access.

Outputs:

* **NotebookEIP**: Public IP address of notebook.
* **SSHAccess**: SSH access target string with user name and IP address.
* **WebAccess**: Web access URL with protocol and port number.
* **JupyterAccessToken**: Jupyter access token for first login. Key is instance ID, and value is access token.

### Log Policy

`aws-cfn-cwlog-policy.yml` is a template of Lambda functions to keep CloudWatch Logs clean.
Lambda functions are invoked by CloudWatch Events periodically.

This template has two functions:

1. Put retention policy on Log Group without "retentionInDays" property.
2. Remove empty log groups and log streams.

To create stack, it also requires permission to create IAM Roles for Lambda functions.
Prepare deployment role so that CloudFormation can deploy serverless stack, and pass role to it while stack creation.

Parameters:

* **LogGroupPrefix**:
    Prefix string of target log group names.
    (Default: `/aws/lambda/`)
* **RetentionInDays**:
    Days to set retention policy.
    (Default: `7`)
* **ScheduleInDays**:
    Days to invoke Lambda functions in the Events Rule ScheduleExpression format.
    (Default: `2 days`)

Outputs:

* **PutRetentionPolicy**:
    Name of Lambda function to put retention policy.
* **RemoveEmptyLogs**:
    Name of Lambda function to remove empty log groups and log streams.
