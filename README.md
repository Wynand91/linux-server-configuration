# PROJECT: Linux Server Configuration

Baseline Installation of a Linux server hosting the Catalog application created earlier in the Nanodegree.

## _Important information:_
   - Access site at: [http://ec2-18-195-96-160.eu-central-1.compute.amazonaws.com/](http://ec2-18-195-96-160.eu-central-1.compute.amazonaws.com/)
   - IP address: 18.195.96.160 (**Do not access site only by IP address! - OAuth will not work**)
   - SSH port: 2200 
   

## Set up server on Amazon Lightsail
   1. Navigate to [Amazon Lightsail](https://aws.amazon.com/lightsail/)
   2. click _Create instance_.
   3. Click Ubuntu instance image, OS Only and Ubuntu 16.04 LTS.
   4. Choose a instance plan (Lowest tier is free for one month).
   5. Give instance a hostname.
   6. Click _Create_ button to create the instance.


## SSH into the server
   - For now you can SSH into server with the button on your instance, but let's set up private key, because that will soon change.
   
   1. download the Default Private Key from the Account menu on your instance to your local machine.
   2. Move the downloaded file into the ~/.ssh directory and rename it to _amazon_pk.rsa_
   3. Change permissions of file `chmod 600 ~/.ssh/amazon_pk.rsa`
   - You can now SSH into server with `ssh -i ~/.ssh/amazon_pk.rsa ubuntu@<instance_public_ip_address>`
   
   
## Upgrade packages:
   
   ```
   sudo apt-get update
   sudo apt-get upgrade
   sudo apt-get dist-upgrade
   ```

## Change SSH port from 22 to 2200:
   
  `sudo nano /etc/ssh/sshd_config`
   
 Search for Port and change it from 22 to 2200
 
## Configure Uncomplicated Firewall (UFW):

   1. `sudo ufw default deny incoming`
   2. `sudo ufw default allow outgoing`
   3. `sudo ufw allow 2200/tcp`
   4. `sudo ufw deny 22`
   5. `sudo ufw allow www`
   6. `sudo ufw allow ntp`  
    (You can double check changes with: `sudo ufw show added`)
   
   7. `sudo ufw enable`
   8. On Amazon lighsail Instance, under Networking tab, update firewall to match changes.
   
   **Note:** From now on you have to SSH into server with `ssh -i ~/.ssh/amazon_pk.rsa -p 2200 ubuntu@<instance_public_ip_address>` 
   
## Configure timezone:

   `sudo dpkg-reconfigure tzdata` and follow commands to find your timezone.
   
## Create grader user and assign sudo privilege:

   1. `sudo adduser grader` and enter password.
   2. `sudo visudo`
   3. Add this to file: `grader ALL=(ALL:ALL) ALL` underneath root permissions.
   
## Create SSH key for grader:

   **On local machine:**
   
   1. Run `ssh-keygen` and enter a file (path including) to save the key to.
        - I used: ~/.ssh/grader_keypair
        - Two files will be created in the ~/.ssh directory: grader_keypair and grader_keypair.pub
   2. Run `cat ~/.ssh/grader_keypair.pub` and copy contents of file.
   
   **Log into server as grader:**
   
   1. In home directory run: `mkdir .ssh`
   2. `nano .ssh/authorized_keys` and paste contents of grader_keypair.pub that you copied.
   3. Set permissions: `chmod 700 .ssh` and `chmod 644 .ssh/authorized_keys`
   
   - **Make sure key based authentication is forced and Deacativate root remote login:**
        1. `sudo nano /etc/ssh/sshd_config` and make sure 'passwordAuthentication' is set to no
        2. `sudo su`
        3. `nano /etc/ssh/sshd_config` and change 'PermitRootLogin' line to 'PermitRootLogin no'
        4. `sudo service ssh restart`
   
   **NOTE:** From now on you can SSH into server with keypair:
   `ssh -i ~/.ssh/grader_keypair -p 2200 grader@<instance_public_ip_address>` 
   
