# Gitlab guide notes

Note: These tutorials are created by Okta team members and may not be 100% accurate. We will do our best to provide updates and make sure they are accurate as the product evolves.

# Purpose & Goal:

This tutorial will guide you through setting up Advanced Server Access and GitLab to provide a security CI/CD pipline by eliminating the need to manage SSH keys. The service account used by the gitlab-runner leverages ASA's ephemeral credential mechanism to authenticate to ASA protected servers. There are no SSH keys to create, deploy, revoke or rotate, and no passwords to steal.

The tutorial assumes that you have the following in place;

Base ASA Environment including; 2 Linux Servers (Ubuntu in this example)

- 1 Assigned Web Server (WEBSERVER)
- 1 Assigned Gitlab Runner Server (GITLABRUNNERSERVER) Gitlab Account (https://gitlab.com/users/sign_up)

# Setup GitLab

Create a Repository using the code in this folder ([GitLab Repo](https://github.com/Okta-PAM-Resource-Kit/tutorials/tree/main/service%20users/gitlab/code/nginx)). Edit the 'gitlab-ci.yml' file so that SERVERNAME is replaced by the hostname of your Assigned Web Server then save repo.

**Setup GitLab:**

1. Once you have created a project on gitlab.com, we will need to create a new project runner.

2. Navigate in the project to 
    a. Settings -> CI/CD -> Click on **Expand** next to **Runners** -> New Project Runner 

3. In the New project runner page, select the **Linux** radio button

4. Under **Details (optional)** enter in a description for the Runner (e.g.: This runner will be used to demonstrate ASA’s Gitlab Runner implementation)

5. Make sure the 'Run untagged jobs' option is selected

6. Now click on **Create runner**

7. Copy the code listed in **Step 1.** Additionally, make sure the runner token (listed below the code box in Step 1 matches the code you copied.

8. Select **Go to runners page**

**Enroll your Servers with ASA:**

If you haven’t done so already, you can use the scripts found here to enroll your servers with ASA: 

https://github.com/Okta-PAM-Resource-Kit/scripts/tree/main/installation/linux

************************************Prepare WEBSERVER:************************************

1. Navigate to ASA, and connect to the WEBSERVER server

2. Once connected, run the following command: `sudo apt-get install nginx`

**Prepare GitLab Runner server** 

1. Now connect to the GITLABRUNNERSERVER server via ASA

2. Install Gitlab Runner on the server by downloading the binary for your gitlab-runner server with the following command:

`sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64`

3. Give gitlab-runner permissions to execute with the following command:

    `sudo chmod +x /usr/local/bin/gitlab-runner`

4. Check that gitlab-runner is installed by running `gitlab-runner -v`

5. We now need to create a GitLab Runner user by running the following command (make sure there is a space between —shell and /bin):
`sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash`

6. Install and run gitlab-runner as a service with following command:
`sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner sudo gitlab-runner start`

7. Now we need to register GitLab Runner. Remember that code we copied earlier with the Runner token? We will now need to copy and paste it into our Gitlab Runner Server.

    The code will look something like this (replace XX with your unique runner token): 
`sudo gitlab-runner register --url [https://gitlab.com](https://gitlab.com/) --token XX`  

    a. Enter the Gitlab instance URL as **https://gitlab.com/**
    
    b. Enter **gitlab-runner** as the name of the runner
    
    c. You can use **docker** as the executor for this demo

    d. Enter in the default Docker image as the example: **ruby:2.7**  

8. We will now need to get the id of the gitlab-runner user

    a. To do this you will need to Sudo to the gitlab-runner user using: `sudo su gitlab-runner`

    b. Now enter in the following command:`id` 

    c. Take note of the UID/GID for the **gitlab-runner user**

**Create ASA Service User:**

1. Create a service user account in ASA by navigating to Users > Service Users

2. Click on Create Service User

3. Give your Service User the **gitlab** username

4. Now you will need to create an API key for the service user by selecting the user you just created and then clicking on the **Create API key** button at the bottom of the page.

5. The API Key ID and Key Secret will be shown. Make sure to take note of both as the Key Secret will not be shown again once you close this window.

**Create group for the ASA Service User:**

1. In ASA navigate to Groups, then click on **Create Group**

2. Use the group name gitlab

3. Now click on the Create Group button

4. You will now see the page for the newly created group. From here, you will need to click on the Users tab and enter the username of the gitlab service user we just created. Then click on Add User. You should now see the gitlab user listed as Active.

**Add GitLab Service User group to servers:**

1. In ASA navigate to Projects, then select the Gitlab Project

2. Click on the Servers tab

3. Click on the Hostname for the gitlab-runner server, then click on the **Services** tab

4. Click on Add Service, then select the gitlab service user from the drop down menu

5. You will now need to enter in the UID you that you got from running the `id` command on the gitlab-runner server earlier into the **Server UID box**

6. Click Submit

**Add GitLab group to gitlab project:**

1. In ASA navigate to Projects, then select the Gitlab Project

2. Click on the Groups tab and then select **Add Group to Project**

3. Enter the gitlab group we created earlier into the Group dropdown box

4. Set the Server Account Permissions as **Admin**

5. Then click on Add Group

**Finalize the Gitlab-runner Server Setup:**

1. Go back to the GITLABRUNNERSERVER terminal window 

2. Make sure you are still using the gitlab-runner user via su. if not, run the following commands again:
    a. `sudo su gitlab-runner`
    b. You can type `whoami` in the console to make sure you are using the gitlab-runner user

3. Now run the following command: `sft config service_auth.enable true`

4. Type in `cd ~`

5. Then `mkdir .ssh`

6. Now run this command: `sft ssh-config >> .ssh/config`

7. Test that you can connect to the WEBSERVER by running: `sft ssh WEBSERVER`

8. You should now be able to authenticate as service user **gitlab** from GITLABRUNNER to WEBSERVER, while connected to gitlab-runner server via ASA.

**Testing:**

Navigate to Gitlab Click Build > Pipelines Click Run Pipeline
