# Setup CodeBuild with Jenkins
### Prerequisites
* You have an AWSAccount

#### Spin up a Jenkins Server in that account
1. Navigate to IAM Console
2. Click **Role**
3. Click **Create Role**
4. Choose **EC2** as the service that will use your role
5. Click **Next:Permissions**
6. Give the role **AWSCodeBuildAdminAccess**, **AmazonS3FullAccess**, and **CloudWatchLogsFullAccess** 
7. Click **Next:Tags**
8. Click **Next:Review** 
9. Name role **CodeBuildDemo**
10. Click **Create Role**

Create Jenkins Server:
1. Navigate to CloudFormation Console
2. Click **Create Stack**
3. Click **Specify an Amazon S3 template URL** and use `https://s3.amazonaws.com/proberts-public/jenkins_build.yaml`
4. Fill in each value for the template and create the stack.
5. You will receive the password to login to your jenkins host on your phone.
6. Navigate to EC2 and find the instance name Jenkins public DNS
7. Connect to the dns at port 8080

Replace values in walk through with customer values
* #{region} -> region you would like to create resources in
* #{account-id} -> your account id
* #{project-name} -> AWSCodeBuildProject name such as CodeBuildJekyllExample

Assume customer has a working Jenkins server.
## Setup Github repo
Create a new repository

`git clone https://github.com/johnhanks1/jekyll_example`

`cd jekyll_example`

`git checkout -b master`

`git remote set-url origin https://github.com/#{github-repo-name}`

or if using ssh for auth

`git remote set-url origin git@github.com:#{github-repo-name}`

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
 files: "**/*"
 base-directory: _site
```
## Setup CodeBuild Resources
Create a bucket for our CodeBuild Artifacts to be publish to:

1. Navigate to https://s3.console.aws.amazon.com/s3/home?region=#{region}
2. Click **Create Bucket**
3. Enter Bucket name
4. Click **Next**
5. Select Versioning
6. Click **Next**
7. uncheck the checkbox next to **Block new public bucket policies (Recommended)**
8. uncheck the checkbox next to **Block public and cross-account access if bucket has public policies (Recommended)**
9. Click **Next**
10. Click **Create Bucket**
11. Select **Bucket From list**
12. Select **Properties Tab**
13. Select Static website Hosting
14. Select **Use this Bucket to host a website**
15. Add **index.html** to **Index Document**
16. Add **404.html** to **Error Document**
17. Note the endpoint
18. Click **Save**

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
  3. In the **Path** text box enter a **.**
  4. Click Checkbox **Remove Artifact Encryption**
6. Click **Create build Project**

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
2. Set **Name**
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
8. Make sure build succeeded
9. Check S3 Bucket website and see the newly created blog


### Update Blog
We will now create a new post that will get automatically built.
1. Open **_posts/2018-11-14-welcome-to-jekyll.markdown**
2. Edit the posts Title and contents
3. Commit and push changes to repository
4. Watch for build to automatically be triggered by Jenkins
5. Once build is complete look at the S3 website to see the update
