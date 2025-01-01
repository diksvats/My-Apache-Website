_Host a HTML website using Apache server, push it on Github and integrate it through Jenkins CI/CD Pipeline_

*Install Apache/Httpd server on Linux (using Amazon linux 2)*

1.sudo su
2.sudo yum update
3.sudo yum install httpd -y
4.sudo systemctl start httpd
5.sudo systemctl status httpd

*Create a index.html file at /var/www/html/*

6.cd /var/www/html
7.nano index.html
8.Paste the following html script in index file:

<!DOCTYPE html>
<html>
<head>
<title>My Website</title>
</head>
<body>
<h1>Welcome to my first website</h1>
</body>
</html>

*Install Git and Push index file*

9.yum install git -y
10. git config --global user.name "your name"
11. git config --global user.email "your email@gmail.com"
12. git init
13. git add .
14. git commit -m "initial commit" .
15. git remote add origin <your repository>
16. git push origin master

*Install Jenkins*

17. sudo wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
18. sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
19. sudo amazon-linux-extras enable java-openjdk11
20. sudo yum install java-17-amazon-corretto-devel -y
21. sudo yum upgrade -y
22. sudo yum install jenkins -y
23. export JAVA_HOME=/usr/lib/jvm/java-17-amazon-corretto.x86_64
24. source ~/.bash_profile
25. java -version
26. sudo systemctl start jenkins
27. sudo systemctl status Jenkins

*Setup Jenkins*

28. Enter this in browser:
http://<your-ec2-instance-public-ip>:8080
29. cat <given location>
and enter the result as administrator password.
30. Choose Install suggested plugins on the next window.
31. Enter required details and move to next window
32.Go to Manage Jenkins > Plugins and Install the following Plugins:
-Git server
-Pipeline: REST API
-Generic Webhook Trigger
-GitHub Integration
33. In your Github Repo, add webhook:
Settings > Webhook
Payload URL: http://<your-ec2-instance-public-ip>:8080/github-webhook/
Content Type: application/json
and click on Add Webhook

*Build Pipeline*

34. In Jenkins Dashboard, Create a new Pipeline job
35. Choose GitHub hook trigger for GITScm polling option in Build Triggers.
36. Add the following in Pipeline Script:

pipeline {
    agent any
    
    environment {
        GIT_REPO = ‘<your github repo url>'  // Replace with your repo URL
        BRANCH = 'master'
        HTTPD_DIR = '/var/www/html'  // Default HTTPD web root
        BACKUP_DIR = '/var/www/backup'
        DEPLOY_USER = 'jenkins'
    }
    
    stages {
        stage('Cleanup Workspace') {
            steps {
                cleanWs()
            }
        }
        
        stage('Clone Repository') {
            steps {
                git branch: "${BRANCH}",
                    url: "${GIT_REPO}"
            }
        }
        
        stage('Backup Current Deployment') {
            steps {
                sh '''
                    TIMESTAMP=$(date +%Y%m%d_%H%M%S)
                    sudo mkdir -p ${BACKUP_DIR}
                    if [ -d "${HTTPD_DIR}/*" ]; then
                        sudo cp -r ${HTTPD_DIR} ${BACKUP_DIR}/backup_${TIMESTAMP}
                    fi
                '''
            }
        }
        
        stage('Deploy to Httpd') {
            steps {
                sh '''
                    # Ensure httpd directory exists
                    sudo mkdir -p ${HTTPD_DIR}
                    
                    
                    # Remove old files (optional)
                    sudo rm -rf ${HTTPD_DIR}/*
                    
                    # Copy new files
                    sudo cp -r ./* ${HTTPD_DIR}/
                    
                    # Set proper permissions
                    sudo chown -R apache:apache ${HTTPD_DIR}
                    sudo chmod -R 755 ${HTTPD_DIR}
                    
                    # Reload Apache to apply changes 
                    sudo systemctl reload httpd
                '''
            }
        }
        
        stage('Verify Deployment') {
            steps {
                sh '''
                    # Check if httpd is running
                    if sudo systemctl is-active --quiet httpd; then
                        echo "Httpd is running"
                    else
                        echo "httpd is not running, starting it..."
                        sudo systemctl start httpd
                    fi
                    
                    # Test httpd configuration
                    sudo httpd -t
                    
                    # Reload httpd to apply changes
                    sudo systemctl reload httpd
                '''
            }
        }
    }
    
    post {
        success {
            echo 'Deployment successful!'
        }
        failure {
            sh '''
                # Restore from latest backup if deployment fails
                if [ -d "${BACKUP_DIR}" ]; then
                    LATEST_BACKUP=$(ls -t ${BACKUP_DIR} | head -n1)
                    if [ ! -z "${LATEST_BACKUP}" ]; then
                        sudo cp -r ${BACKUP_DIR}/${LATEST_BACKUP}/* ${HTTPD_DIR}/
                    fi
                fi
            '''
            echo 'Deployment failed! Restored from backup.'
        }
    }
}


37. Apply and Save
38. In ec2 instance, 
sudo visudo
39. Add following at the end of file:
jenkins ALL=(ALL) NOPASSWD: /usr/bin/mkdir -p /var/www/html
jenkins ALL=(ALL) NOPASSWD: /usr/bin/rm -rf /var/www/html/*
jenkins ALL=(ALL) NOPASSWD: /usr/bin/cp -r * /var/www/html/
jenkins ALL=(ALL) NOPASSWD: /bin/chown -R apache apache /var/www/html
jenkins ALL=(ALL) NOPASSWD: /bin/chmod -R 755 /var/www/html
jenkins ALL=(ALL) NOPASSWD: /usr/sbin/nginx
jenkins ALL=(ALL) NOPASSWD: /bin/systemctl start httpd
jenkins ALL=(ALL) NOPASSWD: /bin/systemctl stop httpd
jenkins ALL=(ALL) NOPASSWD: /bin/systemctl reload httpd
jenkins ALL=(ALL) NOPASSWD: /bin/systemctl status httpd
jenkins ALL=(ALL) NOPASSWD: /bin/mkdir
jenkins ALL=(ALL) NOPASSWD: /bin/cp -r
jenkins ALL=(ALL) NOPASSWD: /bin/rm -rf
jenkins ALL=(ALL) NOPASSWD: /bin/chown -R
jenkins ALL=(ALL) NOPASSWD: /bin/chmod -R
jenkins ALL=(ALL) NOPASSWD: ALL

40. Click on build now in jenkins.
