---
title: Deploy with Existing IAM Policies
category: Guides
order: 3
---

As of vXXX, the deployer supports the ability to use externally-created IAM policies. Note, we're currently assuming no cloud-aware kuberenetes on installs using this feature.

All policies should have roles with ec2 instance profiles attached. The standard set of policies would have a general, executor and S3 policy, and then roles/instance profiles for executor and general that both attach the S3 policy, ie

Instance profile list:
- stagename-general
    - attaches stagename-general and stagename-S3 policies
- stagename-executor
    - attaches stagename-executor and stagename-S3 policies

To turn this on, you'll want to set provisioning.aws.use_existing_iam to 'true, and then be sure to set the instance_profile_name for all instances in the deploy manifest.

Example:

```yaml
provisioning:
  stage: 'iamtest'
  aws:
    region: 'us-west-2'
    ...
    use_existing_iam: 'true'
  private_key: '~/.ssh/domino-test.pem' # private key created as part of the deploy
  public_key: '~/.ssh/domino-test.pub' # public key created as part of the deploy
  instances:
    central:
      services:
        salt-master:
        docker-registry:
        elasticsearch:
        logging-elasticsearch:
        git:
        mongodb:
        dispatcher:
      instance_config:
        data_volume_size: 1000
        instance_type: 'm4.2xlarge'
        encrypt_ebs_volumes: 'true'
        instance_profile_name: 'iamtest-general'
...etc
```

### Policy Set

Below are the policies we're currently testing for climate. This is an existing vpc install using existing subnets, k8s-master, and imports existing security groups. An example of creating/importing the security groups is shown after the Deployer IAM policy.

Deployer IAM policy

```json
# This is to give customers for the deployer
# This version is for minimal access with no access to IAM
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "elbDeny",
            "Action": [
                "elasticloadbalancing:Delete*"
            ],
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Sid": "elbAllow",
            "Action": "elasticloadbalancing:*",
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Sid": "ec2Allow",
            "Action": [
                "ec2:DescribeSecurityGroups",
                "ec2:DescribeSubnets",
                "ec2:DescribeAvailabilityZones",
                "ec2:DescribeInstance*",
                "ec2:CreateTags",
                "ec2:DescribeTags",
                "ec2:CreateSnapshot",
                "ec2:CreateImage",
                "ec2:DescribeImages",
                "ec2:DescribeSnapshots",
                "ec2:DeleteSnapshot",
                "ec2:AssignPrivateIpAddresses",
                "ec2:AllocateAddress",
                "ec2:AssociateAddress",
                "ec2:DescribeAddresses",
                "ses:SendRawEmail",
                "ec2:CreateKeyPair",
                "ec2:DescribeKeyPairs",
                "ec2:ImportKeyPair",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DescribeNetworkInterfaceAttribute",
                "ec2:CreateNetworkInterface",
                "ec2:Describe*",
                "ec2:ModifyInstanceAttribute"
            ],
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Sid": "EC2TagLimitedDelete",
            "Effect": "Allow",
            "Action": [
                "ec2:DeregisterImage",
                "ec2:DeleteTags"
            ],
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "ec2:ResourceTag/Name": "<STAGENAME_HERE>-*"
                }
            }
        },
        {
            "Sid": "ec2SGIngress",
            "Action": [
                "ec2:AuthorizeSecurityGroupEgress",
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:RevokeSecurityGroupEgress",
                "ec2:RevokeSecurityGroupIngress"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:security-group/<STAGE-domino-dis-int SG ID HERE>",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:security-group/<STAGE-executor SG ID HERE>",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:security-group/<STAGE-domino_fe_ext_elb SG ID HERE>",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:security-group/<STAGE-domino_fe_int SG ID HERE>",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:security-group/<STAGE-general SG ID HERE>",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:security-group/<STAGE-executor-efs SG ID HERE>",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:security-group/<STAGE-general-efs SG ID HERE>"
                ]
        },
        {
            "Sid": "ec2InstanceAllow",
            "Action": [
                "ec2:StartInstances",
                "ec2:RunInstances",
                "ec2:ReplaceIamInstanceProfileAssociation"
            ],
            "Effect": "Allow",
            "Condition": {
                "ForAnyValue:StringLike": {
                    "ec2:InstanceProfile": [
                        "arn:aws:iam::<AWS_ACCOUNT_ID_HERE>:instance-profile/<STAGENAME_HERE>-*"
                    ]
                }
            },
            "Resource": "*"
        },
        {
            "Sid": "ec2RunInstanceAllow",
            "Action": [
                "ec2:RunInstances"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:key-pair/<STAGENAME_HERE>-*",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:network-interface/*",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:security-group/*",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:subnet/*",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:volume/*",
                "arn:aws:ec2:us-east-1::image/ami-3bdd502c",
                "arn:aws:ec2:us-east-2::image/ami-019abc64",
                "arn:aws:ec2:us-west-1::image/ami-48db9d28",
                "arn:aws:ec2:us-west-2::image/ami-1d14807d",
                "arn:aws:ec2:eu-west-1::image/ami-ed82e39e",
                "arn:aws:ec2:eu-central-1::image/ami-26c43149",
                "arn:aws:ec2:sa-east-1::image/ami-dc48dcb0"
                "arn:aws:ec2:<AWS_REGION_HERE>::image/<ANY OTHER BASE AMIS GO HERE>",
            ]
        },
        {
            "Sid": "support",
            "Action": [
                "support:*",
                "sts:DecodeAuthorizationMessage"
            ],
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::<STAGENAME_HERE>-blobs*",
                "arn:aws:s3:::<STAGENAME_HERE>-log-snaps*",
                "arn:aws:s3:::<STAGENAME_HERE>-backups*"
            ]
        },
        {
            "Sid": "efsDeny",
            "Action": "elasticfilesystem:Delete*",
            "Effect": "Deny",
            "Resource": [
                "*"
            ]
        },
        {
            "Sid": "efsAllow",
            "Action": "elasticfilesystem:*",
            "Effect": "Allow",
            "Resource": [
                "*"
            ]
        },
        {
            "Sid": "iamAllow",
            "Action": [
                "iam:AddRoleToInstanceProfile",
                "iam:Get*",
                "iam:ListRoles",
                "iam:ListEntitiesForPolicy",
                "iam:ListPolicyVersions",
                "iam:ListInstanceProfiles",
                "iam:ListInstanceProfilesForRole",
                "iam:ListRolePolicies",
                "iam:PassRole",
                "iam:RemoveRoleFromInstanceProfile"
            ],
            "Effect": "Allow",
            "Resource": [
                "arn:aws:iam::<AWS_ACCOUNT_ID_HERE>:policy/<STAGENAME_HERE>*",
                "arn:aws:iam::<AWS_ACCOUNT_ID_HERE>:role/<STAGENAME_HERE>*",
                "arn:aws:iam::<AWS_ACCOUNT_ID_HERE>:instance-profile/<STAGENAME_HERE>*"
            ]
        }
    ]
}
```

