# CloudGuard integration with CICD pipeline on AWS using CodePipeline

Docker images 

In this tutorial, I'll do a step-by-step walk-through of integrating CloudGuard SHIFTLEFT into your CICD PipeLine on AWS. The integration will happen at the build stage. C

This Github repo contains source code (zip) of a sample docker image.

# Pre-requisites
 You need the following tools on your computer:

* AWS CLI [Install AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html).

Note: This is an **ALL-AWS** tutorial which means we'll be using CICD services provided by **AWS ONLY**. However, CloudGuard can be integrated with any other automation tools that can create CICD pipeline.

### AWS and CloudGuard 

* AWS Account
* Access to Check Point Infinity portal (For now, the scan result can be viewed only on Infinity portal. Hopefully, we'll make it available on CloudGuard console soon.)

### AWS IAM Roles needed for the following AWS services

The role(s) will be created as part of creating a codepipeline. Please take note that the role used by codebulid requires permission to access to a number of AWS resources such as S3. 

* CodeBuild

- For CodeBuild Role, two additional policies need to be attached to it on top of of the policies that were attahed when it was created.

1. AmazonEC2ContainerRegistryPowerUser 
2. An Inline Policy that allows it to "PUT OBJECT" to S3 Bucket. This is for uploading scan result to S3. (See JSON below)

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "s3:PutObject",
            "Resource": "*"
        }
    ]
}
```


# What exactly we will be doing

In this tutorial, we'll be doing the followings;

1. Create AWS ECR repo \
(Yes if you'd like to follow along my ALL-AWS tutorial, you'll need to create a ECR repo which will store the docker image.)
2. Create a CodeCommit Repo
3. Create a Codebuild Project
4. Test the Codebuild with SHIFTLEFT



## 1. Create a ECR Repository
First you'll need to create a ECR on AWS. Your docker image (after build stage) will be stored in the ECR repo.

You can create the ECR repo on AWS web console or you can just execute the following command.

```bash
aws ecr create-repository --repository-name project-a/Your-App
```

**Take note of the Docker Image URI!**

## 2. Create a CodeCommit Repository

Then you'll need to create a CodeCommit on AWS. We need the CodeCommit repo to store the "source" files that we will build into a docker image. 

 You can do it on AWS web console or you can just execute the following command.

```bash
aws codecommit create-repository --repository-name my-docker-repo --repository-description "My Docker Repo"
```

Then you'll need to do 'git clone your codepipline repo' via either SSH or HTTP.  It'll be an empty repository first. Then you will need to download the source files (zip) into your local repo [here](https://github.com/jaydenaung/CloudGuard-ShiftLeft-CICD-AWS/blob/main/src.zip) 

- Unzip the source files. You'll need to **make sure that "src" folder and Dockerfile are in the same root directory**.
- Remove the zip file 
- Download the Dockerfile & buildspec.yml 

**So in your CodeCommit local dirctory, you should have the following folder and files**.

1. src (directory where source codes are)
2. Dockerfile
3. buildspec.yml (This file isn't needed for Docker image however, it is required to CodeBuild)

- Then you'll need to do `git init`, `git add -A`, `git commit -m "Your message"` and `git push`
- All the above files should now be uploaded to your CodeCommit repo.


### CLOUDGUARD API KEY AND SECRET

SHIFTLEFT requires CloudGuard's API key and API secrets. In Build stage, we'll need to export it in buildspec.yml. You can generate CloudGuard API key and API secrets on CloudGuard console. 

### S3 Bucket
You'll also need to create an S3 bucket to uplaod and store a copy of SHIFTLEFT vulnerability scan result.

```bash
aws s3 mb s3://Your-Bucket-Name
```


## [buildspec.yml](https://github.com/jaydenaung/CloudGuard-ShiftLeft-CICD-AWS/blob/main/buildspec.yml)

Buildspec.yml instructs CodeBuild in build stage in terms of what to do. Basically, buildspec.yml will instruct AWS CodeBuild to automatically scan the docker image for vulnerability during build stage. So this an important configuration file. 

**[IMPORTANT]** In the buildspec.yml, look for #UPDATE comment and replace the values with your own values accordingly.


```
version: 0.2  
 
