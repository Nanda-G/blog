+++
title = "TravisCI: Deploying to server using SSH"
date = "2020-07-26"
+++

## What is Travis-CI?
Well, TravisCI is what we call a Continuous Integration(CI) tool. CI is a practice of merging in small code changes frequently instead of doing it at the end of a development cycle where there is a ton of code and you are bound to run into issues which will be harder to debug.
But, having to do frequent changes makes it harder to run tests and deploy it to your server and that's where Travis steps in. Travis makes it easier to automate your building, deploying and testing stages.

## How does it work?
Travis clones your GitHub repository into a new virtual environment and carries out the series of tasks you write out in your Travis testing file.(.travis.yml)

To learn more about the builds & job lifecycle, check out these two posts:

1. [Builds, Stages, Jobs and Phases](https://docs.travis-ci.com/user/for-beginners/#builds-stages-jobs-and-phases)
2. [The Job Lifecycle](https://docs.travis-ci.com/user/job-lifecycle/#the-job-lifecycle)

## Our Goals

+ Deploy to a server.
+ Run any required scripts on the server if required.

## Assumptions

+ You are using GitHub.

## First-time Travis setup

If it's your first time using Travis then you need to define what OS, distribution and/or language your virtual environment will be using. For example:

  ```
    os: linux
    dist: bionic
    language: python
    python:
      -'3.7'
  ```
      
To read more about this, check out [Travis' Getting Started Guide](https://docs.travis-ci.com/user/tutorial/#to-get-started-with-travis-ci-using-github) and [Building a Python Project Guide](https://docs.travis-ci.com/user/languages/python/) or the guide specific to what programming language you want to use.

## Setup SSH encryption keys

To be able to deploy and run commands on server, you need to have SSH access to it which means you will need an SSH keypair but of course we can't leave our private key lying around in our repository for everyone to see, so we need to encrypt it so that only the Travis CI can read it. For that, we need to use the Travis CLI to encrypt our files. [Here's the relevant Travis Documentation page for it.](https://docs.travis-ci.com/user/encrypting-files/)

1. Install Travis

`$ gem install travis`

2. Log-in to Travis and authorise your GitHub account with it

`$ travis login --com`

The com argument is because we are using travis-ci.com and not travis-ci.org, make sure you are careful around the documentation since travis-ci.com and travis-ci.org have a few differences. [Here is an answer on StackOverflow explaining what the difference is.](https://devops.stackexchange.com/a/4305)

3. Set up SSH keypair

+ Generate the keypair

`$ ssh-keygen -t rsa -b 4096 -C 'build@travis-ci.com' -f ./deploy_rsa`

+ Encrypt the deploy_rsa file

`$ travis encrypt-file deploy_rsa --add`
 
The `--add` flag automatically adds the required lines in the .travis.yml file.

+ Install Public SSH key on server

`ssh-copy-id -i deploy_rsa.pub <USERNAME>@<DEPLOY-HOSTNAME>`

+ DELETE the keypair

`rm -f deploy_rsa deploy_rsa.pub`

This is very IMPORTANT, make sure that the private key is not on the repository.

Your .travis.yml file must be looking something like this:

```
addons:
  ssh_known_hosts: <DEPLOY-HOSTNAME>

before_deploy:
- openssl aes-256-cbc -K $encrypted_<...>_key -iv $encrypted_<...>_iv -in deploy_rsa.enc -out /tmp/deploy_rsa -d
- eval "$(ssh-agent -s)"
- chmod 600 /tmp/deploy_rsa
- ssh-add /tmp/deploy_rsa
```

Alright, before we go ahead with deployment, lets add some secrets to the repository, you can do it two ways.

1. Using the Travis CLI tool:

`travis encrypt --pro SOMEVAR="secretvalue"`

But, that adds the values in the .travis.yml file which looks kinda ugly so I chose to add it using the travis-ci.com website, it really depends on what you want to do.

2. Using the travis-ci.com website

..* Go to your TravisCI dashboard and pick the repository you want to work with. Click on the "More Options" hamburger menu and go into settings. There you will see an Environment Variables section, add your server's HOSTNAME and USERNAME to it just so you don't reveal your IP address in the .travis.yml file. ;)

## Deployment

Before we start writing the deployment script, it would be nice to create a `scripts/` folder and then place a `deploy.sh` file in that folder. Yes, the deployment script is just a single line but it would be easier to add more commands in there without populating up the `.travis.yml` file.

The single line you would use to deploy is:

`ssh -i /tmp/deploy_rsa $USERNAME@$HOSTNAME "cd <repo-directory>; git pull origin <branch-name>"`

We need to put this script somewhere in the .travis.yml file, but it can't be just anywhere. We need to follow Travis' build lifecycle events. So, we will use the deploy script lifecycle event which looks like this:

```
deploy:
    - provider: script
      skip_cleanup: true
      script:  scp -r $TRAVIS_BUILD_DIR $USERNAME@$HOSTNAME:/path/to/repository/files
      on:
      branch: master
```

Or, if you created a separate `scripts/deploy.sh`, add the single `scp` command to the deploy.sh file and then your deploy script would look something like this:

```
deploy:
    - provider: script
      skip_cleanup: true
      script: bash scripts/deploy.sh
      on:
        branch: master
```

That's all for deployment and if you want to run any commands post deployment, you could add an extra line in your deploy script:

`ssh -i /tmp/deploy_rsa $USERNAME@$HOSTNAME  "cd <repo-directory>; <run any command(s) or script(s)>;"`

Make sure there aren't any process that linger or else the command will not exit and the build will fail.

### Finishing up the script

Just above both the `before_deploy:` stage you would include a `jobs:` section since the `jobs:` section entails a sequence of stages, i.e., your `before_deploy:` and `deploy:` stage.

It would look something like this:

```
jobs:
    include:
      - stage: deploy
        if: branch = master AND type != pull_request
        script: skip
```
        
What this does is that it runs the script only on a push to master and NOT if it's a pull request.

If you want to deploy multiple branches, you could add an extra condition, for example:
`if: branch = master OR branch=testing AND type != pull_request`

And your deploy script would look like:

```
deploy:
       - provider: script
         skip_cleanup: true
         script: bash scripts/deploy_master.sh
         on:
           branch: master
       - provider: script
         skip_cleanup: true
         script: bash scripts/deploy_testing.sh
         on:
           branch: testing
```

## Conclusion

I hope you were able to understand how Travis works, its workflow and how it makes it very easy to deploy and run things without putting in much effort.

Make sure to put some time reading the Travis Documentation, they are really good and it will generally help you do things properly and make the entire process much more effortless.

For example, if someone creates a pull request but their code is really ugly and not properly linted, or want to run tests to make sure nothing broken is being pushed to master, you could add an extra linting and testing stage just above the deploy stage. It would look something like this:

```
jobs:
   include:
   - stage: flake8 test
     if: type = pull_request
     script: flake8 .
```
     
This runs a flake8 styling test and if the code doesn't abibe by it, the Travis build will fail.

Here's an example .travis.yml that I use to push a [Telegram Bot written in Python](https://github.com/Super-Serious/bot) to a server:

https://github.com/Super-Serious/bot/blob/master/.travis.yml

## Contact

If you find something wrong in the article or have any doubts, please don't hesitate to [contact me.](https://blog.vnandag.me/about/#need-to-get-in-contact-with-me)


Thank you for reading!
