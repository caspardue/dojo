# ado-k8s-aws-getstarted
This is sample code for getting up and running with sample code that lets you create an s3 bucket, create a docker image and read from the bucket from within a container running on kubernetes. All through a pipeline.

## AWS resources
Sign in with saml2aws

### Windows
```Powershell
saml2aws login --force
$env:AWS_PROFILE = 'saml'
```

### Unix
```bash
saml2aws login --force
export AWS_PROFILE=saml
```

Go to the terraform-1 folder:
```bash
cd terraform-1
```

Inside the terraform-1 folder you will find a simple terraform file which will include a region to provision in and the recipe for an s3 bucket.

If running by yourself I would recommend changing the bucket name since it has to be unique across AWS.

From within this folder run a terraform init followed by a plan.

```bash
terraform init
terraform plan
```

The init sets up the API providers to enable terraform to talk with AWS and the plan will show you what will happen if you apply the terraform.

You should see something similar to this:

```bash
------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + aws_s3_bucket.test-bucket
      id:                          <computed>
      acceleration_status:         <computed>
      acl:                         "private"
      arn:                         <computed>
      bucket:                      "dfds-k8sworkshop-bucket"
      bucket_domain_name:          <computed>
      bucket_regional_domain_name: <computed>
      force_destroy:               "false"
      hosted_zone_id:              <computed>
      region:                      <computed>
      request_payer:               <computed>
      versioning.#:                <computed>
      website_domain:              <computed>
      website_endpoint:            <computed>


Plan: 1 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------
```


The important to notice part here is the line with the plan. It will inform you if your terraform will create new resources, change some or even delete.

If you do mistakes in your code you will very likely experience that your resources will be listed as destroy.
This can become a huge issue if it targets your database in production and accidentially delete your whole database.
It is a very good idea to do the plan and look it over before moving on.

For this it shouldn't be an issue and you can safely do the apply.

```bash
terraform apply
```

It will show you the plan again and ask if you want to apply it. Do so by typing in "yes".

For pipelines etc it is a good idea to use the auto approve since it can be troublesome to require user input. It is done like this:

```bash
terraform apply -auto-approve
```

End this part by cleaing up with
```
terraform destroy
```

Go back to the root workshop folder
```bash
cd ..
```

## Terraform for teams and pipelines
This is all very nice but due to the nature of terraform it saves something called a state. The state keeps track of the resources created with terraform and uses it to handle changes afterwards.
By default the state is saved in the folder with the terraform files which would be a huge problem if used in a team with more than 1 computer or if done within a pipeline.

To solve this we can introduce a concept of shared state.

Go to terraform-2

```bash
cd terraform-2
```
If you look inside the main.tf file you will notice it is very similar to the previous one. Except for the fact that we have included a terraform block with a backend called s3.

Notice that inside the block there is a bucket name and a key path.
Change these to something that makes sense for you, and remember that buckets needs to be unique across AWS.

This means that it will store the state of the things it create inside an s3 bucket instead of on your local machine. Unfortunately it won't create the bucket by itself.
There are tools that can do this for you but let's do it manually just to grasp the concept. Remember to change the bucket name to match what you used inside the block:

```bash
aws s3 mb s3://dfds-k8sworkshop-bucket-state --region eu-west-1
```

Now we got a bucket. But just in case something goes wrong like your computer locking up, internet get cut or what ever issues we IT people face, let's also put versioning on our bucket.

If the state is somehow corrupted we can easily role back so we got a working state of our terraform resources.

Remember to change the bucket name and then run this command:
```bash
aws s3api put-bucket-versioning --bucket dfds-k8sworkshop-bucket-state --versioning-configuration Status=Enabled
```

Go a head and follow the same steps as before:
```bash
terraform init
terraform plan
terraform apply
```

Don't forget to clean up
```bash
terraform destroy
cd ..
```

## Docker image

Next we are going to build a docker image that can use our aws s3 bucket.

Go to the docker-3 folder

```bash
cd docker-3
```

Inspect the Dockerfile.
You will see it is build from different components.
It got a base image which is an image built by somebody else.
In this case python which we will use to install AWS CLI.

Next we make sure the image is patched and upgraded. Just to follow best practices.

Then we install curl followed by AWS CLI which we will be using.

And last we preset the folders for the image.

Build this image by running:

```bash
docker build -t awscli .
```

Once the image is built we can try use it

```bash
docker run -it awscli
```

This will return the help text from the AWS CLI since we haven't specified a command.
Seeing this means that our image works and is ready to accept input.

Return to the root folder

```bash
cd ..
```

## Combining AWS and Docker

So far we should have learned how to create AWS resources and how to create and run docker images.
But how do we mix it?

```bash
cd docker-4
```

If you inspect the terraform file you will notice another resource is added to the file.
This one is a bucket file and in this case a simple txt file.

Remember to change the bucket name of the bucket and the state and let's provision it.

```bash
terraform init
terraform plan
terraform apply -auto-approve
```

Next let's see if we can access the file from our docker image

```bash
docker run \
-e AWS_ACCESS_KEY_ID=$(aws configure get saml.aws_access_key_id) \
-e AWS_SECRET_ACCESS_KEY=$(aws configure get saml.aws_secret_access_key) \
-e AWS_SECURITY_TOKEN=$(aws configure get saml.aws_session_token) \
-e X_PRINCIPAL_ARN=$(aws configure get saml.x_principal_arn) \
awscli s3 cp s3://dfds-k8sworkshop-bucket/testfile.txt -
```

This should return the message hello from s3 which corrospond to the content of the file created through terraform.

What we do here is running the docker image as a container, mounting our credentials we got from saml2aws as environment variables inside the container and executing the command s3 copy from path to terminal output.

This is very handy for local development where we can change the file by editing and applying our terraform and handle the file in different ways sending commands to our container.

But wouldn't it be nice if the container just did what we wanted it to without having to write the command?

Ïnspect the Dockerfile.
You will notice that a script is added and the entrypoint is changed.
Inspect the script found in get-file.sh.
You will notice that this file include our "application" which prints the content of an s3 bucket every 5 seconds.
It finds the content based on an environment variable.

Let's rebuild the image with the changes:
```bash
docker build -t awscli .
```
One of the neat features with docker build is that it is based on layers. It will find the changes performed and only rebuild everything after that point.
If you structure it  correctly you can speed up your built process by a lot!

Lets try to run this container with the path to our file. Remember to change bucket and file name.

```bash
docker run \
-e AWS_ACCESS_KEY_ID=$(aws configure get saml.aws_access_key_id) \
-e AWS_SECRET_ACCESS_KEY=$(aws configure get saml.aws_secret_access_key) \
-e AWS_SECURITY_TOKEN=$(aws configure get saml.aws_session_token) \
-e X_PRINCIPAL_ARN=$(aws configure get saml.x_principal_arn) \
-e path_to_file="s3://dfds-k8sworkshop-bucket/testfile.txt" \
awscli
```

You can exit by clicking "ctrl+c" on your keyboard. 

End this part by cleaing up with
```
terraform destroy
```

Go back to the root workshop folder
```bash
cd ..
```