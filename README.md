# Setup CodeBuild and CodeDeploy with Jenkins

Replace vaules in walk through with customer values
* #{region} -> region you would like to create resources in
* #{account-id} -> your account id
* #{project-name} -> AWSCodeBuildProject name such as CodeBuildJekyllExample

Assume customer has a working Jenkins server.
## Setup Github repo
Create a new repository

`git clone https://github.com/jekyll/jekyll`

`cd example`

`git checkout -b master`

`git remote set-url origin https://github.com/#{github-repo-name}`

`git push --set-upstream origin master`

Now we will add a couple of files that will be used with CodeBuild and Jenkins

buildspec.yml
```
version: 0.2

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
 discard-paths: yes
```
## Setup CodeBuild Resources
Create a bucket for our CodeBuild Artifacts to be publish to:

1. Navigate to https://s3.console.aws.amazon.com/s3/home?region=#{region}
2. Click **Create Bucket**
3. Enter Bucket name
4. Click **Next**
5. Select Versioning
6. **Click Next**
7. Click **Next**
8. Click Create Bucket
10. Select **Bucket From list**
11. Select **Properties Tab**
12. Select Static website Hosting
13. Select **Use this Bucket to host a website**
14. Add **index.html** to **Index Document**
15. Add **404.html** to **Error Document**
16. Note the endpoint
16. Click **Save**

Add Public Read to bucket 
1. Click **Permissions**
2. Click **Bucket Policy**
3. Paste policy below make sure to replace bucket name

```
{
    "Version": "2008-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::#{bucket-name}/*"
        }
    ]
}
```

`aws s3api create-bucket --bucket jekyll-example-artifacts-#{account-id}-#{region} --region #{region}  --create-bucket-configuration LocationConstraint=#{region}`

Login to AWSConsole and navigate to:
https://#{region}.console.aws.amazon.com/codesuite/codebuild/project/new?region=#{region}
1. Set **Project Name** to #{project-name}
2. Navigate to Source section
  1. Select **GitHub**
  2. Follow OAuth flow to connect your GitHub account to CodeBuild
3. Select the repository created above.
4. Navigate to environment select **Ubuntu -> Ruby -> aws/codebuild/ruby:2.5.1**
5. Navigate to **Artifacts**
  1. Select **Amazon S3** as an artifact type
  2. Choose the bucket we created above: jekyll-example-artifacts-#{account-id}-#{region}
  3. Click Checkbox **Remove Artifact Encryption**
6. Click **Create build Project**

## Setup Jenkins
Install CodeBuild and CodeDeploy plugins on jenkins server.
1. Navigate in Jenkins to **Manage Jenkins -> Manage Plugins -> Available Tab**
2. Search for **AWS CodeDeploy**
  * Click **Install Checkbox**
3. Search **AWS CodeBuild**
  * Click **Install Checkbox**
4. Search **GitHub**
  * Click **Install Checkbox**
5. Search **Blue Ocean**
  * Click **Install Checkbox**
6. Click **Download now and install after restart**


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
2. Set **Name **
3. Select **Pipeline**
4. Click **OK**
5. Navigate to **Build Triggers**
  * Click **Poll SCM**
  * Enter "* * * * *" this will pull github every minute for changes
6. Navigate to **Pipeline**
  * Click Definition and select **Pipeline Script from SCM**
  * Click **SCM** select **Git**
  * Enter **Repository URL**
  * Click **Save**
7. Now wait up to 1 minute and a build should be kicked off.

