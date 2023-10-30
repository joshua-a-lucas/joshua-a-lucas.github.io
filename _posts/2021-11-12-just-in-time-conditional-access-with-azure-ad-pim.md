---
title: "Just-in-time conditional access with Azure AD Privileged Identity Management"
date: "2021-11-12"
categories: 
  - "Azure"
  - "Azure AD"
tags: 
  - "azure-ad"
  - "exchange-online"
  - "sharepoint-online"
---

## Introduction

Microsoft curate a list of [common conditional access policies](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/concept-conditional-access-policy-common) that align with their best-practice recommendations for securing Azure Active Directory, including requiring multi-factor authentication for all users and blocking legacy authentication protocols, just to name a few. These policies are great, but in practise they can be difficult to implement. One of the challenges that many organisations face is the ability to handle exclusions in a manner that is compliant with their regulatory requirements, such as the need to audit all exemptions, ensure they go through a proper approval process and enforce compensating controls.

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/conditional-access.png)

Consider a business that has configured a conditional access policy to [require users login from compliant work-managed devices](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-policy-compliant-device) in order to access company resources. In the event their laptop breaks, employees need to be able to request temporary exemptions to this policy so that they can continue to work from a personal device while a replacement laptop is prepared. Exemptions need to be auditable, time-based and explicitly approved by management, and IT needs to enforce technical restrictions that prevent the user from downloading company documents to their personal device in the interim. Unfortunately, conditional access policy exclusions currently lack any significant auditing functionality, cannot facilitate expirations and are difficult to manage at scale. How can we meet these business requirements while minimising the administrative overhead on our IT team?

Our answer lies with [Azure AD Privileged Identity Management](https://docs.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-configure). To explore this scenario, we'll develop a process in which users can temporarily elevate to membership of a privileged access group that is excluded from the compliant device requirement, and instead imposes app enforced controls in SharePoint, OneDrive and Exchange Online to restrict their ability to download company documents for the duration of the exemption.

## Creating the privileged access group

Privileged Identity Management (PIM) is a service that enables you to manage, control, and monitor access to important Azure AD roles, Azure RBAC roles and privileged access groups in order to mitigate the risk of permanently assigning users excessive or unnecessary permissions. We're specifically interested in [privileged access groups](https://docs.microsoft.com/en-us/azure/active-directory/privileged-identity-management/groups-features), which are typically used to allow the activation of multiple role assignments in a single request - for example, an auditor who needs to activate the Global Reader role in Azure AD and the Reader role in many different Azure subscriptions. Rather activate each role individually, they can elevate to membership of a privileged access group that has been assigned the required permissions.

In our case, we're going to create a privileged access group for all users who are exempt to the compliant device requirement (and should therefore have limited, browser-only access to company documents) by adding it as an exclusion/inclusion to the respective conditional access policies.

Step one is to navigate to the [Azure Portal](https://portal.azure.com) and create a role-assignable security group for our browser-only users. While we won't actually be assigning any roles to this group, it must be role-assignable in order to subsequently enable it as a privileged access group:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/new-group.png)

Next, open the group and enable it for privileged access:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/new-privileged-access-group.png)

Once privileged access group has been enabled, it will show up in the [Privileged Identity Management console](https://portal.azure.com/#blade/Microsoft_Azure_PIMCommon/CommonMenuBlade/aadgroup):

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/privileged-access-group-1-1024x376.png)

Click the newly-enabled group and navigate to the Settings page. Privileged access group allow you to configure two roles for activation - Owner and Member. We're specifically interested in the Member role, since group membership is how Azure AD will determine which conditional access policies to apply to our users during sign-in, so select that role for now.

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/privileged-access-group-settings-1024x331.png)

Before we configure the Member role, we need to understand the difference between an _eligible_ and an _active_ role assignment. Essentially, an eligible role assignment is one that requires a user to perform an action before they can actually use the role (e.g. go through an approval process), whereas having an active role assignment means the user has those privileges enabled for use. Refer to Microsoft's [terminology page](https://docs.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-configure#terminology) for more details if you're unsure.

In our case we're going to allow both eligible and active role assignments, since we might need to use both depending on the circumstance. Click the Member role and configure it per the settings below:

