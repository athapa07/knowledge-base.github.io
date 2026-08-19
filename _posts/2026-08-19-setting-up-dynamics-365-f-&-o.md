---
layout: post
title: Setting Up a Dynamics 365 Finance & Operations Dev Environment
subtitle: From an empty Microsoft 365 tenant to a working F&O sandbox, with a Windows VM for the tooling that won't run on a Mac
tags: [tools, power-platform, dynamics-365, dataverse, azure, microsoft-365, finance-and-operations, vm]
author: Anil Thapa
---

# Setting Up a Dynamics 365 Finance & Operations Dev Environment: A Step-by-Step Walkthrough

Spinning up a proper Dynamics 365 Finance and Operations (F&O) sandbox from scratch touches half a dozen different Microsoft services — Microsoft 365, Azure, the Power Platform Admin Center, and eventually a full Windows VM for the SQL Server and Visual Studio tooling. None of it is hard on its own, but the pieces aren't obviously connected unless you've done it before.

Here's the full path I followed, from an empty Microsoft 365 tenant to a working F&O sandbox environment.

## 1. Create a Microsoft 365 Tenant (Business Basic Trial)

Everything starts with a tenant. I signed up for a **Microsoft 365 Business Basic trial**, which gives you the base tenant and admin center access needed for the rest of the setup.

> **Tip:** Once the trial is active, go into billing settings and turn **auto-renew off** immediately. This stops you from being silently charged once the trial period ends.

<!-- https://snapshot.opc.com.au/BkY2HnFs -->

## 2. Activate the Dynamics 365 Finance Premium License

With the tenant in place, activate a **Dynamics 365 Finance Premium** trial license from the Microsoft 365 admin center (`admin.cloud.microsoft`). This is the license that unlocks Finance and Operations apps for the environment you'll create later.

Same rule applies here — once the trial is active, switch **auto-renew off** so it doesn't convert into a paid subscription automatically.

<!-- https://snapshot.opc.com.au/smfv06Dp -->
## 3. Create an Azure Account (Free $200 Credit)

F&O environments run on Azure infrastructure under the hood, so you'll need an Azure subscription linked to the same tenant. Signing up as a new Azure customer gets you a **$200 free credit** for the first 30 days, which is more than enough to get a sandbox running and tested.

<!-- https://snapshot.opc.com.au/M78t6nbP -->

## 4. Link the Azure Subscription in the Power Platform Admin Center

This is the step that ties Microsoft 365, Dynamics 365, and Azure together. In the **Power Platform Admin Center**, link your Azure subscription so that Dataverse capacity and environments can be billed against it.

<!-- https://snapshot.opc.com.au/x8mJj0rf -->

## 5. Create a Dataverse Billing Plan

Before you can create a sandbox environment, Dataverse needs a billing plan tied to your Azure subscription:

**Licensing → Dataverse → Manage Capacity → Default Environment → New Billing Plan**


## 6. Create the Sandbox Environment

Now for the environment itself, from **Environments → New**:

- **Type:** Sandbox
- **Security group:** None
- **Enable Dynamics 365 apps:** Yes
- **Enable Dataverse:** Yes

## 7. Install the Finance and Operations Platform Tools

Once the environment is provisioned, go to:

**Environments → [your new environment] → Resources → Dynamics 365 Apps → Dynamics 365 Finance and Operations Platform Tools**

Install it — this step takes roughly **10 minutes**.

## 8. Install the Finance and Operations Provisioning App

Next, still under Dynamics 365 Apps, install **Dynamics 365 Finance and Operations Provisioning App**. This is the long pole in the process — expect it to take **1 to 2 hours**.

*[Insert screenshot: Provisioning app install progress]*

## 9. Set Up a Windows VM for SQL Server and Visual Studio

The remaining tools — **SQL Server Management Studio** and **Visual Studio Professional 2022** — only ship as `.exe` installers, so if you're on a Mac, you'll need a Windows VM.

I used **UTM** for this:

- Install UTM following the [macOS installation guide](https://docs.getutm.app/installation/macos/)
- Set up a **Windows 11** VM following the [Windows guide](https://docs.getutm.app/guides/windows/)

### Fixing networking inside the VM

After installation, networking may not work out of the box inside the VM. If that happens, install the missing network driver and restart the VM — it resolves the connectivity issue.
For reference: [https://snapshot.opc.com.au/hBcdTf7G](https://snapshot.opc.com.au/hBcdTf7G)

### Installing the tooling inside the VM

Once the VM is up and networked:

- Install **SQL Server Management Studio** from the [official Microsoft docs](https://learn.microsoft.com/en-us/ssms/install/install)
- Download **Visual Studio** via `my.visualstudio.com` (subscription details required — pull these from your Visual Studio subscription portal)

---

## Wrap-up

At this point you've got a full loop: a licensed Microsoft 365 tenant, an Azure subscription funding Dataverse capacity, a provisioned F&O sandbox environment, and a Windows VM ready for the SQL Server and Visual Studio tooling needed for deeper customization work.

The whole process is mostly waiting — the platform tools and provisioning app installs alone account for over an hour of that — so it's worth kicking off step 7 and 8 and using the downtime to get your VM set up in parallel.
