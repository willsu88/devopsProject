## Motivation Behind Project
I've been particularly interested in CI/CD pipelines, especially as I touched upon them quite often in my previous job at Tableau, Salesforce. At the time, the CI/CD tool we used was GitLab CI/CD. I've also had to deal with Docker containers here and there before, but never had to manually create Dockerfile's. AWS was another tool I used often at school and at Tableau. In this project, I want to combine all of these tools and technologies together and try to build my own end-to-end pipeline - from pulling new commits and auto-builds to deploying them onto a docker container. Here's the general architecture of the project:

![alt text](/images/architecture.png)
## Step 1: Jenkins, Maven, Git
Jenkins was the tool I chose to auto-build and deploy my code. For this project, the Java code itself isn't too important. It's simply a Hello World kind of HTML page. I am merely using it to test that whennever I make a new commit, my deployed webpage is automatically reflected as well.

Here I created an EC2 instance on AWS to host my Jenkins. (Note to self: open the inbound TCP ports for `port 22` and `port 8080`. One to use for ssh, the other to expose Jenkins on). On the EC2, I installed java (which was necessary before installing Jenkins) and Jenkins. 

On the Jenkins GUI, I downloaded a Maven plugin. This is the tool Jenkins will use to actually build the Java tool. I've used Maven in school and at work multiple times, so this wasn't too unfamiliar. I also downloaded the Git plugin, which will be used to pull code from my repository.

Then I created a new job on Jenkins. This job 1. pulls code from the Git repo, 2. builds the Java code with the Maven goal `clean install`. I ran it, and verified the a .war file was successfully built.

![alt text](/images/jenkins-build.png)
## Step 2: Tomcat | Automation
Apache Tomcat is a free and open-source implementation of the Jakarta Servlet, Jakarta Expression Language, and WebSocket technologies. It provides a "pure Java" HTTP web server environment in which Java code can also run. I will be using it to run my Java webpage.

Again, I started an EC2 instance for my Tomcat server. I installed Tomcat Plugin for Jenkins. In the Post-build Actions of the Jenkins job configuration page, I pointed the Deploy War action to my Tomcat EC2's public IP address. Now my Jenkins job,  Now whenever I `git push` some new code and run this Jenkins job, it not only builds code from my Git repo, it also automatically deploys it onto a Tomcat server. I verified by checking that the webpage on my Tomcat server was updating.

However, it quickly became annoying to have to click on the build job everytime I push new code. As programmers say, never do anything more than thrice. I created a cron job on Jenkins based on this rule
`* * * * *`, which basically means it's always polling the git repo, listening for changes. Once I did this, now everytime I push code, Jenkins automatically pulls the code, builds it, and deploys it. My first pipeline is working!

## Step 3: Docker
To really round the edges of my pipeline, I really should put my Tomcat hosted server in a Docker container. This way I don't need to reconfigure the details for a Tomcat server everytime I want to build and deploy an app. And so I moved on setting up another EC2 instance as to host my Docker service.

(Note: Edit the Security Group: open up the Inboud TCP Ports from 8081-9000. This provides some convinience for me to host docker containers and for me to access them via the browser.)

In my Docker instance, make sure to create a Dockerfile ready to use. 
```
FROM tomcat:latest
RUN cp -R /usr/local/tomcat/webapps.dist/* /usr/local/tomcat/webapps
COPY ./*.war /usr/local/tomcat/webapps
```

This copies the artifact war file sent by Jenkins in our docker host to our docker container. I've configured Jenkins to send the built war file to EC2 instance via public SSH with password auth (there are probably other methods to do this but for now its okay). Here I had to configure a new `dockeradmin` user for the docker EC2 instance. This is for Jenkins to login to our EC2 instance. (By default EC2 doesn't allow password auth ssh. It's probably better to use a SSH key for better safety). I verified that it is indeed a new .war file here every time I push new code here.

![alt text](/images/docker-new-war.png)

Then I configured Jenkins to execute some docker commands that will help me:
1. build a docker image using that war file
2. build a docker container using image
A note: We have to make sure to delete old containers in this command too, bececause we are constantly using the same container name, which doesn't work for Docker.

Now my pipeline 1. pulls from Git 2. builds using Maven 3. deploys and sent to a Docker host 4. creates a Docker container with the necessary Tomcat dependencies to host my code. Nice! 

![alt text](/images/jenkin-docker-cmds.png)
