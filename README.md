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
    EKS_VERSION=1.17
    EKS_CLUSTER_NAME=myEksTest
    EC2_KEY_PAIR_NAME=EKSNodesKeyPair
    WORKER_NODE_TYPE=m5.large
    MIN_NODE=1
    MAX_NODE=5
    DESIRED_NODE=3
    REGION=us-east-1
    CICD_CFN_STACK=eks-cicd
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
    --parameters ParameterKey=EksClusterName,ParameterValue=$EKS_CLUSTER_NAME ParameterKey=EksDeploymentName,ParameterValue=web-server-depl \
    --capabilities CAPABILITY_NAMED_IAM  --region $REGION
    ``` 
1. Create Kubernetes resources
    ```
    The YAML file for different kubernetes objects are defined in "yaml" directory.
    You can cd to "yaml" directory and use below kubernetes command to create the respurces:
    
    kubectl apply -f <resource_definition_file_name.yaml>
    ```
    1. 1.0_web-pod.yaml <- This file has pod definition and it can be used to create a standalong pod.
    1. 1.5_mysql_secrets.yaml <- This file has definition for kubernetes secrets. This secrets will be used while creating Deployment.
    1. 2.0_web-server-depl.yaml <- This file has definition for deployment which creates a deployment for the demo application. It usages the secrets defined above
    1. 2.25_configmap_colour.yaml <- This file has config map definition. This is used while creating the next deployment.
    1. 2.5_web-server-depl-with-colour.yaml <- This file has definition for deployment which usages the config map defind above.
    1. 3.0_web-server-svc.yaml <- This file has definition for ClusterIP service.
    1. 4.0_web-server-svc-clb.yaml <- This file has definition for service of type LoadBalancer.
    1. 4.5_web-server-svc-nlb.yaml <- This file has definition for service of type LoadBalancer but it creates NLB instead of an CLB.
    1. 5.0_redis.yaml <- This file has definition for redis deployment and redis service.
    1. 6.0_mysql.yaml <- This file has definition for mysql deployment and mysql service.
    
    Once all the above resource are created you can access the demo application using the URL of the LoadBalancer Service.
    
1. Prepare EKS Cluster to authorised CodeBuild to deploy new version of application.
    ```
    AccountID=$(aws sts get-caller-identity --output text | awk '{print $1}')
    RoleName=$(aws cloudformation describe-stack-resource --stack-name $CICD_CFN_STACK --logical-resource-id EksCodeBuildKubectlRole --query 'StackResourceDetail.PhysicalResourceId' --output text  --region $REGION)
    RoleARN="arn:aws:iam::$AccountID:role/$RoleName"
    ROLE="    - groups:\n      - system:masters\n      rolearn: $RoleARN\n      username: build"
    kubectl get -n kube-system configmap/aws-auth -o yaml | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > /tmp/aws-auth-patch.yml
    kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
    ```
1. Clone CodeCommit Repo, add code to it and push source code along with buildspec.yml file to CodeCommit to start the CICD deployment.
    ```
    CodeRepo=$(aws codecommit list-repositories --output text --region $REGION| grep $(aws cloudformation describe-stack-resource --stack-name $CICD_CFN_STACK --logical-resource-id MyCodeRepo --query 'StackResourceDetail.PhysicalResourceId' --output text  --region $REGION) | awk '{print $3}')
    RepoCloneURL="https://git-codecommit.${REGION}.amazonaws.com/v1/repos/$CodeRepo"
    git clone $RepoCloneURL
    cd $CodeRepo
    cp ../src/* ./
    sed -i '/ENV/d' Dockerfile
    echo 'ENV COLOUR=Blue' >> Dockerfile
    git add . && git commit -m "Modified Docker file" && git push
    ```
    Note: To push code to CodeCommit, you would need CodeCommit credential which you can generate from IAM console of your account for your IAM user. Instructions are available in: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_ssh-keys.html#git-credentials-code-commit
    
    After pushing the code to ComeCommit, the codeBuild would have triggered. You can view the same in AWS console using below URL:
    https://console.aws.amazon.com/codesuite/codebuild/projects
    
    Kubernetes deployment status and history can be checked using below commands:
    ```
    kubectl rollout status deployment web-server-depl
    kubectl rollout history deployment web-server-depl
    ```
    
    Once the CodeBuild stage is complete, refresh the browser tab, and notice the background of the page. Did it change to Blue?
1. Cleanup Resources
    ```
    ECR_REPO=$(aws cloudformation describe-stack-resource --stack-name $CICD_CFN_STACK --logical-resource-id DockerImageRepo --query 'StackResourceDetail.PhysicalResourceId' --output text  --region $REGION)
    Images=$(aws ecr list-images --repository-name $ECR_REPO --region $REGION --output text | awk '{printf "imageTag=%s ", $NF}')
    aws ecr batch-delete-image --repository-name $ECR_REPO --image-ids $Images --region $REGION
    aws cloudformation delete-stack --stack-name $CICD_CFN_STACK --region $REGION
    eksctl delete cluster $EKS_CLUSTER_NAME
    ```
