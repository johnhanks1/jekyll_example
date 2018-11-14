# Setup CodeBuild and CodeDeploy with Jenkins

Replace vaules in walk through with customer values
* #{region} -> region you would like to create resources in
* #{account-id} -> your account id
* #{project-name} -> AWSCodeBuildProject name such as CodeBuildJekyllExample

Assume customer has a working Jenkins server.
## Setup Github repo
Create a new repository

`git clone https://github.com/jekyll/example`

`cd example`

`git checkout -b master`

`git remote set-url origin https://github.com/#{github-repo-name}`

`git push --set-upstream origin master`

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
artifacts:
 files: _site/*
```
## Setup CodeBuild Resources
Create a bucket for our CodeBuild Artifacts to be publish to:

`aws s3api create-bucket --bucket jekyll-example-artifacts-#{account-id}-#{region} --region #{region}  --create-bucket-configuration LocationConstraint=#{region}`

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


### Setup Jenkins pipeline resources 
Create an IAM user that can call CodeBuild
1. Navigate to https://console.aws.amazon.com/iam/home?region=#{region}#/home
2. Select **Users**
3. Click **Add user**
4. Add *name* and select **Programmatic access** check box
5. Click **Next: Permissions**
6. Click **Attach Existing Policies directly**
  1. Search for **AWSCodeBuildAdminAccess**
  2. Click check box
7. Click **Next: Review**
8. Click **Create User**
9. Note Access Key Id and Secret Access Key 

### Add user to Jenkins

1. Navigate To Jenkins main menu
2. Click **Credentials**
3. Click **System**
4. Click **Global credentials**
5. Click **Add Credentials**
6. Click **Kind** and select **CodeBuild Credentials**
7. Enter **ID** and note the id
8. Add AccessKey and Secret Access Key from above leaving all other areas blank.
9. Click **OK**


We now need to add a Jenkinsfile to our repository. This will have the information that
jenkins will need to call CodeBuild.

Jenkinsfile

Make sure to replace values with noted values from above

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
                       sourceControlType: 'project'

        }
      }
    }
}
```

### Push Changes into repo
`git add buildspec.yml`

`git add Jenkinsfile`

`git push`


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

