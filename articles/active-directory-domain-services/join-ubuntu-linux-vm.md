---
title: 'Azure Active Directory Domain Services: Join an Ubuntu VM to a managed domain | Microsoft Docs'
description: Join an Ubuntu Linux virtual machine to Azure AD Domain Services
services: active-directory-ds
documentationcenter: ''
author: iainfoulds
manager: daveba
editor: curtand

ms.assetid: 804438c4-51a1-497d-8ccc-5be775980203
ms.service: active-directory
ms.subservice: domain-services
ms.workload: identity
ms.tgt_pltfrm: na
ms.devlang: na
ms.topic: conceptual
ms.date: 05/20/2019
ms.author: iainfou

---
# Join an Ubuntu virtual machine in Azure to a managed domain
This article shows you how to join an Ubuntu Linux virtual machine to an Azure AD Domain Services managed domain.

[!INCLUDE [active-directory-ds-prerequisites.md](../../includes/active-directory-ds-prerequisites.md)]

## Before you begin
To perform the tasks listed in this article, you need:  
1. A valid **Azure subscription**.
2. An **Azure AD directory** - either synchronized with an on-premises directory or a cloud-only directory.
3. **Azure AD Domain Services** must be enabled for the Azure AD directory. If you haven't done so, follow all the tasks outlined in the [Getting Started guide](tutorial-create-instance.md).
4. Ensure that you have configured the IP addresses of the managed domain as the DNS servers for the virtual network. For more information, see [how to update DNS settings for the Azure virtual network](tutorial-create-instance.md#update-dns-settings-for-the-azure-virtual-network)
5. Complete the steps required to [synchronize passwords to your Azure AD Domain Services managed domain](tutorial-create-instance.md#enable-user-accounts-for-azure-ad-ds).


## Provision an Ubuntu Linux virtual machine
Provision an Ubuntu Linux virtual machine in Azure, using any of the following methods:
* [Azure portal](../virtual-machines/linux/quick-create-portal.md)
* [Azure CLI](../virtual-machines/linux/quick-create-cli.md)
* [Azure PowerShell](../virtual-machines/linux/quick-create-powershell.md)

> [!IMPORTANT]
> * Deploy the virtual machine into the **same virtual network in which you have enabled Azure AD Domain Services**.
> * Pick a **different subnet** than the one in which you have enabled Azure AD Domain Services.
>


## Connect remotely to the Ubuntu Linux virtual machine
The Ubuntu virtual machine has been provisioned in Azure. The next task is to connect remotely to the virtual machine using the local administrator account created while provisioning the VM.

Follow the instructions in the article [How to sign in to a virtual machine running Linux](../virtual-machines/linux/mac-create-ssh-keys.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json).


## Configure the hosts file on the Linux virtual machine
In your SSH terminal, edit the /etc/hosts file and update your machine’s IP address and hostname.

```console
sudo vi /etc/hosts
```

In the hosts file, enter the following value:

```console
127.0.0.1 contoso-ubuntu.contoso.com contoso-ubuntu
```

Here, 'contoso.com' is the DNS domain name of your managed domain. 'contoso-ubuntu' is the hostname of the Ubuntu virtual machine you are joining to the managed domain.


## Install required packages on the Linux virtual machine
Next, install packages required for domain join on the virtual machine. Perform the following steps:

1.  In your SSH terminal, type the following command to download the package lists from the repositories. This command updates the package lists to get information on the newest versions of packages and their dependencies.

    ```console
    sudo apt-get update
    ```

2. Type the following command to install the required packages.

    ```console
      sudo apt-get install krb5-user samba sssd sssd-tools libnss-sss libpam-sss ntp ntpdate realmd adcli
    ```

3. During the Kerberos installation, you see a pink screen. The installation of the 'krb5-user' package prompts for the realm name (in ALL UPPERCASE). The installation writes the [realm] and [domain_realm] sections in /etc/krb5.conf.

    > [!TIP]
    > If the name of your managed domain is contoso.com, enter CONTOSO.COM as the realm. Remember, the realm name must be specified in UPPERCASE.


## Configure the NTP (Network Time Protocol) settings on the Linux virtual machine
The date and time of your Ubuntu VM must synchronize with the managed domain. Add your managed domain's NTP hostname in the /etc/ntp.conf file.

```console
sudo vi /etc/ntp.conf
```

In the ntp.conf file, enter the following value and save the file:

```console
server contoso.com
```

Here, 'contoso.com' is the DNS domain name of your managed domain.

Now sync the Ubuntu VM's date and time with NTP server and then start the NTP service:

```console
sudo systemctl stop ntp
sudo ntpdate contoso.com
sudo systemctl start ntp
```


## Join the Linux virtual machine to the managed domain
Now that the required packages are installed on the Linux virtual machine, the next task is to join the virtual machine to the managed domain.

1. Discover the AAD Domain Services managed domain. In your SSH terminal, type the following command:

    ```console
    sudo realm discover CONTOSO.COM
    ```

   > [!NOTE]
   > **Troubleshooting:**
   > If *realm discover* is unable to find your managed domain:
   >   * Ensure that the domain is reachable from the virtual machine (try ping).
   >   * Check that the virtual machine has indeed been deployed to the same virtual network in which the managed domain is available.
   >   * Check to see if you have updated the DNS server settings for the virtual network to point to the domain controllers of the managed domain.

2. Initialize Kerberos. In your SSH terminal, type the following command:

    > [!TIP]
    > * Ensure that you specify a user who belongs to the 'AAD DC Administrators' group. If needed, [add a user account to a group in Azure AD](../active-directory/fundamentals/active-directory-groups-members-azure-portal.md)
    > * Specify the domain name in capital letters, else kinit fails.
    >

    ```console
    kinit bob@CONTOSO.COM
    ```

3. Join the machine to the domain. In your SSH terminal, type the following command:

    > [!TIP]
    > Use the same user account you specified in the preceding step ('kinit').
    >
    > If your VM is unable to join the domain, make sure that the VM's network security group allows outbound Kerberos traffic on TCP + UDP port 464 to the virtual network subnet for your Azure AD DS managed domain.

    ```console
    sudo realm join --verbose CONTOSO.COM -U 'bob@CONTOSO.COM' --install=/
    ```

You should get a message ("Successfully enrolled machine in realm") when the machine is successfully joined to the managed domain.


## Update the SSSD configuration and restart the service
1. In your SSH terminal, type the following command. Open the sssd.conf file and make the following change
    
    ```console
    sudo vi /etc/sssd/sssd.conf
    ```

2. Comment out the line **use_fully_qualified_names = True** and save the file.
    
    ```console
    # use_fully_qualified_names = True
    ```

3. Restart the SSSD service.
    
    ```console
    sudo service sssd restart
    ```


## Configure automatic home directory creation
To enable automatic creation of the home directory after signing in users, type the following commands in your PuTTY terminal:

```console
sudo vi /etc/pam.d/common-session
```

Add the following line in this file below the line 'session optional pam_sss.so' and save it:

```console
session required pam_mkhomedir.so skel=/etc/skel/ umask=0077
```


## Verify domain join
Verify whether the machine has been successfully joined to the managed domain. Connect to the domain joined Ubuntu VM using a different SSH connection. Use a domain user account and then check to see if the user account is resolved correctly.

1. In your SSH terminal, type the following command to connect to the domain joined Ubuntu virtual machine using SSH. Use a domain account that belongs to the managed domain (for example, 'bob@CONTOSO.COM' in this case.)
    
    ```console
    ssh -l bob@CONTOSO.COM contoso-ubuntu.contoso.com
    ```

2. In your SSH terminal, type the following command to see if the home directory was initialized correctly.
    
    ```console
    pwd
    ```

3. In your SSH terminal, type the following command to see if the group memberships are being resolved correctly.
    
    ```console
    id
    ```


## Grant the 'AAD DC Administrators' group sudo privileges
You can grant members of the 'AAD DC Administrators' group administrative privileges on the Ubuntu VM. The sudo file is located at /etc/sudoers. The members of AD groups added in sudoers can perform sudo.

1. In your SSH terminal, ensure you are signed in with superuser privileges. You can use the local administrator account you specified while creating the VM. Execute the following command:
    
    ```console
    sudo vi /etc/sudoers
    ```

2. Add the following entry to the /etc/sudoers file and save it:
    
    ```console
    # Add 'AAD DC Administrators' group members as admins.
    %AAD\ DC\ Administrators ALL=(ALL) NOPASSWD:ALL
    ```

3. You can now sign in as a member of the 'AAD DC Administrators' group and should have administrative privileges on the VM.


## Troubleshooting domain join
Refer to the [Troubleshooting domain join](join-windows-vm.md#troubleshoot-domain-join-issues) article.


## Related Content
* [Azure AD Domain Services - Getting Started guide](tutorial-create-instance.md)
* [Join a Windows Server virtual machine to an Azure AD Domain Services managed domain](active-directory-ds-admin-guide-join-windows-vm.md)
* [How to sign in to a virtual machine running Linux](../virtual-machines/linux/mac-create-ssh-keys.md?toc=%2fazure%2fvirtual-machines%2flinux%2ftoc.json).