[![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/member-settings-activation.png)](https://joshua-lucas.com/wp-content/uploads/2021/11/member-settings-activation.png)

[![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/member-settings-assignment.png)](https://joshua-lucas.com/wp-content/uploads/2021/11/member-settings-assignment.png)

I've configured my role to require eligible users [perform multi-factor authentication](https://docs.microsoft.com/en-us/azure/active-directory/privileged-identity-management/groups-role-settings#require-multifactor-authentication), provide a business justification and have the request approved by an administrator upon activation of the Member role. Permanent eligible assignments are allowed, since we always want the role available to internal users for activation, but I'm also allowing temporary active assignments in case administrators need to assign long-term exemptions (eligible role assignments can only be activated for up to 24 hours at a time, though they can be [extended or renewed](https://docs.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-configure#extend-and-renew-assignments)). I've left the default notification settings as-is.

Once configured, your Member role settings should look something like this:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/member-settings-summary.png)

Our last step is to add an eligible role assignment to the privileged access group for our internal users. I've created a [dynamic security group](https://docs.microsoft.com/en-us/azure/active-directory/enterprise-users/groups-dynamic-membership) using the (user.userType -eq "Member") membership rule for this purpose, but any applicable group will do. Navigate to the Assignments page and add an eligible role assignment as follows:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/add-assignment-membership.png)

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/add-assignment-setting.png)

## Configuring the conditional access policies

Now that we've set up the privileged access group, we need to configure our conditional access policies so that the group is excluded from the [compliant device requirement](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/howto-conditional-access-policy-compliant-device), and create a new policy that imposes app enforced restrictions on when they access company documents in SharePoint, OneDrive and Exchange Online. Here's a snapshot of my existing conditional access policy as an example:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/require-compliant-device-users-1.png)

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/require-compliant-device-apps-1.png)

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/require-compliant-device-grant-1024x648.png)

I've updated the policy so that it targets all users accessing Office 365 (excluding external users, Global Administrators and most importantly, members of the 'Browser-Only Users' group) and only grants access if their device is marked as compliant in Microsoft Endpoint Manager. Next, we need to create a new conditional access policy that limits exempted users' access to Office 365 from a browser only and restricts their ability to download company documents from SharePoint, OneDrive and Exchange Online using app enforced restrictions.

[Application enforced restrictions](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/concept-conditional-access-session#application-enforced-restrictions) are a session control that allows conditional access policies to pass device information to selected cloud apps, which lets us restrict certain actions (e.g. the ability to download documents, or even access the resource at all) from unmanaged devices. Currently, only SharePoint Online (which includes OneDrive for Business, which is built upon SharePoint) and Exchange Online support this session control. We'll start with Exchange Online first.

In order to implement [app enforced restrictions in Exchange Online](https://techcommunity.microsoft.com/t5/outlook-blog/conditional-access-in-outlook-on-the-web-for-exchange-online/ba-p/267069), we need to update the ConditionalAccessPolicy setting in our Outlook Web App mailbox policy. This setting has the following values:

- **Off**: No conditional access policy is applied to Outlook on the web. This is the default value.
- **ReadOnly**: Users can't download attachments to their local computer, and can't enable Offline Mode on non-compliant computers. They can still view attachments in the browser.
- **ReadOnlyPlusAttachmentsBlocked**: All restrictions from ReadOnly apply, but users can't view attachments in the browser.

[Connect to Exchange Online PowerShell](https://docs.microsoft.com/en-us/powershell/exchange/connect-to-exchange-online-powershell?view=exchange-ps#connect-to-exchange-online-powershell-using-modern-authentication-with-or-without-mfa) and run the _Get-OwaMailboxPolicy_ cmdlet to view your existing policies - if you haven't created any custom ones, the default policy is called 'OwaMailboxPolicy-Default':

```powershell
Get-OwaMailboxPolicy | Select-Object Identity,ConditionalAccess*
```

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/get-owamailboxpolicy.png)

Use the _Set-OwaMailboxPolicy_ cmdlet to update your mailbox policy with the desired ConditionalAccessPolicy setting. In my case I'm using the ReadOnly value since I still want users to be able to view attachments in the Outlook Web App, just not download them:

```powershell
Set-OwaMailBoxPolicy -Identity <mailbox_policy> -ConditionalAccessPolicy <value>
```

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/set-owamailboxpolicy.png)

That's it for Exchange Online. In order to implement [app enforced restrictions in SharePoint Online](https://docs.microsoft.com/en-us/sharepoint/control-access-from-unmanaged-devices), navigate to your tenant's SharePoint admin center and select Policies → Access control → Unmanaged devices. You will be prompted to select one of the following choices:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/sharepoint-unmanaged-devices-1024x421.png)

