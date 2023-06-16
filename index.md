## In this post, I'm going to show you how to enable server-side LDAPS for your AWS Managed Microsoft AD directory using your existing onpremise Microsoft PKI infrastructure (cross-forest certificate enrolment).

### Use your existing on-premises Microsoft PKI infrastructure to enable LDAPs on AWS Managed Microsoft AD:
#### Prerequisites
1.	An existing self-managed Active Directory environment with [two-tier Microsoft PKI infrastructure](https://docs.microsoft.com/en-us/previous-versions/windows/it-pro/windows-server-2012-r2-and-2012/hh831348(v%3Dws.11)).
2.	An existing two-way [forest trust](https://docs.aws.amazon.com/directoryservice/latest/admin-guide/ms_ad_tutorial_setup_trust.html) between AWS Managed Microsoft AD and your self-managed AD.

#### Step 1: Configure your on-premises firewall
You must configure your on-premises firewall so that the following ports are open to the CIDRs for all subnets used by the VPC that contains your AWS Managed Microsoft AD and also from the private IP of the RSAT instance that you are going to use to manage the AWS Managed Microsoft AD.

*	TCP 135 (RPC)
* TCP 1024 – 65535 (RPC randomly allocated high TCP ports)
* TCP/UDP 53 (DNS)
* TCP/UDP 88 (Kerberos authentication)
* TCP/UDP 389 (LDAP)
* TCP 445 (SMB)

#### Step 2: Prepare your on-premises CA
In this step, you prepare your on-premises Enterprise Subordinate CA for cross-forest certificate enrollment by enabling LDAP referral.
1.	Log in to your on-premises Enterprise CA.
2.	Open a command prompt as an administrator and run the following command:
3.	certutil -setreg Policy\EditFlags +EDITF_ENABLELDAPREFERRALS
4.	To restart the certificate service, run the following command:
5.	net stop certsvc && net start certsvc

#### Step 3: Create and publish the certificate template
In Step 5, you created and published the certificate template in the on-premises Enterprise Subordinate CA. To add enroll and auto-enroll permissions on the certificate template so that AWS Managed Microsoft AD Domain controllers can auto-enroll the certificate, complete the following steps.
1.	Connect to the Subordinate CA using RDP with an on-premises CA admin.
2.	Open the Run dialog box, enter certtmpl.msc, and select OK.
3.	In the Certificate Templates Console window, right-click the LDAPoverSSL certificate template that was created in Step 5 and choose Properties.
4.	Choose the Security tab and then choose Add.
 
 <img width="270" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/72ecdbd4-340c-4f09-ac9f-8c0a6bba639a">

5.	Change the location to the AWS Managed Microsoft AD domain. For Enter the object names to select, enter Domain Controllers, choose Check names, and then choose OK.
 
 <img width="309" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/77ed323a-07e4-4984-9378-b9cab5707d1f">

6.	On the Properties of New Template screen, choose the Security tab, and under Group or user names, select Domain controllers (amazondomains\Domain Controllers). In the Permissions for Domain Controllers section, check the boxes in the Allow column for Read, Enroll, and Autoenroll, choose Apply, and then choose OK.
 <img width="272" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/fde12294-adb7-4e94-9b6a-5c75f2d845c9">
 
#### Step 4: Configure AWS Managed Microsoft AD
In Step 6d, you perform multiple operations to configure the amazondomains.com domain for cross-forest certificate enrollment.
##### Step 4.1: Add an on-premises Enterprise CA computer object in the Cert Publishers group of the amazondomains.com domain
1.	Take RDP to the RSAT instance with the AWS Managed Microsoft AD Admin user.
2.	Open the Run dialog box, enter dsa.msc, and choose OK.
3.	Navigate to amazondomains.com > Users, right click the group named Cert Publishers, and select Properties.
 <img width="507" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/72c9be02-c489-497a-97d0-cd15b35f26ad">
 
4.	In the Cert Publishers Properties box, choose the Members tab and then choose Add.
 <img width="375" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/f0809349-fa12-4e0a-bfd3-fbc655ff7fab">

5.	In the Select Users, Computers, Service Accounts, or Groups box, provide the following details:
*	For Select this object type, choose Object Types, and then choose Computers.
*	For From this location, choose Locations and then choose onprem.example.com.
*	For Enter the object names to select, enter the name of the computer where the Subordinate CA is installed, select Check Names, and then select OK.
 <img width="310" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/067f5700-fcc0-4413-9bd1-89ceeef8c25a">
 
6.	Choose Apply and then choose OK.

##### Step 4.2: Share the Root CA certificate with the RSAT instance
1.	Take RDP to the on-premises server where the Root CA is configured as a local administrator of the server.
2.	Upload the Root CA certificate (.crt files) from the c:\windows\system32\certsrv\certnroll folder to the S3 bucket created in Step 2 by following the steps in Uploading objects.
##### Step 4.3: Publish the Root CA certificate in AWS Managed Microsoft AD
1.	Take RDP to the RSAT instance with the Admin user.
2.	Download the Root CA certificate from the S3 bucket by following the documentation in Downloading an object, and then save the certificates in a new folder named c:\certconfig.
3.	To publish the certificate in the amazondomains.com domain, open the command prompt as an administrator and run the following commands. Replace <placeholder values> with your values.
4.	certutil -dspublish -f <root-ca-cert-filename.cer> RootCA
5.	certutil -addstore -f root <root-ca-cert-filename.crt>
For the example in this post, I ran the following commands:
certutil -dspublish -f c:\certconfig\RootCA_RootCA.crt RootCA
certutil -addstore -f root c:\certconfig\RootCA_RootCA.crt
##### Step 4.4: Publish the Subordinate CA certificate in AWS Managed Microsoft AD
1.	Take RDP to the RSAT instance with the Admin user.
2.	To publish the Subordinate CA certificate in the amazondomains.com domain, open the command prompt as an administrator and run the following commands. Replace <placeholder values> with your values.
3.	certutil -config <Computer-Name>\<Enterprise-CA-Name> -ca.cert <enterprise-ca-cert-filename.cer>
4.	certutil -addstore -f CA <enterprise-ca-cert-filename.cer>
5.	certutil -dspublish -f <enterprise-ca-cert-filename.cer> NTAuthCA
6.	certutil -dspublish -f <enterprise-ca-cert-filename.cer> SubCA
For the example in this post, I ran the following commands:
certutil -config SubordinateCA.onprem.example.com\onPremSubordinateCA -ca.cert c:\certconfig\subcacert.cer
certutil -addstore -f CA c:\certconfig\subcacert.cer 
certutil -dspublish -f c:\certconfig\subcacert.cer NTAuthCA
certutil -dspublish -f c:\certconfig\subcacert.cer SubCA
##### Step 4.5: Download and run the PKIsync script from Microsoft to sync CA objects from the on-premises AD to AWS Managed Microsoft AD.
1.	Download the PKISync.ps1 script from Microsoft.
2.	At the top of the code section, select Copy code.
3.	In Notepad or a similar text editor, paste the code, and save the file with the name PKISync.ps1.
4.	Upload the PKISync.ps1 file to the S3 bucket.
5.	Take RDP to the RSAT instance of AWS Managed Microsoft AD and download the PKISync.ps1 file from the S3 bucket to the folder c:\certconfig.
6.	Open Powershell as an administrator and run the following command.
7.	Set-Location C:\Certconfig
##### Step 4.6: To copy the certificate template from the on-premises domain to AWS Managed Microsoft AD, run the following command. Replace <placeholder values> with your values.
.\PKISync.ps1 -sourceforest <onPrem domain DNS> -targetforest <AWS managed AD domain DNS> -type Template -cn <certificate template common name> -f
For the example in this post, I ran the following command:
.\PKISync.ps1 -sourceforest onprem.example.com -targetforest amazondomains.com -type Template -cn LDAPoverSSL -f
Step 6d-7: To copy the OID from the on-premises domain to AWS Managed Microsoft AD, run the following command. Replace <placeholder values> with your values.
.\PKISync.ps1 -sourceforest <onPrem domain DNS> -targetforest <AWS managed AD domain DNS> -type Oid -f
For the example in this post, I ran the following command:
.\PKISync.ps1 -sourceforest onprem.example.com -targetforest amazondomains.com -type Oid -f
Step 6d-8: To copy the Enterprise CA object from the on-premises domain to AWS Managed Microsoft AD, run the following command. Replace <placeholder values> with your values.
.\PKISync.ps1 -sourceforest <onPrem domain DNS> -targetforest <AWS managed AD domain DNS> -type CA -cn <enterprise CA sanitized>
For the example in this post, I ran the following command:
.\PKISync.ps1 -sourceforest onprem.example.com -targetforest amazondomains.com -type CA -cn onPremSubordinateCA -f

#### Step 7: Configure AWS security group rules
In this step, you configure AWS security group rules so that your directory domain controllers can connect to the Subordinate CA to request a certificate. To do this, you must add outbound rules to your directory’s AWS security group (in this case, sg-014fee4c22b6ff511) to allow all outbound traffic to SubordinateCA’s AWS security group (in this case, sg-06693e3b64a7e32f4 ) so that your directory domain controllers can connect to SubordinateCA for requesting a certificate. You also must add inbound rules to SubordinateCA’s AWS security group to allow all incoming traffic from your directory’s AWS security group so that the Subordinate CA can accept incoming traffic from your directory domain controllers.
Follow these steps to configure AWS security group rules:
1.	Log in to the Management instance as Admin.
2.	Navigate to the EC2 console.
3.	In the left pane, choose Network & Security > Security Groups.
4.	In the right pane, choose the AWS security group (in this case, sg-6fbe7109) of SubordinateCA.
5.	Switch to the Inbound tab and choose Edit.
6.	Choose Add Rule. Choose All traffic for Type and Custom for Source. Enter your directory’s AWS security group (in this case, sg-4ba7682d) in the Source box. Choose Save.
  <img width="1457" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/16e77bbc-d015-4ab8-ac9d-c17d2dd427b6">

7.	Now choose the AWS security group (in this case, sg-4ba7682d) of your AWS Managed Microsoft AD directory, switch to the Outbound tab, and choose Edit.
8.	Choose Add Rule. Choose All traffic for Type and Custom for Destination. Enter your directory’s AWS security group (in this case, sg-6fbe7109) in the Destination box. Choose Save.
 <img width="1451" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/7a55e57c-6b52-486d-9c6e-1460f9fe8e8c">

You have completed the configuration of AWS security group rules to allow traffic between your directory domain controllers and SubordinateCA.
The AWS Managed Microsoft AD domain controllers will automatically request a certificate based on the template created on the Microsoft Enterprise Subordinate CA in Step 5: Create a certificate template. It can take up to 30 minutes for the directory domain controllers to auto-enroll the available certificates. Once the certificates are issued to the directory domain controllers LDAPS will become functional. This completes the setup of LDAPS for the AWS Managed Microsoft AD directory. The LDAP service on the directory is now ready to accept LDAPS connections!

#### Step 5: Test LDAPS access by using the LDP tool
In this step, you test the LDAPS connection to the AWS Managed Microsoft AD directory by using the LDP tool. The LDP tool is available on the Management machine where you installed Active Directory Administration Tools. Before you test the LDAPS connection, you must wait up to 30 minutes for the Microsoft Enterprise Subordinate CA to issue a certificate to your domain controllers.
To test LDAPS, you connect to one of the domain controllers using port 636. Here are the steps to test the LDAPS connection:
1.	Log in to Management as Admin.
2.	Launch the Microsoft Windows Server Manager on Management and navigate to Tools > Active Directory Users and Computers.
3.	Switch to the tree view and navigate to corp.example.com > CORP > Domain Controllers. In the right pane, right-click on one of the domain controllers and choose Properties. Copy the DNS name of the domain controller.
 <img width="764" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/cd84d406-a762-4375-85e4-c6a5aacc5271">

4.	Launch the LDP.exe tool by launching Windows PowerShell and running the LDP.exe command.
 <img width="647" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/2af9c2b5-35a2-4fbb-904c-2bf4ca573845">

5.	In the LDP tool, choose Connection > Connect.
 <img width="580" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/613e18a9-bb76-4962-b694-fb9580c0b0d2">

6.	In the Server box, paste the DNS name you copied in Step 2. Type 636 in the Port box. Choose OK to test the LDAPS connection to port 636 of your directory.
 <img width="582" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/82d8f6c6-f0bd-40e3-bb07-7cfefa9f6d69">

7.	You should see the following message to confirm that your LDAPS connection is now open.
 <img width="676" alt="image" src="https://github.com/ckatyal17/ldaps_with_cross_forest_enrollment/assets/68083582/2fcf0d47-7345-4d11-a2fe-b0de37a0201c">

You have completed the setup of LDAPS for your AWS Managed Microsoft AD directory! You can now encrypt LDAP communications between your Windows and Linux applications and your AWS Managed Microsoft AD directory using LDAPS.
