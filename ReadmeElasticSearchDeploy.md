Deploying to Elastic Beanstalk via CircleCi 2.0
------

I got to here after spending hours trying to deploy to an Elastic Beanstalk instance via CircleCi 2.0 so I thought I'd write up what worked for me to hopefully help others. Shout out to RobertoSchneiders who's [steps for getting it to work with CircleCi 1.0](https://gist.github.com/RobertoSchneiders/9e0e73e836a80d53a21e) were my starting point. 

For the record, I'm not the most server-savvy of developers so there may be a better way of doing this.


#### Setup a user on AWS IAM to use for deployments
- [Add user here](https://console.aws.amazon.com/iam/home#/users$new?step=details)
- Set a username and select `Programmatic access` as the Access type
- Click 'Create group' on the user permissions page
- Set a group name and search for the `AdministratorAccess-AWSElasticBeanstalk` and `AmazonS3FullAccess` policies and select them
    - Note: This used to be just `AWSElasticBeanstalkFullAccess`, but that's since been deprecated
- Create the group so it's assigned to your new user 
- Review and create the user 


#### Setup an Elastic Beanstalk application 
- 'Create New Application'
- 'Create New Environment'
- 'Create web server'
- On the Environment Information page where it asks you to set the 'Environment name', set it to something with the Git branch name in e.g. `BRANCHNAME-my-application`
    - I do this as I have a staging branch and the master branch so in our EB config, we'll be replacing `BRANCHNAME` with the $CIRCLE_BRANCH environment variable provided by CircleCi so when deploying the staging branch for example, it will know to deploy to the `staging-my-application` environment on Elastic Beanstalk
- Follow the rest of the setup as you require 


#### Add deployment user environment variables to CircleCi
- Project Settings > Environment Variables
    - AWS_ACCESS_KEY_ID
    - AWS_SECRET_ACCESS_KEY


#### Add `.elasticbeanstalk/config.yml` config to application code 
- Update the values in the below snippet to match what you setup. 

```
branch-defaults:
  master:
    environment: $CIRCLE_BRANCH-my-application
global:
  application_name: My Application
  default_ec2_keyname: XXXXXX
  default_platform: 64bit Amazon Linux 2017.09 v2.6.2 running Ruby 2.4 (Passenger Standalone)
  default_region: eu-west-1
  sc: git
```

Note: Ensure the `application_name` is exactly what you called your application in Elastic Beanstalk when you did the 'Create New Application' step.

Also Note: Do not set a `profile:` value here, the profile will be set based on the AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY environment variables you setup.


#### Update your `.circleci/config.yml`
- The following is what I think is the bare minimum I needed to get only the `master` or `staging` branches on your Git repo to deploy.
- Update `$CIRCLE_BRANCH-my-application` to the environment name you set in Elastic Beanstalk.

```
version: 2
jobs:
  deploy:
    working_directory: ~/app
    docker:
      - image: circleci/ruby:2.4.3
    steps:
      - checkout

      - run:
          name: Installing deployment dependencies
          working_directory: /
          command: |
            sudo apt-get -y -qq update
            sudo apt-get install python3-pip python3-dev build-essential
            sudo pip3 install awsebcli

      - run:
          name: Deploying
          command: eb deploy $CIRCLE_BRANCH-my-application

workflows:
  version: 2
  build:
    jobs:
      - deploy:
          filters:
            branches:
              only:
                - staging
                - master

```


#### Cross fingers
- Commit and let CircleCi do it's thing. If all goes well you should see it updating on the Elastic Beanstalk dashboard as the 'Deploying' step is running in CircleCi.