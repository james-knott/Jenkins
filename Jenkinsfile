#!/usr/bin/env groovy

pipeline {

  agent {
    label 'custom-rbna-image'
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '5'))
    disableConcurrentBuilds()
   }

  parameters {
    string(name: 'BUCKET_NAME', defaultValue: 'linuxdeploy-servername-s3', description: "S3 Bucket for state file. Not Managed by Terraform Setup once per Job")
    string(name: 'TABLE_NAME', defaultValue: 'linuxdeploy-dynamodb-table-state-locking', description: "DynamoDB Table for State Locking. Not Managed by Terraform Setup once per Job")
    string(name: 'FOLDER_NAME', defaultValue: 'ec2/commonec2', description: "This will be the name of the folder that is used for the terrafrom apply. This will stay the same for the life of the job")
    string(name: 'SERVER_NAME', defaultValue: 'servername', description: "This will be the default name to use to do multiple deployments.")
    string(name: 'AWS_REGION', defaultValue: 'us-west-2', description: "This will be the name of the creds ID from Jenkins credentials. Will change when you want to deploy to another region.")
    string(name: 'VPC_NAME', defaultValue: 'USPDXVPC01B', description: "This is just informational. Tags will be derived from this vpc name")
    string(name: 'TAG_ENVIRONMENT', defaultValue: 'P', description: "This is for production")
    string(name: 'VPC_ID', defaultValue: 'vpc-50204929', description: "This is the vpc id where you want to deploy the instance.")
    string(name: 'SUBNET_CIDRS', defaultValue: '10.16.4.0/24', description: "This is the subnet cidr where you want to deploy your instance")
    string(name: 'TAG_LIFECYCLE', defaultValue: 'Permanent', description: "Tag Lifecycle")
    string(name: 'SUBNET_IDS', defaultValue: 'subnet-3733ef4e', description: "This is the subnet ID where you want to deploy your instance")
    string(name: 'TAG_CONTACT', defaultValue: 'IT Ops / itoperations@us.redbull.com', description: "Tag Contacts")
    string(name: 'ALLOWED_IPS', defaultValue: '0.0.0.0/0', description: "Allowed IPs to the instance")
    string(name: 'BASE_AMI', defaultValue: 'ami-e4485b9d', description: "AMI ID")
    string(name: 'KEYPAIR', defaultValue: 'USPDXVPC01-KEY-OPSCLUSTER-001-P', description: "Keypair used to connect to the box")
    string(name: 'INSTANCE_TYPE', defaultValue: 't2.medium', description: "Instance Type")
    string(name: 'ROLE', defaultValue: 'UBNTST', description: "Role the instance will play")
    string(name: 'INSTANCE_PROFILE', defaultValue: 'USPDXVPC01B-IRG-USPDXUBNTST-P', description: "Role the instance will play")
    string(name: 'SECURITY_GROUP_ID', defaultValue: 'sg-e9e1d096', description: "Role the instance will play")
    string(name: 'TAG_PROVISIONING', defaultValue: 'Auto', description: "Tag Provisioning")
    string(name: 'TAG_SERVICENOWID', defaultValue: 'Operations Cluster', description: "Tag Service Now ID")
    credentials(name: 'CredsToUse', description: 'An AWS account to build with Terraform', defaultValue: 'vpc01', credentialType: "AWS Credentials", required: true )
  }

  environment {
        TF_VAR_vpc_name               = "${params.VPC_NAME}"
        TF_VAR_environment        = "${params.ENVIRONMENT}"
        TF_VAR_vpc_id             = "${params.VPC_ID}"
        TF_VAR_subnet_cidrs            = "${params.SUBNET_CIDRS}"
        TF_VAR_subnet_ids         = "${params.SUBNET_IDS}"
        TF_VAR_allowed_ips        = "${params.ALLOWED_IPS}"
        TF_VAR_tag_environment    = "${params.TAG_ENVIRONMENT}"
        TF_VAR_base_ami           = "${params.BASE_AMI}"
        TF_VAR_tag_servicenowid   = "${params.TAG_SERVICENOWID}"
        TF_VAR_instance_type      = "${params.INSTANCE_TYPE}"
        TF_VAR_instance_count     = "${params.INSTANCE_COUNT}"
        TF_VAR_tag_provisioning   = "${params.TAG_PROVISIONING}"
        TF_VAR_role               = "${params.ROLE}"
        TF_VAR_instance_profile   = "${params.INSTANCE_PROFILE}"
        TF_VAR_security_group_id  = "${params.SECURITY_GROUP_ID}"
        TF_VAR_tag_lifecycle      = "${params.TAG_LIFECYCLE}"
        TF_VAR_tag_contact        = "${params.TAG_CONTACT}"
        TF_VAR_key_pair           = "${params.KEYPAIR}"
        TF_VAR_region = "${params.AWS_REGION}"
        AWS_DEFAULT_REGION = "${params.AWS_REGION}"
    }

  stages {
     stage('Terraform Init') {
      steps {
       withCredentials([[ $class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'vpc01', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY' ]])
          {
        sh '''#!/bin/bash
        rm -rf .terraform
        rm -rf terraform.tfstate
        rm -rf terraform.tfstate.backup
        rm -rf backend.tf
        echo "terraform { backend \\"s3\\" { bucket = \\"${BUCKET_NAME}\\" dynamodb_table = \\"${TABLE_NAME}\\" region = \\"us-west-2\\" key = \\"production/linuxdeploy/${SERVER_NAME}/terraform.tfstate\\" }}" > $WORKSPACE/${FOLDER_NAME}/backend.tf && ls -al $WORKSPACE/${FOLDER_NAME}
        cat $WORKSPACE/${FOLDER_NAME}/backend.tf
        aws s3 ls s3://${BUCKET_NAME} || aws s3 mb s3://${BUCKET_NAME}
        aws dynamodb describe-table --table-name ${TABLE_NAME} || aws dynamodb create-table --table-name ${TABLE_NAME} --attribute-definitions AttributeName=LockID,AttributeType=S --key-schema AttributeName=LockID,KeyType=HASH --provisioned-throughput ReadCapacityUnits=1,WriteCapacityUnits=1
        cd $WORKSPACE/${FOLDER_NAME} && ls -al && terraform init -no-color -input=false
        '''
         }
      }
    }
    stage('Terraform Plan') {
       steps {
          withCredentials([[ $class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'vpc01', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY' ]])
          {
        sh '''#!/bin/bash
        echo "terraform { backend "s3" { bucket = ${BUCKET_NAME} dynamodb_table = ${TABLE_NAME} region = "us-west-2" key = "production/linuxdeploy/${SERVER_NAME}/terraform.tfstate" }}" > backend.tf
        '''
        sh "cd $WORKSPACE/${params.FOLDER_NAME} && ls -al && terraform plan -no-color -input=false"
        sleep 10
      }
      }
    }
    stage('User Input') {
      steps {
        mail (to: 'dominador.sarenas@redbull.com',
         subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is waiting for input",
         body: "Please go to ${env.BUILD_URL}.");
         input 'Do you want to build this infrastructure?';
           }
     }
    stage('Terraform Apply') {
      steps {
          withCredentials([[ $class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'vpc01', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY' ]])
          {
        sh '''#!/bin/bash
        echo "terraform { backend "s3" { bucket = ${BUCKET_NAME} dynamodb_table = ${TABLE_NAME} region = "us-west-2" key = "production/linuxdeploy/${SERVER_NAME}/terraform.tfstate" }}" > backend.tf
        '''
        sh "cd $WORKSPACE/${params.FOLDER_NAME} && ls -al && terraform apply -no-color -input=false"
        sleep 10
      }
      }
    }
    stage('User Input for Destroy') {
      steps {
        mail (to: 'dominador.sarenas@redbull.com',
         subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) is waiting for input",
         body: "Please go to ${env.BUILD_URL}.");
         input 'Do you want to destroy the infrastructure? Click Proceed to destroy all the infrastructure so you can start over or Abort if everything looks good.';
           }
     }
     stage ('Terraform Destroy') {
      steps {
         withCredentials([[ $class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'vpc01', accessKeyVariable: 'AWS_ACCESS_KEY_ID', secretKeyVariable: 'AWS_SECRET_ACCESS_KEY' ]])
          {
        sh '''#!/bin/bash
        echo "terraform { backend "s3" { bucket = ${BUCKET_NAME} dynamodb_table = ${TABLE_NAME} region = "us-west-2" key = "production/linuxdeploy/${SERVER_NAME}/terraform.tfstate" }}" > backend.tf
        '''
        sh "cd $WORKSPACE/${params.FOLDER_NAME} && ls -al && terraform destroy -force -no-color -input=false"
      }
      }
  }

  stage ('Notification') {
    steps {
        mail (to: 'dominador.sarenas@redbull.com',
         subject: "Job '${env.JOB_NAME}' (${env.BUILD_NUMBER}) has been completed",
         body: "To view the build status. Please go to ${env.BUILD_URL}.");
           }
      }
    }
}

