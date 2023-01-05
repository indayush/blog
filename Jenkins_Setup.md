## Jenkins Server Setup - 

Create an Ubuntu EC2 with correct IAM role


Setting up the ubuntu user with basic configs

***```root```*** user
```bash
sudo su -
sudo apt update
sudo apt upgrade -y

# Setting IST timezone
sudo timedatectl set-timezone Asia/Kolkata
date

sudo apt install -y software-properties-common zip unzip git curl gnupg2 htop awscli jq

# Add deployer user and give password
# sudo adduser deployer

username=deployer
# Enter deployer password here -
password=deployer_pwd

adduser --gecos "" --disabled-password $username
chpasswd <<<"$username:$password"


echo 'deployer ALL=(ALL) ALL' >> /etc/sudoers
echo 'deployer ALL=(ALL:ALL)ALL,NOPASSWD:/bin/sed,/usr/bin/tee' >> /etc/sudoers

# Changing the SSH port to 722
sudo vi /etc/ssh/sshd_config -c ':10,20s/#Port 22/Port 722/g' -c ':wq'
sudo service ssh restart

sudo ufw allow 722
echo "y" | sudo ufw enable
sudo apt update
```
Adding a new user in a single line command - [link](https://askubuntu.com/a/1307156/1617734)

Installing Jenkins on the Vanilla Ubuntu Server 

***```ubuntu```*** user
```bash
sudo apt install -y software-properties-common zip unzip git curl gnupg2 htop awscli jq
wget -p -O - https://pkg.jenkins.io/debian/jenkins.io.key | sudo apt-key add -
sudo sh -c 'echo deb http://pkg.jenkins.io/debian-stable binary/ > /etc/apt/sources.list.d/jenkins.list'

sudo apt update
sudo apt install -y openjdk-11-jdk
sudo apt install -y jenkins

sudo systemctl start jenkins

sudo ufw allow 8080
sudo ufw allow 22
sudo ufw allow 722
sudo ufw allow 41081
echo "y" | sudo ufw enable

jenkins_url=`curl http://169.254.169.254/latest/meta-data/public-ipv4`
jenkins_pwd=`sudo cat /var/lib/jenkins/secrets/initialAdminPassword`
sudo vi /lib/systemd/system/jenkins.service -c ':60,70s/JENKINS_PORT=8080/JENKINS_PORT=41081/g' -c ':wq'

sudo systemctl daemon-reload
sudo service jenkins restart

echo $jenkins_url:41081
echo $jenkins_pwd
```

Go to `Jenkins Url` and Enter the `Password` spit out from ubuntu console.

Click on the `Install suggested plugins` button. Wait for the process to complete. After that enter the following details for the respective fields -

- Username - deployer
- Password - deployer_pwd
- Confirm password - deployer_pwd
- Full name - deployer
- E-mail address - demo_mail@gmail.com

Then click the `Save and Finish` and `Start Using Jenkins` button


Go back to the Server SSH Console and configure the following -

***```root```*** user
```bash
sudo su
sudo apt update

jenkins_username=admin_user
jenkins_password=admin_pwd
chpasswd <<<"$jenkins_username:$jenkins_password"

# Enter the new password - e.g. - admin
sudo passwd jenkins

# Add the jenkins user in sudoers file
echo 'jenkins ALL=(ALL) ALL' >> /etc/sudoers

sudo apt update
exit
```

***```jenkins```*** user
```bash
sudo su - jenkins

aws configure set default.region ap-southeast-1
aws configure set default.output json
sudo apt-get install -y software-properties-common zip
sudo apt install -y git curl gnupg2 htop
sudo apt update
# Enter password if prompted

# RVM 2.7.2 INSTALL
gpg2 --keyserver hkp://keyserver.ubuntu.com --recv-keys 409B6B1796C275462A1703113804BB82D39DC0E3 7D2BAF1CF37B13E2069D6956105BD0E739499BDB
\curl -sSL https://get.rvm.io -o rvm.sh
cat rvm.sh | bash -s stable --rails
source ~/.rvm/scripts/rvm		# This will reload the shell to start using the RVM command-line tool


# RUBY INSTALL
rvm install "ruby-2.7.2"
rvm use "ruby-2.7.2" --default
rvm uninstall ruby-3.0.0
export PATH="$PATH:$HOME/.rvm/rubies/ruby-2.7.2/bin"

# RAILS INSTALL
gem install rails -v 6.1.6.1

# NVM INSTALL
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.1/install.sh | bash

export NVM_DIR="$([ -z "${XDG_CONFIG_HOME-}" ] && printf %s "${HOME}/.nvm" || printf %s "${XDG_CONFIG_HOME}/nvm")"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" # This loads nvm
# https://github.com/nvm-sh/nvm#ansible

# LATEST NODE INSTALL
nvm install --lts
# NPM INSTALL
sudo apt install -y npm

exit
```

After the setup, we need to install the Nginx.
Download guidlines available here - [link](https://www.nginx.com/resources/wiki/start/topics/tutorials/install/)

***```root```*** user
```bash
sudo su -
echo "deb https://nginx.org/packages/ubuntu/ focal nginx" | sudo tee -a /etc/apt/sources.list
echo "deb-src https://nginx.org/packages/ubuntu/ focal nginx" | sudo tee -a /etc/apt/sources.list
key=ABF5BD827BD9BF62
sudo apt update
```

If Here we will get an error saying -
 **_signatures couldn't be verified because the public key is not available_**, get the key from error message and assign value to ```key``` variable 

```bash
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys $key
sudo apt update
sudo apt install -y nginx
sudo service nginx start
```

## Create new users for Jenkins login - 

***```jenkins```*** user
```bash
cd ~

# Downloading the jenkins-cli.jar
JENKINS_PORT_NUM=41081
wget localhost:$JENKINS_PORT_NUM/jnlpJars/jenkins-cli.jar

jenkins_admin_user=deployer
jenkins_admin_pwd=admin

echo 'jenkins.model.Jenkins.instance.securityRealm.createAccount("ayush.adarsh@healthgraph.in", "ayush@jenkins")' | java -jar ./jenkins-cli.jar -s "http://localhost:"$JENKINS_PORT_NUM -auth $jenkins_admin_user:$jenkins_admin_pwd -noKeyAuth groovy = –
```


## Exporting the existing Jenkins jobs - 

First get all the names of jobs needed to be exported. Add all the job names to the array ```array_of_jobs``` and run the command.

Pre-requisite: The ```jenkins-cli.jar``` must be downloaded in the home location. After all jobs the are exported into the export location, zip them up & then transfer to the new jenkins machine. 

***```jenkins```*** user
```bash
cd ~

EXPORT_LOCATION=~/jenkins_export_jobs
PORT=41081
jenkins_admin_user=deployer
jenkins_admin_pwd=deployer_pwd

mkdir $EXPORT_LOCATION

declare -a array_of_jobs=("admin-app-deployment-pipeline" "ASG-GO-LIVE" "ASG-PP" "ASG-PROD-JENKINS" "asg-QA" "asg-QA1" "asg-QA2" "asg-QA3" "asg-QA4" "asg-QA5" "asg-QA6" "asg-QA7" "asg-STAGE" "dev-non-prod-deployment-pipeline" "sdev-non-prod-deployment-pipeline" "qa-deployment-pipeline" "sqa-deployment-pipeline")

for JOB_NAME in "${array_of_jobs[@]}"
do
    echo "$JOB_NAME"
    java -jar ~/jenkins-cli.jar -s "http://localhost:$PORT" -auth $jenkins_admin_user:$jenkins_admin_pwd get-job $JOB_NAME > $EXPORT_LOCATION/$JOB_NAME.xml    
done

echo "All jobs exported to location = $EXPORT_LOCATION"
```


## Importing the existing Jenkins jobs - 

Get the zip file containing all the exported jobs and unzip it. Then put them in the home location ```(~)``` in a folder named as mentioned in ```IMPORT_LOCATION```. 

Pre-requisite: The ```jenkins-cli.jar``` must be downloaded in the home location.

***```jenkins```*** user
```bash
cd ~

IMPORT_LOCATION=~/jenkins_export_jobs
PORT=41081
jenkins_admin_user=deployer
jenkins_admin_pwd=deployer_pwd

declare -a array_of_jobs=("admin-app-deployment-pipeline" "ASG-GO-LIVE" "ASG-PP" "ASG-PROD-JENKINS" "asg-QA" "asg-QA1" "asg-QA2" "asg-QA3" "asg-QA4" "asg-QA5" "asg-QA6" "asg-QA7" "asg-STAGE" "dev-non-prod-deployment-pipeline" "sdev-non-prod-deployment-pipeline" "qa-deployment-pipeline" "sqa-deployment-pipeline")

for JOB_NAME in "${array_of_jobs[@]}"
do
    echo "$JOB_NAME"
    java -jar ~/jenkins-cli.jar -s "http://localhost:$PORT" -auth $jenkins_admin_user:$jenkins_admin_pwd create-job $JOB_NAME < $IMPORT_LOCATION/$JOB_NAME.xml

done

echo "All jobs imported to location = $IMPORT_LOCATION"
```
