# sftp
Upload data automatically to Amazon S3 from Outlook

On this post I will explain how to solve the problem that a lot of companies are facing at the moment of syncronizing and uploading automatically data from Microsoft Office tools like Outlook or Sharepoint to AWS.

The different secctions on this article:

Create a new Amazon S3 Bucket
Create a new IAM Role and Policy for accessing the S3 bucket
Creae a new SFTP server using AWS Transfer for SFTP
Add user on the SFTP server
Test the connection to SFTP using WinSCP
Create a Power Automate Flow and connect it to Amazon S3 via SFTP
The first we will need to do is to create an Amazon S3 Bucket and configure SFTP using AWS Transfer for SFTP.

1. Create a new Amazon S3 Bucket
Sign in to the AWS Management Console and open the Amazon S3 console at https://console.aws.amazon.com/s3/.
Choose Create bucket. The Create bucket wizard opens.
In Bucket name, enter a DNS-compliant name for your bucket.The bucket name must:
Be unique across all of Amazon S3.
Be between 3 and 63 characters long.
Not contain uppercase characters.
Start with a lowercase letter or number.
In Region, choose the AWS Region where you want the bucket to reside.
Under Object Ownership disable ACLs
In Bucket settings for Block Public Access, choose the Block Public Access settings that you want to apply to the bucket.
You can have some extra optional configuration such as Default encryption or Tags but for the purpose of this blog post this are not needed.
Finally click Create bucket

2. Create a new IAM Role and Policy for accessing the S3 bucket
Sign in to the AWS Management Console and open the IAM Management Console.
Click on Create role
Select AWS Service
On Use case select Transfer and click on the bullet point in Transfer below.

5. Click on Next (on the bottom-right)
6. On this page we need to select the permissions policies. You can just go wild and select AmazonS3FullAccess and then the role will have full access to all the buckets and will be able to do everything. That’s not recommended, actually, in cloud we apply The Principal of Least Privileges therefore in this case we will create a custom policy with limited access. For that we need to click on Create policy (on the top-right).

Once the new tab we click on the JSON tab and we paste the following code:

{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:PutObject",
                "s3:GetObject",
                "s3:AbortMultipartUpload",
                "s3:CreateBucket",
                "s3:ListBucket",
                "s3:DeleteObject",
                "s3:ListMultipartUploadParts"
            ],
            "Resource": [
                "arn:aws:s3:::[s3-bucket-name]/*",
                "arn:aws:s3:::[s3-bucket-name]"
            ]
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "s3:ListAllMyBuckets",
            "Resource": "*"
        }
    ]
}
Please, note you need to replace [s3-bucket-name] with the name of the bucket you created previously. With this JSON script what we are doing is creating a policy that will provide FULL access to ONLY the bucket created before but not the rest of buckets we have. The only thing the policy is able to do on other buckets is to list all the bucket names we have but only that.

In my case it looks like this:


7. Then we click on Next:Tags and then Next:Review.
8. Finally we have to set the policy name and click on Create policy. For the policy names it’s recommended to use a naming convention so it’s easier to maintain them.
9. Once we have create the policy we have to go back on the page Create role, click on the Refresh button next to the Create policy button and then select the policy we just created and then click Next.
10. Finally we have to put a name to the role and click Create role.

Once the role has been created, we can click on it and see details like the permissions policies attached to it:


3. Creae a new SFTP server using AWS Transfer for SFTP
Sign in to the AWS Management Console and open the AWS Transfer for SFTP console at https://aws.amazon.com/sftp/.
Click on Create server.
Next we need to select the Identity provider. In this case we have 2 options but for this blog we will use Service managed because it’s much easier to configure and doesn’t require any exisiting LDAP or Active Directory.
On the next step we will set as Publicly accessible endpoint type
You can add a DNS configuration using Amazon Route 53 but in this case we don’t need it. That would be nice if you want to use a DNS name at the moment of connecting instead of an IPv4 (e.g: sftp.mysite.com)
Then we have to choose the domain, in this case we will use Amazon S3
Last step on the configuration is to provide a role for logging into CloudWatch. Here you can just click Create a new role or select an exisiting role if you already have one.
Finally we click on Create server

4. Add user on the SFTP server
Now we can add a new user! For that we need to select the server we just created and then Add user.


Then we just need to fill all the data on the form:

Username
Select the Role that can access the S3 bucket we just created before.
Select the S3 Bucket created on the first step on Home directory.

After filling all this information you will have something similar to this:

Then we have to add an SSH public key, this key will be used as a method of authentication. Basically, when we connect to the SFTP server we will use the Username and the Key we will generate.

For generating a key there are multiple way to do it and you can check them here: https://docs.aws.amazon.com/transfer/latest/userguide/key-management.html#sshkeygen

For this tutorial I will use a tool called PuTTY Key Generator. You can download this from: https://www.puttygen.com/

Once it’s installed you open and click on Generate. Then you have to move the mouse to generate entropy so it can createa “real” random numbers to generate the key.

