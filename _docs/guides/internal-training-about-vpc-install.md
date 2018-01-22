---
title: Internal Training About VPC Install/Deployment
category: Guides
order: 2
---

Table of contents here?

## Pre-pre Requirements

### I'm about to do a deploy - what version of the deployer, Domino artifacts, and manifest template should I use?

1. Find out the right version that is "Gold" from Product wiki. If you're unsure, you can ask a CS team lead
2. Go to https://github.com/cerebrotech/domino/releases to get the deployer docker image tag and artifacts dir to use in the process below. Use the "Latest release"
3. Manifest template:
    a. VPC? Depending on your deploy layout and options, use one of the manifests from [Standard AWS VPC Deploys]()
    b. On-Prem? You must be Ozzy Johnson, you dont need no stinkin' template

### Setting ourselves up for success

This is a team effort. Long story short, we prefer managed VPCs where Domino is in full control.

See [How is Domino deployed in the wild?]() for more information

See [Pre-Install Call]() process.

### Pre-requisites

- Domino 101 & 102 should be completed as part of new Domino employee on-boarding – often during your first or second week
- Domino Architecture & Admin Training should be completed (preferrably as part of a live group discussion) – often in your first few weeks as a CSE/SE

### Overview of a Deploy

1. Pre-deploy
    1. Sales handoff, CSM stuff
    2. Pre-install / solution architecture questionnaire/negotiation (Deploys Process - Documentation & SOW requirements)
    3. Manifest (configuration file) creation
    4. AWS Account setup
    5. Deploy server setup
2. Deploy
    1. Provision step (Terraform & Kops)
        1. dry
        2. normal
    2. Deploy step (Salt Stack)
        1. normal
    3. Sanity checks
3. Post deploy
    1. Configuration and CS checks (CS-1909)
    2. Smoke tests (currently manual)
    3. UAT (with an actual user)
4. Ongoing support
5. Upgrade (Upgrade Checklist)

## Deploy Process in Detail

### Pre-Deploy

#### AWS account setup and hygeine

- Setup AWS sub account (usually only for Managed VPC sub accounts)
    - Create new account from domino root account using an `ops+CUSTOMER_NAME@dominodatalab.com` email
    - Go to aws.amazon.com, click login with root creds, and reset the password for the account
    - Login as root for the new sub account and enable MFA (take a screenshot of the MFA activation barcode for the next step)
    - Set a custom IAM sign-in link (domino-CUSTOMERNAME.signin.aws....)
    - Share the email address, root password, and a screenshot of the MFA barcode w/ Ozzy, Joaquin, and Mark securely over keybase
    - Create a domino-staff group that has the admin policy and create named IAM accounts for the right people – always include Ozzy, Joaquin, and Mark
- Log into AWS console for first time

##### Request Service Limit Increase

While this isn't strictly required, it's generally a really good idea to request limit increases really early because they take some time to get through.

Increase the following limits:

| Limit	                          | Recommended Value |
|:--------------------------------|:------------------|
| EC2 Instance Limit: m4.large    | 20
| EC2 Instance Limit: m4.xlarge	  | 20
| EC2 Instance Limit: m4.2xlarge  | 20
| EC2 Instance Limit: m4.4xlarge  | 10
| EC2 Instance Limit: c4.4xlarge  | 10
| EC2 Instance Limit: r4.4xlarge  | 10
| EC2 Instance Limit: p2.xlarge   | 5
| EC2 Instance Limit: p2.8xlarge  | 5
| EC2 Instance Limit: m4.10xlarge | 5
| EC2 Instance Limit: r4.8xlarge  | 5
| (If public IPs required) VPC Elastic IP Addresses	| 6 if dedicated account; 10 if shared
| EBS General Purpose (SSD) gp2 Storage	 | 40 TB
| SES Sending Limit	| Just...increase it?

(Verify support+CUSTOMER@dominodatalab.com in the region where SES is used, verification link will come through Zendesk)

##### Route53 / DNS

> **WARNING!** Unless you are deploying to _customer.domino.tech_, please read this section carefully!

