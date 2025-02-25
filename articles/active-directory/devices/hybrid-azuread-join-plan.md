---
title: Plan hybrid Azure Active Directory join - Azure Active Directory
description: Learn how to configure hybrid Azure Active Directory joined devices.

services: active-directory
ms.service: active-directory
ms.subservice: devices
ms.topic: conceptual
ms.date: 06/10/2021

ms.author: joflore
author: MicrosoftGuyJFlo
manager: daveba
ms.reviewer: sandeo

ms.collection: M365-identity-device-management
---
# How To: Plan your hybrid Azure Active Directory join implementation

In a similar way to a user, a device is another core identity you want to protect and use it to protect your resources at any time and from any location. You can accomplish this goal by bringing and managing device identities in Azure AD using one of the following methods:

- Azure AD join
- Hybrid Azure AD join
- Azure AD registration

By bringing your devices to Azure AD, you maximize your users' productivity through single sign-on (SSO) across your cloud and on-premises resources. At the same time, you can secure access to your cloud and on-premises resources with [Conditional Access](../conditional-access/overview.md).

If you have an on-premises Active Directory (AD) environment and you want to join your AD domain-joined computers to Azure AD, you can accomplish this by doing hybrid Azure AD join. This article provides you with the related steps to implement a hybrid Azure AD join in your environment. 

## Prerequisites

This article assumes that you are familiar with the [Introduction to device identity management in Azure Active Directory](./overview.md).

> [!NOTE]
> The minimum required domain controller version for Windows 10 hybrid Azure AD join is Windows Server 2008 R2.

Hybrid Azure AD joined devices require network line of sight to your domain controllers periodically. Without this connection, devices become unusable.

Scenarios that break without line of sight to your domain controllers:

- Device password change
- User password change (Cached credentials)
- TPM reset

## Plan your implementation

To plan your hybrid Azure AD implementation, you should familiarize yourself with:

> [!div class="checklist"]
> - Review supported devices
> - Review things you should know
> - Review controlled validation of hybrid Azure AD join
> - Select your scenario based on your identity infrastructure
> - Review on-premises AD UPN support for hybrid Azure AD join

## Review supported devices

Hybrid Azure AD join supports a broad range of Windows devices. Because the configuration for devices running older versions of Windows requires additional or different steps, the supported devices are grouped into two categories:

### Windows current devices

- Windows 10
- Windows Server 2016
  - **Note**: Azure National cloud customers require version 1803
- Windows Server 2019

For devices running the Windows desktop operating system, supported version are listed in this article [Windows 10 release information](/windows/release-information/). As a best practice, Microsoft recommends you upgrade to the latest version of Windows 10.

### Windows down-level devices

