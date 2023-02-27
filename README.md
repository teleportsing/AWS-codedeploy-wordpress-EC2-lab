# AWS-codedeploy-wordpress-EC2-lab
Deploy the WordPress application to an Amazon EC2 instance using Amazon codedeploy.

# Overview

This lab introduces you to AWS Codedeploy In this lab you will use AWSCodedeploy to deploy an application to an Amazon EC2 instance

# Topics covered
 By the end of this lab , you will be able to:

  * Verify that the Codedeploy agent has been installed
  * Configure application source content to be deployed to Codedeploy
  * Create an Amazon S3 bucket , then upload a Wordpress application to the bucket
  * Deploy a Wordpress application to an Amazon EC2 instance
  * Monitor a Wordpress application deploymentUpdate a Wordpress application and then redeploy it

# AWS Codedeploy

 AWS Code Deploy is a deployment service that automates applicationdeployments to Amazon EC2 instances, on-premises instances, or serverlessambda functions

 You can deploy a nearly unlimited variety of application content, such as code,serverless AWS Lambda functions, web and configuration files, executables,packages, scripts, multimedia files, and so on. AWS Codedeploy can deployapplication content that runs on a server and is stored in Amazon S3 buckets,Github repositories, or Bitbucket repositories. AWS Codedeploy can also deploya serverless Lambda function. You do not need to make changes to yourexisting code before you can use AWS Codedeploy

 AWS Code Deploy makes it easier for you to:

  * Rapidly release new features
  * Update AWS Lambda function versions
  * Avoid downtime during application deployment
  * Handle the complexity of updating your applications , without many of therisks associated with error-prone manual deployments


# Task 1 : Connect to your Linux EC2instanceAs 

As part of this lab , a Linux EC2 Instance has been automatically created

 In this task you will connect to your EC2 instance using AWS Systems Manager - Session manager.
  * Copy the Ec2lnstancesessionurl value from the list to the left of thesenstructions and paste it into a new web browser tab

A new web browser tab opens with a AWS Systems Manager-sessionManager console connecting to your EC2 instance . A set of commands are runautomatically when you connect to the instance that change to the users homedirectory and display the path of the working directory , similar to this

```
  /home/ec2-user
  [ec2-user@ip-10-0-1-137~]$
 `

<i class="far fa-thumbs-up" style="color:#008296"></i> Congratulations! You have successfully connected to a AWS System Manager Session Manager terminal session on the lab EC2 instance. Visit the link to know more about [Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)

---

# Task 2: Check your environment

In this task, you will verify that:

- The CodeDeploy agent is running on your EC2 instance.
- Your EC2 security group has been configured properly

### Verify the CodeDeploy agent is running

3. [3]Ensure the CodeDeploy agent is running on your instance.

`bash
sudo service codedeploy-agent status
```


The AWS Code Deploy agent is a software package that , when installed andconfigured on an instance , enables that instance to be used in AWSCodedeploy deployments The AWS Codedeploy agent is required only if youdeploy to an EC2 / On-premises compute platform . The agent is not required fordeployments that use the AWS Lambda compute platform

Check your security groups

4. At the top of the AWS Management Console , in the search bar , search forand choose EC2

5. In the left navigation pane , click Security Groups

6. Select Codedeploysg

7. Click the Inbound rules tab

You should see a rule that allows HTTP access from Source-0.0.0.0/0


# Task 3 : Configure your applicationsource content  

 In this task you will:
  
  * Configure your source application content to be deployed

  * Create scripts that will run when your application is deployed

  * Create a Codedeploy Appspec file


Download the Wordpress code

8. Copy the Wordpress source code to your Linux instance.

```
wget https://s3-us-west-2.amazonaws.com/us-west-2-aws-training/courses/spl-82/v1.4.5.prod-2bbeba30/scripts/WordPress-master.zip
```

9. Unpack the master .zip file into the / tmp / Wordpress _ Temp folder

```
unzip Wordpress-master .zip - d /tmp/Wordpress_Temp
```

10. Create a Wordpress destination folder

```
mkdir -p /tmp/WordPress
```

11. Copy its unzipped contents to the / tmp / Word Press destination folder

```
cp -paf /tmp/WordPress_Temp/WordPress-master/* /tmp/WordPress
```

12. Delete the temporary / tmp / Wordpress _ Temp folder and master file

```
rm -rf /tmp/WordPress_Temp
rm -f WordPress-master.zip
```


Create scripts to run your application

13. Create a scripts directory in your copy of the Wordpress source code

```
mkdir -p /tmp/WordPress/scripts
```