If you to a non-Domino domain (i.e., to a customer's DNS), then some extra prep work will be necessary:

- If the customer is allowing us to be delegated DNS to the domino subdomain (domino.customer.com), then we'll 
    - Go to Route53
    - Create a new hosted zone
    - Type 'domino.customer.com' into the domain
    - Select Public Hosted Zone
    - When this is complete, you'll need to send a note to the customer to actually delegate DNS.  Here's a template note to send:

       > Hi ${CUSTOMER_ADMIN} -
       >  
       > I have created a 'domino.customer.com' hosted zone. Here are the NS records:
       >
       > ns-1826.awsdns-36.co.uk.
       >
       > ns-1496.awsdns-59.org.
       >
       > ns-515.awsdns-00.net.
       > ns-400.awsdns-50.com.
       > Can you please create NS record under your 'customer.com' zone that points to those hosts? This will allow us to create certificates for your Domino environment.
       > Thanks,
       > ${DEPLOY_ENGINEER}

You can verify that they've successfully done this by running this command:

```
dig cloud-shiny.domino.tech NS

; <<>> DiG 9.9.7-P3 <<>> cloud-shiny.domino.tech NS
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25109
;; flags: qr rd ra; QUERY: 1, ANSWER: 4, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;cloud-shiny.domino.tech. IN NS

;; ANSWER SECTION:
cloud-shiny.domino.tech. 21599 IN NS ns-1103.awsdns-09.org.
cloud-shiny.domino.tech. 21599 IN NS ns-1935.awsdns-49.co.uk.
cloud-shiny.domino.tech. 21599 IN NS ns-273.awsdns-34.com.
cloud-shiny.domino.tech. 21599 IN NS ns-636.awsdns-15.net.

;; Query time: 143 msec
;; SERVER: 172.24.1.1#53(172.24.1.1)
;; WHEN: Fri Jan 19 13:20:24 EST 2018
;; MSG SIZE rcvd: 192
```

If you need to setup Route 53 manually (as opposed to using the native hosted zone functionality in the deployer): _TBD_


##### SSL Certificate

If the customer is not providing the SSL cert or ACM ARN, you will need to request a certificate for customer.domino.tech
and *.customer.domino.tech.   

> **Note:** You can self generate a domino.tech cert, after doing so via AWS Certificate Management capture the ARN to use for deployment wizard or MANIFEST.

#### SSH Key Pair

Generate and securely store (keybase for now) EC2 private key pair. You can do this in AWS console / EC2 / Key Pairs:

```
chmod 0600 domino-CUSTOMER.pem
```

from terminal, use the following to generate the key pair's public key:

```
ssh-keygen -y -f domino-CUSTOMER.pem > domino-CUSTOMER.pub
```

#### Mixpanel and NewRelic

- Create a Mixpanel project just for this customer. Mixpanel credentials can be found here.
- Create a NewRelic sub-account for the customer
    - To retrieve New Relic "API key"
    - Create a new account for the company. Go here: https://rpm.newrelic.com/accounts/1210898
    - Click Servers
    - Click on Ubuntu
    - Scroll down and find the license key. This license key is the "API key" that you put in the manifest

#### Preparing the Deployer Instance

We must now prepare the instance from which the deployer will run.  While the deployer is technically capable of running
from your local machine, it is generally recommended that you provision an instance for it separately.

> **WARNING!** Are you using private IPs only? See VPC Deploy: Private IPs only (provisio bastion, deploy from it)

1. Create an ec2 t2.micro with the existing keypair and a SG to allow only incoming on port 22 with the latest ubuntu server AMI; 16gb root drive should be enough
2. SSH into the server, become root i.e., sudo su
3. Run the following:

   ```
   mkdir -p /domino/deployment/keys && cd /domino/deployment/keys
   ```
4. cp the public and private key over to /domino/deployment/keys
    - `chmod 0600 /domino/deployment/keys/domino`
5. Stage Helper Scripts & Install Docker (this is an unversioned tar file that inclused helper scripts to install docker and download the deployer)

   ```
   cd /domino/deployment \
     && wget https://s3-us-west-2.amazonaws.com/domino-operations/deployment/deployment.tar.gz \
     && tar -xzvf deployment.tar.gz \
     && sh install_docker.sh \
     && chmod +x run_deployer.sh \
     && docker login -u="domino+dominoinstaller" -p="D38WABZUBGGTIXL3OI65BX9WGLMARROMYSZ3MF5CUV6IY5U2JX9UCMVYMVIXTC6E" quay.io
   ```


#### Prepare the Deployment Directory

> **WARNING!** STAGE_NAME must be between 6 and 17 Characters long.  Stage may contain only lowercase characters, digits and hyphen.

- put your previously created manifest YAML file at /domino/deployment/deploy/STAGE_NAME/deploy.yaml
    mkdir -p deploy/<stage_name>
    cd deploy/<stage_name>
    vim deploy.yaml
    Copy and paste the edited manifest
    Be sure sure sure you have added the things mentioned in the deployment notes. For 2.1, this relates directly to workspaces
- cd /domino/deployment and edit run_deployer.sh to have the right version of the deployer you want to use and the proper STAGE_NAME:
- Edit top of run_deployer.sh and set the 3 below vars to the correct values (this should be changed to use env or a single sourced env setup)

  ```
  DEPLOY_IMAGE="quay.io/domino/deployer:v<deployer version tag>"
  DEPLOY_PREFIX="domino-deployer-<deployer version>"
  DOMINO_STAGE="<stage name>"
  ```

### Manifest Setup

#### Wizard

If you are running >= 2.3.4 of the Deployer, you can use the new manifest wizard.  To start it, from inside the Deployer container, run:

```
bin/domino-deployment plan wizard
```

This will start an interactive wizard that leads you through the major decisions of configuring a deployment:

![](/img/guides-wizard.png)

There is inline documentation on each screen of the wizard, accessible through <F1> or Ctrl-O.

#### Advanced Manual Configuration

> **Proceed with caution!** Configuring the manifest by hand can be very tricky.

If you are running a deployer older than 2.3.4 or you cannot use the wizard for some reason, then you'll need to manually configure the manifest file. 

- All manifest templates are inside the Deployer image at the path `/domino-deployer/manifests`.  It is highly
  recommended that you use the version of the manifest template in that image (rather than going directly to our main
  app repo). There are many templates, so pick the correct starting template:

  | Status        | Template Name	            | Purpose
  |:--------------|:----------------------------|:---
  | Supported     | cs-central-template.yaml	| THE STANDARD: A small (< 25 user) production Domino deployment
  | Supported     | cs-fanned-out-template.yaml	| A large (>= 25 user) production Domino deployment
  | Informational | cs-central-govcloud.yaml	| A production Domino deployment deployed into a govcloud region.  Working when Pulu added, but not actively maintained
  | Informational | dev.yaml	                | Pure internal testing Domino deployment for application developers to test pre-release code.  Optimized for size and speed of deployment
  | Informational | dev-onprem.yaml	            | Pure internal testing Domino deployment simulating an on premise deployment.  
  |               | All other templates	        | Not actively maintained; presume to be totally broken

- This will take some time. Go through each line thoroughly and make sure you understand the implications of changing it.
- Make heavy use of the manifest checks! 
- If they are using public IPs (ie they want to be able hit without internal routing), make sure to include `use_elb: true`
- Try to configure email and LDAP (if applicable) at this point as well

### Pre-Deploy Check

Do not proceed to deploy until these are all met!

- DNS: is the customer ready to set DNS when we have the ELBs' CNAMEs to give them?
- SSL: have we received the needed SSL certificate in ACM from the customer
- AWS: Do we have root, admin, or sufficiently permissive access to AWS so we can do the deploy? Have limits for EC2 machines been granted by AWS?
- Network access/routing: Is there clarity on how users will enter the VPC? Is it consistent across users and does Domino have a path in too? What about accessing data sources in other VPCs and data centers?
- CSE- have you read the PRD deploy notes and deployment notes end-to-end? They are linked from the github release page.

## Deploy

### Running the deployer code

- `./run_deployer.sh` (DOMINO_STAGE, DEPLOY_IMAGE and DEPLOY_PREFIX are set in run_deployer.sh, edit at top of script)
- You are now inside the deployer docker container
- export your AWS access key id and secret for the deployer to use: 

  ```
  export AWS_ACCESS_KEY_ID=___
  export AWS_SECRET_ACCESS_KEY=___
  ```
- NOTE: If you have MFA enabled for your account, you'll have to get temporary credentials this way. https://aws.amazon.com/premiumsupport/knowledge-center/authenticate-mfa-cli/ 

  Additionally, you will need to set the new AWS_ACCESS_KEY_ID,AWS_SECRET_ACCESS_KEY and AWS_SESSION_TOKEN that is returned in the JSON payload from the aws sts command shown below.

  ```bash
  #Install AWS Client
  pip install awscli --upgrade --user

  #Command to get keys when you have mfa from iam/security credentials tab on AWS Console
  #36 Hr Timeout
  aws sts get-session-token --serial-number arn:aws:iam::198832413611:mfa/john.lynch --token-code <code from token> --duration-seconds 129600
  ```
- cd /domino-deployer
- Provision, then deploy

  ```
  bin/domino-deployment plan check-manifest ./deploys/$DOMINO_STAGE/deploy.yaml
  bin/domino-deployment provision $DOMINO_STAGE --dry # Dry provision
  bin/domino-deployment provision $DOMINO_STAGE # Provision
  # bin/domino-deployment provision $DOMINO_STAGE --force # Force provision. Only use if you have double and triple checked the changes.

  bin/domino-deployment deploy $DOMINO_STAGE
  ```
- If managed VPC and using Domino's SSL and DNS, set up Route 53 in domino-test sub account. Otherwise, ask customer to
  create similar CNAME records)
    - In AWS console, on our domino-test environment, go to Route 53 / Hosted Zones.
    - Click into domino.tech. hosted zone
       - (It will take a while for all the records to load)
    - Create Record Set (Type will be CNAME for all four)
       - <customer>.domino.tech
           - Name: <customer>.domino.tech
           - Value: <address for FE ELB>
       - *.<customer>.domino.tech
           - Name: *.<customer>.domino.tech
           - Value: <address for FE ELB>
       - dispatcher.<customer>.domino.tech
           - Name: dispatcher.<customer>.domino.tech
           - Value: <address for dispatcher ELB>
       - models.<customer>.domino.tech
           - Name: models.<customer>.domino.tech
           - Value: <address for model manager ELB, it's the crazy looking one>

