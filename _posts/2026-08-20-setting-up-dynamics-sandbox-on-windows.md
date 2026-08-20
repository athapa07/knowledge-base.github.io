---
layout: post
title: Configuring Visual Studio and SSMS for Dynamics 365 Finance & Operations (Part 2)
subtitle: Ditching the Mac VM for a native Windows machine, then wiring up Visual Studio, Dataverse, and your first model
tags: [tools, power-platform, dynamics-365, dataverse, visual-studio, ssms, finance-and-operations, x++]
author: Anil Thapa
---

# Configuring Visual Studio and SSMS for Dynamics 365 Finance & Operations

In the [previous post](https://athapa07.github.io/knowledge-base.github.io/2026-08-19-setting-up-dynamics-365-f-&-o/), I walked through provisioning a Dynamics 365 Finance and Operations sandbox — tenant, licensing, Azure credit, and a Power Platform environment. The next step is getting the actual development tooling connected to that environment: Visual Studio, SQL Server Management Studio, and your first F&O model.

Virtualizing Windows 11 on a Mac turned out to be more trouble than it was worth — enough app-access issues that I dropped the VM approach entirely and moved to a native Windows machine instead. If you're on Apple Silicon and hitting similar walls with UTM or another hypervisor, this is the easier path.

## A Note on Virtualization

Running Windows 11 in a VM on Mac (via UTM or similar) works in principle, but in practice it caused enough app-access problems — driver issues, performance, tooling that just wouldn't cooperate — that it wasn't worth pursuing further. If you have access to a physical Windows machine, use it. It'll save you hours of troubleshooting.

## Installing the Tools

On the Windows machine, install:

- **SQL Server Management Studio (SSMS)**
- **Visual Studio Professional 2022**

When installing Visual Studio, be deliberate about the workload and components you select:

- Enable the **.NET desktop development** workload
- Make sure **Entity Framework 6 tools**, the **Modeling SDK**, and the **DGML editor** are included
- Sign in with your Microsoft account
- Install the **Reporting Services** and **Power Platform Tools** extensions

## Configuring Power Platform Tools

Once the extensions are installed, go to **Tools → Options** and search for **Power Platform Tools**. Turn on the following:

- Filter out plugin assemblies from Microsoft
- Display detailed log data
- Capture detailed Dataverse communications log
- Skip discovery when connecting to Dataverse
- Download logs for operations when using a Unified environment
- Enable auto setup for Dynamics 365 when using a Unified environment

These settings make troubleshooting much easier later — the detailed logging in particular is worth having on from the start rather than turning it on after something breaks.

## Connecting to the Sandbox

With the extension configured, connect Visual Studio to your Dataverse sandbox:

1. **Tools → Connect to Dataverse** — this opens a new connection window
2. Set **Deployment Type** to **Office 365**
3. Check **Display list of available organizations**
4. The organization URL is often picked up automatically. If it isn't, grab it manually from the Power Platform Admin Center
5. Select your sandbox Dataverse environment — this should happen automatically, but confirm it if you're prompted
6. For the solution, don't leave the default selected — choose the one created for you (it'll have an auto-generated name like `Cr7f2222`)
7. Give it a few minutes. Once connected, all your tables will be listed

### If something goes wrong

Check **View → Output** to see what's actually happening. If components are missing, install them, and run **Tools → Download Dynamics 365 FinOps assets**.

This step can take a while — it's pulling down a large volume of reference data. To confirm it's actually progressing, check:

```
C:\Users\<username>\AppData\Local\Microsoft\Dynamics365\<version>\
```

Look at the combined properties for that folder — once complete, it should total around **24 GB**.

## Setting Up Metadata

With the local connection sorted, configure your metadata paths:

**Extensions → Dynamics 365 → Configure Metadata**

- Update the description
- Point to a folder for your own custom metadata (this can be any folder you choose)
- Point to the folder for reference metadata — typically:
  ```
  C:\Users\<username>\AppData\Local\Microsoft\Dynamics365\<version>\PackagesLocalDirectory
  ```

## Creating Your First Model and Package

Now you're ready to create a working model:

**Extensions → Dynamics 365 → Model Management → Create Model**

Fill in the parameters, for example:

- **Model name:** `ppac`
- **Model publisher:** your name
- **Description:** a short summary of the model's purpose

From there, create a new package and select it as a **reference package**. The three most commonly referenced packages are:

- **Application Foundation**
- **Application Platform**
- **Application Suite**

Verify everything on the summary screen before confirming.

### Verifying the Project

Once creation finishes, you'll be prompted to confirm the project name and location. Double-check:

- The **custom metadata folder** you defined during project creation
- The **Git repo folder** defined during model creation

Then, in **Solution Explorer**:

- Right-click the project and **Build**
- Run **Synchronize** on your project
- **Deploy** the model for the project

## Connecting to the Database Directly

For direct SQL access against your F&O environment:

1. Go to **Tools → SQL Credentials for Dynamics 365 FinOps**
2. Select your solution and request access
3. Temporary credentials are generated, valid for **1 day**
4. Open **SQL Server Management Studio**, sign in with your Microsoft account, and enter the provided credentials

From there, you can query the underlying tables directly — useful for debugging data issues without going through the application layer.

---

## Wrap-up

At this point you've got a fully connected local development setup: Visual Studio talking to your Dataverse sandbox, a working model and package structure, and direct SQL access for when you need to look under the hood. The heaviest part of this stage is the initial FinOps asset download — once that 24 GB of reference metadata is in place, everything else moves quickly.

Next up: building out custom extensions against the `Application Suite` reference package.