14. Create an install _ dependencies . sh script by running

```
nano /tmp/WordPress/scripts/install_dependencies.sh
#!/bin/bash
yum install -y httpd php mariadb-server php-mysqlnd
```

15. Save the dependencies.sh script and exit nano by:

 * Pressing Ctrl + X
 * ntering y
 * Pressing Enter


 The install _ dependencies . sh script installs Apache , Mariadb , and PHP. It alsoadds MYSQL support to PHP.

 16. Create a start _ server . sh script by running

```
nano /tmp/WordPress/scripts/start_server.sh
#!/bin/bash
service httpd start
service mariadb start
```

This start _ server . sh script starts Apache and MariaDB.


17. Save the start server sh file and exit nano:

 * Pressing Ctrl + X
 * Entering y
 * Pressing Enter

18. Create a stop _ server . sh script by running:

```
nano /tmp/WordPress/scripts/stop_server.sh
#!/bin/bash
isExistApp=`pgrep httpd`
if [[ -n  $isExistApp ]]; then
   service httpd stop
fi
isExistApp=`pgrep sql`
if [[ -n  $isExistApp ]]; then
    service mariadb stop
fi
```

19. Save the stop _ server.sh script and exit nano

The stop_server.sh script stops Apache and MariadbDB. 

20. Create a create _ test _ db . sh script by running:

```
nano /tmp/WordPress/scripts/create_test_db.sh
#!/bin/bash
mysql -uroot <<CREATE_TEST_DB
CREATE DATABASE IF NOT EXISTS test;
CREATE_TEST_DB
```

21. Save the create_test_db.sh script and exit nano

This create test_db.sh script uses MariaDB to create a test database for Wordpress to use.

22. Create a change_permissions.sh script by running:

```
nano /tmp/WordPress/scripts/change_permissions.sh
#!/bin/bash
chmod -R 777 /var/www/html/WordPress
```

23. Save the change_permissions.sh file and exit nano

This script is used to change the folder permissions in Apache

24. Make all of your scripts executable

```
chmod +x /tmp/WordPress/scripts/*
```

25. List your scripts.

```
ls -la /tmp/WordPress/scripts
```

You should see five scripts that are executable



Create a Codedeploy Appspec file

 AWS Codedeploy uses the appspec.yml file to:

  * Map the source files in your application revision to their destinations on thetarget Amazon EC2 instance

  * Specify custom permissions for deployed files

  * Specify scripts to be run on the target Amazon EC2 instance during thedeployment

The Appspec file must be named appspec yml It must be placed in the rootdirectory of the applications source code . In this lab , the root directory is/tmp/Wordpress


The App Spec file is unique to AWS Code Deploy . It defines the deploymentactions you want AWS Codedeploy to execute You bundle your deployablecontent and the Appspec file into an archive file , and then upload it to anAmazon S3 bucket or a Github repository . This archive file is called anappllcation revision (or Simply a revision).


26. Create an application specific file in/tmp/Wordpress


27. Add the following text to your appspec.ym/file , then save


```
version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html/WordPress
hooks:
  BeforeInstall:
    - location: scripts/install_dependencies.sh
      timeout: 300
      runas: root
  AfterInstall:
    - location: scripts/change_permissions.sh
      timeout: 300
      runas: root
  ApplicationStart:
    - location: scripts/start_server.sh
    - location: scripts/create_test_db.sh
      timeout: 300
      runas: root
  ApplicationStop:
    - location: scripts/stop_server.sh
      timeout: 300
      runas: root
```


28. Save the appspec yml file and exit nano


In your appspec yaml file you should see the scripts that your created earlierThese scripts will be run when your application is deployed.



# Task 4: Deploy your Wordpress application

In this task , you will:


 * Create an Amazon S3 bucket to host your application source content

 * Upload your application source content to your Amazon S3 bucket

 * Deploy your Wordpress application to an instance running in youenvironment named Codedeploy



In AWS Codedeploy , a deployment is the process , and the componeinvolved in the process , of installing content on one or more instances . Thiscontent can consist of code , web and configuration files , executables ,packages , scripts , and so on . AWS Codedeploy deploys content that is storeda source repository , according to the configuration rules you specify.


Create your Codedeploy application29. 

29. Create a Codedeploy application

`aws deploy create-application --application-name WordPress_App`


Provision an Amazon S3 bucket


Amazon S3 is object storage built to store and retrieve any amount of datafrom anywhere on the Internet . Its a simple storage service that offers anextremely durable , highly available , and infinitely scalable data storageinfrastructure at very low costs.


