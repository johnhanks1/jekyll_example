# Setup CodeBuild and CodeDeploy with Jenkins

Replace vaules in walk through with customer values
* #{region} -> region you would like to create resources in
* #{account-id} -> your account id
* #{project-name} -> AWSCodeBuildProject name such as CodeBuildJekyllExample

Assume customer has a working Jenkins server.
## Setup Github repo
Create a new repository

`git clone https://github.com/jekyll/example`
`git remote set-url origin https://github.com/#{github-repo-name}`
`git push`

Now we will add a couple of files that will be used with CodeBuild and Jenkins

buildspec.yml
```
version: 2.0

phases:
  install:
    commands:
      - gem install jekyll jekyll-paginate jekyll-sitemap jekyll-gist
      - bundle install
    build:
      commands:
        - echo "******** Building Jekyll site ********"
        - jekyll build
     post_build:
       commands:
         - echo "******** Uploading to S3 ********"
         - aws s3 sync _site/ s3://jenkinspipelineexample
artifacts:
 files: _site/*
```

Please fill in attributes and save values for use later in the demo
Jenkinsfile
//TODO add CodeDeploy steps to Jenkinsfile
```
pipeline {
  agent any
    stages {
      stage('Build') {
        steps {
          awsCodeBuild projectName: '#{project-name}',
                       credentialsId: '#{credential-id}',
                       credentialsType: 'jenkins',
                       region: '#{region}',
                       sourceControlType: 'jenkins'

        }
      }
    }
}
```

## Setup CodeBuild Resources
Create a bucket for our CodeBuild Artifacts to be publish to:

`aws s3api create-bucket --bucket jekyll-example-artifacts-#{account-id}-#{region} --region #{region} `
//Add cloudformation template but for now console setup
Login to AWSConsole and navigate to:
https://#{region}.console.aws.amazon.com/codesuite/codebuild/project/new?region=#{region}
1. Set Project Name to #{project-name}
2. Select GitHub
3. Follow OAuth flow to connect your GitHub account to CodeBuild
4. Select public repository created above.
5. Navigate to environment select Ubuntu -> Ruby -> aws/codebuild/ruby:2.5.1
6. Enter a role name or choose and existing role in your account
7. Navigate to Artifacts 
  1. Select S3 as an artifact
  2. Choose the bucket we created above: jekyll-example-artifacts-#{account-id}-#{region}
  3. //TODO maybe set artifacts packaging type to zip to use with CodeDeploy.

## Setup Jenkins
Install CodeBuild and CodeDeploy plugins on jenkins server.
1. Navigate in Jenkins Manage Jenkins -> Manage Plugins -> Available Tab
2. Search for AWS CodeDeploy
  * Click Install Checkbox
3. Search AWS CodeBuild
  * Click Install Checkbox
4. Search GitHub
  * Click Install Checkbox
5. Search Blue Ocean
  * Click Install Checkbox
6. Click Download now and install after restart


### Create Jenkins Pipeline
1. Click New Item 
2. Set name to CodeBuildJekyllExample
3. Select Pipeline
4. Click OK
5. Navigate to Build Triggers
  * Click Poll SCM
  * Enter "* * * * *" this will pull github every minute for changes
6. Navigate to Pipeline
  * Click Definition and select Pipeline Script from SCM
  * Click SCM select Git
  * Enter Repository URL
  * Click Save
