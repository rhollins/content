---
title: "Proven methods to reduce Azure costs for medium size companies and enterprise clients"
date: 2023-03-19T10:31:41Z
draft: false
tags: [azure, costs]
# 5
---

Table of contents

* [Compute costs](#compute-costs)
    * [Utilize Spot VMs](#utilize-spot-vms)
    * [Utilize Burstable VM sizes](#utilize-burstable-vm-sizes)
    * [Use VM Reserved Instances or Savings Plans](#use-vm-reserved-instances-or-savings-plans)
    * [Reduce Azure Virtual Desktop (AVD) costs](#reduce-azure-virtual-desktop-avd-costs)
* [Backup costs](#backup-costs)
    * [Reduce the amount of data stored through backup policies](#reduce-the-amount-of-data-stored-through-backup-policies)
    * [Utilize the archive tier for VM backups](#utilize-the-archive-tier-for-vm-backups)
* [Storage costs](#storage-costs)
    * [Replace Premium managed disks with Standard SSD or Ephemeral where possible](#replace-premium-managed-disks-with-standard-ssd-or-ephemeral-where-possible)
    * [Use reservation for larger managed disks](#use-reservation-for-larger-managed-disks)
    * [Utilize lifecycle and cheaper tiers on storage accounts](#utilize-lifecycle-and-cheaper-tiers-on-storage-accounts)
    * [Take care when enabling Microsoft Defender for Storage](#take-care-when-enabling-microsoft-defender-for-storage)
* [Ingestion costs](#ingestion-costs)
    * [Monitor Azure Defender ingestion rate into Log Analytics Workspace](#monitor-azure-defender-ingestion-rate-into-log-analytics-workspace)
    * [Reduce performance counters collection rate](#reduce-performance-counters-collection-rate)
    * [Reduce flow of container logs running in Azure Kubernetes Service](#reduce-flow-of-container-logs-running-in-azure-kubernetes-service)
* [Last thoughts](#last-thoughts)

After working with many clients over the years on their Azure cloud adoption journey, I found that cost optimization was an essential part of many projects I was working on. In general, I like to include cost analysis at the proof of concept (POC) stage as sometimes the costs themselves might outweigh the benefits of what we are building once it will scale up.

Another reason for including detailed costs analysis in your design decisions is that as much as it is a good thing that cloud providers are trying to make new functionality easy to enable pressing "that one button" to switch something on can sometimes cause your monthly bill to jump up by thousands of dollars due to the less obvious costs.

Let me share some of my findings which I hope you will find useful and perhaps one day can help you avoid unnecessary expense.

## Compute costs

### Utilize Spot VMs

Azure Spot VMs can save you up to 90% on your compute costs so as an example you can use one of the latest and most advanced generations of Azure VMs (v5) with 32 CPU and 128 MEM for just $180 or less per month versus $1500 regular pay-as-you-go price.

However here are the conditions:
* VM can be deallocated or deleted anytime with as low as just 30 seconds of notification.
* There is no guarantee that you will be able to create a new machine of a specific size when you need it.
* You have to request Microsoft to increase your Spot vCPU family quotas (and it is not guaranteed).

The good news is that despite Azure assigned quota all other factors depend mostly on technical skills and level of automation to enable you to take advantage of those low costs VMs.

So far, I found that they work really well with Machine Learning workloads and CI/CD build pipelines.
I also wrote a blog post on how to utilize spot VMs as Jenkins agents running on Kubernetes, [here](https://rhollins.github.io/content/posts/jenkins-on-azure-kubernetes-cluster-aks-flexible-and-cost-effective-jenkins-agents-with-kubernetes-cluster-autoscaler-and-azure-spot-virtual-machines/) it is if you like to learn more details.

Make sure to also pay attention to monthly *eviction rates* for a given VM size which usually does not excess 20% and in some cases is as low as 5%.

### Utilize Burstable VM sizes

After using Spot VMs and provisioning VMs only when you need them using Burstable sizes is the next best way to cut those compute costs.

I would recommend them for all your non-production environments while keeping an eye on CPU credits balance if your application is consuming CPU at a constantly higher rate than the General sizes might be a better choice.

### Use VM Reserved Instances or Savings Plans

Azure VM Reserved Instances allows you to save up to 30% of VM compute costs on average if you commit to having this machine switched on for at least a year, or save even more if you were to opt-in for 3 years commitment.

Based on my experience using reserved instances could become maintenance heavy once you get into a range of hundreds of servers or above.

Here are a few points to highlight:
* Azure Advisor in the portal was not always accurate enough with reservation suggestions - at least in my experience.
* Prepare yourself to produce on regular basis an inventory of your VMs including sizes, locations as well as last month's uptime all these are crucial to making the right decision when buying reservations.
* Remember that you can also reserve Azure VM Scale Sets (for example Azure Kubernetes Service nodes qualify here).
* If you change VM sizes often there might be penalties for cancelling reservations.
* Make sure to have oversight of the process of changing VM sizes in your company as well as keep an eye on reservation expirations - this will ensure that you are not increasing costs due to reservations.

If you will find Azure VM reservation to be a bad choice for your company however you still want to save costs on permanently running workloads you can utilize Azure Saving Plans instead. In this case, you simply commit to a specific amount of compute usage over a year or 3 years.

### Reduce Azure Virtual Desktop (AVD) costs

[AVD](https://learn.microsoft.com/en-us/azure/virtual-desktop/overview) became popular especially during the pandemic either by replacing employees physical desktops while work from home (WFH) or by replacing the company's physical disaster recovery sites with virtual machines in Azure.

Some of what I already covered applies here as well but worth to mention a few more things:

* Since AVD are usually Windows 10/11 machines, you potentially could qualify for Azure Hybrid Benefits. You should check on that with your Microsoft account manager or representative.

* You can quite easily implement automation which will deallocate AVD when it wasn't used (user session was not available, the user did not log in) for a specific amount of time and it could be provisioned up again when the user tries to log in. From my experience, this provides significant cost reduction

* Keep in mind that Development team machines usually will require a docker desktop installed with docker engine enabled and only specific usually more expensive VM sizes can allow this functionality.

* Depending on your configuration, AVD might be treated the same as your servers if it is about Azure Defender for Cloud, therefore, a large number of those machines will increase your ingestions costs and will be included in your Defender Plan which in some cases can significantly increase overall costs.

* If you enabled Microsoft Defender for Storage with a "per transaction" rate for your Azure subscription and you are using AVD profiles based on FSLogix rather than local one, you might see a significant spike in your scanned transaction costs, so please plan accordingly.

## Backup costs

### Reduce the amount of data stored through backup policies

This might sound obvious, but it is not always that straightforward. In highly regulated environments it is not uncommon to have requirements to store data for 10 years, however, you might be able to store data in less frequent backup timestamps the further they go back in time.

Before making such a change it would be good to check on that with your company compliance officer or someone fulfilling a similar function.

Once it is established, Azure backup policies in both Azure Recovery Service Vault and Azure Backup Vault are quite flexible and could allow you to, for example store only monthly backup after 1 year and annual backup after 3 years in some cases, like large file server or database server backup, this could greatly reduce storage costs.

### Utilize the archive tier for VM backups

For VMs with higher churn (data change rate), it might be beneficial to move old data to the [archive tier](https://learn.microsoft.com/en-us/azure/backup/archive-tier-support) however they will be at this point converted from incremental to full recovery points. Similarly to buying a VM reservation it is a good idea to do some analysis to ensure that it will not cost more in the end.

## Storage costs

### Replace Premium managed disks with Standard SSD or Ephemeral where possible

As a rule of thumb, I would recommend using Standard SSD first, and only moving to Premium disks when you see that performance starts to suffer. Switching the SKU of the managed disk is easy and the difference in cost is quite substantial.

Ephemeral OS disks are free and ideal for stateless applications. I am using them very effectively as disks for Azure Kubernetes cluster nodes since nodes themself should not be used to store any data, there is no risk of data loss, and at the same, you get the benefit of lower latency and reduced costs compared to managed disks backed K8s nodes.
The only caveat is that you cannot use them with Burstable B sizes which are usually used for non-production environments.

### Use reservation for larger managed disks

If you have any managed permanent disks of size 1TB or larger it probably is a good candidate to purchase reservations for.

### Utilize lifecycle and cheaper tiers on storage accounts

Azure Files and Blob storage offer cheaper costs of storing data that are accessed less frequently and stored for longer periods (like for example backups, and logs). Azure Files offer Cool tier while Blobs offers Cool and Archive tier keep in mind that with the latter there is a rehydration period involved.

If you are heavily utilizing snapshots and versioning functionality in Blob store it will increase the amount of storage that you need to pay for. I would recommend using lifecycle management and periodically removing older versions and snapshots that might be no longer needed.

### Take care when enabling Microsoft Defender for Storage

Microsoft Defender for Storage is worth enabling if you take into account that a big chunk of security breaches in any cloud provider is due to a misconfiguration in the storage access layer.

Currently, there are two modes you can use where you pay either "per transaction" or "per storage account". The "per transaction" mode can be very costly if you have a large number of transactions, especially on a large number of storage accounts.

Before you switch it on check the number of monthly transactions on all your storage accounts (you can do it in Azure Portal Monitoring under the Insight section). If it exceeded millions of transactions on each of your storage accounts regularly it might be better to switch to "per storage account" pricing.

If you have for example only a few accounts that generate a huge amount of transactions it might be better to leave "per transaction" mode switched on on all accounts and temporarily disable Microsoft Defender on those few. Then ensure account access configuration is not modified outside of change control process while you work on a solution to reduce those transactions numbers.

## Ingestion costs

### Monitor Azure Defender ingestion rate into Log Analytics Workspace

When you enable Microsoft Defender and for example VM insight on your VMs it might be sometimes hard to say how much data each of your sources are sending to the log analytics workspace. In my case, I remember being surprised once, when it turns out that only one server was responsible for around 20% of overall ingestion charges. You can use Log Analytics Workspace Insights which could help you identify high ingestion sources.

### Reduce performance counters collection rate

Both legacy "Log Analytics Agent" as well as new "Azure Monitoring Agents" allow reducing the rate at which performance counters are being sent to Log Analytics Workspace, which again depending on your scale could have quite a big impact on overall workspace storage costs.

### Reduce flow of container logs running in Azure Kubernetes Service

Clusters that are running a large number of containers can very quickly start ingesting a huge amount of data into the log analytics workspace that you set up as the target for container insight.
When that happens you can change the existing container insight setting on running the AKS cluster by modifying the configmap. Among other options you could exclude specific namespace from collecting logs at all or only collect logs from stderr output, more information can be found [here](https://learn.microsoft.com/en-us/azure/azure-monitor/containers/container-insights-agent-config#configure-and-deploy-configmaps).

## Last thoughts

In this article, I tried to cover some of the most common resources and costs, however every resource in Azure has its own specific ways to optimize its costs, so I would always start by reading the Azure pricing page for that particular offering.

