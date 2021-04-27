# Set up the workshop environment

![](media/ssm-aws-logo.png)

NOTE: You will incur charges as you go through either of these workshops, as they will exceed the [limits of AWS free tier](http://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-limits.html).

## Table of Contents

- [Summary](#summary)
- [Manual instructions](#manual-instructions)
    - [Create a test Windows EC2 instance](#create-a-test-windows-ec2-instance)
    - [Enable Quick Setup to create and attach the IAM role](#enable-quick-setup-to-create-and-attach-the-iam-role)
    - [Create the KMS key](#create-the-kms-key)
    - [Add KMS permissions to the IAM role](#add-kms-permissions-to-the-iam-role)
    - [Confirm the EC2 instance is registered](#confirm-the-ec2-instance-is-registered]
- [CloudFormation instructions](#cloudformation-instructions)
    - [Create test resources using CloudFormation](#create-test-resources-using-cloudformation)
- [Next Section](#next-section)

## Summary

In this section of the workshop you create an Amazon Elastic Compute Cloud (EC2) Windows instance with a PowerShell script that will stress test the CPU and consume CPU credits, if any are available. In the next section, you will create an Amazon CloudWatch alarm that monitors CPU usage. You can (1) manually launch an EC2 instance and configure AWS Systems Manager or (2) you create these resources using [AWS CloudFormation](https://aws.amazon.com/cloudformation/).

To create the test resources manually, skip to the section [Manual Instructions](#manual-instructions).

To create the resources using CloudFormation, skip to the section [CloudFormation instructions](#cloudformation-instructions).

## Manual Instructions

### Create a test Windows EC2 instance

1. Open the Amazon Elastic Compute Cloud (EC2) console at https://console.aws.amazon.com/ec2/v2/home.

    - **Important**: Ensure you are using the N. Virginia (us-east-1) region.

1. Choose **Launch Instance** and choose **Launch instance**.
1. In the list of Amazon Machine Images (AMI), choose **Select** for the **Microsoft Windows Server 2019 Base** AMI.
    - **Note**: You may need to scroll down.
    
    ![](/media/episode-03-windows-ami.png)
    
1. Choose the instance type **t3.medium** and choose **Next: Configure Instance Details**.

    ![](/media/ep03-p10.png)

1. On the **Step 3: Configure Instance Details** screen, perform the following steps:
    - Expand the **Advanced Details** section and copy/paste the following PowerShell script in to the **User data**.
    - **Note**: The PowerShell script below will download a test PowerShell script to stress test the CPU.

```
<powershell>
#
#Download the script and save to the c: drive
#
Invoke-WebRequest `
    https://notasdofelip.s3.amazonaws.com/loop-and-stress.ps1 `
    -OutFile c:\loop-and-stress.ps1

#
#Create Registry Entry needed by the stress script
#
New-Item -Path HKLM:\SOFTWARE -Name "benfelip"
Set-ItemProperty -Path HKLM:\SOFTWARE\benfelip -Type DWORD -Name CrazyLogs -Value 1

#Setup the trigger to run everytime it boots
$trigger = New-JobTrigger -AtStartup -RandomDelay 00:00:30
Register-ScheduledJob -Trigger $trigger -FilePath c:\loop-and-stress.ps1 -Name StressCPU

</powershell>
```

    ![](/media/episode-03-userdata.png)

1. Leave all the other the settings as default and choose **Next: Add Storage**.
1. For the **Step 4: Add Storage** page, leave the default settings and choose **Next: Add Tags**.
1. For the **Step 5: Add Tags** page, choose **click to add a Name tag**, enter ```TestWindowsInstance``` for the **Value**, choose **Next: Configure Security Group**.

    ![](/media/episode-03-tags.png)

1. For the **Step 6: Configure Security Group** page, choose **Create a new security group** and change the **Source** from **Custom** to **My IP**. Choose **Review and Launch**.

    ![](/media/episode-03-security-group.png)

1. Choose **Launch**.
1. For**Select an existing key pair or create a new key pair**, choose **Proceed without a key pair** from the drop-down list, choose **I acknowledge that I will not be able to connect to this instance unless I already know the password built into this AMI.** and choose **Launch Instances**.
    - **Note**: In the next section you will use Session Manager to remotely connect to the EC2 instance which does not require a key pair.

### Enable Quick Setup to create and attach the IAM role

Use AWS Systems Manager Quick Setup to quickly configure frequently used AWS services and features with recommended best practices. You can use Quick Setup in an individual account or across multiple accounts and AWS Regions by integrating with AWS Organizations. Quick Setup simplifies setting up services, including AWS Systems Manager, by automating common or recommended tasks. These tasks include, for example, creating required AWS Identity and Access Management (IAM) instance profile roles and setting up operational best practices, such as periodic patch scans and inventory collection.

**To setup Quick Setup Host Management**

1. Open the AWS Systems Manager console at https://console.aws.amazon.com/systems-manager/.
1. In the navigation pane, choose [**Quick Setup**](https://console.aws.amazon.com/systems-manager/quick-setup).
1. Choose **Get started**.
1. Choose **Create**.
1. Choose **Host Management** and choose **Next**.

    - **Note**: If your AWS account is part of an AWS Organization and you are logged in to the root organization account, you also have an option to configure the Change Manager capability of Systems Manager.
    
1. On the **Customize Host Management configuration options**, ensure the defaults for **Systems Manager** are enabled:

    - Update Systems Manager (SSM) Agent every two weeks.
    - Collect inventory from your instances every 30 minutes.
    - Scan instances for missing patches daily.

1. In the **Targets** section, choose **Current Region** and choose **All instances**.
1. Choose **Create**.

    ![](/media/quick-setup-create.png)
    
    - Systems Manager Quick Setup will begin deploying a CloudFormation stack which creates the corresponding resources to enable Systems Manager best practices. After a few moments, the **Host Management** page should refresh and you can see the **Configuration deployment status**, **Configuration association status**, and **Configuration details** for the **Quick Setup** deployment.

    ![](/media/quick-setup-host-mgmt.png)

1. Choose the radio button for the current account and Region and choose **View details**.    
1. On the **Association drilldown** page, you can review the **Association status**, **Instances per status**, **Schedule rate**, and **Last updated** timestamp for each association created by **Quick Setup**.

    ![](/media/quick-setup-drilldown.png)

### Create the KMS key

In order to fully manage the instance with Fleet Manager, you need to configure a KMS Key and I assume you are following this in order.

1. Open the AWS Key Management Service (KMS) console [at](https://console.aws.amazon.com/kms/)
1. In the navigation pane, choose Customer managed keys
1. Choose Create key
1. On the Configure key page, leave the default values and choose **Next**
1. On the Add labels page, for Alias enter session-manager, and choose **Next**
1. On the Define key administrative permissions page, select your IAM role TeamRole, and choose **Next**.
1. On the Define key usage permissions page, select the EC2 IAM instance profile role **AmazonSSMRoleForInstancesQuickSetup**, and choose **Next**
On the Review page, choose **Finish**
1. Once the KMS key is created, note the Amazon Resource Name (ARN) of the KMS key and copy the value
    ![](https://workshop.aws-management.tools/ssm/capability_hands-on_labs/media/fleet-manager-kms.png)

### Add KMS permissions to the IAM role
1. Open the AWS Identity and Access Management (IAM) console at https://console.aws.amazon.com/iam/home
1. In the navigation pane, choose Roles
1. Select the **AmazonSSMRoleForInstancesQuickSetup** role from the list
1. On the Summary page, choose **Add inline** policy
1. On the Create policy page, choose the **JSON tab**.
1. Replace the default content with the following. **Important:** replace key-name with the ARN of the custom KMS key you created previously.
```
{
"Version": "2012-10-17",
"Statement": [
    {
        "Effect": "Allow",
        "Action": [
            "kms:Decrypt"
        ],
        "Resource": "key-name"
    }
]
}

```
### Confirm the EC2 instance is registered
1. Open System Manager console at https://console.aws.amazon.com/iam/home 
1. Go to Session Manager and click **Preferences** tab.
1. Click edit and for KMS Key, choose the key you created in the previous session.
    ![](/media/ep03-p14.png)
1. Click **Save** at the bottom at the page. 
1. On the Right menu click **Fleet Manager**.
1. You should see your managed instance listed
    (It might take a couple of minutes)
    ![](/media/ep03-p15.png)
1. System Manager and Fleet Manager are ready to go.
1. Reboot the instance so the script you installed through user data can be triggered and proceed to the next session.

You have now completed the manual setup instructions for this workshop. You can now proceed to the [Next Section](#next-section).

## CloudFormation instructions

The [CloudFormation template](misc/create-instance-unmanaged.yaml) creates a test Windows EC2 instance and IAM instance profile role for System Manager. if you launched the CloudFormation, skip to **Configure System Manager Fleet Manager** in the next [section](episode-03-step-02-troubleshoot.md)

### Create test resources using CloudFormation

**To save the CloudFormation template locally**
    
1. Open the CloudFormation template [ssm-workshop-resources-episode-01.yml](cfntemplates/ssm-workshop-resources-episode-01.yml).
1. Choose **Raw**.

    ![](/media/github-raw.png)

1. Open Notepad and copy the entire text.
1. Save the file to your local machine as ```ssm-workshop-resources-episode-01.yml```.

The CloudFormation template will create the resources depicted in the diagram below.

![](/media/ep01-st01.png)

**To create the test EC2 instances**
    
1. Open the [AWS CloudFormation console](https://console.aws.amazon.com/cloudformation/home).
1. Choose **Create stack**.
1. For **Specify template**, choose **Upload a template file**, choose the file you saved locally ```ssm-workshop-resources-episode-01.yml```, and choose **Next**.

    ![](/media/cloudformation-create-stack-ep01.png)

1. For **Stack name**, enter ```ssm-workshop```, and choose **Next**.
1. On the **Configure stack options** page, leave the defaults and choose **Next**.
1. On the **Review** page, choose **Create stack**.

CloudFormation will begin provisioning the resources specified within the CloudFormation template and once complete, you will have two Amazon Linux 2 EC2 instances to test using Systems Manager. You can also use the refresh button to see the latest events related to the CloudFormation stack. Once the status of the CloudFormation stack changes to ```CREATE_COMPLETE```, you can proceed with the next steps. This process should complete within 5 minutes.

You have now completed the CloudFormation setup instructions for this workshop. You can now proceed to the [Next Section](#next-section).

## Next Section

**Congratulations** You ready to start configuring System Manager and troubleshoot this instance.

Click the link below to go to the next section.

[![](media/episode-03-troubleshoot.png)](/episode-03-step-02-troubleshoot.md)