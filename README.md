# Deliverypipeline [![Circle CI](https://circleci.com/gh/lachatak/deliverypipeline/tree/master.svg?style=svg)](https://circleci.com/gh/lachatak/deliverypipeline/tree/master) [![Coverage Status](https://coveralls.io/repos/lachatak/deliverypipeline/badge.svg?branch=master)](https://coveralls.io/r/lachatak/deliverypipeline?branch=master)

## Continuous Delivery on cloud ##
This is an experimental project to test how we could achieve continuous delivery pipeline with open source cloud based tools to enable zero downtime release.
The main motivation was this documentation from AWS:
http://docs.aws.amazon.com/elasticbeanstalk/latest/dg/using-features.CNAMESwap.html

### Tool set ###
- [AWS Elastic Beanstalk](http://aws.amazon.com/elasticbeanstalk/) to host our application
- [Docker](https://www.docker.com/) to make the application more portable
- [Dockerhub](https://hub.docker.com/) to store Docker images produced by the CI
- [Mongolab](https://mongolab.com/) to have a cloud based mongo store for the test application to store event source snapshots 
- [CircleCI](https://circleci.com/) to build and deploy the application
- [Ansible](http://www.ansible.com/home) to manage AWS instances
- [Loggly](https://www.loggly.com/simplify-log-management-with-loggly/) to be able to easily analize app logs
- [Coveralls](https://coveralls.io/) cloud code coverage history and stats
- [HipChat](https://www.hipchat.com/) to see build notifications
- [Github](https://github.com/lachatak) to store the application source
- The application itself is a [Spray](http://spray.io/) and [Akka](http://akka.io/) based simple REST application

The running application is available [here](http://deliverypipeline-prod.elasticbeanstalk.com/)

## The process ##

![Alt text](pics/deliverypipeline_flow.jpg?raw=true "Pipeline Flow")

1. Start provisioning AWS EBS application and environments with ***Ansible***
2. AWS with the help of Ansible creates 2 preconfigured instances for the environments
3. Push modifications to the github
4. Notifications goes from the github to the ***HipChat*** about pushed code
5. ***CircleCI*** discovers that there is a modification in the codebase. It goes through the build process and deploy new version to AWS
6. CircleCI runs all the unit tests
7. CircleCI pushes code coverage results to Coveralls
8. CircleCI builds and tests the new Docker image
9. After the successfull test the new image is pushed to ***Dockerhub***
10. CircleCI uploads the modified **Dockerrun.aws.json** to ***S3***
11. CircleCI triggers new application version creation
12. AWS creates a new application version based on the **Dockerrun.aws.json** downloaded from the ***S3***
13. CicrleCI kickstarts the deployment on the staging environment. The staging environment is picked based on the public URL. Its URL is ***deliverypipeline-staging.elasticbeanstalk.com***
14. AWS pull the new version from the Dockerhub. The image version is defined in the **Dockerrun.aws.json**
15. The docker image is deployed to the AWS instances
16. CircleCI verifies the deployment and swaps the staging and production URL if the verification was successful. However, DNS propagation requires some time to happen. Notification will be sent to ***HipChat*** about the result of the build
17. The user goes to the production URL and hits the page. The application updates its internal state and persists the state to the ***Mongolab*** mongo database. This configuration is coming from the **/app/application.conf** live configuration which is mapped into the Docker image via the **Dockerrun.aws.json**
18. The application is logging using **logback-ext-loggly** so every log generated by the application goes to ***Loggly***. It is easier to track and debug issues using Loggly than login into the AWS instance and tail the log. Obviously there are cases when it is inevitable
19. Updated status badge for the build state on github
20. Updated codecoverage result badge for the build on github
21. Booom!! Make profit ;)

Obviously in a real application it could be even more complex but it is a good basic solution for further development.

## The ingredients ##

### The application ###
I have a simple REST based application. It provides some basic information about its running environment:
- Deploy history
- Aggregated number of calls since the first version was deployed
- Currently deployed version
- Host name

<img src="./pics/app.png" width="800"/>

The application has Akka mongo persistence. Every time the application URL is called the internal state will be modified and persisted to a mongo store which is hosted by Mongolab. If I deploy a new version of the application it is going to use the same mongo store and fetch the previously persisted state.

### sbt plugins ###
To achieve the well desired goal I had to add some sbt plugins:
- [sbt-docker](https://github.com/marcuslonnberg/sbt-docker) to manage Docker image generation
- [sbt-buildinfo](https://github.com/sbt/sbt-buildinfo) to generate a class that contains static build time information like git version number, generation time. It is used by the server when the user hits a endpoint
- [sbt-git](https://github.com/sbt/sbt-git) to modify the application version number to contain git version number

### Ansible ###
- Ansible is used to prepare the entire infrstructure before the first deployment happens.
- It creates the AWS Elastic Beanstalk application and the environments
- It copies all the neccessary configuration files to the instances like **/app/logback.xml** and the **/app/application.conf** files. Those two files will be mounted as volumes to the Docker process
- The Ansible playbook can be run with the [setup.sh](ansible/setup.sh)
- Sensitive application data like Loggly token, MongoLab credentials are stored as an encrypted [Ansible vault](http://docs.ansible.com/playbooks_vault.html).
- After the preparation the new version of the application can be deployed. The hosting environment is ready

<img src="./pics/ansible.png" width="800"/>

### AWS Elastic Beanstalk ###
- After the initial environment preparation step there will be an AWS Elastic Beanstalk application called ***deliverypipeline*** and the required two environments
- The environments have ***deliverypipeline-node-1*** and ***deliverypipeline-node-2*** names. The first has ***deliverypipeline-prod.elasticbeanstalk.com*** public URL meanwhile the other has ***deliverypipeline-staging.elasticbeanstalk.com***
- Both of the environments will host the Dockerized version of the aforementioned Spray REST application. 
- The dockerized application requires two properties to be configured. ***-Dlogback.configurationFile=/app/logback.xml*** which points to the production logback file contains Loggly configurations. The other is ***-Dakka.configuration=/app/application.conf*** which contains configuration for the mongo backed persistence. Both of the files are mapped within the ***Dockerrun.aws.json*** file as a **volume** for the generated Docker image. That is the way how the live application has proper configuration relevant to the environment

#### Application is installed to AWS Elastic Beanstalk ####

<img src="./pics/ebs1.png" width="800"/>

<img src="./pics/ebs2.png" width="800"/>

<img src="./pics/version.png" width="800"/>

### Continous Integration ###
Before the first deployment the environment should be prepared to host the application! See the previous two paragraph.
When ever I push a modification to the github repository my cloud based CircleCI is going to pick up the modification and build the new version of the application.
The build has the following steps:
- Prepare the build environment hosted by CircleCI
- Deploy required dependencies like awscli
- Run application unit tests and check code coverage
- Upload code coverage results to Coveralls
- Build docker image from the application and run it on CircleCI build box for testing purpose
- Run integration tests agains the previously constructed docker image to very that the image is functional and all the ports are properly exposed. The test currently is just a simple curl request but it could be a more complex integration test or even load test for complex builds. This environment doesn't have ***/app/backlog.xml*** and ***/app/application.conf*** so the application will use the default configuration provided inside the fat jar which by default points to a local LevelDB for storing event stanpshots.
- If all the test pass the image will be uploaded to Dockerhub
- Update Dockerrun.aws.json to point to the newly created docker image version in Dockerhub
- Create a new application version in AWS using the new, modified Dockerrun.aws.json
- Pick the staging environment for deployment based on the public URL
- Deploy the new application version to the staging environment
- After the deployment wait as long as the environment is ready again 
- Swap the staging and production URL. The staging environment will become the new production and vica versa. For the next deployment the new staging system will be used 
- Notifies configured HipChat room about the result of the build process

<img src="./pics/circleci.png" width="800"/>

All the steps described here can be followed in the [CircleCi configuration file](circle.yml) added to the projects root directory. 

There is one extra thing worth mentioning. As you can see in the CircleCI configuration there are couple of referenced environmental variables like AWS keys, Dockerhub credentials. All those variables are coming from the CircleCI project configuration to avoid exposing sensitive data to the wide audience.

### Dockerhub ###
Docker Hub manages the lifecycle of distributed apps with cloud services for building and sharing containers and automating workflows.

<img src="./pics/dockerhub.png" width="800"/>

### HipChat ###
It is always good to have a central place for team communication. It is even better when this channel can be feed by build tools like CircleCI. It send messages about the result of the build to the predefined HipChat room.

<img src="./pics/hipchat.png" width="800"/>

### Loggly ###
Loggly provides a cloud based log management system. Behind the scene **logback** is configured the way to be able to send log messages to Loggly.

<img src="./pics/loggly.png" width="800"/>

### Coveralls ###
Coveralls provides a cloud based code coverage stats for the freshly built application

<img src="./pics/coveralls.png" width="800"/>

## Zero downtime release with a lots of shinny extra features delivered!!! ##
