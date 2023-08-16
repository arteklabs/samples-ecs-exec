# samples-ecs-exec

Arteklabs Samples: AWS ECS Exec

## pre-requirements

Install the dependencies specified at the [ECS Exec docs](https://aws.amazon.com/blogs/containers/new-using-amazon-ecs-exec-access-your-containers-fargate-ec2/).

## getting started

KMS key to encrypt the ECS Exec data channel

```shell
KMS_KEY=$(aws kms create-key --region $AWS_REGION --profile $AWS_PROFILE)
KMS_KEY_ARN=$(echo $KMS_KEY | jq --raw-output .KeyMetadata.Arn)
aws kms create-alias \
    --alias-name alias/$PROJECT-kms-key \
    --target-key-id $KMS_KEY_ARN \
    --region $AWS_REGION \
    --profile $AWS_PROFILE
echo "The KMS Key ARN is: "$KMS_KEY_ARN
# arn:aws:kms:eu-central-1:*****:key/*****
```

ECS Cluster

```shell
aws ecs create-cluster \
    --cluster-name $PROJECT-cluster \
    --region $AWS_REGION \
    --configuration executeCommandConfiguration="{logging=OVERRIDE,\
                                                kmsKeyId=$KMS_KEY_ARN,\
                                                logConfiguration={cloudWatchLogGroupName="/aws/ecs/$PROJECT",\
                                                                s3BucketName=$ECS_EXEC_BUCKET_NAME,\
                                                                s3KeyPrefix=exec-output}}" \
    --profile $AWS_PROFILE
# {
#     "cluster": {
#         "clusterArn": "arn:aws:ecs:eu-central-1:*****:cluster/samples-ecs-exec-cluster",
#         "clusterName": "samples-ecs-exec-cluster",
#         "configuration": {
#             "executeCommandConfiguration": {
#                 "kmsKeyId": "arn:aws:kms:eu-central-1::*****:key/:*****",
#                 "logging": "OVERRIDE",
#                 "logConfiguration": {
#                     "cloudWatchLogGroupName": "/aws/ecs/samples-ecs-exec",
#                     "cloudWatchEncryptionEnabled": false,
#                     "s3BucketName": "samples-ecs-exec-output-:*****",
#                     "s3EncryptionEnabled": false,
#                     "s3KeyPrefix": "exec-output"
#                 }
#             }
#         },
#         "status": "ACTIVE",
#         "registeredContainerInstancesCount": 0,
#         "runningTasksCount": 0,
#         "pendingTasksCount": 0,
#         "activeServicesCount": 0,
#         "statistics": [],
#         "tags": [],
#         "settings": [
#             {
#                 "name": "containerInsights",
#                 "value": "disabled"
#             }
#         ],
#         "capacityProviders": [],
#         "defaultCapacityProviderStrategy": []
#     }
# }
```

CloudWatch log group

```shell
aws logs create-log-group \
    --log-group-name /aws/ecs/$PROJECT \
    --region $AWS_REGION \
    --profile $AWS_PROFILE
```

S3 bucket

```shell
aws s3api create-bucket \
    --bucket $ECS_EXEC_BUCKET_NAME \
    --region $AWS_REGION \
    --create-bucket-configuration LocationConstraint=$AWS_REGION \
    --profile $AWS_PROFILE
```

allow traffic on port 80 (nginx server)

```shell
ECS_EXEC_DEMO_SG=$(aws ec2 create-security-group \
    --group-name $PROJECT-SG \
    --description "ECS exec demo SG" \
    --vpc-id $VPC_ID \
    --region $AWS_REGION \
    --profile $AWS_PROFILE)
ECS_EXEC_DEMO_SG_ID=$(echo $ECS_EXEC_DEMO_SG | jq --raw-output .GroupId)
aws ec2 authorize-security-group-ingress \
    --group-id $ECS_EXEC_DEMO_SG_ID \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0 \
    --region $AWS_REGION \
    --profile $AWS_PROFILE
# {
#     "Return": true,
#     "SecurityGroupRules": [
#         {
#             "SecurityGroupRuleId": "sgr-0ba6a29f04adb2238",
#             "GroupId": "sg-0f818f41b6355ef4e",
#             "GroupOwnerId": "401886413957",
#             "IsEgress": false,
#             "IpProtocol": "tcp",
#             "FromPort": 80,
#             "ToPort": 80,
#             "CidrIpv4": "0.0.0.0/0"
#         }
#     ]
# }
```

iam roles

```shell
aws iam create-role \
    --role-name $PROJECT-task-execution-role \
    --assume-role-policy-document file://ecs-tasks-trust-policy.json \
    --region $AWS_REGION \
    --profile $AWS_PROFILE
# {
#     "Role": {
#         "Path": "/",
#         "RoleName": "samples-ecs-exec-task-execution-role",
#         "RoleId": "*****",
#         "Arn": "arn:aws:iam::*****:role/samples-ecs-exec-task-execution-role",
#         "CreateDate": "*****",
#         "AssumeRolePolicyDocument": {
#             "Version": "2012-10-17",
#             "Statement": [
#                 {
#                     "Effect": "Allow",
#                     "Principal": {
#                         "Service": [
#                             "ecs-tasks.amazonaws.com"
#                         ]
#                     },
#                     "Action": "sts:AssumeRole"
#                 }
#             ]
#         }
#     }
# }

aws iam create-role \
    --role-name $PROJECT-task-role \
    --assume-role-policy-document file://ecs-tasks-trust-policy.json \
    --region $AWS_REGION \
    --profile $AWS_PROFILE
# {
#     "Role": {
#         "Path": "/",
#         "RoleName": "samples-ecs-exec-task-role",
#         "RoleId": "*****",
#         "Arn": "arn:aws:iam::*****:role/samples-ecs-exec-task-execution-role",
#         "CreateDate": "*****",
#         "AssumeRolePolicyDocument": {
#             "Version": "2012-10-17",
#             "Statement": [
#                 {
#                     "Effect": "Allow",
#                     "Principal": {
#                         "Service": [
#                             "ecs-tasks.amazonaws.com"
#                         ]
#                     },
#                     "Action": "sts:AssumeRole"
#                 }
#             ]
#         }
#     }
# }
```