30. AWS Systems Manager-session manager tab , create an S3 bucket by:

 * Pasting aws s3 mb s3://codedeploybucketnumber

 * Replacing NUMBER with a random number

 * Pressing ENTER


31. Copy the name of your S3 bucket to your text editor

32. Navigate to the folder that contains your Wordpress files

`cd /tmp/WordPress`


Upload your Wordpress application to Amazon S3


33. Paste the following push command into your text editor


```
aws deploy push --application-name WordPress_App --description "This is a revision for the application WordPress_App" --ignore-hidden-files --s3-location s3://BUCKET/WordPressApp.zip --source .
```

34. Replace BUCKET With your bucket name

35. Copy and paste the updated push command into your AWS SystemManager Session Manager terminal session

36. Press Enter


This command does the following:


 * Associates the bundled files with an application named Wordpress_App

 * Attaches a description to the revision

 * Igores hidden files

 * Names the revision Wordpressapp zip and pushes it to a bucket yourbucket

 * Bundles all files in the root directory into the revision


Deploy your application

37. Paste the following create-deployment-group command into your text editor

```
aws deploy create-deployment-group --application-name WordPress_App --deployment-config-name CodeDeployDefault.AllAtOnce --deployment-group-name WordPress_DG --ec2-tag-filters Key=Name,Value=CodeDeploy,Type=KEY_AND_VALUE --service-role-arn ROLE
```

38. Replace ROLE with the value of Codedeployrolearn located to the left ofthese instructions

39. Copy and paste the updated create-deployment-group command into yourerminal session

40. Press Enter.

This creates your Codedeploy deployment group

In an EC2 / On-premises deployment , a deployment group is a set ofndividual instances targeted for a deployment . A deployment group containsdividually tagged instances , Amazon EC2 instances in Amazon EC2 AutoScaling groups , or both . In an AWS Lambda deployment , a deployment groupdefines a set of AWS Codedeploy configurations for future serverless Lambdadeployment to the group


41. Paste the following create-deployment command into your text editor.


```
aws deploy create-deployment --application-name WordPress_App --s3-location bucket=BUCKET,key=WordPressApp.zip,bundleType=zip --deployment-group-name WordPress_DG  --description "This is a revision for the application WordPress_App"
```


42. Replace BUCKET With your bucket name

43. Copy and paste the updated create-deployment command into your terminalsessio

44. Press Enter

This will deploy your Wordpress application


# Task 5 : Monitor your deployment

 In this task , you will monitor your Codedeploy deployment using the AWS Codedeploy management console

 AWS provides various tools that you can use to monitor AWS Codedeploy Youcan configure some of these tools to do the monitoring for you , while some ofthe tools require manual intervention . We recommend that you automatemonitoring tasks as much as possible

 You can also use the following automated monitoring tools to watch AWS Codedeploy and report when something is wrong

  * Amazon Cloudwatch Alarms - Watch a single metric over a time periodthat you specify , and perform one or more actions based on the value of themetric relative to a given threshold over a number of time periods . Theaction is a notification sent to an Amazon Simple Notification ServiceAmazon SNS ) topic or Amazon EC2 Auto Scaling policy . Cloud Watchalarms do not invoke actions simply because they are in a particular statethe state must have changed and been maintained for a specified numberf periods

  * Amazon Cloudwatch Events - Match events and route them to one or moretarget functions or streams to make changes , capture state information ,and take corrective action

  * AWS Cloud Trail Log Monitoring - Share log files between accounts , monitorCloud Trail log files in real time by sending them to Cloudwatch Logs , writelog processing applications in Java , and validate that your log files have nochanged after delivery by CloudTrai

  * Amazon Simple Notification Service-configure event-driven triggers toreceive SMS or email notifications about deployment and instance events ,such as success or failure


45. At the top of the AWS Management Console , in the search bar search forand choose Codedeploy


