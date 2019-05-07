# Introducing to CloudFormation
* Allow to configure, manage and provision your AWS infrastructure as code. Resources are defined using CloudFormation template.
* Benefits:
  * Infrastructure is provisioned consistently, with fewer mistakes
  * Less times than manual configuration
  * You can version control (good for backup) and peer review your templates
  * Free
  * Can be used to manage updates & dependency (ensure resources are created in correct order)
  * Can be use to rollback and delete the entire stack.
    * _Note_: rollback can be triggered based on an alarm threshold
  * Can be used to replicate infrastructure in different region
  * Good for disaster recovery
* CloudFormation templates:
  * Can use YAML or JSON
  * Describe the endstate of your infrastructure to provision or update
  * CloudFormation template must be updated to S3
  * CloudFormation reads from S3 the template to make APIs call
  * The resulting resources are called a Stack
* Templates structure (YAML). Just need to know the utility of each section:

    ```yaml
    AWSTemplateFormatVersion: "2010-09-09"
    Description: "Template to create EC2 instance"
    Metadata:
      Instances:
        Description: "Web Server Instance"

    Parameters: #Input custom values
      EnvType:
        Description: "Environment type"
        Type: String
        AllowedValues:
          - prod
          - test

    Conditions: #e.g. provision resources based on environment
      CreateProdResources: !Equals [ !Ref EnvType, prod ]

    Mappings: #e.g. define set of values based on region, based on environment...
      RegionMap:
        eu-west-1:
          "ami": "ami-Obcccfe12qvc"

    Transform: #include snippets of code from S3 (lambda code, template snippet code, AWS provided snippet of code)
      Name: "AWS::Include"
      Parameters:
        Location: "s3://MyS3Bucket//myFile.yaml"

    Resources: #AWS resource to create. This is the only one mandatory section
      EC2Instance:
        Type: AWS::EC2::Instance
        Properties:
          InstanceType: t2.micro
          ImageId: ami-Obcccfe12qvc

    Output: #Output. Can use function like Fn::Join to joint string, Fn::GetAttr to retrieve server name, etc.
      InstanceID:
        Description: "The instance ID"
        Value: !Ref EC2Instance
    ```
* CloudFormer (beta): create CloudFormation template from existing AWS resources in your account.

# CloudFormation lab
* Go in CloudFormation
* Click on create new stack:
  * Templates selection: can design a new one, can choose one proposed by AWS (LAMP stack, Wordpress blog...), upload a template or specify location in S3 of a template. Upload this template:

    ```yaml
    AWSTemplateFormatVersion: 2010-09-09
    Description: Template to create an EC2 instance and enable SSH
    Parameters:
      KeyName:
        Description: Name of SSH KeyPair
        Type: 'AWS::EC2::KeyPair::KeyName'
        ConstraintDescription: Provide the name of an existing SSH key pair
    Resources:
      InstanceSecurityGroup:
        Type: 'AWS::EC2::SecurityGroup'
        Properties:
          GroupDescription: Enable SSH access via port 22
          SecurityGroupIngress:
            IpProtocol: tcp
            FromPort: 22
            ToPort: 22
            CidrIp: 0.0.0.0/0
      MyEC2Instance:
        Type: 'AWS::EC2::Instance'
        Properties:
          InstanceType: t2.micro
          ImageId: ami-0bdb1d6c15a40392c #AMI ID are regional. Update according to your region
          SecurityGroups: !Ref InstanceSecurityGroup
          KeyName: !Ref KeyName
          Tags:
            - Key: Name
              Value: My CF Instance
    Outputs:
      InstanceID:
        Description: The Instance ID
        Value: !Ref MyEC2Instance
    ```
  * Provide a name for the stack
  * Select value for "KeyName" parameter: select in pick list the key pair you want for the instance
  * Rollback on failure: Yes (default value). Even rollback the resources successfully created. You can also enable/disable via API using `DisabledRollback` or using `--disable-rollback` in AWS CLI. Disable rollback is useful to troubleshooting a problem.
  * Click on "Create" to execute the template
* Finally, you can delete the stack: it will delete the resources created

# What is Elastic Beanstalk
* Service for deploying a scalable Web applications
  * Language: Java, .NET, Node.js, Python, Docker, Ruby, Go, PHP
  * Platform: Tomcat, Puma, Passenger, IIS...
  * Free
  * Fastest and simplest way to deploy an application in AWS
* Use following services: EC2, Auto scaling, ELB, RDS, SQS, CloudFront
* User doesn't need to administrate Tomcat, Java. The OS can automatically updated by AWS if requested.
* AWS handle: deployment, capacity provisioning, load balancing, automatic auto-scaling and application health. You retain full control of underlying resources if you want.
* Deployment:
  * Can be done from upload files or repository
  * Can have several environments for one app (dev, prod, etc.)
* Configuration:
  * Can configure disk
  * Can configure auto scaling (AZ, nb instances...)
  * Can configure ELB (stickiness session, https, certificate...)
    * **Connection draining**: maximum of time connection stay open between ELB and EC2 instances (default: 300 sec).
  * Can configure update (e.g. rolling: replace instances one by one every 5 min with new application version, replace all instances at once...)
  * Can configure security: role, EC2 key to use
  * Can configure network: change VPC, select AZ, security group to use
* Possibility to monitor and manage application health via a dashboard
* Integrated with CloudWatch and X-Ray (end-to-end view through all service to identify bug/performance problems) for performance data and metrics

# Elastic Beanstalk lab
* Create a new Web app:
  * Provide name
  * Platform: PHP
  * Upload a Zip file containing an "index.php" file
* If you want to handle S3, SNS...: it is better to use CloudFormation.

# OpsWorks
* Service which allow you to automate your server configuration using Puppet or Chef code. Flexible way to create and manage resources as code. Allow to automate, monitor and maintain the deployment.
* Service uses managed instance of Puppet or Chef (you don't need to configure your own Puppet/Chef environment). It's a fully managed service: don't need to configure and operate your own configuration management environment
* Work with your existing Chef or Puppet code on Linux/Windows
* Anatomy:
  * **Stacks**: represent set of resources to manage as group. Example: stack for dev and prod.
  * **Layers**: used to represent and configure a component of a stack. Chef recipes are added to layers:
    * Recipes: describe a machine setup through a code (Ruby). Recipes are run at certain events within a stack: setup (at instance boot), configure (when instances enter/leave inline state), deploy (when we deploy an app), un-deploy (when we delete an application), shutdown (when we shutdown an instance).
  * **Instances**: associate with a least one layer. Can run: 24/7, load-based or time-based.
  * **Apps**: deployed in application layer from GIT repository, SVN or even S3. Example: you can deploy an application on a layer and recipes will run to prepare the instance.
* See https://aws.amazon.com/fr/opsworks/stacks/faqs/
