

# Python/HTML web Application CI/CD Pipeline with Jenkins

## Task
Create a CI/CD Pipeline with the following steps:
- Save code artifacts to S3.
- Deploy the application to a web server.
- Jenkins and application servers should be different.
![image](https://github.com/adeelshafi79/python-app-CICD-Jenkins/assets/49460005/daf4d985-1926-454b-bf4b-0ac5b8360f9e)


## Solution

1. **Provision EC2 Instances:**
   - Provisioned two EC2 instances: one for Jenkins server and the other for the web application.

2. **Install Jenkins:**
   - Installed Jenkins on the Jenkins EC2 server using the guide at [Installing Jenkins on Linux](https://www.jenkins.io/doc/book/installing/linux/).

3. **Install Jenkins Plugins:**
   - Installed necessary plugins in Jenkins, including plugins for webhooks, AWS credentials, and ssh-agent for accessing the remote EC2 web server.

4. **Install AWS CLI:**
   - Installed AWS CLI on the Jenkins server EC2 to access the S3 bucket provisioned in AWS.

5. **GitHub Repository Setup:**
   - Created a GitHub repository [python-app-CICD-Jenkins](https://github.com/adeelshafi79/python-app-CICD-Jenkins/tree/master).
   - Configured a webhook in the repository settings.
     - Note: Update the payload URL with the live IP of the Jenkins server.

6. **Generate GitHub Token:**
   - Generated a GitHub token to configure it with Jenkins and saved it securely.

7. **Jenkins Credentials:**
   - Saved credentials in Jenkins for GitHub token, SSH agent, and AWS credentials.

   - GitHub Token:
     - Copied GitHub token as secret text in managed credentials.

   - SSH Agent:
     - Copied the private key of the EC2 web application server in credentials as SSH credentials.

   - AWS Credentials:
     - Copied the access and secret key to credentials under AWS login credentials.

8. **Jenkins Pipeline Configuration:**
   - Created a new pipeline item in Jenkins.
   - Configured Jenkins pipeline for this project.
     - Checked the webhook trigger box.
     - Selected the token credentials and enabled "GitHub hook trigger for GITScm polling."

9. **Pipeline Script:**
   ```groovy
   pipeline {
       agent any

       stages {
           stage('Checkout') {
               steps {
                   git 'https://github.com/adeelshafi79/python-app-CICD-Jenkins.git'
               }
           }

           stage('Deploy to EC2') {
               steps {
                   sshagent(credentials: ['cdd7bb48-5996-4067-a06a-05beaaf49476']) {
                       sh 'ssh -o StrictHostKeyChecking=no ubuntu@54.252.189.146 "mkdir -p /home/ubuntu/app && chmod 777 /home/ubuntu/app"'
                       sh 'scp -o StrictHostKeyChecking=no -r ./* ubuntu@54.252.189.146:/home/ubuntu/app'
                       sh 'ssh -o StrictHostKeyChecking=no ubuntu@54.252.189.146 "sudo apt-get update && sudo apt-get install -y apache2"'
                       sh 'ssh -o StrictHostKeyChecking=no ubuntu@54.252.189.146 "sudo cp /home/ubuntu/app/index.html /var/www/html/"'
                       sh 'ssh -o StrictHostKeyChecking=no ubuntu@54.252.189.146 "sudo chown www-data:www-data /var/www/html/index.html"'
                       sh 'ssh -o StrictHostKeyChecking=no ubuntu@54.252.189.146 "sudo chmod 644 /var/www/html/index.html"'
                       sh 'ssh -o StrictHostKeyChecking=no ubuntu@54.252.189.146 "sudo systemctl restart apache2"'
                   }
               }
           }

           stage('Upload to S3') {
               steps {
                   script {
                       sshagent(credentials: ['cdd7bb48-5996-4067-a06a-05beaaf49476']) {
                           sh 'ssh -o StrictHostKeyChecking=no ubuntu@54.252.189.146 "[ -f /home/ubuntu/app/index.html ] && echo File exists || echo File does not exist"'
                           sh 'scp -o StrictHostKeyChecking=no ubuntu@54.252.189.146:/home/ubuntu/app/index.html .'
                           archiveArtifacts artifacts: 'index.html', allowEmptyArchive: true
                           withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: '6ab16cf1-6ea6-466c-88f5-7fd1a4eb77d5', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                               sh 'aws s3 cp index.html s3://projects3static/'
                           }
                       }
                   }
               }
           }
       }
   }



