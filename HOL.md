<a name='handsonlab' />
# Deploying Active Directory in Windows Azure (PowerShell) #
---

<a name='Overview' />
## Overview ##

In this hands-on lab, you will walk through the steps  required to deploy an Active Directory domain in the cloud and provision new virtual machines into this domain. 

<a name='Objectives' />
### Objectives ###

In this hands-on lab, you will learn how to:

- Configure Virtual Networking
- Deploy a Domain Controller 
- Create new Virtual Machines in the Domain

<a name="Prerequisites"></a> 
### Prerequisites ###

The following is required to complete this hands-on lab:

- [Windows PowerShell 2.0][1]
- [Windows Azure PowerShell CmdLets][2]
- A Windows Azure subscription with the Virtual Machines Preview enabled - you can sign up for free trial [here](http://bit.ly/WindowsAzureFreeTrial)

[1]: http://microsoft.com/powershell/
[2]: http://msdn.microsoft.com/en-us/library/windowsazure/jj156055

<a name="Setup"></a> 
### Setup ###

The Windows Azure PowerShell Cmdlets are required for this lab. If you have not configured them yet, please see the **Automating VM Management** hands-on lab in the **Automating Windows Azure with PowerShell** module. 

>**Note:** In order to run through the complete hands-on lab, you must have network connectivity. 

<a name='Exercises' />
## Exercises ##

This hands-on lab includes the following exercises:

1. [Configuring Virtual Networking](#Exercise1)
1. [Deploying the first Domain Controller](#Exercise2)
1. [Provisioning new Virtual Machines into the Domain](#Exercise3)

Estimated time to complete this lab: **60 minutes**.

<a name='gettingstarted' />
### Getting Started: Obtaining Subscription's Credentials ###

In order to complete this lab, you will need your subscription’s secure credentials. Windows Azure lets you download a Publish Settings file with all the information required to manage your account in your development environment.

#### Task 1 - Downloading and Importing a Publish-settings File ####

> **Note:** If you have done these steps in a previous lab on the same computer you can move on to Exercise 1.


In this task, you will log on to the Windows Azure Portal and download the publish-settings file. This file contains the secure credentials and additional information about your Windows Azure Subscription to use in your development environment. Then, you will import this file using the Windows Azure Cmdlets in order to install the certificate and obtain the account information.

1.	Open an Internet Explorer browser and go to <https://windows.azure.com/download/publishprofile.aspx>.

1.	Sign in using the **Microsoft Account** associated with your **Windows Azure** account.

1.	**Save** the publish-settings file to your local machine.

	![Downloading publish-settings file](images/downloading-publish-settings-file.png?raw=true 'Downloading publish-settings file')

	_Downloading publish-settings file_

	> **Note:** The download page shows you how to import the publish-settings file using Visual Studio Publish box. This lab will show you how to import it using the Windows Azure PowerShell Cmdlets instead.

1.In the start menu under **Windows Azure**, right-click **Windows Azure PowerShell** and choose **Run as Administrator**.

1.	Change the PowerShell execution policy to **RemoteSigned**. When asked to confirm press **Y** and then **Enter**.
	
	<!-- mark:1 -->
	````PowerShell
	Set-ExecutionPolicy RemoteSigned
	````

	> **Note:** The Set-ExecutionPolicy cmdlet enables you to determine which Windows PowerShell scripts (if any) will be allowed to run on your computer. Windows PowerShell has four different execution policies:
	>
	> - _Restricted_ - No scripts can be run. Windows PowerShell can be used only in interactive mode.
	> - _AllSigned_ - Only scripts signed by a trusted publisher can be run.
	> - _RemoteSigned_ - Downloaded scripts must be signed by a trusted publisher before they can be run.
	> - _Unrestricted_ - No restrictions; all Windows PowerShell scripts can be run.
	>
	> For more information about Execution Policies refer to this TechNet article: <http://technet.microsoft.com/en-us/library/ee176961.aspx>

	
1.	The following script imports your publish-settings file and generates an XML file with your account information. You will use these values during the lab to manage your Windows Azure Subscription. Replace the placeholder with your publish-setting file's path and execute the script.

	<!-- mark:1 -->
	````PowerShell
	Import-AzurePublishSettingsFile '[YOUR-PUBLISH-SETTINGS-PATH]'   
	````

1. Execute the following commands and take note of the Subscription name and the storage account name you will use for the exercise.

	<!-- mark:1-3 -->
	````PowerShell
	Get-AzureSubscription | select SubscriptionName
	Get-AzureStorageAccount | select StorageAccountName 
	````

1. Execute the following command to set your current storage account for your subscription.

	````PowerShell
	Set-AzureSubscription -SubscriptionName '[YOUR-SUBSCRIPTION-NAME]' -CurrentStorageAccount '[YOUR-STORAGE-ACCOUNT]'
	````

<a name='Exercise1'></a>
### Exercise 1: Configuring Virtual Networking ###

Running an Active Directory Domain requires persistent IP addresses and for clients of the Active Directory Domain to point to an AD enabled DNS server. The default internal DNS service (iDNS) in Windows Azure is not an acceptable solution because the IP address assigned to each virtual machine is not persistent. For this solution you will define a virtual network where you can assign the virtual machines to specific subnets. Using this technique you can plan on what the IP address of a specific VM will be and know that it will be persistent. 

The network configuration used for this lab defines the following:

- A Virtual Network Named ADVNET with an address prefix of: 192.168.0.0/16
- A subnet named ADSubnet with an address prefix of: 192.168.1.0/24
- A subnet named AppSubnet with an address prefix of: 192.168.2.0/24


<a name='Ex1Task1' />
#### Task 1 - Creating an Affinity Group ####

1. Execute the following command to retrieve the Available Data Center Locations. 

	````PowerShell
	Get-AzureLocation | select name
	````

1. Define a variable ($dclocation) and set its value with the name of the data center you want to deploy to.

	````PowerShell
	$dclocation = '[YOUR-LOCATION]'
	````

1. The first step is to create an affinity group with the same name specified in ad-vnet.xml (adag). 

	````PowerShell
	$affinityGroup = 'adag'
	New-AzureAffinityGroup -Name $affinityGroup -Location $dclocation
	````

1. Next, apply the virtual network settings in the file **ad-vnet.xml** under **Source\Assets** folder, to your subscription.

	````PowerShell
	$ConfigPath = 'c:\WATK\Labs\DeployingActiveDirectoryPS\Source\Assets\ad-vnet.xml' 
	Set-AzureVNetConfig -ConfigurationPath $ConfigPath 
	````

1. Create a storage account in the same affinity group as the virtual network. The storage account you create must be unique.

	````PowerShell
	New-AzureStorageAccount -StorageAccountName 'someuniquename' -AffinityGroup $affinityGroup
	````

<a name='Exercise2'></a>
### Exercise 2: Deploying the first Domain Controller ###

We can choose whether to use the Windows Azure Portal or PowerShell to provision the virtual machine that will be our domain controller. In this exercise, you will use PowerShell since you will be using it in the next exercise to demonstrate domain join automation.



<a name='Ex2Task1' />
#### Task 1 - Creating the First VM and Deployment with Networking Settings ####

1. Run the following command to return the available images.

	````PowerShell
	Get-AzureVMImage | Select ImageName
	````

1. Choose one of the Windows Server 2008 R2 Images and specify it as the value to **$imgname** below.

	````PowerShell
	$imgname = 'ImageNameGoesHere'
	````

1. Next run the following commands to create the domain controller in the correct virtual network and subnet.

	````PowerShell

	$cloudsvc = 'some-unique-name'
	$vmname1 = 'ad-dc'
	$subnet = 'ADSubnet'
	$vnet = 'ADVNET'
	$pwd = '[YOUR-PASSWORD]'

	New-AzureVMConfig -Name $vmname1 -InstanceSize Small -ImageName $imgname |
		Add-AzureProvisioningConfig -Windows -Password $pwd |
		Set-AzureSubnet -SubnetNames 'ADSubnet' |
		Add-AzureDataDisk -CreateNew -DiskSizeInGB 20 -DiskLabel 'DITDrive' -LUN 0 |
		New-AzureVM –ServiceName $cloudsvc -AffinityGroup 'adag' -VNetName 'ADVNET'

	````

<a name='Ex2Task3' />
#### Task 3 - Creating the Domain Controller ####

1. Login to the newly created virtual machine in the Windows Azure Portal by clicking on Virtual Machines, the VM **ad-dc**, and click **Connect** at the bottom.

1. Once logged in, start a console session and run the command **IPConfig** and copy the IPv4 IP Address returned. You will use it later to provision new VMs in the domain.

	![VM-IP](images/vm-ip.png?raw=true "VM ip")

	_Virtual Machine IP_

1. Open **Computer Manager** by going to **Start** | **All Programs** | **Administrative Tools** | **Computer Manager**.

1. Expand the **Storage** node and select **Disk Management**.

	![Opening Disk Management](images/opening-disk-management.png?raw=true)

	_Opening Disk Management_

1. Right-click the **Disk 2** and select **Initialize Disk**. Click **Ok** to start the disk initialization.

	![Initializing Disk](images/initializing-disk.png?raw=true)

	_Initializing the disk_

1. Right-click over the unallocated disk space and select **New Simple Volume**.

	![Creating a new simple volume](images/creating-a-new-simple-volume.png?raw=true)

	_Creating a New Simple Volume_

1. Follow the **New Simple Volume Wizard** and set the **Volume label** to _DIT_. Click **Finish**. The disk will be formatted and ready to be used.

	![Formatted disk](images/formatted-disk.png?raw=true)

	_Formatted disk_

1. Now, you will start the **Active Directory Domain Services Installation Wizard**. To do this, click start and run and type in _DCPromo_ and press **enter**.

	![dcpromostart](images/dcpromostart.png?raw=true)	

	_Active Directory Domain Services Installation Wizard_

1. Click **Next** Two Times. 

1. Choose **Create a new domain in a new forest** and click **Next**.

	![newforest](images/newforest.png?raw=true)

	_Creating a new domain in a new forest_

1. Name the **Forest Root Domain** _contoso.com_ and click **Next**.

	![setting the domain name](images/setting-the-domain-name.png?raw=true)

	_Setting the domain name_

1. Set the functional level to **Windows Server 2008 R2**.

	![functionallevel](images/functionallevel.png?raw=true)

	_Selecting the forest functional level_

1. Choose the Default of Creating a DNS Server

	![DNSSelection](images/dnsselection.png?raw=true)

	_Selecting additional options_

	>**Note:** Choose **Yes**, the computer will use an IP address automatically assigned by a DHCP server (not recommended).
	>
	> ![dhcp-address](images/dhcp-address.png?raw=true)
	>
	>Using Virtual Networks with Windows Azure IaaS the IP address has a lifetime of the virtual machine lease. Do NOT set the IP address to static.

1. Since you are not integrating into an existing AD environment in this lab, click **Yes**.

	![dns-warning](images/dns-warning.png?raw=true)

	_DNS Creation Warning_

1. Set the **Database**, **Log files** and the **SYSVOL** folders location to the recently formatted data disk (for example _F:\\NTDS_ for Database, _F:\\NTDSLogs_ for Logs and _F:\\SYSVOL_ for SYSVOL). Click **Next** to continue.

	![dit-location](images/dit-location.png?raw=true)

	_Setting values for Database, Logs and SYSVOL folders_

1. Type in the same password used for provisioning the original machine and click **Next**.

	![ad-password](images/ad-password.png?raw=true)

	_Setting Domain Administrator password_

1. Finally, click **Next** again and allow for Active Directory to be configured. When prompted to allow reboot choose **Restart Now**.



<a name='Exercise3'></a>
### Exercise 3: Provisioning new Virtual Machines into the Domain ###

Once the domain controller has finished booting you will now be able to provision virtual machines and have them automatically join the domain when they are provisioned. This is accomplished by creating a new cloud service as the container for the new virtual machines. The DNS servers for the VMs can be automatically configured by specifying DNS settings during the initial deployment of the cloud service. 


<a name='Ex3Task1'></a>
#### Task 1 - Provisioning a Virtual Machine that is Domain Joined on Boot ###

The example below demonstrates how you can automatically provision new virtual machines that are joined to the Active Directory domain at boot.


1. The **Add-AzureProvisioningConfig** also takes a **-MachineObjectOU** parameter which if specified (requires the full distinguished name in AD) allows for setting group policy settings on all of the virtual machines in that container. Ensure that you replace **storageaccountname** with the storage account you created earlier.

	>**Note:** To get the IP Address of the Domain Controller, connect to the VM you have created. In a command prompt, enter **IPConfig** and copy the IPv4
	>
	> 	![VM-IP](images/vm-ip.png?raw=true "VM ip")
	> 

	````PowerShell
	# Point to IP Address of Domain Controller Created Earlier
	$dns1 = New-AzureDns -Name 'ad-dc' -IPAddress '[Domain-Controller IP Address]'
	
	
    # Configuring VM to Automatically Join Domain
	$advm1 = New-AzureVMConfig -Name 'advm1' -InstanceSize Small -ImageName $imgname | 
		Add-AzureProvisioningConfig -WindowsDomain -Password '[YOUR-PASSWORD]' `
	        -Domain 'contoso' -DomainPassword '[YOUR-PASSWORD]' `
			 -DomainUserName 'administrator' -JoinDomain 'contoso.com' |
		Set-AzureSubnet -SubnetNames 'AppSubnet' 
   
    # New Cloud Service with VNET and DNS settings
    New-AzureVM –ServiceName 'someuniqueappname' -AffinityGroup 'adag' `
							-VMs $advm1 -DnsSettings $dns1 -VNetName 'ADVNET' 
	
	````

    ![AD Architecture](images/ad-architecture.png?raw=true)
    
    _Resulting Architecture_

1. Once the VM is provisioned, open a remote desktop connection.

1. Open the Initial Configuration Tasks and verify the domain is _contoso.com_. 

	![Initial configuration tasks](images/initial-configuration-tasks.png?raw=true "Initial configuration tasks")

	_Initial configuration tasks_
	

---

<a name='Summary'/>
## Summary ##

In this lab, you walked through the steps of deploying a new Active Directory Domain using Windows Azure virtual machines and a simple virtual network. This lab also demonstrated how once in place virtual machines could be provisioned joined to the domain at boot time.