## Post Deploy

- Run the sanity checks (see Runbook Sanity Checks Runbook for more details)
  ```
  root@deployer-image:/domino-deployer# bin/diag-domino -s $DOMINO_STAGE
  ```
- We are currently tracking deploy time. Add details to [this spreadsheet](https://docs.google.com/spreadsheets/d/1CVquSR8tuDmxuO9kRRpVQJPCDYZaOi6cMHEMm7Uk9D8/edit?ts=5a270581#gid=0)
- Backup the deployer by uploading the deploys, keys, and scripts folders and the install_docker.sh and run_deploy.sh files:
    - Click here to create a new private repo called "deploy-CUSTOMER" under the company org
        - Add the customer-deployers team to the Github repo with Admin access. (Settings > Collaborators and teams)
        - Prior to following the steps in the code block below, address the following authentication pre requisite steps:
        - Rather than using Personal Access Tokens as before (specific to the user), Git now has Deploy Keys (specific to the repo). This will provide a more secure and flexible form of access since it isn't tied to individuals.
            - reference https://gist.github.com/zhujunsan/a0becf82ade50ed06115 to work through the steps as root (clarifying notes to be used in combination with the article are below):
                - Step 1: Instead of generating a key pair, use the key pair under /domin/deployment/keys
                - Step 2: use /domino/deployment/keys/domino.pub (make sure to check Allow write access)
                - Step 3: cp /domino/deployment/keys/domino ~/.ssh
                - Additional Step: vim ~/.ssh/config (this is a new file, paste the following)
                  ```
                  host github.com
                  HostName github.com
                  IdentityFile ~/.ssh/domino
                  User git
        - Then:
          ```
          cd /domino/deployment
          git init
          git add keys deploy run_deployer.sh install_docker.sh
          git commit -m "Installed 2.3.3 <or Upgraded from 2.0.1 to 2.3.3>"
          git remote add origin git@github.com:cerebrotech/deploy-<customer name>.git
          git push origin master
          ```
- The private deployment hygiene checklist is very important; follow it thoroughly: https://dominodatalab.atlassian.net/browse/CS-1909
- If needed for access to data sources in other VPCs, configure VPC peering including SG tweaks and route table updates
- Update the SFDC record for the deploy please  :)

## Destroy a Deploy

Our deployer creates a lot of infrastructure above and beyond the EC2 instances. It is strongly advised you use the deployer "destroy" command (as described below) to destroy a deploy and all of its assets – terminating EC2 instances is not enough.

> **Be careful...** this will cause data loss. 

1 .In the AWS console, go to EC2 instances and disable termination protection on all servers
2. Get onto the deployer box and into the deployer docker container
   ```
   cd /domino-deployer
   bin/domino-deployment destroy $DOMINO_STAGE
   ```
   Follow prompts (say "yes" to confirmation()

3. Go back into the AWS console and empty the S3 buckets and then delete them too

