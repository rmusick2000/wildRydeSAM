# WildRydeSAM

YAML templates for AWS's serverless web app tutorial 'Wild Rydes'.

The Wild Rydes tutorial is a step by step walkthrough of creating a serverless web app for unicorn rides, by using the AWS console.  The tutorial can be found here:

https://aws.amazon.com/getting-started/projects/build-serverless-web-app-lambda-apigateway-s3-dynamodb-cognito/

This project contains working AWS SAM templates for the Wild Rydes SAM tutorial above.  The YAML files found here are a SAM template approach to creating the same application from the command line, rather then click-by-click on the AWS console.


## Files:

   #### S3StaticWeb.yaml:
   SAM template to create an S3 bucket for hosting a static website.

   #### wildRyde.yaml:
   SAM template to create the serverless resources needed for the wildRydes web app.

   #### RequestUnicorn.js:
   The lambda handler for wildRydes.
   
   #### package.json:
   Needed to package and build the lambda handler.
   
   #### README.md:
   This file.


## Directions for use (Linux):

NOTE: us-east-1 region has been used throughout the example below.  Replace this if needed with the AWS region of your choice.

1. Install and test the command line interface for AWS SAM.

   https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/serverless-sam-cli-install.html

   Note the name of your SAM deployment bucket.   Below this is referred to as \<mySAMDeployBucket\>.


2. install npm, nodejs for 'sam build' step.

   For ubuntu: 
   ```bash
   sudo apt-get update
   sudo apt install npm
   sudo npm install -g npm@latest
   sudo apt-get install nodejs
   ```


3. Create & populate wildRyde directory structucture:

   Change to the github directory for this project.  For example: 
   ```bash
   cd ~/wildRydeSAM
   ```

   Get static wildRyde web files.
   ```bash
   aws s3 sync s3://wildrydes-us-east-1/WebApplication/1_StaticWebHosting/website wildRydeApp/.
   cd wildRydeApp
   ```

   Create subdirectory for the lambdaHandler.  We want this to be a small leaf directory, to minimize the data transfer that occurs in the 'sam package' step.  NOTE: the original RequestUnicorn.js file can be found here: https://github.com/aws-samples/aws-serverless-workshops/blob/master/WebApplication/3_ServerlessBackend/requestUnicorn.js
   ```bash
   mkdir lambdaHandler
   mv ../RequestUnicorn.js lambdaHandler/.
   mv ../package.json lambdaHandler/.
   ```


4. Create a wildRyde static web bucket with SAM.

   Change to the github directory for this project.  For example: 
   ```bash
   cd ~/wildRydeSAM
   ```

   Edit S3StaticWeb.yaml, changing \<myStaticWebBucketName\> to the external-facing name of your bucket (i.e 'mydomain.net'), and changing \<myStaticWebTemplateName\> to the alpha-numeric internal logical name (i.e 'myDomainNet'). Note, bucket names are unique and lower-case only.  It make take a few tries to get an unused name.
   ```
   <EDIT S3StaticWeb.yaml>
   ```

   Validate.  NOTE that validation catches some template errors, but not all.  For example, SAM templates are space-sensitive.  The 'Events' block is valid both as a peer of Properties, or as a child of Properties.  In our case, only the child version is correct - the peer version fails without useful warnings.
   ```bash
   sam validate -t S3StaticWeb.yaml
   ```

   Package, and deploy, replacing \<mySAMDeployBucket\> with the bucket created in step 1.
   ```bash
   sam package --template-file S3StaticWeb.yaml --output-template S3StaticWebPack.yaml --s3-bucket <mySAMDeployBucket>
   sam deploy --template-file S3StaticWebPack.yaml --region us-east-1 --capabilities CAPABILITY_IAM --stack-name S3StaticWeb
   aws cloudformation describe-stacks --stack-name S3StaticWeb
   ```

5. Create a wildRyde serverless infrastructure with SAM. 

   Change to the github directory for this project.  For example: 
   ```bash
   cd ~/wildRydeSAM
   ```

   Edit wildRyde.yaml to update the CodeUri location under WRLambda to be consistent with step 3.
   ```
   <EDIT wildRyde.yaml>
   ```

   Validate, build, package, deploy, replacing  \<mySAMDeployBucket\> with the bucket created in step 1.
   ```bash
   sam validate -t wildRyde.yaml
   sam build -t wildRyde.yaml
   sam package --template-file wildRyde.yaml --output-template wildRydePack.yaml --s3-bucket <mySAMDeployBucket>
   sam deploy --template-file wildRydePack.yaml --region us-east-1 --capabilities CAPABILITY_IAM --stack-name wildRydeApp
   ```

6. Connect the static pages created in step 4 with the serverless resources created in step 5.

   Change to the github directory for this project.  For example: 
   ```bash
   cd ~/wildRydeSAM
   ```

   Print a few of the AWS resource IDs created in step 5.
   ```bash
   aws cloudformation describe-stacks --stack-name wildRydeApp
   ```

   Edit wildRydeApp/js/config.js that you created in step 3.  Note, this connects cognito authorization, and the backend lambdaHandler offered by the serverless API, to the static web pages.
   ```
   <EDIT wildRydeApp/js/config.js, replacing values with those printed out in the 'describe-stacks' command above>
   ```

   This is an example of a fully configured config.js file
   ```javascript
      window._config = {
         cognito: {
             userPoolId: 'us-east-1_8nV7gSCxH',
             userPoolClientId: '25jg7hgjom7qkbu409bj913fc8',
             region: 'us-east-1' 
         },
         api: {
             invokeUrl: 'https://ufs3aiw8sl.execute-api.us-east-1.amazonaws.com/prod'  # WildRydeApiExecution
         }
      };
   ```

   Copy the static web files upstream to your S3 static web bucket, replacing \<myStaticWebBucketName\> with the bucket created in step 3.
   ```bash
   aws s3 sync wildRydeApp s3://<myStaticWebBucketName> --region us-east-1 
   ```

7. Test WildRydesSam!

   Find the 'WebsiteURL' value in the 'describe-stacks' output in step 6.
   ```
   <ENTER the 'WebsiteURL' value into your browser, and test>
   ```

8. Delete resources.
   ```bash
   aws cloudformation delete-stack --stack-name wildRydeApp
   ```

   wait for wildRydeApp to be deleted, else this fails.
   ```
   <WAIT>
   ```   

   Stack delete fails unless bucket is force-deleted first.
   ```bash
   aws s3 rb s3://<myStaticWebBucketName> --force
   aws cloudformation delete-stack --stack-name S3StaticWeb
   ```
