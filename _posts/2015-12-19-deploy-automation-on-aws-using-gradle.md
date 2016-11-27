---
layout: post
title:  "Deploy automation on AWS using Gradle"
date:   2015-11-22 00:00:00
author: Gioia Ballin
categories: programming
tags:	aws gradle deploy
cover:  "/assets/lego.jpg"
---

According to the [Gartner MQ 2015 report](https://aws.amazon.com/resources/gartner-2015-mq-learn-more/){:target="_blank"}, Amazon with its Amazon Web Services is currently the market leader provider of IaaS services. Furthermore, it can now boast well known "unicorns" (e.g.: Airbnb, Netflix, Slack, Pinterest) among its [clients](https://aws.amazon.com/solutions/case-studies/){:target="_blank"}. Apart from these success stories, relying on Amazon Web Services is also becoming a popular choice for small, innovative, high-tech businesses.

Different AWS tools aim to address the deployment automation problem. Here, I want to focus on [AWS CodeDeploy](http://docs.aws.amazon.com/codedeploy/latest/userguide/welcome.html){:target="_blank"}. AWS CodeDeploy lets you to easily deploy applications from S3 buckets (or GitHub repositories) to EC2 instances. It's a really easy-to-use tool but, it involves the execution of repetitive manual actions.

Let's say we have a Java application built using [Gradle](https://gradle.org/){:target="_blank"}. And, let's say we want to use AWS CodeDeploy to deploy our application JAR from a S3 bucket to a EC2 instance. In order to accomplish this objective, we have to:

1. build the application using Gradle, thus obtaining the JAR file
2. move the JAR file into our deployment bucket on S3
3. sign-in to the AWS Management Console
4. open the AWS CodeDeploy console on the Deployments section
5. choose "Create New Deployment"
6. in the "Create New Deployment" section, insert:

    - the application name
    - the deployment group
    - the revision type
    - the revision location
    - the deployment configuration
7. click "Deploy Now".

__Each time__ we have to deploy our application, we should follow these steps in which either we do mechanical actions (e.g.: moving a file from our local development environment to the S3 deployment bucket) or we insert information known in advance. Think about having the test and staging environments running on EC2 instances. It's very likely that performing tests in those stages will flush out bugs, thus implying a code review. In turn, any code changes will require new tests, thus entailing the deployment of the code to the test/staging deployment groups again, and again. Therefore, reducing the amount of "boilerplate actions" that should be performed to accomplish the deployment can effectively streamline the overall develop-test-and-deploy process.

> Since we’re using Gradle to build our application, can we use it to create tasks that simplify our lives?

The answer is definitely yes. We can code Gradle tasks providing the capability to automatically perform specific kind of actions but, in order to do so, we need a tool for interacting with AWS services through scripts.

> Does Amazon provide any tool serving our ends?

Again, the answer is yes. The tool we're looking for is the [AWS Command Line Interface](https://aws.amazon.com/cli){:target="_blank"}. The AWS CLI is shipped as a python package (so it can be installed via <tt>pip</tt>) and it provides a complete series of commands devoted to manage, drive and control AWS services.

> What's the main idea, then?

Gradle gives the chance to execute shell scripts, thus it's possible to write very simple bash scripts that performs the mechanical actions we wouldn't like to do anymore and having them executed by Gradle.

Note: If we want to automate the deployment process in this way, the AWS CLI installation and configuration into our system is a prerequisite. Please refer to the [Amazon guide](http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-set-up.html){:target="_blank"} to have your environment properly set up.

> Let's start! We want to build our distribution and move it to the right S3 bucket.

First of all, we need to write a Gradle task that produces the distribution file. The logic applied to create the distribution may vary from case to case so, I don’t want to go deeper into this implementation. Long short story, let's say we have a Gradle task named __buildDistribution__ that accomplishes the purpose of making a package of our application with the AWS CodeDeploy scripts bundled inside.

Now, we can write a Gradle task that [dependsOn](https://docs.gradle.org/current/dsl/org.gradle.api.Task.html#org.gradle.api.Task:dependsOn(java.lang.Object[])){:target="_blank"} __buildDistribution__ and moves the distribution created by the <strong>buildDistribution</strong> task to the right S3 bucket.

{% highlight groovy linenos %}
// Creates a new bundle and copies it to S3
task buildAndCopyDistribution(type:Exec, dependsOn: [buildDistribution]) {
    // Defines the path into the project of the move.sh script
    def move = "automation/move.sh"
    // Defines the distribution name based on the jar section
    def distName = "${jar.baseName}-${jar.version}.zip"
    // Defines the path into the project of the distribution file
    def localDist = "$buildDir/distributions/${distName}"

    // Moves the distribution file to the right S3 bucket
    executable "bash"
    args "-c", "sh ${move} ${localDist}"
}
{% endhighlight %}

The task definition is pretty straightforward. First of all, the task is defined to be an [Exec](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.Exec.html){:target="_blank"} task, meaning that it's a task that *"executes a command line process"* and, of course, it depends on a previous execution of the __buildDistribution__ task. Then, the task defines two important variables:

- the __move__ variable contains the path to the __move.sh__ script. The reference folder for the path is the one in which the __build.gradle__ file is contained. The __move.sh__ script will perform the actual move of the file to the S3 bucket.
- the __localDist__ variable contains the path to the distribution file. Again, the reference folder is the one in which the __build.gradle__ file is contained.

Finally, the task executes the __move.sh__ bash script with the __localDist__ variable as input.

> What does the __move.sh__ script actually do?

Let's see it directly.

{% highlight bash linenos %}
#!/bin/bash

AWS="/usr/local/bin/aws"

APPLICATION_NAME="cn-app"

BUCKET_NAME="coding-nights-releases"
FULL_BUCKET="s3://${BUCKET_NAME}/${APPLICATION_NAME}/"
SOURCE_FILE=$1

echo "Application Name: ${APPLICATION_NAME}"
echo "Full Bucket: ${FULL_BUCKET}"
echo "Source file: ${SOURCE_FILE}"

# Copies the distribution to the right S3 Bucket
$AWS s3 cp ${SOURCE_FILE} ${FULL_BUCKET}

# Checks the result of the copy
if [ $? -eq 0 ]; then
    echo "Copy successful!"
else
    echo "Copy Failed!"
fi
{% endhighlight %}

Basically, the script defines some well known parameters like, the application name and the release bucket name. By merging these two constants, the full S3 bucket path is obtained. __It isn't required to build such S3 path__, it essentially depends on the standard you chose to organize and manage your S3 buckets. Furthermore, you can of course decide to define the variable __FULL_BUCKET__ only. Once the script has acquired the bucket location and the location on your environment of the distribution file, it executes the AWS CLI command devoted to move a file to an s3 bucket and, finally, it checks the result and returns.

Summing up, by executing the task __buildAndCopyDistribution__ we package a new distribution for our application and we move it to the right S3 bucket.

> Now, how can we automate the deploy of our <tt>cn-app</tt> to the deployment group <tt>cn-app-test</tt> (representing our test environment)?

The approach is the same used for automating the transfer of the distribution bundle. So, we need to write a Gradle task that performs the deploy of our bundle to the test environment.

{% highlight groovy linenos %}
// Deploys a given revision to the test environment
task deployOnTest(type:Exec) {
    // Defines the path into the project of the deploy.sh script
    def deploy = "automation/deploy.sh"
    // Defines the distribution name based on the jar section
    def distName = "${jar.baseName}-${jar.deployVersion}.zip"

    // Deploys the given distribution to the test deployment group
    executable "bash"
    args "-c", "sh ${deploy} test ${distName}"
}
{% endhighlight %}

Again, we're facing with a simple task definition. First, the task is defined to be an [Exec](https://docs.gradle.org/current/dsl/org.gradle.api.tasks.Exec.html){:target="_blank"} task, exactly like the __buildAndCopyDistribution__ task. Then, the task defines two important variables:

- the __deploy__ variable contains the path to the __deploy.sh__ script. The reference folder for the path is the one in which the __build.gradle__ file is contained. The __deploy.sh__ script will perform the deploy of our bundle to the <tt>cn-app-test</tt> deployment group.
- the __distName__ variable contains the name of the distribution bundle.

Finally, the task executes the __deploy.sh__ bash script by feeding it with the constant string "__test__", which identifies the target deployment group, and with the name of the distribution bundle.

You may be wondering why the constant string is needed. Indeed, it isn't. However, think about having more deployment groups each one representing a stage in the deployment process of the same application. In this case, the __deploy.sh__ script is expected to perform the very same computations for every deployment group but we don't want to cut and paste the same code changing only a constant string.

> Finally, we've reached the end. What does the __deploy.sh__ script actually do?

Let's see it.

{% highlight bash linenos %}
#!/bin/bash

AWS="/usr/local/bin/aws"

APPLICATION_NAME="cn-app"
APPLICATION_ENV=$1
DEPLOYMENT_GROUP="cn-app-${APPLICATION_ENV}"
BUCKET_NAME="coding-nights-releases"
BUNDLE_NAME=$2

echo "Application Name: ${APPLICATION_NAME}"
echo "Application Environment: ${APPLICATION_ENV}"
echo "Deployment Group: ${DEPLOYMENT_GROUP}"
echo "Bucket Name: ${BUCKET_NAME}"
echo "Bundle Name: ${BUNDLE_NAME}"

$AWS s3api head-object --bucket ${BUCKET_NAME} --key "${APPLICATION_NAME}/${BUNDLE_NAME}"

# Checks whether the distribution bundle exists on the expected S3 bucket
if [ $? -eq 0 ]; then
    # If the bundle exists...
    # Deploys it to the right application/deployment group
    $AWS deploy create-deployment \
        --application-name ${APPLICATION_NAME} \
        --deployment-group-name ${DEPLOYMENT_GROUP} \
        --deployment-config-name CodeDeployDefault.OneAtATime \
        --s3-location bucket="${BUCKET_NAME}",bundleType="zip",key="${APPLICATION_NAME}/${BUNDLE_NAME}"

else
    # Returns an error message
    echo "Distribution not found!"
fi
{% endhighlight %}

The __deploy.sh__ script allows to automate the deploy of the distribution bundle stored into a S3 bucket. The automation is accomplished following three main steps:

- first, constant and input parameters are defined, collected and combined together in order to get the variables that really matter to the deployment process (the application name, the deployment group, the bucket name and the bundle name).
- then, the existence of the distribution bundle into the expected S3 bucket is checked by executing the [head-object](http://docs.aws.amazon.com/cli/latest/reference/s3api/head-object.html){:target="blank"} AWS CLI command.
- if the distribution bundle already exists, the deploy is accomplished by means of the [create-deployment](http://docs.aws.amazon.com/cli/latest/reference/opsworks/create-deployment.html) AWS CLI command otherwise, an error message is returned.

You may see that the __create-deployment__ command requires a couple of further information, i.e. the bundle type and the deployment configuration. It's up to you to decide which kind of bundle you want to distribute and which deployment flow you want to follow, exactly like it's up to you to decide the naming conventions that should apply to the application name, the bucket name and the object keys.

> Is that all?

Yes, it is! We're done with the deployment automation! Now, in order to deploy our application to the test deployment group, we have to first run the __buildAndCopyDistribution__ task and then run the __deployOnTest__ task.

> Last but not the least... Why having two Gradle tasks instead of only one?

You can of course assemble tasks so the the bundle you're currently building is immediately deployed into your target group. However, sooner or later a bug will get out (yes, it's sad, I know), and maybe a quick recover is even needed in that case. You cannot benefit of the deployment automation tools if you have only one macro-task which builds, moves and deploys the distribution package. So, at the end, having two tasks provides you with the chance to choose which distribution version should be shipped.
