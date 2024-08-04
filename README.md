# Deploying a Flask API

This is the project starter repo for the course Server Deployment, Containerization, and Testing.

In this project you will containerize and deploy a Flask API to a Kubernetes cluster using Docker, AWS EKS, CodePipeline, and CodeBuild.

The Flask app that will be used for this project consists of a simple API with three endpoints:

- `GET '/'`: This is a simple health check, which returns the response 'Healthy'.
- `POST '/auth'`: This takes a email and password as json arguments and returns a JWT based on a custom secret.
- `GET '/contents'`: This requires a valid JWT, and returns the un-encrpyted contents of that token.

The app relies on a secret set as the environment variable `JWT_SECRET` to produce a JWT. The built-in Flask server is adequate for local development, but not production, so you will be using the production-ready [Gunicorn](https://gunicorn.org/) server when deploying the app.



## Prerequisites

* Docker Desktop - Installation instructions for all OSes can be found <a href="https://docs.docker.com/install/" target="_blank">here</a>.
* Git: <a href="https://git-scm.com/downloads" target="_blank">Download and install Git</a> for your system.
* Code editor: You can <a href="https://code.visualstudio.com/download" target="_blank">download and install VS code</a> here.
* AWS Account
* Python version between 3.7 and 3.9. Check the current version using:
```bash
#  Mac/Linux/Windows
python --version
```
You can download a specific release version from <a href="https://www.python.org/downloads/" target="_blank">here</a>.

* Python package manager - PIP 19.x or higher. PIP is already installed in Python 3 >=3.4 downloaded from python.org . However, you can upgrade to a specific version, say 20.2.3, using the command:
```bash
#  Mac/Linux/Windows Check the current version
pip --version
# Mac/Linux
pip install --upgrade pip==20.2.3
# Windows
python -m pip install --upgrade pip==20.2.3
```
* Terminal
   * Mac/Linux users can use the default terminal.
   * Windows users can use either the GitBash terminal or WSL.
* Command line utilities:
  * AWS CLI installed and configured using the `aws configure` command. Another important configuration is the region. Do not use the us-east-1 because the cluster creation may fails mostly in us-east-1. Let's change the default region to:
  ```bash
  aws configure set region us-east-2
  ```
  Ensure to create all your resources in a single region.
  * EKSCTL installed in your system. Follow the instructions [available here](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html#installing-eksctl) or <a href="https://eksctl.io/introduction/#installation" target="_blank">here</a> to download and install `eksctl` utility.
  * The KUBECTL installed in your system. Installation instructions for kubectl can be found <a href="https://kubernetes.io/docs/tasks/tools/install-kubectl/" target="_blank">here</a>.


## Initial setup

1. Fork the <a href="https://github.com/udacity/cd0157-Server-Deployment-and-Containerization" target="_blank">Server and Deployment Containerization Github repo</a> to your Github account.
1. Locally clone your forked version to begin working on the project.
```bash
git clone https://github.com/SudKul/cd0157-Server-Deployment-and-Containerization.git
cd cd0157-Server-Deployment-and-Containerization/
```
1. These are the files relevant for the current project:
```bash
.
├── Dockerfile
├── README.md
├── aws-auth-patch.yml #ToDo
├── buildspec.yml      #ToDo
├── ci-cd-codepipeline.cfn.yml #ToDo
├── iam-role-policy.json  #ToDo
├── main.py
├── requirements.txt
├── simple_jwt_api.yml
├── test_main.py  #ToDo
└── trust.json     #ToDo
```

## Test the python app locally
Set the environment variables
```bash
$ export JWT_SECRET='myjwtsecret'
$ export LOG_LEVEL=DEBUG
```

Run the API:
```bash
python main.py
```


```bash
export TOKEN=`curl --data '{"email":"abc@xyz.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:8080/auth  | jq -r '.token'`
```
Explain the bash command:

**Send POST request for authentication**
```
curl --data '{"email":"abc@xyz.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:8080/auth
```

**Extract the token from the response**
```
curl --data '{"email":"abc@xyz.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:8080/auth | jq -r '.token'
```
**Store the token in an environment variable**
```
export TOKEN=$(curl --data '{"email":"abc@xyz.com","password":"mypwd"}' --header "Content-Type: application/json" -X POST localhost:8080/auth | jq -r '.token')
```

### Gunicorn
Gunicorn is recommended for production environments: Compared with Flask's own development server, Gunicorn provides higher performance and stability, and is suitable for high-concurrency processing in production environments.
Gunicorn is a program that runs on a server operating system.
Its role is to act as a middleman between the client and the Python web application, processing HTTP requests and forwarding them to the web application, and then returning the web application's response to the client.



## Project Steps

Completing the project involves several steps:

### 1. Write a Dockerfile for a simple Flask API
### 2. Build and test the container locally
### 3. Create an EKS cluster
```bash
eksctl create cluster --name simple-jwt-api --nodes=2 --instance-types=t2.medium --region=us-east-2
```

get the aws account ID:
```bash
aws sts get-caller-identity --query Account --output text
```

Create a role:'UdacityFlaskDeployCBKubectlRole', using the `trust.json` trust relationship:
```bash
aws iam create-role --role-name UdacityFlaskDeployCBKubectlRole --assume-role-policy-document file://trust.json --output text --query 'Role.Arn'
```
This AWS CLI command creates an IAM role with the name 'UdacityFlaskDeployCBKubectlRole':

* role-name UdacityFlaskDeployCBKubectlRole: Specifies the name of the IAM role.
* assume-role-policy-document file://trust.json: Uses the policy document trust.json to define who
can assume this role.
* output text: Formats the output as plain text.
* query 'Role.Arn': Extracts and displays only the ARN (Amazon Resource Name) of the created role.

Since we have to add the new created IAM rule into the EKS cluster's aws-auth configMap.

The main purpose of the aws-auth ConfigMap is to manage RBAC (role-based access control) in the EKS cluster. It allows you to associate IAM roles and users with permissions for the Kubernetes cluster, thereby controlling who can access and manage resources in the EKS cluster.

**Fetch** -  get the current configmap and save it to a file:
```
kubectl get -n kube-system configmap/aws-auth -o yaml > /tmp/aws-auth-patch.yml
```
The file will be created at `/System/Volumes/Data/private/tmp/aws-auth-patch.yml` path and then we **edit** the `.yml` file.

Add the following group in the data → mapRoles section of this file. YAML is indentation-sensitive, therefore refer to the snapshot below for a correct indentation:

```bash
   - groups:
       - system:masters
     rolearn: arn:aws:iam::<ACCOUNT_ID>:role/UdacityFlaskDeployCBKubectlRole
     username: build
```

**Update** the cluster's configmap:
```bash
kubectl patch configmap/aws-auth -n kube-system --patch "$(cat /tmp/aws-auth-patch.yml)"
```

### 4. Store a secret using AWS Parameter Store
We need a way to pass your text secret to the app in kubernetes securly. You will be using AWS Parameter Store(opens in a new tab) to do this.
Run this command from the project home directory to put secret into AWS Parameter Store
```bash
aws ssm put-parameter --name JWT_SECRET --overwrite --value "myjwtsecret" --type SecureString
```
**Verify**
```bash
aws ssm get-parameter --name JWT_SECRET
```
Once you submit your project and receive the reviews, you can consider deleting the variable from parameter-store using:
```bash
aws ssm delete-parameter --name JWT_SECRET
```
### 5. Create a CodePipeline pipeline triggered by GitHub checkins
### 6. Create a CodeBuild stage which will build, test, and deploy your code

For more detail about each of these steps, see the project lesson.
