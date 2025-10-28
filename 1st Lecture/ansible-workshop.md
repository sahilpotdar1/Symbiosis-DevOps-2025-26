# Ansible to a real-world scenario—deploying an open-source static website from GitHub

#### This demonstrates how Ansible orchestrates tasks like installing software, cloning code, and configuring services

#### GitHub Repository: https://github.com/cloudacademy/static-website-example







##### Step 1: Prepare Environment

* ###### create directory *ansible-workshop* \& *level2-deploy-website*
* ###### Navigate to *level2-deploy-website*
* ###### Setup Inventory file *inventory.ini*

	

        *\[local]*

	*localhost ansible\_connection=local*



* ###### Test ping: *ansible -i inventory.ini local -m ping*



*----------------------------------------------------------------------------*



##### Step 2: Playbook setup

###### **These are the task will be cover in this workshop:**

1. ###### Update package lists (good practice).
2. ###### Install Git (to clone the repo) and Nginx (web server).
3. ###### Clone the GitHub repo to a web-accessible directory.
4. ###### Configure Nginx to serve the site.
5. ###### Restart Nginx and verify it's running.
6. ###### Test the site by fetching it locally.



* ###### Setup playbook: *deploy-website.yml*



*---*

*- name: Deploy Simple Static Website*

  *hosts: local*

  *become: yes  # Run as root for installations*



  *tasks:*

    *# We'll add tasks here one by one*



*- name: Update apt package cache*

  *apt:*

    *update\_cache: yes*

    *cache\_valid\_time: 5*





* ###### Run: *ansible-playbook -i inventory.ini deploy-website.yml*





*-----------------------------------------------------------------------------*



##### Step 3: Install Dependencies (Git and Nginx)



* ###### Update playbook: *deploy-website.yml*



*- name: Install Git*

  *apt:*

    *name: git*

    *state: present*



*- name: Install Nginx*

  *apt:*

    *name: nginx*

    *state: present*



* ###### Run: *ansible-playbook -i inventory.ini deploy-website.yml*
* ###### Confirmation: nginx -v and git --version





--------------------------------------------------------------------------------



##### Step 4: Clone the GitHub Repo



* ###### Update playbook: *deploy-website.yml*



*- name: Clone static website repo*

  *git:*

    *repo: https://github.com/cloudacademy/static-website-example.git*

    *dest: /var/www/static-site*

    *force: yes  # Overwrite if exists*


* ###### Run: *ansible-playbook -i inventory.ini deploy-website.yml*
* ###### Confirmation: ls /var/www/static-site



--------------------------------------------------------------------------------



##### Step 5: Configure Nginx



* ###### Setup configuration file: *nginx-site.conf*



*server {*

    *listen 80;*

    *server\_name localhost;*



    *root /var/www/static-site;*

    *index index.html;*



    *location / {*

        *try\_files $uri $uri/ =404;*

    *}*

*}*





* ###### Update playbook: *deploy-website.yml*



*- name: Copy Nginx config*

  *copy:*

    *src: nginx-site.conf*

    *dest: /etc/nginx/sites-available/static-site*

    *mode: '0644'*



*- name: Enable Nginx site*

  *file:*

    *src: /etc/nginx/sites-available/static-site*

    *dest: /etc/nginx/sites-enabled/static-site*

    *state: link*



*- name: Remove default Nginx site*

  *file:*

    *path: /etc/nginx/sites-enabled/default*

    *state: absent*



* ###### Run: *ansible-playbook -i inventory.ini deploy-website.yml*
* ###### Explanation: ***copy*** moves your local config to the server. ***file*** creates a symlink to enable it and removes the default to avoid conflicts.



----------------------------------------------------------------------------------------------------------





##### Step 6: Restart Nginx and Verify



* ###### Update playbook: *deploy-website.yml*



*- name: Restart Nginx*

  *service:*

    *name: nginx*

    *state: restarted*



*- name: Verify Nginx is running*

  *command: systemctl status nginx*

  *register: nginx\_status*

  *changed\_when: false  # Don't mark as changed*



*- name: Display Nginx status*

  *debug:*

    *msg: "{{ nginx\_status.stdout }}"*




* ###### Run: *ansible-playbook -i inventory.ini deploy-website.yml*
* ###### Explanation: service restarts Nginx. command checks status, register saves output, and debug prints it



--------------------------------------------------------------------------------------------





##### Step 7: Test the Deployed Website



* ###### fetch site: *curl http://localhost*







===================================================================================================================================

xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

===================================================================================================================================



# \[optional] Deploying to a remote machine





### Step 1: Windows (Set Up WSL Ubuntu)



1. ##### open PowerShell (as admin): *wsl --install* + setup username/password
2. ##### Install SSH Server: *sudo apt update + sudo apt install openssh-server*
3. ##### Start and enable SSH: *sudo systemctl start ssh + sudo systemctl enable ssh*   <b>\[confirmation: sudo systemctl status ssh]</b>
4. ##### Get WSL IP Address: *ip addr show* (look for inet under eth0)

##### 5\. Adjust WSL Networking: *netsh interface portproxy add v4tov4 listenport=2222 listenaddress=0.0.0.0 connectport=22 connectaddress=<wsl-ip>*   \[replace <wsl-ip> with the ip]





### Step 2: Set Up SSH Key-Based Authentication



1. ##### Generate SSH Key in Root device: *ssh-keygen -t ed25519 -C "ansible-key"*
2. ##### Copy Key to Remote Machine: *ssh-copy-id student@<student-ip>* \[Example: ssh-copy-id student@192.168.1.100]
3. ##### Test SSH: *ssh student@<student-ip>* \[If it logs in without a password, exit with exit]





### Step 3: Update Ansible Inventory



1. ##### In root device modify the inventory file \[Replace <student-ip> with the IP]: 



###### 	*\[webservers]*

###### 	*localhost ansible\_connection=local*

###### 	*<student-ip> ansible\_user=student ansible\_ssh\_private\_key\_file=~/.ssh/id\_ed25519*



##### 2\. Test the inventory: ansible -i inventory.ini webservers -m ping \[if fail verify: *ssh student@<student-ip>*]





#### Step 4: Verify the Playbook



1. ##### *ansible-playbook -i inventory.ini deploy-website.yml*



##### 2\. If error comes:

* ###### Connection: Check SSH (ssh student@<student-ip>), firewall (sudo ufw allow ssh on student’s machine).
* ###### Sudo: If student’s user needs a password, run with --ask-become-pass.
* ###### Port Conflict: If port 80 is taken, stop other services (sudo systemctl stop apache2)





#### Step 5: Test the Websites on Both Machines



1. ##### Root: curl *http://localhost*
2. ##### From Root WSL: *curl http://<student-ip>* OR want to access from child device: Windows WSL: *curl http://localhost*







