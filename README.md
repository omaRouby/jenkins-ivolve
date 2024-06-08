# README.md

## Jenkins Pipeline to Build and Deploy a Java Application to OpenShift using Shared Library

This README provides a detailed guide on setting up a Jenkins pipeline to build a Java application image, push it to DockerHub, and deploy it to an OpenShift cluster using a shared Jenkins library. The Jenkins agent will run on an Amazon EC2 instance.

### Prerequisites

1. **Jenkins Server**: Jenkins should be installed and running.
2. **Amazon EC2 Instance**: An EC2 instance running a Jenkins agent.
3. **DockerHub Account**: A DockerHub account to push the Docker image.
4. **OpenShift Cluster**: An OpenShift cluster to deploy the application.
5. **Java Application**: A Java application with a Dockerfile.
6. **Shared Jenkins Library**: A Jenkins shared library containing reusable pipeline code.

### Steps

#### Step 1: Set Up Jenkins Agent on Amazon EC2

1. **Launch EC2 Instance**:
    - Launch an EC2 instance with a suitable AMI (e.g., Ubuntu).
    - Ensure the instance has appropriate security group settings to allow Jenkins master to connect to it.
![](https://github.com/omaRouby/jenkins-ivolve/blob/main/pictures/ec2-instance.png)
2. **Install Docker on EC2**:
    ```sh
    sudo apt-get update
    sudo apt-get install -y docker.io
    sudo usermod -aG docker ubuntu
    ```
![](https://github.com/omaRouby/jenkins-ivolve/blob/main/pictures/docker.png)
3. **Install Java on EC2**:
    ```sh
    sudo apt-get install -y openjdk-11-jdk
    ```
![](https://github.com/omaRouby/jenkins-ivolve/blob/main/pictures/open-jdk.png)
4. **Install OC CLI ov EC2**
![](https://github.com/omaRouby/jenkins-ivolve/blob/main/pictures/oc-cli.png)

#### Step 2: Configure DockerHub Credentials in Jenkins

1. **Add DockerHub Credentials**:
    - Go to Jenkins Dashboard > Manage Jenkins > Manage Credentials.
    - Add a new credential with your DockerHub username and password.
![](https://github.com/omaRouby/jenkins-ivolve/blob/main/pictures/dockerhub-cred.png)
#### Step 3: Add OpenShift Service Account Token to Jenkins

1. **Obtain OpenShift Service Account Token**:
    - Retrieve the token from OpenShift for the service account that Jenkins will use.

    ```sh
    oc sa get-token <service-account-name>
    ```

2. **Add OpenShift Token to Jenkins**:
    - Go to Jenkins Dashboard > Manage Jenkins > Manage Credentials.
    - Add a new secret text credential with the OpenShift service account token.
![](https://github.com/omaRouby/jenkins-ivolve/blob/main/pictures/sa-token.png)
#### Step 4: Add EC2 SSH Key to Jenkins

1. **Add EC2 SSH Key**:
    - Go to Jenkins Dashboard > Manage Jenkins > Manage Credentials.
    - Add a new SSH username with private key credential. Use the private key associated with your EC2 instance.
![](https://github.com/omaRouby/jenkins-ivolve/blob/main/pictures/slave-sshkey.png)
#### Step 5: Create Jenkins Shared Library

1. **Set Up Shared Library Repository**:
    - Create a new Git repository for your shared library if you don't have one.

2. **Structure Your Shared Library**:
    - Follow the standard structure for a Jenkins shared library. Create the necessary directories and files.

3. **Example Shared Library**:
    
- Here is the link to the shared library i used in this project https://github.com/omaRouby/shared-lib-oc.git
#### Step 6: Configure Shared Library in Jenkins

1. **Add Shared Library**:
    - Go to Jenkins Dashboard > Manage Jenkins > Configure System.
    - Scroll down to the "Global Pipeline Libraries" section.
    - Add a new library with the name of your shared library, the default version ( `main`), and the repository URL.

#### Step 7: Create Jenkins Pipeline Job

1. **Create a New Pipeline Job**:
    - Go to Jenkins Dashboard > New Item.
    - Enter a name for your job and select "Pipeline".

2. **Configure Pipeline Script**:
    - In the pipeline configuration, select "Pipeline script from SCM".
    - Choose "Git" and enter the repository URL.
    - The jenkins file in the Repository
    
 ```groovy
   @Library('shared-library') _

pipeline {
    agent { 
        label 'ec2-slave'
    }
    environment {
        DOCKER_IMAGE_NAME = "omarrouby/gradle-app"
        DOCKERHUB_CREDENTIALS_ID = "dockerhub"
        OPENSHIFT_PROJECT = "omarrouby"
        OPENSHIFT_CREDENTIALS_ID = "sa-token"
        CLUSTER_URL = "https://api.ocp-training.ivolve-test.com:6443"
    }

    stages {
        stage('Build and Push Docker Image') {
            steps {
                script {
                    def fullImageName = "${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                    buildAndPushImage(fullImageName, DOCKERHUB_CREDENTIALS_ID)
                }
            }
        }

        stage('Edit new image in deployment.yaml file') {
            steps {
                script {
                    dir('openshift') {
                        def fullImageName = "${DOCKER_IMAGE_NAME}:${env.BUILD_NUMBER}"
                        editNewImage(fullImageName)
                    }
                }
            }
        }

        stage('Deploy to OpenShift') {
            steps {
                script {
                    deployToOpenShift(OPENSHIFT_CREDENTIALS_ID, OPENSHIFT_PROJECT, CLUSTER_URL)
                }
            }
        }
    }

    post {
        success {
            echo "${JOB_NAME}-${BUILD_NUMBER} pipeline succeeded"
        }
        failure {
            echo "${JOB_NAME}-${BUILD_NUMBER} pipeline failed"
        }
    }
}
  ```


#### Step 8: Run the Pipeline

1. **Trigger the Pipeline**:
    - Go to the pipeline job and click "Build Now".
    - Monitor the pipeline execution in the console output.
![](https://github.com/omaRouby/jenkins-ivolve/blob/main/pictures/console-output.png)
### Detailed Explanation

1. **Jenkins Agent on EC2**:
    - The Jenkins agent runs on an EC2 instance with Docker and Java installed to handle the build and deployment processes.

2. **DockerHub**:
    - The pipeline logs into DockerHub using stored credentials, builds a Docker image for the Java application, and pushes it to the DockerHub repository.

3. **OpenShift Deployment**:
    - The pipeline logs into the OpenShift cluster using stored credentials, imports the Docker image from DockerHub, and triggers a deployment.

## Step 10: See the output 
- The pipeline succeeded 
![](https://github.com/omaRouby/jenkins-ivolve/blob/main/pictures/pipeline%20success.png)
- Check The application using the Route URL created
![](https://github.com/omaRouby/jenkins-ivolve/blob/main/pictures/route.png)
![](https://github.com/omaRouby/jenkins-ivolve/blob/main/pictures/route-page.png)