Once it’s created you will see something like this:


Then you have to export the key and make sure to save the public and private keys in a safe place, specially the private key is really important to keep secret and safe (like if it was a password).

Then we have to copy the public key and paste it on the Public key textfield.
Finally, we can just click Add at the bottom and our user will be created!

5. Test the connection to SFTP using WinSCP
Before we continue to the next steps, I just want to make sure the connection works and we can upload and download file from it.

For testing this we can use any SFTP client, in my case I am going to use WinSCP.

If you also want to use this client you can download it from the official websie: https://winscp.net/eng/download.php

Once the program is open we have to click on New Session. Then a new form will open, we then click on New Site.

Select File protocol the type SFTP
On Host name you can use your own Custom hostname in case you configured one using Amazon Route 53 or your own. If not, you can just use the Endpoint that is created automatically.
The endpoint format is: [server-id].server.transfer.[region-id].amazonaws.com
The Port number put 22
On User name you have to put the username we just created before
At this point you should have something similar to this (with your own data):


Then we have to click on Advanced… and go to SSH -> Authentication. Here we need to select the Private key file that we generated and saved before, this file extension is PPK.

If all goes well, you should get an alert message to Trust the key, we have to accept it and then we will be able to access the S3 Bucket from our computer using the SFTP Transfer service.

Now you can just play with some files and try to upload, remove and download them.

Here you can see how I uploaded a test.txt file on the folder sftp-rogave on the bucket called sftp.rogave.com. If you want to place the files on the root folder, you have to delete the Home directory folder on the User in the Server page in the AWS Transfer Family page.


At this point we have created:

S3 Bucket
IAM Role
IAM Policy
AWS SFTP Transfer Server
User in the AWS SFTP Transfer Server
RSA Key (Public and Private Key)
6. Create a Power Automate Flow and connect it to Amazon S3 via SFTP
Now we are ready to connect to the Amazon S3 bucket from any SFTP client, as we could see using WinSCP. The idea of this blog is to explain how we can automate the extraction of attachments on emails and send them automatically to Amazon S3 so we can process them using other services.

The last step is to create a Power Automate Flow that will get trigger everytime a new email with attachments arrives in Outlook. The flow will extract the attachment and copy it into our S3 Bucket automatically.


Microsoft Power Automate, formerly known as Microsoft Flow until November 2019, is an iPaaS platform by Microsoft for automation of recurring tasks. It is part of the Microsoft Power Platform line of products together with products such as Power Apps and Power BI.

Now a days, a lot of companies and users are using Microsoft products, specially with Outlook and SharePoint, and the integration of this products with Power Automate is really easy!

If you open for the first time Power Automate you will see different pre-builds flows as a template.


To create the flow we need to click into + Create menu in the left and then select Automated cloud flow and then we set a Flow name and choose the flow trigger to be “When a new email arrives (V2)” or “When a new email arrives (V3)”.


Once we have created the flow we see already the first step on it, also called the Trigger.


Here we can select the folder from where we want to get the emails with attachments. In my case I will use the main inbound called Inbox. An interesting thing you could do here is if you want to only process the files comming from the Company X you can create an Outloook rule and automatically move the emails comming from x@companyx.com into the folder CompanyX in your email account. Then you can just select this folder on the Trigger.

Then we press on + New step and we search for the option Apply to each and then we select Attachments as an output from the previous steps:


Now we click on Add an action inside the Apply to each step, and we search for Create file (SFTP – SSH).


Now we have to fill all the fields with the same data we used at the moment of connecting to the SFTP server using WinSCP. But in this case there is a change on the SSH private key format. We need to transform the SSH private key into OpenSSH Key format. To do that we will use again PuTTY Key Generator.

We first need to load the PPK file by clicking on File -> Load private key. Then we go to Conversions -> Export OpenSSH key and we save file.


Once we have transformed the key, we open it with a Text Editor and we copy all the content into the SSH private key field.

The private key format should be:

-----BEGIN RSA PRIVATE KEY-----
<private-key>
-----END RSA PRIVATE KEY-----
Make sure to copy ALL text of the file, including the begin and end line. Also, you need to check Disable SSH host key validation.

If you followed all the steps you should have something like this:


Finally, click on Create!

Once we have configured to connection, we have to set the Folder path, File name and File content.

The Folder path can be any folder in specific in where you want to store the files, for example /test. On File name you have to select Attachments Name and on File content you have to select Attachment Content.


Now we have to test if it really works, for that we can click the option in the top-right called Test.

Now send and email with an attachment to the folder you have configured.

Congratualtions if the flow worked! In case something doesn’t work, please check the previous steps to make sure you didn’t missed something.

Here is another example of flow that could be done using SharePoint instead of Outlook.


In this case, the Trigger of the flow is When a file is created or modified on SharePoint.

Hope you liked this post! 🙂

For any feedback please feel free to contact me, I will greatly appreciate any feedback!