Creating / importing security groups example:

```bash
[shizuko : 5:26PM] /tmp>aws ec2 create-security-group --vpc-id vpc-1d51ca7b --group-name iamtest-domino_dis_int --description "Security group for the Domino dispatcher internal ELB"
{
    "GroupId": "sg-f22cc38f"
}
[shizuko : 5:27PM] /tmp>aws ec2 create-security-group --vpc-id vpc-1d51ca7b --group-name iamtest-executor --description "Security group for the Domino executor servers"
{
    "GroupId": "sg-f62dc28b"
}
[shizuko : 5:29PM] /tmp>aws ec2 create-security-group --vpc-id vpc-1d51ca7b --group-name iamtest-domino_fe_ext_elb --description "Security group for the Domino external ELB"
{
    "GroupId": "sg-2b2cc356"
}
[shizuko : 5:30PM] /tmp>aws ec2 create-security-group --vpc-id vpc-1d51ca7b --group-name iamtest-domino_fe_int --description "Security group for the Domino internal ELB"
{
    "GroupId": "sg-432dc23e"
}
[shizuko : 5:31PM] /tmp>aws ec2 create-security-group --vpc-id vpc-1d51ca7b --group-name iamtest-general --description "Security group for the Domino general server"
{
    "GroupId": "sg-d92dc2a4"
}
[shizuko : 5:32PM] /tmp>aws ec2 create-security-group --vpc-id vpc-1d51ca7b --group-name iamtest-executor-efs --description "Security group for the Executor EFS access"
{
    "GroupId": "sg-f32ac58e"
}
[shizuko : 5:32PM] /tmp>aws ec2 create-security-group --vpc-id vpc-1d51ca7b --group-name iamtest-general-efs --description "Security group for the Dispatcher EFS access"
{
    "GroupId": "sg-f928c784"
}

# The resulting policy resource section:

            "Resource": [
                "arn:aws:ec2:us-west-2:198832413611:security-group/sg-f22cc38f",
                "arn:aws:ec2:us-west-2:198832413611:security-group/sg-f62dc28b",
                "arn:aws:ec2:us-west-2:198832413611:security-group/sg-2b2cc356",
                "arn:aws:ec2:us-west-2:198832413611:security-group/sg-432dc23e",
                "arn:aws:ec2:us-west-2:198832413611:security-group/sg-d92dc2a4",
                "arn:aws:ec2:us-west-2:198832413611:security-group/sg-f32ac58e",
                "arn:aws:ec2:us-west-2:198832413611:security-group/sg-f928c784"
            ]

# Finally, after doing a dry provision, the import into terraform:
terraform import -state=./deploys/iamtest/terraform.tfstate aws_security_group.domino_dis_int sg-f22cc38f
terraform import -state=./deploys/iamtest/terraform.tfstate aws_security_group.domino_executor sg-f62dc28b
terraform import -state=./deploys/iamtest/terraform.tfstate aws_security_group.domino_fe_ext_elb sg-2b2cc356
terraform import -state=./deploys/iamtest/terraform.tfstate aws_security_group.domino_fe_int sg-432dc23e
terraform import -state=./deploys/iamtest/terraform.tfstate aws_security_group.domino_general sg-d92dc2a4
terraform import -state=./deploys/iamtest/terraform.tfstate aws_security_group.executor_efs sg-f32ac58e
terraform import -state=./deploys/iamtest/terraform.tfstate aws_security_group.general_efs sg-f928c784

# After this, bash /scripts/provision.sh finished successfully...
```