phases: 
  install:
    runtime-versions:
        docker: 18     
    commands: 
      - nohup /usr/local/bin/dockerd --host=unix:///var/run/docker.sock --host=tcp://127.0.0.1:2375 --storage-driver=overlay2&
      - timeout 15 sh -c "until docker info; do echo .; sleep 1; done"
  
  pre_build: 
    commands: 
    - echo Logging in to Amazon ECR.... 
    - aws --version
    # update the following line with your own region
    - $(aws ecr get-login --no-include-email --region ap-southeast-1) 
  build: 
    commands: 
    - echo Downloading SHIFTLEFT
    #UPDATE
    - export CHKP_CLOUDGUARD_ID=YOUR-CLOUDGUARD-ID
     # UPDATE
    - export CHKP_CLOUDGUARD_SECRET=YOUR-SECRET
    - wget https://jaydenstaticwebsite.s3-ap-southeast-1.amazonaws.com/download/shiftleft
    - chmod -R +x ./shiftleft
    - echo Build started on `date` 
    - echo Building the Docker image... 
    # UPDATE the following line with the name of your own ECR repository
    - docker build -t your-docker-image .
    # UPDATE the following line with the URI of your own ECR repository (view the Push Commands in the console)
    - docker tag YOUR-DOCKER-IMAGE:latest ECR-URI-dkr.ecr.ap-southeast-1.amazonaws.com/YOUR-DOCKER-IAMGE:latest
    #Saving the docker image in tar
    - echo Saving Docker image 
    - docker save cyberave-docker -o Your-DOCKER-IAMGE.tar
    # Start Scan
    - echo Starting scan at `date`
    # Update the saved tar file with your docker image name 
    - ./shiftleft image-scan -i Your-DOCKER-IAMGE.tar > result.txt || if [ "$?" = "6" ]; then exit 0; fi
     
  post_build: 
    commands: 
    - echo Build completed on `date` 
    - echo Pushing image to repo
    # UPDATE the following line with the URI of your own ECR repository
    - docker push ECR-URI-dkr.ecr.ap-southeast-1.amazonaws.com/YOUR-DOCKER-IAMGE:latest 

artifacts: 
  files:
    - result.txt
```


# 3. Create a Codebuild Project


![header image](img/codebuild-1.png)


![header image](img/codebuild-2.png)


![header image](img/codebuild-3.png)

### CodeBuild Output

Below is the logs from Codebuild not everything just an excerpt from it - towards the end of the build.

```bash
See any operating system documentation about shared libraries for
more information, such as the ld(1) and ld.so(8) manual pages.
----------------------------------------------------------------------

Build complete.
Don't forget to run 'make test'.

Installing shared extensions:     /usr/local/lib/php/extensions/no-debug-non-zts-20160303/
Installing header files:          /usr/local/include/php/

warning: mbstring (mbstring.so) is already loaded!

