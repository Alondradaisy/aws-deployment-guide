# AWS DEPLOYMENT GUIDE 


---

## CREATE A VPC

1. Confirm you are in your closest availability zone
2. Navigate to VPC (services)
3. Launch VPC WIZARD (important to be in correct availability zone)
4. Choose a VPC with a Single Public Subnet (_for production purposes, choose a VPC with Public and Private Subnets)_
5. Enter a VPC name
6. Click on Create VPC - you will automatically have one subnet and route tables

## CREATE SECURITY GROUP

1. Within VPC, click on 'Security group's on the left vertical sidebar
2. Click on 'Create security group'
3. In 'Create security group' 
* Name the security group
* Fill in Description as 80/443 (ports)
* Choose the current VPC 
* Add 2 inbound rules
  1. Type: HTTP || Destination: Anywhere IPV4
  2. Type HTTPS || Destination: Anywhere IPV4  
* Add optional tag (name: name chosen)
* Create SSH Security Group
* Fill in Description as 22 (port)
* Add Inbound rule 
  1. Type: SSH || Destination: MyIP 
* Add optional tag (name: name chosen)
* Click 'Create security group' button

## Create new EC2 Instance

* Confirm correct region
* Click EC2 (services), then click 'Launch Instance'
* Select Ubuntu Server 20.04 LTS 64-bit (x86)
* Instance type: Choose free tier t2.micro (or t3.micro depending on location)
* Click 'Next: Configure Instance Details' 
* Network: Select the VPC created
* Subnet: automatically selects that current one 
* Auto-assign Public IP: select enable
* Click Enable termination protection
* Click 'Next: Add Storage' 
* Select Volume Type: General Purpose SSD (gp3)
* Add optional tag(s) Name: domain name
* Configure security group
* Select the 2 security groups just created (web & SSH)
* Click 'Review and Launch' || **review**
* Click 'Launch'
* Key Pair pop-up form appears
1. Create new key pair
2. Select ED25519 as type (connected to Github)
3. Name the key pair (same name as instance & no dots in name [use hyphens, if so])
4. Download the key pair. Store it safely and make backups on device(s)
   
   * **.env file** 
   * **keychain**
   * **AWS key(s) directory**

5. In iTerm or Terminal CD into AWS keys folder and type 

        'chmod 400 adwarren.pem'
        
    * **chmod 400 gives read-only access to owner and no permissions given to anyone else**
    * **type 'ls -la' to list al keys**
    * **chmod 777 gives all users permissions** ||  _beware_



* Type yes and hit enter 
## Manually Connect to Instance
- Grab public IP address from instance

* Create SSH shortcut - Manually connect to instance 
        
        ssh -i "adwarren.com" ubuntu@PublicIPAddressFromInstance

## Associate an Elastic IP to an Instance