You should see your application (Word Press_App) . If you do not see yourapplication , verify that you are in the same region that you created you Codedeploy application in . The region code that you created your Codedeploy application in is located to the left of these instruction next to Region . You canuse the [AWS Regions and Endpoints](https://docs.aws.amazon.com/general/latest/gr/rande.html) page to identify the region name thatrepresents your region code . Once you know which region your application wascreated in , you can navigate to it by:


 * Clicking the drop-down vto the left of `Awslabsuser-` at the top
 * Clicking the correct region

 On the Deployments page in the AWS Codedeploy console , you can monitoryour deployments status in the Status column

 46. Scroll down to the Deployment lifecycle events page

 Here you can see the instance id that was used to deploy your application

 47. Click the Instance ID link:

 This is navigate you to the instance that was used to deploy your applicationAs you can see , your application was deployed onto your Codedeploy instance.

 48. Close the tab for your EC2 instancethe 

 49. Codedeploy tab , click the View events link

 On this page , you can monitor all of the events of your deployment

 50. In the left navigation pane , click Deployment

 51. Wait for the Deployment status to display ✅ Succeeded

 52. Verify that you can access your Wordpress application by:


 * Copying the value of Ec2publicip located to the left of these instruction

 * Pasting the value into a new browser tab

 * Appending/Wordpress

 * Pressing ENTER


The value you enter into your browser tab should look similar to: http://34.1.1.1/Wordpress


# Task 6 : Update and redeploy yourWordpress application

Now that you've successfully deployed your application revision , update theWordpress code on the development machine , and then use AWS Codedeployto redeploy the site . Afterward , you should see the code changes on theAmazon EC2 instance


Finish setting up your Wordpress site 


To see the effects of the code change , your will first finish setting up theWordpress site so that you have a fully functional installation


53. On your Wordpress site , click `Let's go!` then configure:

 * Database Name: `test`
 * Username: `root`
 * Password: Delete `password` so that the field is empty
 * Click `Submit`


 54. Click `Run the installation`


 55. At the Welcome page configure:

 * Site Title: `mySite`
 * username: `user1`
 * password: `Enter a password that youmembe`
 * Your Email: `Enter your email address`
 * Click `Install Wordpress`


56. Once Wordpress has been installed, sign into your Wordpress site

57. At top-left of the your screen , hover over mysite , then click Visit Site

The backaround should be White



Modify and redeploy your Wordpress site


In this task , you will modify your Wordpress site and then redeploy it using Codedeploy

During the deployment of the Wordpress application , thechange_permissions.sh script updated permissions of the /tmp/Wordpress folder so that anyone can write to it.


58. In your AWS Systems Manager-session manager tab , run the followingcommand to restrict permissions so that only you , the owner , can write to it.

`sudo chmod -R 755 /var/www/html/WordPress`


59. Navigate to the Wordpress directory

`cd /tmp/WordPress`

60. Modify some of the sites colors , in the wpcontent/themes/twentyfifteen/style .css file


```
sed -i 's/#fff/#768331/g' wp-content/themes/twentyseventeen/style.css
```


The sed command changes this color (#fff) to #768331


61. Call the AWS Codedeploy push command to bundle the updated filestogether , upload the revisions to Amazon S3 , and register information withAWS Codedeploy about the uploaded revision by:


 * Pasting: `aws deploy push --application-name Wordpress_App --s3-locat`
 * Replacing BUCKET with your S3 bucket name
 * Pressing ENTER

62. At the top of the AWS Management Console , in the search bar , search forand choose `Codedeploy`

63. In the left navigation pane , click `Applications`

64. Click `Wordpress_App`

65. Click the `Revisions` tab

66. Select O the revision that has not been deployed yet . This is the latestversion

67. Click `Deploy application`

68. On the Create deployment window , configure

 * Deployment group: Wordpress_DG
 * Scroll to the bottom of the screen, then click `Create deployment`

69. Wait for the Deployment status to display ✅ Succeeded

70. Go back to your Wordpress tab and refresh the screen

71. The backaround of your site should now be gareen

End lab

# Conclusion

 Congratulations! You now have successfully:

  * Verified that the Codedeploy agent has been installed

  * Configured application source content to be deployed to Codedeploy

  * Created an Amazon S3 bucket , then uploaded your Wordpress application tothe bucket

  * Deployed a Wordpress application to an Amazon EC2 instance

  * Monitored a Wordpress application deployment

  * Updated a Wordpress application and then redeployed it

# Resources

 * [AWS Codedeploy](https://aws.amazon.com/codedeploy/)

 * [AWS Codedeploy Application Specification Files](https://docs.aws.amazon.com/codedeploy/latest/userguide/application-specification-files.html#application-specification-files-agent-usage)

 * [Working with Deployment Configuration in AWS Codedeploy]()

 * [Lab Tutoria](https://docs.aws.amazon.com/codedeploy/latest/userguide/tutorials-wordpress-launch-instance.html)

 * [Verify the AWS Codedeploy Agent Is Running](https://docs.aws.amazon.com/codedeploy/latest/userguide/codedeploy-agent-operations-verify.html)
 

 [Introduction to Amazon API Gateway](https://amazon.qwiklabs.com/focuses/53694?parent=catalog)