We're going to chose 'Allow limited, web-only access' since it's perfect for our use case. Note that these settings are global and will apply to all SharePoint sites in your tenant, but you can [block or limit access to a specific SharePoint site or OneDrive account](https://docs.microsoft.com/en-us/sharepoint/control-access-from-unmanaged-devices#block-or-limit-access-to-a-specific-sharepoint-site-or-onedrive) if you wish. There are also a handful of [advanced configurations](https://docs.microsoft.com/en-us/sharepoint/control-access-from-unmanaged-devices#advanced-configurations) you can apply using PowerShell, such as the option to allow users to download files that can't be previewed, such as ZIP archives and executables.

**Warning**

Clicking 'Save' will immediately create two new conditional access policies targeting all users that only allows access to SharePoint from compliant devices, and enforces limited browser-only access from unmanaged devices.

When you're ready, save the setting and head on back to the [Conditional Access console](https://portal.azure.com/#blade/Microsoft_AAD_IAM/ConditionalAccessBlade/Policies) in the Azure Portal. You'll notice SharePoint will have created two new conditional access policies for you without much warning:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/sharepoint-created-policies-1024x386.png)

I'd recommend disabling these policies immediately, since we'll be using our own. Create a new conditional access policy configured as follows:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/app-enforced-controls-users.png)

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/app-enforced-controls-apps.png)

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/app-enforced-controls-conditions-1024x609.png)

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/app-enforced-controls-session-1024x611.png)

In summary, this conditional access policy targets members of the 'Browser-Only Users' group accessing Exchange Online and SharePoint, only grants access if the sign-in originates from a browser (rather than a client application like Microsoft Outlook) and imposes our app enforced restrictions so that users can preview company documents, but not download anything. With all that configured, let's review the user experience.

## Reviewing the user experience

First, our user initially tries to access Office 365 from an unmanaged device but is blocked because their device is not compliant:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/you-cant-get-there-e1637500195708.png)

Reviewing the user's [sign-in logs](https://portal.azure.com/#blade/Microsoft_AAD_IAM/UsersManagementMenuBlade/SignIns), we can see they are still included in the 'Require compliant device' policy, which is expected given the error message:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/sign-in-blocked.png)

Next, the user is instructed to navigate to the [Privileged Identity Management console](https://portal.azure.com/#blade/Microsoft_Azure_PIMCommon/ActivationMenuBlade/aadgroup), select My roles → Privileged access groups → Eligible assignments, and apply to activate the Member role of the 'Browser-Only Users' group. The user is prompted to define the duration of their exemption, provide a business justification, and optionally choose a custom activation start time:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/member-activation-1024x353.png)

Once the user has applied to activate the role, members of the 'Administrators' group (or whoever you chose as an approver) will get an email notification asking them to approve or deny the request:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/email-notification.png)

Clicking the 'Approve or deny request' link in the email redirects the approver to the Privileged Identity Management console to review the request:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/member-activation-approval-1024x501.png)

Once approved, the Member role is activated and the user is temporarily added to the 'Browser-Only Users' privileged access group. If they try to access Office 365 once more, the sign in will be successful since they are now exempt from the compliant device requirement, and our browser-only app enforced restrictions will instead be applied:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/privileged-access-group-members-1-1024x292.png)

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/sign-in-success.png)

Upon accessing their emails in the Outlook Web App, the user is presented with a banner stating that they are unable to download or print attachments from this device:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/outlook-web-app-1024x416.png)

In SharePoint and OneDrive, the user is presented with a similar banner stating that they are unable to download, print or sync company documents and that they can only do so from a compliant, work-managed device:

![](/assets/img/2021-11-12-just-in-time-conditional-access-with-azure-ad-pim/sharepoint-1024x856.png)

## Conclusion

In this post we explored using privileged access groups to implement time-based conditional access policy exclusions and impose app enforced restrictions in SharePoint and Exchange Online.

Before trying to implement this in a production environment, please be aware that conditional access policies do have some [limitations regarding group membership and policy update effective times](https://docs.microsoft.com/en-us/azure/active-directory/conditional-access/concept-continuous-access-evaluation#limitations) that may impact its feasibility in larger tenants due to the delay in replication between Azure AD and resource providers like Exchange Online and SharePoint. In practice, I haven't had any issues in small-medium size tenants (in which the conditional access policies updated almost immediately) but it's worth testing for yourself regardless.