- Confirm correct region
- Navigate to VPC service, click on Elastic IP on the left sidebar menu
- Leave all default settings as is
- Allocate elastic IP
- Click 'Allocate Elastic IP Address'
- Then in 'Actions' associate it by clicking 'Associate Elastic IP address' (select running instance, select IP) 

    _(you can also disassociate it by clicking 'Disassociate Elastic IP address)_
- Navigate to instances and copy the new Associated Elastic IP address and SSH in Terminal or iTerm (ssh -i), type yes, hit enter

## Set Up Route 53 - Hosted Zones / Domains

- Navigate to Route 53 service. Click on 'Hosted Zones'. 

    _(take a look at the 2 records - 'Name Server' and 'Start of Authority')_
- In 'Hosted Zones' click on the domain purchased, then click 'Create Record'
- Copy the elastic IP address and paste in value field 
- Leave all other default settings as is and click 'Create Record'
    1. Create new A Record (alias records) for 'www.yourdomain.com' (Simple Routing) using your new Elastic IP as the value
    2. Create another new A Record (alias records) for 'yourdomain.com' and use 'www.yourdomain.com' as the value 
   
        **_Alias Records can be thought of as the other address that a user may type to reach your domain, which are then routed to your actual IP address where domain is hosted._**

## Create an SSH Shortcut

- In iTerm or Terminal type 'sudo nano ~/.ssh/config' which will prompt you to a text screen

        Host <Name>
            User ubuntu
            Hostname <DNS or elastic IP of instance>
            Port 22
            IdentityFile <path to key file (.pem file)>

    _Example:_

        Host adwarren
            User ubuntu
            Hostname adwarren.com
            Port 22
            Identity File (insert file to my downloaded .pem file)

- Save the text 'cmd + S', 'control + C' to exit that screen, then SSH using shortcut (SSH adwarren)
    * from here you will now be able to use your host name to SSH in ||  _'SSH MySite'_

## Configure Ubuntu Server

- When you SSH into the ubuntu server, the terminal informs you about the updates needed.
- Update & upgrade ubuntu in terminal by ssh-ing in and in the ubuntu server type: 

        sudo apt update && sudo apt upgrade -y
    

    _This will trigger an update and upgrade of the server_
- Reboot / restart the EC2 instance (using drop down menu)
- Then in ubuntu, type the command
    
        sudo apt install nginx -y

    then hit enter.
- Following that, type:

        sudo systemctl enable nginx

    then hit enter.

- Install Node Version Manager from quick Google search 'NVM' or navigate to Github to find the direct curl address (Install and Update section)
        
        curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v16.2.0/install.sh | bash
        
- Exit Ubuntu server from Terminal or iTerm, then re-enter with SSH shortcut
- type the command 

        nvm install --lts

- Check node version you're currently using by running the command: 

        node -v 

- To upgrade to the latest version of node, run:


        nvm install stable

- To change node version, run:

        nvm use stable

**When deploying projects, the node version on cloud server should match the version on the local machine**

## Configure NGINX

- While in the cloud server, cd into nginx, 

        cd  /etc/nginx/sites-available
    then hit enter.

- Run the command: 

        sudo nano yourHostName

- In the text screen that pops up, enter the following

for front-end: 

        server {
        listen 80;
        server_name yourDomain.com www.yourDomain.com;
        location / {
                proxy_pass http://127.0.0.1:3000; 
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
                proxy_redirect off;
               }
             }

for backend: 

        location/api/{ 
            proxy_pass http://127.0.0.1:3001
        }

for both front-end and back-end:

    server {
        listen 80;
        server_name yourDomain.com www.yourDomain.com;
        location / {
                proxy_pass http://127.0.0.1:3000; 
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
                proxy_redirect off;
               }
            

             location/api/{ 
            proxy_pass http://127.0.0.1:3001
            } 
        }

the backend location/api should match routes
the port should match the backend app port

- Save and exit that screen (cmd + S, cmd + C)


- Since the file isn't saved in the sites-enabled, you do so by running a 'symlink'
- Run the command: 

        sudo ln -s /etc/nginx/sites-available/yourHostName /etc/nginx/sites-enabled/yourHostname

- Type:

        ../sites-enabled/

    hit enter, then cd a folder up (cd ..) to view (ls) what's in the sites-enabled folder


- Run the cmd:

        sudo systemctl reload nginx


    you should be running this command anytime you do a configuration change

- Test nginx by running the cmd: 

        sudo nginx -t

    then hit enter.

    _If testing is OK = success_

    _If there is feedback(errors) = error_

    There shouldn't be feedback after running that cmd.
    Check the URL, a '502 Bad Gateway' error should appear _**this means that nginx is working, but there is no website to point to**_

## Upload & Clone Code to Get Proj onto Instance
- Git Clone your project 
- Git Pull
- CD into front-end code from clone
- Make sure you are using correct node version (check with node -v)
    - if you need to change the version to match versions, use the cmd:

            nvm use stable
            or nvm use <specified version>
- Npm i
- Create .env (remember to update your .env on the server when you change it, git will not automatically do it)
    - your .env can include PORT=<port> which will set the port of the app... ie: 8080 
    - You can edit your project to remove localhosts from connection URLs either on the server or locally and push with git 
        
        - Axios: localhost: 3001/api --> /api

- In the front-end, npm start + hit enter
- Test that your project works: you can use multiple iTerm or Terminal tabs that are ssh logged in to monitor front-end + back-end
  
## Configure / Edit NGINX 
#### (Configure once more to intercept API & Forward to the backend port 3001)

- Go back to NGINX config file in terminal. “cd /etc/nginx/sites-available/” enter.
- Then “sudo nano yourHostName” enter.
- Add a second location now in the “nano environment” that pops up.   
- Add the following code in nano environment (*note that the ports may change):
  
        


        server {
        listen 80;
        server_name riegercodes.com www.riegercodes.com;
        location / {
                proxy_pass http://127.0.0.1:3000;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header Host $host;
                proxy_cache_bypass $http_upgrade;
                proxy_redirect off;
         }
         location /api/ {
                proxy_pass http://127.0.0.1:3001;
               }
        }  

- Save & exit
- Test nginx again with cmd:
  
        sudo nginx -t

    hit enter.

- Reload the system once more with cmd:
  
        sudo systemctl reload nginx

    then hit enter.

- Navigate into the back-end and run 

        npm run start

    then hit enter.
    You will notice that it will not run properly because mongoDB is not yet connected or installed.

## Create a create-react-app on the remote server


- SSH into the server.
- Type “npx create-react-app test” then hit enter. It will do it's thing 
- CD into the app “test” and type “npm i” hit enter. 
- “Npm run build” hit enter.
- CD into build/ folder and type ls to make sure there is an index.html present. 
- Type “PWD” (hit enter) present working directory to show path.  
- Open new tab in terminal, run cmd:
  
        cd /etc/nginx/sites-available
    hit enter.

        sudo cp yourHostName static-test 
        
    hit enter. 

        “ls” hit enter. 

        sudo nano static-test
    hit enter.

- Delete everything inside of location. The text should look like this, for reference: 
    - Inside location type “root” the space and paste the working directory into build file and paste after “root.”

        server {
        listen 80;
        server_name adwarren.com www.adwarren.com

         location / {

         }
 }
     






            server {
                listen 80;
                server_name adwarren.com www.adwarren.com;

            location / {
                root /home/ubuntu/test/build;
                index index.html;
                }
            }

- CD into sites-enabled
- Then run the cmd:

        sudo rm adwarren

    hit enter

        sudo ln -s /etc/nginx/sites-available/static-test /etc/nginx/sites-enabled/static-test
    
    hit enter. 
- Then “ls” to list what's inside 
- Test it again by typing cmd:
        
         sudo systemctl reload nginx
         
    hit enter. 

## Change file & Rebuild App (to update a static website)

- CD into the react app named test/src
- Type cmd:

        nano App.js

- Edit file, save changes, then CD back one folder (cd ..) and type cmd:

        npm run build

- Test it again by running sudo reload nginx cmd:

        sudo systemctl reload nginx

    hit enter

## Subdomain Setup or Enable >1 Sites

- Login to AWS, navigate to 'Route 53' -> Hosted Zone -> Domain. Copy the Elastic IP located in the value field. Click 'Create Record'.
- Give the record a name, paste value copied (Elastic IP address)
- In Terminal or iTerm, SSH into ubuntu server, make sure to be in sites-enabled
        
        cd  /etc/nginx/sites-available

    and 
    
        sudo nano static-test
    
    hit enter.
- Edit/change:
        
        server-name subdomain.yourDomain.com

- Do not forget to add the symlink!


## MongoDB Setup

- Go to mongoDB.com and make an account or sign in.
- Click new project, give it a name (AWS-project)
- Create cluster-create database, all free options, AWS, in a zone closest. 
- Name cluster (AWS-cluster).
- Click Network access and add current IP address. 
- Also add AWS elastic IP address. 
- Click on Database access and click Add New Database User.
- Open terminal, CD into aws-deploy-backend and type “nano .env” to create a .env file.
- Paste auto generated password in the .env file.  Save and exit. 
- Also add the password to locally developed backend .env file. 
- Navigate back to the MongoDB created and click on “connect” the “connect to application.”
- Leave default settings, then copy the connection string provided and paste in the .env files into a variable and rename database from “myFirstDatabase” to “talks”
- Replace <password> with the copied password created a few steps earlier. Make sure to also take out angle brackets.
- In .env file, also add “SECRET_KEY=rememberyoursecretkey”
- Exit, then while in the frontend on the server, type “npm start” hit enter.  Will show “MONGODB CONNECTED” if things went well. 
  
**NOTE: This all can be done on local machine as well to use the MONGODB in the cloud created with code on local machine.  Same steps adding code to .env file. When running on local machine, MAKE SURE you type “npm run dev” instead of “npm run start”.**

## Process Management

- SSH into Ubuntu

- Then run the command:
  
        npm i pm2 -g

    then after the system runs through and executes, it runs react project code as a process. _This will monitor and keep the program running to avoid crashing._

        cd backend
    
    to backend folder. 

Installing pm2 globally is a **'Process Manager'** that runs code as a process. 

Backend + Frontend Process (repeat for both)

- Type:

        pm2 start --name Backend npm – start

- Save it by running command:

        pm2 save

- Then,

    to Start:

        pm2 start
    
    To Check Status:

        pm2 status

    To Stop:

        pm2 stop 1
    
    or Stop All: 

        pm2 stop all
        
    To Delete:

        pm2 delete <# of the process>

    To Name: 

        -name<NAME>

    To Rename a Current Process:

        pm2 restart <Current Name> -name<New Name>

    To Restart:

        pm2 restart

- Remember to save:

        pm2 save

- After you've created and saved everything online, reboot EC2 Instance

## Enable HTTPS

- SSH into remote server 

        sudo apt update
        
    hit enter.

- update to confirm you get latest packages, then:


        sudo apt install certbot python3-certbot-nginx
        

    hit enter
- It will prompt you: Do you want to continue? -> Yes. 

        sudo ufw status

    hit enter.

        sudo certbot --nginx -d adwarren.com -d www.adwarren.com” 
    
    hit enter. 
- Answer all questions to include #2 = yes redirect traffic.  
- Refresh website
- You should now be secure. 
  
# Happy Deploying!