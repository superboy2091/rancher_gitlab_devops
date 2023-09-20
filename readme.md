# GitLab pipeline for deployment to Kubernetes cluster on Rancher at AWS.

## Prepare Environments
- Install Git
    
    You can download from [Git site](https://git-scm.com/download/win)
    You can download latest windows version(v2.41.0) from this [link](https://github.com/git-for-windows/git/releases/download/v2.41.0.windows.1/Git-2.41.0-64-bit.exe)

    After installation you can open terminal and test git version

      git version

    And you need to config username and email

      git config --global user.email = <Your Email Address>
      git config --global user.name = <Your Name>

- Install Docker
    
    You can download docker from [docker site](https://www.docker.com/)

    After installation you can check version

      docker version

- Install Terraform
    
    To install terraform in Windows using Powershell, follow below steps:
    
    - First, you have to install chocolately. Open powershell as administrator and run following code.

          Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1')) 
    
    - Second, install terraform using choco installer.

          choco install terraform
        
- Create Rancher server at Amazon Web Service

    You can check steps from this [link](https://ranchermanager.docs.rancher.com/getting-started/quick-start-guides/deploy-rancher-manager/aws)
    
    1. Clone [Rancher Quickstart](https://github.com/rancher/quickstart) to a folder using `git clone https://github.com/rancher/quickstart`.

    2. Go into the AWS folder containing the Terraform files by executing `cd quickstart/rancher/aws`.

    3. Rename the `terraform.tfvars.example` file to `terraform.tfvars`.

    4. Edit `terraform.tfvars` and customize the following variables:

        - `aws_access_key` - Amazon AWS Access Key
        - `aws_secret_key` - Amazon AWS Secret Key
        - `rancher_server_admin_password` - Admin password for created Rancher server (minimum 12 characters)

    5. **Optional:** Modify optional variables within `terraform.tfvars`. See the [Quickstart Readme](https://github.com/rancher/quickstart) and the [AWS Quickstart Readme](https://github.com/rancher/quickstart/tree/master/rancher/aws) for more information.
    Suggestions include:

    - `aws_region` - Amazon AWS region, choose the closest instead of the default (`us-east-1`)
    - `prefix` - Prefix for all created resources
    - `instance_type` - EC2 instance size used, minimum is `t3a.medium` but `t3a.large` or `t3a.xlarge` could be used if within budget
    - `add_windows_node` - If true, an additional Windows worker node is added to the workload cluster

    6. Run `terraform init`.

    7. To initiate the creation of the environment, run `terraform apply --auto-approve`. Then wait for output similar to the following:

        ```
        Apply complete! Resources: 16 added, 0 changed, 0 destroyed.

        Outputs:

        rancher_node_ip = xx.xx.xx.xx
        rancher_server_url = https://rancher.xx.xx.xx.xx.sslip.io
        workload_node_ip = yy.yy.yy.yy
        ```

    8. Paste the `rancher_server_url` from the output above into the browser. Log in when prompted (default username is `admin`, use the password set in `rancher_server_admin_password`).
    9. ssh to the Rancher Server using the `id_rsa` key generated in `quickstart/rancher/aws`.

## Dockerfile ##

Create `Dockerfile` in the project

    FROM python:3.8-slim-buster

    WORKDIR /app

    COPY requirements.txt requirements.txt
    RUN pip install -r requirements.txt
    RUN pip install flask

    COPY . .

    CMD ["flask", "run"]

Create `deploy.manifest.yaml` for kubernetes cluster.

    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: flask-app
    spec:
    replicas: 1
    selector:
        matchLabels:
        app: flask-app
    template:
        metadata:
        labels:
            app: flask-app
        spec:
        containers:
            - name: flask-app
            image: merianda2016/riskmatrix:tagname
            ports:
                - containerPort: 5000
    ---
    apiVersion: v1
    kind: Service
    metadata:
    name: flask-app
    spec:
    selector:
        app: flask-app
    ports:
        - name: http
        port: 80
        targetPort: 5000
    type: LoadBalancer

## GitLab Pipeline ##


Create `.gitlab-ci.yml` in the project

    stages:
    - build
    - deploy

    build:
    stage: build
    image: docker:latest
    services:
        - docker:dind
    script:
        - docker build -t <Your Docker Hub Username>/<Your Docker Hub Repository Name>:tagname .
        - docker login -u <Your Docker Hub Username> -p <Your Docker Hub Password>
        - docker push <Your Docker Hub Username>/<Your Docker Hub Repository Name>:tagname

    deploy:
    stage: deploy
    image: 
        name: josenobile/rancherclikubectl
        entrypoint: [""]
    script:
        - echo "trying rancher login ..."
        - rancher login <Rancher Server Url> -t <Rancher Server token> --skip-verify --context <Context>
        - rancher kubectl apply -f deploy.manifest.yaml

## Push project to GitLab"

    git init
    git add .
    git commit -m "First Commit"
    git remote add origin <Your gitlab repository link>
    git branch -M main
    git push -u origin main

You can check pipeline working in Gitlab's Project detail page's sidebar: `Build / Pipelines`
    
### Date: **Saturday, June 10, 2023**
