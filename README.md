# Instructions for using this repository

### Prerequisite
1. Install and setup [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)
1. Configure AWS CLI with an IAM user which has sufficient previledges for creating and deleteing EKS, EC2, CodeDeploy, CodeCommit, ECR, CloudWatch and IAM resources.
1. Install and setup [eksctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html).
1. Install and setup [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html).

### Usages Instruction
1. Clone this Repo
    ```
    cd ~
    git clone https://github.com/santosh07bec/EKS_SIMPLE_CICD.git
    cd EKS_SIMPLE_CICD
    ```
1. Define Parameters
    ```
    EKS_VERSION=1.18
    EKS_CLUSTER_NAME=myEksTest
    EC2_KEY_PAIR_NAME=EKSNodesKeyPair
    WORKER_NODE_TYPE=m5.large
    MIN_NODE=1
    MAX_NODE=5
    DESIRED_NODE=3
    REGION=us-east-1
    CICD_CFN_STACK=EKS_CICD
    ```
1. Create EC2 KeyPair
    ```
    aws ec2 create-key-pair --key-name $EC2_KEY_PAIR_NAME --query 'KeyMaterial' --output text --region $REGION  > $EC2_KEY_PAIR_NAME.pem
    chmod 400 $EC2_KEY_PAIR_NAME.pem
    ```
1. Launch EKS Cluster using eksctl command
    ```
    eksctl create cluster --name=$EKS_CLUSTER_NAME --region=$REGION --zones="${REGION}a,${REGION}b,${REGION}c" \
    --version=$EKS_VERSION --node-type=$WORKER_NODE_TYPE --nodes-min=$MIN_NODE --nodes-max=$MAX_NODE --nodes=$DESIRED_NODE \
    --node-volume-size=10 --ssh-access --ssh-public-key $EC2_KEY_PAIR_NAME --asg-access --full-ecr-access --appmesh-access --alb-ingress-access
    ```
1. Create CICD CloudFormation Template
    ```
    aws cloudformation create-stack --stack-name $CICD_CFN_STACK --template-body file://./CloudFormationTemplated/EKS_CICD.yml  \
    --parameters ParameterKey=EksClusterName,ParameterValue=myEksTest ParameterKey=EksDeploymentName,ParameterValue=web-server-depl \
    --capabilities CAPABILITY_NAMED_IAM  --region $REGION
    ``` 
1. Create Kubernetes resources
    ```
    ```
1. Push source code along with buildspec.yml file to CodeCommit to start the CICD deployment
    ```
    ```
1. Cleanup Resources
    ```
    ```
