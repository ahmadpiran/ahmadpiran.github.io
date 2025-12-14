---
date: '2025-12-14T15:30:52+03:00'
draft: false
title: 'Troubleshooting the invisible Error: When the Entra Provisioning Agent won’t Sync.'
tags: ["MicrosoftEntra", "CloudSync", "HybridIdentity", "ActiveDirectory", "AzureAD"]
---

## Introduction: The Client's Dilemma**

You’ve just upgraded your hybrid identity setup on Azure, moving from the heavy AAD Connect to the lightweight **Microsoft Entra Provisioning Agent** (Cloud Sync). The installation finished cleanly on your Domain Controller, the service is running, and the Entra portal says "Healthy." Everything looks perfect… except for one critical detail: **zero users are syncing to the cloud.**

The worst part? The logging is uselessly vague. You check the Entra Portal and see a generic "sync errors or failures" status. When you use the "Provision on Demand" tool, you get a frustratingly simple error: *"The object doesn’t exist."* There are no permission failures, no obvious certificate issues—just a wall of silence between your local AD and the cloud.

This isn’t a minor annoyance; it’s a critical blocker. Without a successful sync, you can’t onboard new users, assign Microsoft 365 licenses, or guarantee the integrity of your hybrid environment. In short, the entire identity flow is frozen.

In this case study, we dive into a recent client scenario where we faced this "healthy-but-stuck" issue. I’ll walk you through the steps we used to bypass the vague logs, pinpoint the surprising firewall rule responsible, and get the user provisioning flowing instantly.

## **Deep Diagnosis**

We’re going to diagnose the issue step by step. Sometimes, even the most obvious configuration points can hide the real problem, so we have to check them all.

### **1. The Permission Trap**

Whenever the Entra Provisioning Agent is deployed to a Domain Controller, the first thing every admin checks is the **gMSA** (Group Managed Service Account) permissions. We needed to verify that the auto-generated service account had the critical **"Log on as a service"** right applied via Local Security Policy.

We simply launched `services.msc`, located the **Microsoft Entra Provisioning Agent** service, and checked its status.

- **Is it running?** Yes.
- **The Logon Account:** We right-clicked **Properties** > **Log On** tab. The account name was correct (e.g., `DOMAIN\provAgentMSA$`).

All good. Since the service was running, we knew the issue was downstream from the logon process.

### **2. Hunting in the Event Logs**

We couldn’t find any obvious symptoms in the cloud, so we shifted our focus to the local server. Maybe we could find a clue in the logs on the local Domain Controller.

But Windows Event Logs can be like a dense forest—you can easily get lost looking for a needle in a haystack. Standard filtering often isn't enough; you have to know exactly where to look.

We bypassed the standard Windows System logs and went straight to the specialized channel: the **ProvisioningAgent/Admin** logs.

> Path: Applications and Services Logs → Microsoft → ProvisioningAgent → Admin
> 

**Wait, what are these?** We were faced with dozens of red error messages. The repeated, key error was clear: **"The agent could not establish a connection."**

This immediately eliminated permissions and shifted our focus to the network layer.

### **3. The Targeted Network Test**

We knew from the [official documentation](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/how-to-prerequisites) that we needed to verify the agent could communicate with Azure datacenters.

However, we could already log in to the Azure Portal and Office 365 from that server. Ports 80 and 443 were open, and we knew we didn't need to allow incoming traffic. But upon closer inspection of the docs, we realized we needed to allow access to specific URLs.

We ran targeted `Test-NetConnection` checks in PowerShell to verify:

PowerShell

`# 1. Test connection to Entra ID Login
Test-NetConnection -ComputerName "login.microsoftonline.com" -Port 443
# Result: TcpTestSucceeded: True (Success)

# 2. Test Connection to the App Proxy service
Test-NetConnection -ComputerName "mscrl.microsoft.com" -Port 80
# Result: TcpTestSucceeded: True (Success)

# 3. Test Connection to the Azure Service Bus
Test-NetConnection -ComputerName "servicebus.windows.net" -Port 443
# Result: TcpTestSucceeded: False (FAILURE)`

