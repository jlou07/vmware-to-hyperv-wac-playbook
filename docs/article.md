# From VMware to Hyper-V with Windows Admin Center

Broadcom's acquisition of VMware was a wake-up call. The question is no longer *should we migrate?* but *how do we migrate without breaking everything or losing our minds?*. Caught between expensive third-party solutions and risky manual methods, Microsoft has quietly released a powerful tool: the VM Conversion extension for Windows Admin Center (WAC).

![](https://jlou.eu/wp-content/uploads/2026/03/image-20.png)

After testing the tool — still in preview — I wanted to share my experience. To help you navigate this long article more easily, here are quick links:

- **FAQ**
  - [Why leave VMware?](#FAQ-I)
  - [What is Windows Admin Center?](#FAQ-II)
  - [What is the VM Conversion extension?](#FAQ-III)
  - [How much does VM Conversion cost?](#FAQ-IV)
  - [Which OS versions are supported?](#FAQ-V)
  - [What are the prerequisites for VM Conversion?](#FAQ-VI)
  - [How many VMs can I migrate simultaneously?](#FAQ-VII)
  - [How does the VM Conversion tool work?](#FAQ-VIII)
  - [What about downtime?](#FAQ-IX)
  - [What happens to VMware Tools on the migrated VM?](#FAQ-X)
- **Migration walkthrough**
  - [Step 0 – Prerequisites recap](#Etape-0)
  - [Step I – Installing Windows Admin Center](#Etape-I)
  - [Step II – Installing WAC prerequisites](#Etape-II)
  - [Step III – Connecting to Hyper-V from WAC](#Etape-III)
  - [Step IV – Windows test: VM synchronization](#Etape-IV)
  - [Step V – Windows test: VM migration](#Etape-V)
  - [Step VI – Linux test: VM synchronization](#Etape-VI)
  - [Step VII – Linux test: VM migration](#Etape-VII)

---

<a id="FAQ-I"></a>
## Why leave VMware?

Broadcom's acquisition of VMware was a wake-up call. Many customers saw costs jump sharply after the acquisition, as Broadcom changed the licensing model (for example, shifting from perpetual licenses to a subscription model) and restructured its offerings. Some received renewal quotes several times higher than before for the same needs.

These rapid changes in product structure, bundles, and support have created uncertainty around the VMware roadmap and product future, leading CIOs to rethink their long-term technology choices.

There are quite a few bloggers writing about this exodus:

- [Why Leave VMware in 2025? Alternatives and Migration Guide](https://pandorafms.com/blog/why-leave-vmware-alternatives-2025/)
- [Navigating the VMware Exit: Why OpenStack is the Smart Alternative for 2025 and Beyond | OpenMetal IaaS](https://openmetal.io/resources/blog/navigating-the-vmware-exit/)
- [VMware : 6 options pour en sortir, 1 piste pour y rester](https://www.cio-online.com/actualites/lire-vmware-6-options-pour-en-sortir-1-piste-pour-y-rester-16231.html)
- [Life After VMware: Accelerating VMware Exits with In-Place Migration • Platform9](https://platform9.com/blog/life-after-vmware-accelerating-vmware-exits-with-in-place-migration/)
- [Hyper-V 2026: Not the Same Beast You Ditched in 2012 (And That's a Good Thing) | by Mr.PlanB | Jan, 2026 | Medium](https://medium.com/@PlanB./hyper-v-2026-not-the-same-beast-you-ditched-in-2012-and-thats-a-good-thing-5522709a5c7c)
- [Ditching Broadcom, Hyper-V Server, & Live Migrations | by Rich | Medium](https://happycamper84.medium.com/ditching-broadcom-hyper-v-server-live-migrations-3b41cf2a8830)

<a id="FAQ-II"></a>
## What is Windows Admin Center?

Windows Admin Center (often abbreviated WAC) is a web-based administration tool developed by Microsoft to manage Windows servers, clusters, and hyperconverged environments from a modern browser interface.

![](https://jlou.eu/wp-content/uploads/2026/02/image-403-1024x576.png)

In practice, Windows Admin Center lets you manage — without using RDP — a wide range of Microsoft services:

- Windows Server
- Hyper-V
- Clusters (Failover Clustering)
- Virtual machines
- Remote servers (on-premises or Azure)
- Storage (Storage Spaces Direct)

https://youtu.be/wT2ps\_71VKU

<a id="FAQ-III"></a>
## What is the VM Conversion extension?

It is a new Microsoft tool built into Windows Admin Center that enables agentless migration of virtual machines from VMware vCenter/ESXi to Hyper-V:

![](https://jlou.eu/wp-content/uploads/2026/02/image-400-1024x428.png)

It is designed to minimize downtime through live disk replication while the source VM keeps running.

Note: this extension is not intended for migrations to Azure Local. A separate article covering that migration type is available here: [from VMware to Azure Local](https://jlou.eu/vmwareazurestackhci/).

<a id="FAQ-IV"></a>
## How much does VM Conversion cost?

The extension is currently in preview. There is no additional licensing cost, but Microsoft does not provide guaranteed support for preview features.

<a id="FAQ-V"></a>
## Which OS versions are supported?

Microsoft publishes [a broad compatibility list](https://learn.microsoft.com/en-us/windows-server/manage/windows-admin-center/use/vm-conversion-extension-overview#vcenter-versions-and-guest-operating-systems):

- All VMware vCenter 6.x, 7.x, and 8.x versions are supported
- Supported guest operating systems:
  - Windows Server 2025, 2022, 2019, 2016, 2012 R2
  - Windows 10/11
  - Ubuntu 20.04, 24.04
  - Debian 11, 12
  - Alma Linux
  - CentOS
  - Red Hat Linux 9.0

The extension also works in a **clustered Hyper-V (WSFC)** environment: it is cluster-aware and correctly detects nodes and resources.

It supports migrating VMs from ESXi to Windows Server Failover Clusters (WSFC) running Hyper-V, allowing you to distribute migrated VMs across multiple Hyper-V nodes for HA or performance purposes.

![](https://jlou.eu/wp-content/uploads/2026/02/image-405-1024x683.png)

<a id="FAQ-VI"></a>
## What are the prerequisites for VM Conversion?

- Windows Admin Center side:
  - WAC version v2410 build 2.4.12.10 or later
  - PowerShell and PowerCLI installed
  - VMware VDDK version 8.0.3 installed
  - Visual Studio 2013 and 2015 redistributables installed
- Hyper-V side:
  - The Hyper-V role must be installed on the target host(s)
  - The account used during the process must be a local administrator or a member of the Hyper-V Administrators group
  - For Linux VMs, the Hyper-V modules (hv\_vmbus, hv\_netvsc, hv\_storvsc) and Linux Integration Services (hyperv-daemons or linux-cloud-tools) must be pre-installed in the initramfs before migration

<a id="FAQ-VII"></a>
## How many VMs can I migrate simultaneously?

Up to 10 virtual machines per batch. Machines can be grouped by application dependency, cluster placement, or business criteria (e.g., separating test from production environments), which makes large-scale workload migrations more manageable.

<a id="FAQ-VIII"></a>
## How does the VM Conversion tool work?

Microsoft clearly explains the two phases in the VM Conversion tool:

> **Synchronization**: the extension performs a full initial copy of the virtual machine's disks while the source VM continues running. This phase minimizes downtime by letting you plan the final migration at a convenient time.
>
> **Migration**: the extension uses the Change Block Tracking (CBT) feature to identify and replicate only the blocks that have changed since the last synchronization. During the cutover, the source VM is powered off and a final delta sync captures all remaining changes before importing the VM into Hyper-V.
>
> [Microsoft Learn](https://learn.microsoft.com/en-us/windows-server/manage/windows-admin-center/use/vm-conversion-extension-overview#how-it-works)

![](https://jlou.eu/wp-content/uploads/2026/02/image-407.png)

<a id="FAQ-IX"></a>
## What about downtime?

During the synchronization phase, the source VM keeps running while an initial copy of its disks is transferred to the Hyper-V host, using a snapshot to track changes.

At cutover time, the source VM is shut down for a final delta sync before import. This process reduces the downtime window to a minimum.

<a id="FAQ-X"></a>
## What happens to VMware Tools on the migrated VM?

VMware Tools running under Hyper-V can cause driver conflicts, so it is best to remove them.

- Originally, VMware Tools had to be uninstalled manually after switching the VM to Hyper-V.
- Since [version 1.8.0](https://learn.microsoft.com/en-us/windows-server/manage/windows-admin-center/use/whats-new-vm-conversion-extension), the extension automatically removes VMware Tools from Windows VMs at the end of the migration process.
- For Linux VMs, it is recommended not to reinstall open-vm-tools on the Hyper-V target (rely solely on the Hyper-V drivers instead).

![](https://jlou.eu/wp-content/uploads/2026/03/image-18.png)

As always, the rest of this article walks you through the full migration process step by step:

- [Step 0 – Prerequisites recap](#Etape-0)
- [Step I – Installing Windows Admin Center](#Etape-I)
- [Step II – Installing WAC prerequisites](#Etape-II)
- [Step III – Connecting to Hyper-V from WAC](#Etape-III)
- [Step IV – Windows test: VM synchronization](#Etape-IV)
- [Step V – Windows test: VM migration](#Etape-V)
- [Step VI – Linux test: VM synchronization](#Etape-VI)
- [Step VII – Linux test: VM migration](#Etape-VII)

---

<a id="Etape-0"></a>
## Step 0 – Prerequisites recap

The following prerequisites are required for this exercise covering the migration of a virtual machine from VMware to Hyper-V via Windows Admin Center:

- A VMware environment
- A Hyper-V environment
- Network connectivity between the two hypervisors

Let's start by installing Windows Admin Center on a new VM hosted in VMware.

<a id="Etape-I"></a>
## Step I – Installing Windows Admin Center

On a dedicated virtual machine in your VMware environment, download the latest version of Windows Admin Center from the [official Microsoft page](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-admin-center):

![](https://jlou.eu/wp-content/uploads/2026/03/image-1-1024x401.png)

Select the Express installation option:

![](https://jlou.eu/wp-content/uploads/2026/03/image-3.png)

For this demonstration, use a self-signed certificate:

![](https://jlou.eu/wp-content/uploads/2026/03/image-5.png)

Start the installation:

[![](https://jlou.eu/wp-content/uploads/2026/02/image-319.png)](https://jlou.eu/?attachment_id=24024)

Once installation is complete, open the Windows Admin Center page and sign in with your account:

![](https://jlou.eu/wp-content/uploads/2026/02/image-326.png)

After signing in, wait a few minutes for the pre-installed extensions to finish loading:

![](https://jlou.eu/wp-content/uploads/2026/02/image-328-1024x555.png)

Go to WAC settings to add the VM Conversion extension:

![](https://jlou.eu/wp-content/uploads/2026/02/image-329-1024x644.png)

<a id="Etape-II"></a>
## Step II – Installing WAC prerequisites

As stated in the Microsoft documentation, certain prerequisites must be installed on the Windows Admin Center machine.

Start by installing [Visual C++ Redistributable Packages for Visual Studio 2013](https://www.microsoft.com/en-us/download/details.aspx?id=40784&msockid=3d75fb163d8768940e5fedff3c3f690d):

![](https://jlou.eu/wp-content/uploads/2026/02/image-320.png)

Then install [Visual C++ 2015-2022 Redistributable 14.50.35719.0](https://aka.ms/vc14/vc_redist.x64.exe):

![](https://jlou.eu/wp-content/uploads/2026/02/image-321.png)

Download and extract **VMware Virtual Disk Development Kit (VDDK)** version 8.0.3 into the following WAC folder:

![](https://jlou.eu/wp-content/uploads/2026/02/image-336-1024x767.png)

Then restart the virtual machine running Windows Admin Center.

<a id="Etape-III"></a>
## Step III – Connecting to Hyper-V from WAC

Once the WAC VM has restarted, open the console and click here to add a connection to your Hyper-V server:

![](https://jlou.eu/wp-content/uploads/2026/02/image-330-1024x168.png)

Click **Add**:

![](https://jlou.eu/wp-content/uploads/2026/03/image-8.png)

Enter the **IP address** of your Hyper-V host along with the credentials, then click **Add**:

![](https://jlou.eu/wp-content/uploads/2026/03/image-10.png)

The connection is established — click on it to open it:

![](https://jlou.eu/wp-content/uploads/2026/02/image-333-1024x175.png)

In the left menu, find the VM Conversion extension, check the box to install **PowerCLI**, then click here:

![](https://jlou.eu/wp-content/uploads/2026/02/image-334-1024x689.png)

Wait for PowerCLI installation to complete:

![](https://jlou.eu/wp-content/uploads/2026/02/image-335-1024x640.png)

Once installation is complete, click here to add a connection to your **vCenter**, enter your credentials, then click here:

![](https://jlou.eu/wp-content/uploads/2026/02/image-337-1024x639.png)

Once the **vCenter** connection is established, your virtual machines will appear:

![](https://jlou.eu/wp-content/uploads/2026/02/image-338-1024x641.png)

The test environment is now ready. Let's begin with the migration of a Windows Server virtual machine.

<a id="Etape-IV"></a>
## Step IV – Windows test: VM synchronization

From the VM list, select one of the available Windows VMs and click here to start synchronization:

![](https://jlou.eu/wp-content/uploads/2026/02/image-339-1024x397.png)

Enter the destination folder on your Hyper-V server, then click Synchronize:

![](https://jlou.eu/wp-content/uploads/2026/03/image-14.png)

The pre-migration check phase has started (accessibility, hardware compatibility, OS compatibility, network configuration, ...):

![](https://jlou.eu/wp-content/uploads/2026/02/image-341-1024x310.png)

The initial full copy of VMDK disks to Hyper-V is underway, which will produce VHDX files:

![](https://jlou.eu/wp-content/uploads/2026/02/image-343-1024x325.png)

You can observe the folder being created on the Hyper-V server:

![](https://jlou.eu/wp-content/uploads/2026/02/image-344-1024x544.png)

The process switches to differential synchronization mode, copying only the blocks changed since the initial sync and reducing the data volume to be transferred before cutover:

![](https://jlou.eu/wp-content/uploads/2026/02/image-345-1024x396.png)

Target Hyper-V disk (VHDX) provisioning: the infrastructure is preparing storage before applying the synchronized blocks from the VMware VM:

![](https://jlou.eu/wp-content/uploads/2026/02/image-346-1024x417.png)

Delta sync phase: changed blocks identified via CBT are applied to the Hyper-V disk to align the target VM with the current state of the source VM:

![](https://jlou.eu/wp-content/uploads/2026/02/image-347-1024x412.png)

Network card monitoring confirms active data transfer:

![](https://jlou.eu/wp-content/uploads/2026/02/image-348-1024x776.png)

Synchronization complete (100%): all source VM data is now replicated on the Hyper-V side:

![](https://jlou.eu/wp-content/uploads/2026/02/image-349-1024x292.png)

The target VM is now fully aligned with the VMware VM and can be switched to Hyper-V with a minimal final delta.

We are ready to proceed with the migration.

<a id="Etape-V"></a>
## Step V – Windows test: VM migration

At this point, the VMware virtual machine is still running:

![](https://jlou.eu/wp-content/uploads/2026/02/image-351-1024x429.png)

To test the migration delta, add new files to the source virtual machine:

![](https://jlou.eu/wp-content/uploads/2026/02/image-356-1024x763.png)

Go back to Windows Admin Center to trigger the migration:

![](https://jlou.eu/wp-content/uploads/2026/02/image-363-1024x259.png)

Choose whether to uninstall VMware Tools, then click here:

![](https://jlou.eu/wp-content/uploads/2026/03/image-12.png)

Migration launch: pre-migration checks are running before the final cutover to the target Hyper-V VM:

![](https://jlou.eu/wp-content/uploads/2026/02/image-364-1024x289.png)

Pre-migration checks complete: the final delta sync of changed blocks is in progress before the VM is stopped and switched over:

![](https://jlou.eu/wp-content/uploads/2026/02/image-359-1024x247.png)

A snapshot has been created on the VMware side:

![](https://jlou.eu/wp-content/uploads/2026/02/image-367-1024x313.png)

Delta sync in progress: the last set of changed blocks is being applied to definitively align the target VM before the final cutover to Hyper-V:

![](https://jlou.eu/wp-content/uploads/2026/02/image-365-1024x287.png)

Source VM shutdown: the VMware machine is powered off and the final synchronization phase begins before startup on Hyper-V:

![](https://jlou.eu/wp-content/uploads/2026/02/image-368-1024x282.png)

The virtual machine is now stopped on the VMware side:

![](https://jlou.eu/wp-content/uploads/2026/02/image-369-1024x427.png)

Migration complete (100%): the target Hyper-V VM is created, synchronized, and ready to start in production:

![](https://jlou.eu/wp-content/uploads/2026/02/image-370-1024x293.png)

The migrated VM is now running on Hyper-V, confirming successful cutover from the VMware environment:

![](https://jlou.eu/wp-content/uploads/2026/02/image-371-1024x609.png)

Windows detects an unexpected shutdown related to the cutover, confirming the VM was successfully switched from VMware to Hyper-V:

![](https://jlou.eu/wp-content/uploads/2026/02/image-372.png)

VMware Tools error at first boot: the old VMware components, now incompatible under Hyper-V, need to be uninstalled after migration:

![](https://jlou.eu/wp-content/uploads/2026/02/image-373.png)

Uninstall VMware Tools, which are no longer needed once the VM is running under Hyper-V:

![](https://jlou.eu/wp-content/uploads/2026/02/image-374.png)

Files created before the Migrate command are present after the cutover, confirming the integrity of the final synchronization:

![](https://jlou.eu/wp-content/uploads/2026/02/image-375.png)

The Windows server has been successfully migrated from VMware to Hyper-V. Let's now perform the same operation with a Linux server.

<a id="Etape-VI"></a>
## Step VI – Linux test: VM synchronization

This example was performed on AlmaLinux / RHEL-like. The goal was to:

- Uninstall VMware Tools
- Install Hyper-V components
- Rebuild initramfs if needed
- Reboot

Create a Linux virtual machine running an OS compatible with the migration tool:

![](https://jlou.eu/wp-content/uploads/2026/02/image-376-1024x783.png)

Before migration, add Hyper-V drivers to the initramfs, rebuild with **dracut**, then reboot to ensure a correct startup under Hyper-V:

```bash
echo 'add_drivers+=" hv_vmbus hv_storvsc hv_netvsc "' | sudo tee /etc/dracut.conf.d/hyperv.conf
sudo dracut -f --regenerate-all
sudo reboot
```

Still before migration, install Hyper-V Integration Services (hyperv-daemons) to optimize performance and interaction between the Linux VM and the Hyper-V host.

Example on a RHEL-like distribution (AlmaLinux / Rocky / RHEL). Adapt the command if you are on Ubuntu or Debian:

```bash
sudo dnf install hyperv-daemons -y
```

![](https://jlou.eu/wp-content/uploads/2026/02/image-377-1024x691.png)

Verify that the Hyper-V modules (hv\_\*) are loaded in the Linux kernel to confirm proper Hyper-V support:

![](https://jlou.eu/wp-content/uploads/2026/02/image-378.png)

From the VM list, select one of the available Linux VMs and click here to start synchronization:

![](https://jlou.eu/wp-content/uploads/2026/02/image-379-1024x270.png)

Enter the destination folder on your Hyper-V server, then click Synchronize:

![](https://jlou.eu/wp-content/uploads/2026/03/image-16.png)

The pre-migration check phase has started (accessibility, hardware compatibility, OS compatibility, network configuration, ...):

![](https://jlou.eu/wp-content/uploads/2026/02/image-381-1024x284.png)

The initial full copy of VMDK disks to Hyper-V is underway, which will produce VHDX files:

![](https://jlou.eu/wp-content/uploads/2026/02/image-382-1024x278.png)

You can observe the folder being created on the Hyper-V server:

![](https://jlou.eu/wp-content/uploads/2026/02/image-383.png)

The process switches to differential synchronization mode, copying only the blocks changed since the initial sync and reducing the data volume to be transferred before cutover:

![](https://jlou.eu/wp-content/uploads/2026/02/image-385-1024x288.png)

Delta sync phase: changed blocks identified via CBT are applied to the Hyper-V disk to align the target VM with the current state of the source VM:

![](https://jlou.eu/wp-content/uploads/2026/02/image-387-1024x277.png)

Network card monitoring confirms active data transfer:

![](https://jlou.eu/wp-content/uploads/2026/02/image-388.png)

Synchronization complete (100%): all source VM data is now replicated on the Hyper-V side:

![](https://jlou.eu/wp-content/uploads/2026/02/image-389-1024x284.png)

The target VM is now fully aligned with the VMware VM and can be switched to Hyper-V with a minimal final delta.

We are ready to proceed with the migration.

<a id="Etape-VII"></a>
## Step VII – Linux test: VM migration

Go back to Windows Admin Center to trigger the migration:

![](https://jlou.eu/wp-content/uploads/2026/02/image-394-1024x311.png)

Migration launch: pre-migration checks are running before the final cutover to the target Hyper-V VM:

![](https://jlou.eu/wp-content/uploads/2026/02/image-395-1024x301.png)

Source VM shutdown: the VMware machine is powered off and the final synchronization phase begins before startup on Hyper-V:

![](https://jlou.eu/wp-content/uploads/2026/02/image-396-1024x303.png)

The virtual machine is now stopped on the VMware side:

![](https://jlou.eu/wp-content/uploads/2026/02/image-397-1024x796.png)

Migration complete (100%): the target Hyper-V VM is created, synchronized, and ready to start in production:

![](https://jlou.eu/wp-content/uploads/2026/02/image-398-1024x302.png)

The migrated VM is now running on Hyper-V, confirming successful cutover from the VMware environment:

![](https://jlou.eu/wp-content/uploads/2026/02/image-401-1024x727.png)

---

## Conclusion

Migrating from VMware to Hyper-V is never a trivial decision. It is usually driven by economic, strategic, or organizational factors. But regardless of the reason, it must remain a controlled, structured, and technically sound operation above all.

The **VM Conversion** extension for Windows Admin Center is not just a graphical wizard. It provides an orchestrated and consistent approach: initial synchronization, delta via CBT, then a controlled cutover phase. It is not "magic" and does not remove the need for serious preparation. However, the tool provides a clear framework that substantially simplifies a historically complex process.

In my tests, the experience proved stable and readable. Hyper-V cluster support (WSFC), the progressive synchronization logic, and the unified interface within WAC make migration far more accessible than manual approaches. It remains a preview extension, so you should thoroughly evaluate it in a test environment before any production use — but the underlying technical foundation is solid.

What is clear is that a credible alternative now exists for organizations looking to diversify or reposition their hypervisor strategy. Migration no longer needs to be seen as a leap into the unknown, but as a structured, manageable, and well-documented project.