- Windows 8.1
- Windows 7 support ended on January 14, 2020. For more information, see [Support for Windows 7 has ended](https://support.microsoft.com/en-us/help/4057281/windows-7-support-ended-on-january-14-2020).
- Windows Server 2012 R2
- Windows Server 2012
- Windows Server 2008 R2. For support information on Windows Server 2008 and 2008 R2, see [Prepare for Windows Server 2008 end of support](https://www.microsoft.com/cloud-platform/windows-server-2008).

As a first planning step, you should review your environment and determine whether you need to support Windows down-level devices.

## Review things you should know

### Unsupported scenarios

- Hybrid Azure AD join is not supported for Windows Server running the Domain Controller (DC) role.

- Hybrid Azure AD join is not supported on Windows down-level devices when using credential roaming or user profile roaming or mandatory profile.

- Server Core OS doesn't support any type of device registration.

- User State Migration Tool (USMT) doesn't work with device registration.  

### OS imaging considerations

- If you are relying on the System Preparation Tool (Sysprep) and if you are using a **pre-Windows 10 1809** image for installation, make sure that image is not from a device that is already registered with Azure AD as Hybrid Azure AD join.

- If you are relying on a Virtual Machine (VM) snapshot to create additional VMs, make sure that snapshot is not from a VM that is already registered with Azure AD as Hybrid Azure AD join.

- If you are using [Unified Write Filter](/windows-hardware/customize/enterprise/unified-write-filter) and similar technologies that clear changes to the disk at reboot, they must be applied after the device is Hybrid Azure AD joined. Enabling such technologies prior to completion of Hybrid Azure AD join will result in the device getting unjoined on every reboot

### Handling devices with Azure AD registered state

If your Windows 10 domain joined devices are [Azure AD registered](concept-azure-ad-register.md) to your tenant, it could lead to a dual state of Hybrid Azure AD joined and Azure AD registered device. We recommend upgrading to Windows 10 1803 (with KB4489894 applied) or above to automatically address this scenario. In pre-1803 releases, you will need to remove the Azure AD registered state manually before enabling Hybrid Azure AD join. In 1803 and above releases, the following changes have been made to avoid this dual state:

- Any existing Azure AD registered state for a user would be automatically removed <i>after the device is Hybrid Azure AD joined and the same user logs in</i>. For example, if User A had an Azure AD registered state on the device, the dual state for User A is cleaned up only when User A logs in to the device. If there are multiple users on the same device, the dual state is cleaned up individually when those users log in. In addition to removing the Azure AD registered state, Windows 10 will also unenroll the device from Intune or other MDM, if the enrollment happened as part of the Azure AD registration via auto-enrollment.
- Azure AD registered state on any local accounts on the device is not impacted by this change. It is only applicable to domain accounts. So Azure AD registered state on local accounts is not removed automatically even after user logon, since the user is not a domain user. 
- You can prevent your domain joined device from being Azure AD registered by adding the following registry value to HKLM\SOFTWARE\Policies\Microsoft\Windows\WorkplaceJoin: "BlockAADWorkplaceJoin"=dword:00000001.
- In Windows 10 1803, if you have Windows Hello for Business configured, the user needs to re-setup Windows Hello for Business after the dual state clean up.This issue has been addressed with KB4512509

> [!NOTE]
> Even though Windows 10 automatically removes the Azure AD registered state locally, the device object in Azure AD is not immediately deleted if it is managed by Intune. You can validate the removal of Azure AD registered state by running dsregcmd /status and consider the device not to be Azure AD registered based on that.

### Hybrid Azure AD join for single forest, multiple Azure AD tenants

To register devices as hybrid Azure AD join to respective tenants, organizations need to ensure that the SCP configuration is done on the devices and not in AD. More details on how to accomplish this can be found in the article [controlled validation of hybrid Azure AD join](hybrid-azuread-join-control.md). It is also important for organizations to understand that certain Azure AD capabilities will not work in a single forest, multiple Azure AD tenants configurations.
- [Device writeback](../hybrid/how-to-connect-device-writeback.md) will not work. This affects [Device based Conditional Access for on-premise apps that are federated using ADFS](/windows-server/identity/ad-fs/operations/configure-device-based-conditional-access-on-premises). This also affects [Windows Hello for Business deployment when using the Hybrid Cert Trust model](/windows/security/identity-protection/hello-for-business/hello-hybrid-cert-trust).
- [Groups writeback](../hybrid/how-to-connect-group-writeback.md) will not work. This affects writeback of Office 365 Groups to a forest with Exchange installed.
- [Seamless SSO](../hybrid/how-to-connect-sso.md) will not work. This affects SSO scenarios that organizations may be using on cross OS/browser platforms, for example iOS/Linux with Firefox, Safari, Chrome without the Windows 10 extension.
- [Hybrid Azure AD join for Windows down-level devices in managed environment](./hybrid-azuread-join-managed-domains.md#enable-windows-down-level-devices) will not work. For example, hybrid Azure AD join on Windows Server 2012 R2 in a managed environment requires Seamless SSO and since Seamless SSO will not work, hybrid Azure AD join for such a setup will not work.
- [On-premises Azure AD Password Protection](../authentication/concept-password-ban-bad-on-premises.md) will not work.This affects ability to perform password changes and password reset events against on-premises Active Directory Domain Services (AD DS) domain controllers using the same global and custom banned password lists that are stored in Azure AD.


### Additional considerations

- If your environment uses virtual desktop infrastructure (VDI), see [Device identity and desktop virtualization](./howto-device-identity-virtual-desktop-infrastructure.md).

- Hybrid Azure AD join is supported for FIPS-compliant TPM 2.0 and not supported for TPM 1.2. If your devices have FIPS-compliant TPM 1.2, you must disable them before proceeding with Hybrid Azure AD join. Microsoft does not provide any tools for disabling FIPS mode for TPMs as it is dependent on the TPM manufacturer. Please contact your hardware OEM for support. 

- Starting from Windows 10 1903 release, TPMs 1.2 are not used with hybrid Azure AD join and devices with those TPMs will be considered as if they don't have a TPM.

- UPN changes are only supported starting Windows 10 2004 update. For devices prior to Windows 10 2004 update, users would have SSO and Conditional Access issues on their devices. To resolve this issue, you need to unjoin the device from Azure AD (run "dsregcmd /leave" with elevated privileges) and rejoin (happens automatically). However, users signing in with Windows Hello for Business do not face this issue.

## Review controlled validation of hybrid Azure AD join

When all of the pre-requisites are in place, Windows devices will automatically register as devices in your Azure AD tenant. The state of these device identities in Azure AD is referred as hybrid Azure AD join. More information about the concepts covered in this article can be found in the article [Introduction to device identity management in Azure Active Directory](overview.md).

Organizations may want to do a controlled validation of hybrid Azure AD join before enabling it across their entire organization all at once. Review the article [controlled validation of hybrid Azure AD join](hybrid-azuread-join-control.md) to understand how to accomplish it.

## Select your scenario based on your identity infrastructure

Hybrid Azure AD join works with both, managed and federated environments depending on whether the UPN is routable or non-routable. See bottom of the page for table on supported scenarios.  

### Managed environment

A managed environment can be deployed either through [Password Hash Sync (PHS)](../hybrid/whatis-phs.md) or [Pass Through Authentication (PTA)](../hybrid/how-to-connect-pta.md) with [Seamless Single Sign On](../hybrid/how-to-connect-sso.md).

These scenarios don't require you to configure a federation server for authentication.

> [!NOTE]
> [Cloud authentication using Staged rollout](../hybrid/how-to-connect-staged-rollout.md) is only supported starting Windows 10 1903 update

### Federated environment

A federated environment should have an identity provider that supports the following requirements. If you have a federated environment using Active Directory Federation Services (AD FS), then the below requirements are already supported.

- **WIAORMULTIAUTHN claim:** This claim is required to do hybrid Azure AD join for Windows down-level devices.
- **WS-Trust protocol:** This protocol is required to authenticate Windows current hybrid Azure AD joined devices with Azure AD. 
When you're using AD FS, you need to enable the following WS-Trust endpoints: 
  `/adfs/services/trust/2005/windowstransport`  
  `/adfs/services/trust/13/windowstransport`  
  `/adfs/services/trust/2005/usernamemixed` 
  `/adfs/services/trust/13/usernamemixed`
  `/adfs/services/trust/2005/certificatemixed` 
  `/adfs/services/trust/13/certificatemixed` 

> [!WARNING] 
> Both **adfs/services/trust/2005/windowstransport** or **adfs/services/trust/13/windowstransport** should be enabled as intranet facing endpoints only and must NOT be exposed as extranet facing endpoints through the Web Application Proxy. To learn more on how to disable WS-Trust Windows endpoints, see [Disable WS-Trust Windows endpoints on the proxy](/windows-server/identity/ad-fs/deployment/best-practices-securing-ad-fs#disable-ws-trust-windows-endpoints-on-the-proxy-ie-from-extranet). You can see what endpoints are enabled through the AD FS management console under **Service** > **Endpoints**.

> [!NOTE]
> Azure AD does not support smartcards or certificates in managed domains.

Beginning with version 1.1.819.0, Azure AD Connect provides you with a wizard to configure hybrid Azure AD join. The wizard enables you to significantly simplify the configuration process. If installing the required version of Azure AD Connect is not an option for you, see [how to manually configure device registration](hybrid-azuread-join-manual.md). 

Based on the scenario that matches your identity infrastructure, see:

- [Configure hybrid Azure Active Directory join for federated environment](hybrid-azuread-join-federated-domains.md)
- [Configure hybrid Azure Active Directory join for managed environment](hybrid-azuread-join-managed-domains.md)

## Review on-premises AD users UPN support for Hybrid Azure AD join

Sometimes, your on-premises AD users UPNs could be different from your Azure AD UPNs. In such cases, Windows 10 Hybrid Azure AD join provides limited support for on-premises AD UPNs based on the [authentication method](../hybrid/choose-ad-authn.md), domain type and Windows 10 version. There are two types of on-premises AD UPNs that can exist in your environment:

- Routable users UPN: A routable UPN has a valid verified domain, that is registered with a domain registrar. For example, if contoso.com is the primary domain in Azure AD, contoso.org is the primary domain in on-premises AD owned by Contoso and [verified in Azure AD](../fundamentals/add-custom-domain.md)
- Non-routable users UPN: A non-routable UPN does not have a verified domain. It is applicable only within your organization's private network. For example, if contoso.com is the primary domain in Azure AD, contoso.local is the primary domain in on-premises AD but is not a verifiable domain in the internet and only used within Contoso's network.

> [!NOTE]
> The information in this section applies only to an on-premises users UPN. It isn't applicable to an on-premises computer domain suffix (example: computer1.contoso.local).

The table below provides details on support for these on-premises AD UPNs in Windows 10 Hybrid Azure AD join

| Type of on-premises AD UPN | Domain type | Windows 10 version | Description |
| ----- | ----- | ----- | ----- |
| Routable | Federated | From 1703 release | Generally available |
| Non-routable | Federated | From 1803 release | Generally available |
| Routable | Managed | From 1803 release | Generally available, Azure AD SSPR on Windows lock screen is not supported. The on-premises UPN must be synced to the     `onPremisesUserPrincipalName` attribute in Azure AD |
| Non-routable | Managed | Not supported | |

## Next steps

> [!div class="nextstepaction"]
> [Configure hybrid Azure Active Directory join for federated environment](hybrid-azuread-join-federated-domains.md)
> [Configure hybrid Azure Active Directory join for managed environment](hybrid-azuread-join-managed-domains.md)

<!--Image references-->
[1]: ./media/hybrid-azuread-join-plan/12.png