**Oops.**
While the server could reach the basic authentication endpoints, the test against the critical infrastructure endpoint failed: `servicebus.windows.net`.

This is the **Azure Service Bus**, Microsoft’s cloud-based messaging service. The agent uses this to maintain a persistent connection to the cloud. This immediately told us the story: The agent was installed correctly and authenticated, but the corporate firewall was blocking the necessary outbound communication channel required for Cloud Sync to receive its commands from Microsoft.

## **The Fix and Validation**

So, we had our "smoking gun." The Entra Provisioning Agent architecture is fundamentally different from the old AAD Connect. It relies entirely on the Azure Service Bus to maintain a persistent outbound "heartbeat." Because the firewall was treating this like standard web traffic, it blocked the infrastructure signals while allowing the login traffic.

### **The Specific Solution**

The fix wasn't to just "open the internet." We needed a targeted rule. I directed the network administrator to whitelist **Outbound HTTPS (Port 443)** specifically for:

- `.servicebus.windows.net`

### **Closing the Loop**

Once the firewall rule was applied, we didn’t just sit and hope. We had to reset the connection logic. I went back to `services.msc` and restarted the **Microsoft Entra Provisioning Agent** service.

Then, the moment of truth. I navigated back to that "forest" of logs—the **ProvisioningAgent/Admin** channel.

- **Before:** Red "Failed to connect" errors.
- **Now:** A fresh Information event stating: **"The agent successfully established a connection with the service."**

The wall of silence was broken.

### **The "Pro Tip" Validation: Provision on Demand**

A rookie mistake here is to sit and wait for the scheduled sync cycle (which runs every 10 minutes). But we needed to know *right now*.

I went back to the Entra Portal, opened the Cloud Sync configuration, and clicked **Provision on Demand**. I entered the Distinguished Name of our stuck test user and hit **Provision**.

Previously, this screen gave us the "object doesn't exist" error. This time?

- **Step 1: Import user** ... ✅ Success.
- **Step 2: Determine if user is in scope** ... ✅ Success.
- **Step 3: Match user** ... ✅ Success.
- **Step 4: Action** ... ✅ Provision.

In less than 5 seconds, the user appeared in the Microsoft 365 admin center, ready for a license.

### **The Cleanup: Handling the "Split-Brain" Identity**

We weren't quite done. During the downtime, the client had tried to "force" things to work by manually creating a user in Office 365 with the same email address as the local AD user. This created a classic **"Split-Brain" identity**:

1. **Local AD User:** Used for laptop login (the source of truth).
2. **Cloud User:** A totally separate, cloud-only account created manually.

If we just let the sync run, it might have resulted in a "soft match" error or a duplicate `user1234@onmicrosoft.com` identity. The fix was surgical:

1. We **deleted** the manually created cloud-only user (and removed it from the Deleted Users recycle bin to avoid conflicts).
2. We re-ran **Provision on Demand** for that specific user.
3. The Agent created a pristine, perfectly linked cloud identity mapped to the local AD user.
4. The client assigned the license, and Outlook connected instantly using the user's local domain password.

## **Conclusion: It’s Only Invisible If You Don’t Look**

Migrating to the Entra Provisioning Agent is supposed to be simpler, and usually, it is. But "simpler" often means the error messages are hidden behind user-friendly "Healthy" statuses.

If you ever find yourself with a healthy agent but zero synced users, remember this case study. Don’t get lost in the generic logs.

1. **Check the `ProvisioningAgent/Admin` event log** immediately.
2. **Verify the GMSA permissions**, but don't stop there.
3. **Test the Service Bus.** If `Test-NetConnection servicebus.windows.net -Port 443` fails, no amount of restarting services will fix it.

It turns out, the error wasn't invisible after all—it was just waiting for us to ask the right question.

**Resources**
1. [Prerequisites for Microsoft Entra Cloud Sync](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/how-to-prerequisites)
2. [Cloud sync troubleshooting](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/how-to-troubleshoot)
3. [On-demand provisioning - Active Directory to Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity/hybrid/cloud-sync/how-to-on-demand-provision)