find . -name \*.gcno -o -name \*.gcda | xargs rm -f
find . -name \*.lo -o -name \*.o | xargs rm -f
find . -name \*.la -o -name \*.a | xargs rm -f
find . -name \*.so | xargs rm -f
find . -name .libs -a -type d|xargs rm -rf
rm -f libphp.la       modules/* libs/*
Removing intermediate container c9800506bba3
 ---> ecb661b8e7c5
Step 12/14 : RUN a2enmod rewrite
 ---> Running in aace68afc4f0
Enabling module rewrite.
To activate the new configuration, you need to run:
  service apache2 restart
Removing intermediate container aace68afc4f0
 ---> e04362b2faaf
Step 13/14 : RUN a2enmod ssl
 ---> Running in dabd18966a41
Considering dependency setenvif for ssl:
Module setenvif already enabled
Considering dependency mime for ssl:
Module mime already enabled
Considering dependency socache_shmcb for ssl:
Enabling module socache_shmcb.
Enabling module ssl.
See /usr/share/doc/apache2/README.Debian.gz on how to configure SSL and create self-signed certificates.
To activate the new configuration, you need to run:
  service apache2 restart
Removing intermediate container dabd18966a41
 ---> 35cad561b896
Step 14/14 : RUN service apache2 restart
 ---> Running in 867053eb27be
Restarting Apache httpd web server: apache2AH00558: apache2: Could not reliably determine the server's fully qualified domain name, using 172.18.0.2. Set the 'ServerName' directive globally to suppress this message

Removing intermediate container 867053eb27be
 ---> 7fdfbf48c822
Successfully built 7fdfbf48c822
Successfully tagged cyberave-docker:latest

[Container] 2020/10/09 08:12:08 Running command docker tag cyberave-docker:latest 116489363094.dkr.ecr.ap-southeast-1.amazonaws.com/cyberave-docker:latest

[Container] 2020/10/09 08:12:08 Running command echo Saving Docker image
Saving Docker image

[Container] 2020/10/09 08:12:08 Running command docker save cyberave-docker -o cyberave-docker.tar

[Container] 2020/10/09 08:12:14 Running command echo Starting scan at `date`
Starting scan at Fri Oct 9 08:12:14 UTC 2020

[Container] 2020/10/09 08:12:14 Running command ./shiftleft image-scan -i cyberave-docker.tar > result.txt || if [ "$?" = "6" ]; then exit 0; fi
INFO   [09-10-2020 08:12:17.476] blade image-scan updated (0.0.130)           
INFO   [09-10-2020 08:12:17.594] SourceGuard Scan Started!                    
INFO   [09-10-2020 08:12:19.471] Project name: cyberave-docker path: /tmp/SourceGuard247824249 
INFO   [09-10-2020 08:12:19.471] Scan ID: 35cad561b8968f02ac5a2e0cb05b1b2fb417b756376dc04164ec5958908d23d8-SvBL8P 
INFO   [09-10-2020 08:12:31.647] Scanning ...                                 
INFO   [09-10-2020 08:12:48.892] Analyzing ...                                

[Container] 2020/10/09 08:14:25 Phase complete: BUILD State: SUCCEEDED
[Container] 2020/10/09 08:14:25 Phase context status code:  Message: 
[Container] 2020/10/09 08:14:25 Entering phase POST_BUILD
[Container] 2020/10/09 08:14:25 Running command echo Build completed on `date`
Build completed on Fri Oct 9 08:14:25 UTC 2020

[Container] 2020/10/09 08:14:25 Running command echo Pushing image to repo
Pushing image to repo

[Container] 2020/10/09 08:14:25 Running command docker push 116489363094.dkr.ecr.ap-southeast-1.amazonaws.com/cyberave-docker:latest
The push refers to repository [116489363094.dkr.ecr.ap-southeast-1.amazonaws.com/cyberave-docker]
98d3c49f13ab: Preparing
96fd7149e6a8: Preparing
5c008103b520: Preparing
897144f6e2ff: Preparing
eac34fee20ac: Preparing
c34869bde755: Preparing
bd51b78ec5a5: Preparing
ae9ead4be184: Preparing
36a33866c585: Preparing
4c4801c52898: Preparing
9a078a1d1b01: Preparing
a42cb226a41b: Preparing
0d678d51888b: Preparing
0817436a8f49: Preparing
3385a426f542: Preparing
35c986c7de74: Preparing
53bab0663330: Preparing
606c36b65880: Preparing
ab99fcc1a184: Preparing
9691e5d7a4c7: Preparing
6a4d393f0795: Preparing
e38834ac7561: Preparing
ec64f555d498: Preparing
840f3f414cf6: Preparing
17fce12edef0: Preparing
831c5620387f: Preparing
c34869bde755: Waiting
bd51b78ec5a5: Waiting
ae9ead4be184: Waiting
36a33866c585: Waiting
4c4801c52898: Waiting
9a078a1d1b01: Waiting
a42cb226a41b: Waiting
0d678d51888b: Waiting
0817436a8f49: Waiting
3385a426f542: Waiting
35c986c7de74: Waiting
53bab0663330: Waiting
606c36b65880: Waiting
ab99fcc1a184: Waiting
9691e5d7a4c7: Waiting
6a4d393f0795: Waiting
e38834ac7561: Waiting
ec64f555d498: Waiting
840f3f414cf6: Waiting
17fce12edef0: Waiting
831c5620387f: Waiting
5c008103b520: Pushed
96fd7149e6a8: Pushed
eac34fee20ac: Pushed
c34869bde755: Pushed
98d3c49f13ab: Pushed
bd51b78ec5a5: Pushed
897144f6e2ff: Pushed
4c4801c52898: Pushed
36a33866c585: Pushed
ae9ead4be184: Pushed
0817436a8f49: Layer already exists
3385a426f542: Layer already exists
35c986c7de74: Layer already exists
53bab0663330: Layer already exists
ab99fcc1a184: Layer already exists
606c36b65880: Layer already exists
9a078a1d1b01: Pushed
a42cb226a41b: Pushed
6a4d393f0795: Layer already exists
9691e5d7a4c7: Layer already exists
e38834ac7561: Layer already exists
ec64f555d498: Layer already exists
17fce12edef0: Layer already exists
840f3f414cf6: Layer already exists
831c5620387f: Layer already exists
0d678d51888b: Pushed
latest: digest: sha256:a46642efce16bbed3015725bfd40cbabb2115529ceb0e0a0430ca44b74f8453f size: 5740

[Container] 2020/10/09 08:14:29 Phase complete: POST_BUILD State: SUCCEEDED
[Container] 2020/10/09 08:14:29 Phase context status code:  Message: 
[Container] 2020/10/09 08:14:29 Expanding base directory path: .
[Container] 2020/10/09 08:14:29 Assembling file list
[Container] 2020/10/09 08:14:29 Expanding .
[Container] 2020/10/09 08:14:29 Expanding file paths for base directory .
[Container] 2020/10/09 08:14:29 Assembling file list
[Container] 2020/10/09 08:14:29 Expanding result.txt
[Container] 2020/10/09 08:14:29 Found 1 file(s)

```

Finally, you can check and verify that SHIFTLEFT has


## 4. Check the SHIFTLEFT scan result

On AWS Console, go to "S3", and the S3 bucket that we've created, and defined as "artifacts" in the CodeBuild stage. In the "output" folder, you should see "result.txt" which basically is the scan result of the SHIFTLEFT.

 


A copy of the result has been sent to Check Point Infinity Portal. If you have access to infinity portal, you should can view the scan result.

## A Sample Scan Result (Excerpt)

```
		  CVEs Findings:
			- ID: CVE-2018-21232
			Description: re2c before 2.0 has uncontrolled recursion that causes stack consumption in find_fixed_tags.
			Severity: MEDIUM
			Last Modified: 2020-05-14T12:15:00Z
		- ncurses-base  6.1+20181013-2+deb10u2
		  Severity: MEDIUM
		  Line: 3524
		  CVEs Findings:
			- ID: CVE-2019-17594
			Description: There is a heap-based buffer over-read in the _nc_find_entry function in tinfo/comp_hash.c in the terminfo library in ncurses before 6.1-20191012.
			Severity: MEDIUM
			Last Modified: 2019-12-26T15:35:00Z
			- ID: CVE-2019-17595
			Description: There is a heap-based buffer over-read in the fmt_entry function in tinfo/comp_hash.c in the terminfo library in ncurses before 6.1-20191012.
			Severity: MEDIUM
			Last Modified: 2019-12-23T19:26:00Z
		- libbrotli1  1.0.7-2
		  Severity: MEDIUM
		  Line: 1350
		  CVEs Findings:
			- ID: CVE-2020-8927
			Description: A buffer overflow exists in the Brotli library versions prior to 1.0.8 where an attacker controlling the input length of a "one-shot" decompression request to a script can trigger a crash, which happens when copying over chunks of data larger than 2 GiB. It is recommended to update your Brotli library to 1.0.8 or later. If one cannot update, we recommend to use the "streaming" API as opposed to the "one-shot" API, and impose chunk size limits.
			Severity: MEDIUM
			Last Modified: 2020-09-30T18:15:00Z
		- util-linux  2.33.1-0.1
		  Severity: UNKNOWN
		  Line: 3899
		  CVEs Findings:
			- ID: CVE-2007-5191
			Description: mount and umount in util-linux and loop-aes-utils call the setuid and setgid functions in the wrong order and do not check the return values, which might allow attackers to gain privileges via helpers such as mount.nfs.
			Severity: UNKNOWN
			Last Modified: 2018-10-15T21:41:00Z
			- ID: CVE-2001-1494
			Description: script command in the util-linux package before 2.11n allows local users to overwrite arbitrary files by setting a hardlink from the typescript log file to any file on the system, then having root execute the script command.
			Severity: UNKNOWN
			Last Modified: 2017-10-11T01:29:00Z
		- m4  1.4.18-2
		  Severity: UNKNOWN
		  Line: 3407
		  CVEs Findings:
			- ID: CVE-2008-1688
			Description: Unspecified vulnerability in GNU m4 before 1.4.11 might allow context-dependent attackers to execute arbitrary code, related to improper handling of filenames specified with the -F option.  NOTE: it is not clear when this issue crosses privilege boundaries.
			Severity: UNKNOWN
			Last Modified: 2017-08-08T01:30:00Z
			- ID: CVE-2008-1687
			Description: The (1) maketemp and (2) mkstemp builtin functions in GNU m4 before 1.4.11 do not quote their output when a file is created, which might allow context-dependent attackers to trigger a macro expansion, leading to unspecified use of an incorrect filename.
			Severity: UNKNOWN
			Last Modified: 2017-08-08T01:30:00Z
Please see full analysis: https://portal.checkpoint.com/Dashboard/SourceGuard#/scan/image/35cad561b8968f02ac5a2eabcderdfdkfndkfndk 
```

**Congratulations!** You've successfully integrated CloudGuard SHIFTLEFT into CICD pipeline on AWS!

![header image](img/cloudguard-1.png) 


## Issues

1. One of the issues you might probably encounter in Build is the build stage might fail due to IAM insufficient permissions. Ensure that the IAM role has the following additional policies attached to it

1. ECR 
2. S3 Put Bucket 

2. Make sure that all required software & dependencies are installed. (e.g. AWS CLI.


![header image](img/cloudguard.png) 

## Resources

1. [Check Point CloudGuard Workload Protection](https://www.checkpoint.com/products/workload-protection/#:~:text=CloudGuard%20Workload%20Protection%2C%20part%20of,automating%20security%20with%20minimal%20overhead.)

2. [CloudGuard SHIFTLEFT](https://github.com/dome9/protego-examples)

3. Here is another good tutorial you might want to check out - [CloudGuard integration with Jenkins by Dean Houari](https://github.com/chkp-dhouari/AWSCICD-CLOUDGUARD)