General IAM policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:RunInstances"
            ],
            "Resource": [
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:subnet/<DOMINO GENERAL SUBNET HERE>",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:subnet/<DOMINO EXECUTOR SUBNET HERE>",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:instance/*",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:network-interface/*",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:volume/*",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:key-pair/<STAGENAME_HERE>-access-key",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:vpc/<DOMINO VPC HERE>",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:security-group/<DOMINO GENERAL SECURITY GROUP HERE>",
                "arn:aws:ec2:<AWS_REGION_HERE>:<AWS_ACCOUNT_ID_HERE>:security-group/<DOMINO EXECUTOR SECURITY GROUP HERE>"
            ]
        },
        {
            "Action": [
                "ec2:RunInstances"
            ],
            "Resource": [
                "arn:aws:ec2:<AWS_REGION_HERE>::image/*"
            ],
            "Effect": "Allow",
            "Condition": {
                "StringLike": {
                    "ec2:ResourceTag/Name": "<STAGENAME_HERE>-*"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeInstance*",
                "ec2:CreateTags",
                "ec2:DescribeTags",
                "ec2:CreateSnapshot",
                "ec2:CreateImage",
                "ec2:DescribeImages",
                "ec2:DescribeSnapshots",
                "ec2:DeleteSnapshot",
                "ec2:AssignPrivateIpAddresses",
                "ec2:AssociateAddress",
                "ec2:DescribeAddresses",
                "ses:SendRawEmail"
            ],
            "Resource": "*"
        },
        {
            "Sid": "EC2TagLimitedDelete",
            "Effect": "Allow",
            "Action": [
                "ec2:DeregisterImage",
                "ec2:DeleteTags"
            ],
            "Resource": "*",
            "Condition": {
                "StringLike": {
                    "ec2:ResourceTag/Name": "iamtest-*"
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:StartInstances",
                "ec2:StopInstances",
                "ec2:RebootInstances",
                "ec2:TerminateInstances"
            ],
            "Condition": {
                "StringLike": {
                    "ec2:ResourceTag/domino.stage": "?*"
                },
                "ForAnyValue:StringLike": {
                    "ec2:InstanceProfile": [
                        "arn:aws:iam::<AWS_ACCOUNT_ID_HERE>:instance-profile/<STAGENAME_HERE>-executor"
                    ]
                }
            },
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "iam:PassRole",
            "Resource": "arn:aws:iam::<AWS_ACCOUNT_ID_HERE>:role/<STAGENAME_HERE>-executor"
        }
    ]
}
```

Executor IAM Policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:DescribeTags"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:StopInstances"
            ],
            "Condition": {
                "StringLike": {
                    "ec2:ResourceTag/domino.stage": "?*"
                }
            },
            "Resource": "*"
        }
    ]
}
```

S3 IAM Policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::<STAGENAME_HERE>-blobs*",
                "arn:aws:s3:::<STAGENAME_HERE>-log-snaps*",
                "arn:aws:s3:::<STAGENAME_HERE>-backups*"
            ]
        }
    ]
}
```
