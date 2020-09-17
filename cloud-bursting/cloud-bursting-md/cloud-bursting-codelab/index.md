---
id: cloud-bursting-codelab
summary: In this codelab, you'll prototype cloud bursting from two clusters in GCP.
status: [published]
authors: Kim Wong
categories: Web
tags: cds17,chrome-dev-summit-2016,devfest18,gdd17,io2016,io2017,io2018,io2019,jsconfeu,kiosk,pwa-dev-summit,pwa-roadshow,tag-web,typtwd17,web

---

# Cloud Bursting via Slurm Federated Clusters




## Overview



This lab updates the original codelab called  [Building Federated HPC Clusters with Slurm](https://codelabs.developers.google.com/codelabs/hpc-slurm-federated-on-gcp). This update is necessary for a couple of reasons:

* The original codelab did not contain complete information and as a result was difficult for novices to follow
* The Google Cloud Platform Console interface had been updated, resulting in divergence between the lab and the platform that can easily trip up students

I renamed this codelab to Cloud Bursting via Federated Slurm Clusters because this is the relevant end goal for my Center and for many other academic computing centers. Credit for this lab should extend to the original creators. As you can see, I shamelessly borrowed from the original and enhanced it where I see the need. If you discover areas that need improvement, please send it my way and I'll do my best to incorporate suggestions.  Let's begin.

Welcome to the Google Codelab for creating two federated Slurm clusters on Google Cloud Platform! By the end of this cloud lab you should have a solid understanding of the ease of provisioning and configuring Slurm clusters with federation, and submitting jobs between clusters.

<img src="img/c16fa310c142ac6f.png" alt="c16fa310c142ac6f.png"  width="312.00" />

Google Cloud teamed up with [SchedMD](https://www.schedmd.com/) to release a set of tools that make it easier to launch the Slurm workload manager on Compute Engine, and to expand your existing cluster when you need extra resources. This integration was built by the experts at [SchedMD](https://www.schedmd.com/) in accordance with Slurm best practices.

If you're planning on using the [Slurm on Google Cloud Platform](https://github.com/SchedMD/slurm/tree/master/contribs/gcp) integrations, or if you have any questions, please consider joining our [Google Cloud & Slurm Community Discussion Group](https://groups.google.com/forum/#!forum/google-cloud-slurm-discuss)!

### **About Slurm**

Slurm is one of the leading workload managers for HPC clusters around the world. Slurm provides an open-source, fault-tolerant, and highly-scalable workload management and job scheduling system for small and large Linux clusters. Slurm requires no kernel modifications for its operation and is relatively self-contained. As a cluster workload manager, Slurm has three key functions:

1. It allocates exclusive or non-exclusive access to resources (compute nodes) to users for some duration of time so they can perform work.

2. It provides a framework for starting, executing, and monitoring work (normally a parallel job) on the set of allocated nodes.

3. It arbitrates contention for resources by managing a queue of pending work.

> aside positive
> **Learn More About Slurm**
> *  [SchedMD Homepage](https://www.schedmd.com/)
> *  [Slurm on GCP ReadMe](https://github.com/SchedMD/slurm/blob/master/contribs/gcp/README.md)
> *  [Slurm Quickstart Guide](https://slurm.schedmd.com/quickstart.html)
> *  [Slurm MAN Pages](https://slurm.schedmd.com/man_index.html)
> *  [Slurm Command Summary](https://slurm.schedmd.com/pdfs/summary.pdf) (PDF)
> *  [Slurm Accounting Guide](https://slurm.schedmd.com/accounting.html)
> *  [Slurm Troubleshooting Guide](https://slurm.schedmd.com/troubleshoot.html)
> *  [Slurm Users Discussion Group](https://groups.google.com/forum/#!forum/slurm-users)

### **What you'll learn**

* How to use the GCP Deployment Manager Service
* How to run a job using SLURM
* How to query cluster information and monitor running jobs in SLURM
* How to create a federation between two clusters and submit federated jobs
* Where to find help with Slurm

### **Prerequisites**

* Google Cloud Platform Account and a Project with Billing
* Basic Linux Experience
* Completed the " [Deploy an Auto-Scaling HPC Cluster with Slurm](https://codelabs.developers.google.com/codelabs/hpc-slurm-on-gcp/)" codelab


## Seeing the Big Picture



### **Why federate Slurm clusters?**

Many users have existing clusters in their environment, either on-premise with user-managed hardware, or already running in Google Cloud Platform. In these cases many users want to supplement their existing resources with burstable cloud-based nodes, possibly in different regions, and with different hardware or software configurations. This use case is solved by using Slurm's federation capabilities alongside the new Slurm Auto-Scaling capabilities in Google Cloud Platform.  The basic architectural diagram of two federated Slurm Clusters in Google Cloud Platform is shown below.

<img src="img/6e62315487c4d73b.png" alt="6e62315487c4d73b.png"  width="624.00" />

In order to test Slurm's federation capabilities in Google Cloud Platform we'll need to set up two clusters, one called **on-prem** and the other called **gcp-burst**.  The two clusters will exist in their own project and have **different slurm-network subnet range**:
**on-prem** | 10.10.0.0/16
--- | ---
**gcp-burst** | 10.20.0.0/16

For this codelab we will use a cross-project VPN, and we will use internal IPs. Customers may use external IPs as well. This is detailed in the  [Slurm on GCP Readme](https://github.com/SchedMD/slurm-gcp/blob/master/README.md).  The VPN permits Slurm on the **on-prem** cluster to federate jobs to the **gcp-burst** cluster.


## Deploying the on-prem cluster



### Our goal.

We want to provision a cluster that represents our on-premise cluster. This deployment is representative of on-premise because it will be in its own network space and will have the following subnet ranges 10.10.0.0/16.  The following sections follow the codelab " [Deploy an Auto-Scaling HPC Cluster with Slurm](https://codelabs.developers.google.com/codelabs/hpc-slurm-on-gcp/)".

### Create a new project called on-prem

From the Google Cloud Console, click on **Select a project** → **NEW PROJECT**. Let's name it on-prem and provision it by clicking the **CREATE** button.

<img src="img/de5bff5b6c623121.png" alt="de5bff5b6c623121.png"  width="624.00" />

<img src="img/fac73e5619af9e9f.png" alt="fac73e5619af9e9f.png"  width="530.40" />

### **Configure the VPC Network**

Next, we will configure the VPC with the subnet range 10.10.0.0/16. If you have other projects under your account, make sure that the current one selected is called **on-prem**. Click on the Google Cloud Console hamburger icon (the three horizontal lines on the far left) and navigate to the NETWORKING section **VPC network** → **VPC networks**.

<img src="img/6dc5a799b0b9bc1b.png" alt="6dc5a799b0b9bc1b.png"  width="624.00" />

Click on **+CREATE VPC NETWORK**, name it *on-prem*, set the Subnet creation mode to *custom*, name the subnet as *slurm-network*, set the IP address range to *10.10.0.0/16*, and then click the **CREATE** button.

<img src="img/47d784ed3687961e.png" alt="47d784ed3687961e.png"  width="624.00" />

<img src="img/3500ffc4d7d88436.png" alt="3500ffc4d7d88436.png"  width="530.40" />

### **Configure the Firewall Rules**

Next we will configure the necessary firewall rules for the on-prem VPC network.

<img src="img/c240cbab435d90f1.png" alt="c240cbab435d90f1.png"  width="624.00" />

<img src="img/bb3ce7a2c75619fc.png" alt="bb3ce7a2c75619fc.png"  width="624.00" />

<img src="img/11db328c55796b19.png" alt="11db328c55796b19.png"  width="624.00" /> <img src="img/8ada47fdf90fbab4.png" alt="8ada47fdf90fbab4.png"  width="624.00" /> <img src="img/c2fff0b42ba594ac.png" alt="c2fff0b42ba594ac.png"  width="624.00" /> <img src="img/4b153f30f0b38e94.png" alt="4b153f30f0b38e94.png"  width="624.00" />

### **Configure the NAT Gateway**

Next we will configure the NAT Gateway to provide the on-prem VPC network access to the internet. From the Google Cloud Console hamburger menu, navigate to the NETWORKING **Network services** → **Cloud NAT** panel. Click on **+CREATE NAT GATEWAY** and fill in the field using the following information:
**Gateway name**  | on-prem-us-central1-gateway
--- | ---
**VPC network** | on-prem
**Cloud Router** | on-prem-us-central1-router

Leave the remaining fields in the default setting and click on the **CREATE** button.

<img src="img/d5fad8e9530273f3.png" alt="d5fad8e9530273f3.png"  width="624.00" /> <img src="img/c9a57cf23a2b11e6.png" alt="c9a57cf23a2b11e6.png"  width="624.00" /> <img src="img/a926f91dc773b1a3.png" alt="a926f91dc773b1a3.png"  width="624.00" />

### **Provision the on-prem cluster**

Next we will deploy the **on-prem** Slurm cluster using the deployment manager scripts as described in the " [Deploy an Auto-Scaling HPC Cluster with Slurm](https://codelabs.developers.google.com/codelabs/hpc-slurm-on-gcp/)" codelab. When deployed, the login node, controller, and image nodes will show up as VM instances under Compute Engine.  Let us start by navigating to the [Compute] **Compute Engine** → **VM instances panel**. 

<img src="img/726f9440d1e26ce0.png" alt="726f9440d1e26ce0.png"  width="624.00" />

Click on the **Activate Cloud Shell icon** on the upper right side of the toolbar. This will bring up a commandline interface to the **on-prem** project.

<img src="img/b6b7330e33ed1561.png" alt="b6b7330e33ed1561.png"  width="624.00" />

Next, we will clone the deployment-manager files and cd into the directory to modify the deployment configuration YAML file **slurm-cluster.yaml**.  You can use your typical command line editors, such as vi, nano, or emacs or use the Cloud Console Code Editor  <img src="img/373bb0f74110d3dc.png" alt="373bb0f74110d3dc.png"  width="87.75" /> to modify the file.

    git clone https://github.com/SchedMD/slurm-gcp.git on-prem
    cd on-prem
    vi slurm-cluster.yaml

<img src="img/4a39b7d56c56db5c.png" alt="4a39b7d56c56db5c.png"  width="624.00" />

Modify the slurm-cluster.yaml file with the following parameters (be sure to uncomment the **vpc_net** and **vpc_subnet** lines):
**cluster_name** | on-prem
--- | ---
**zone** | us-central1-b
**vpc_net** | on-prem
**vpc_subnet** | slurm-network

Additionally, change the name of the first partition from **debug** to **test** and uncomment the block under **# Additional partition** and update the values as follows:

        # Additional partition
    
          - name           : smp
            machine_type   : n1-standard-64
            max_node_count : 42
            zone           : us-central1-b

The full modified **slurm-cluster.yaml** file should look like the following:

    # Copyright 2017 SchedMD LLC.
    # Modified for use with the Slurm Resource Manager.
    #
    # Copyright 2015 Google Inc. All rights reserved.
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.
    
    # [START cluster_yaml]
    imports:
    - path: slurm.jinja
    
    resources:
    - name: slurm-cluster
      type: slurm.jinja
      properties:
        cluster_name            : on-prem
    
        zone                    : us-central1-b
    
      # Optional network configuration fields
      # READ slurm.jinja.schema for prerequisites
        vpc_net                   : on-prem
        vpc_subnet                : slurm-network
        # shared_vpc_host_project   : < my-shared-vpc-project-name >
    
        controller_machine_type : n1-standard-2
        # controller_disk_type      : pd-standard
        # controller_disk_size_gb   : 50
        # external_controller_ip    : False
        # controller_labels         :
        #   key1 : value1
        #   key2 : value2
        # controller_service_account: default
        # controller_scopes         :
        # - https://www.googleapis.com/auth/cloud-platform
        # cloudsql                  :
        #   server_ip: <cloudsql ip>
        #   user: slurm
        #   password: verysecure
        #   # Optional
        #   db_name: slurm_accounting
    
        login_machine_type        : n1-standard-2
        # login_disk_type           : pd-standard
        # login_disk_size_gb        : 20
        # external_login_ips        : False
        # login_labels              :
        #   key1 : value1
        #   key2 : value2
        # login_node_count          : 0
        # login_node_service_account: default
        # login_node_scopes         :
        # - https://www.googleapis.com/auth/devstorage.read_only
        # - https://www.googleapis.com/auth/logging.write
    
      # Optional network storage fields
      # network_storage is mounted on all instances
      # login_network_storage is mounted on controller and login instances
        # network_storage           :
        #   - server_ip: <storage host>
        #     remote_mount: /home
        #     local_mount: /home
        #     fs_type: nfs
        # login_network_storage     :
        #   - server_ip: <storage host>
        #     remote_mount: /net_storage
        #     local_mount: /shared
        #     fs_type: nfs
    
        compute_image_machine_type  : n1-standard-2
        # compute_image_disk_type   : pd-standard
        # compute_image_disk_size_gb: 20
        # compute_image_labels      :
        #   key1 : value1
        #   key2 : value2
    
      # Optional compute configuration fields
        # external_compute_ips      : False
        # private_google_access     : True
    
        # controller_secondary_disk         : True
        # controller_secondary_disk_type    : pd-standard
        # controller_secondary_disk_size_gb : 300
    
        # compute_node_service_account : default
        # compute_node_scopes          :
        #   -  https://www.googleapis.com/auth/devstorage.read_only
        #   -  https://www.googleapis.com/auth/logging.write
    
        # Optional timer fields
        # suspend_time              : 300
    
        # slurm_version             : 19.05-latest
        # ompi_version              : v3.1.x
    
        partitions :
          - name              : test
            machine_type      : n1-standard-2
            max_node_count    : 10
            zone              : us-central1-a
    
        # Optional compute configuration fields
    
            # cpu_platform           : Intel Skylake
            # preemptible_bursting   : False
            # compute_disk_type      : pd-standard
            # compute_disk_size_gb   : 20
            # compute_labels         :
            #   key1 : value1
            #   key2 : value2
            # compute_image_family   : custom-image
    
        # Optional network configuration fields
            # vpc_subnet                : default
    
        # Optional GPU configuration fields
    
            # gpu_type               : nvidia-tesla-v100
            # gpu_count              : 8
    
    
        # Additional partition
    
          - name           : smp
            machine_type   : n1-standard-64
            max_node_count : 42
            zone           : us-central1-b
    
        # Optional compute configuration fields
    
            # cpu_platform           : Intel Skylake
            # preemptible_bursting   : False
            # compute_disk_type      : pd-standard
            # compute_disk_size_gb   : 20
            # compute_labels         :
            #   key1 : value1
            #   key2 : value2
            # compute_image_family   : custom-image
            # network_storage        :
            #   - server_ip: none
            #     remote_mount: <gcs bucket name>
            #     local_mount: /data
            #     fs_type: gcsfuse
            #     mount_options: file_mode=664,dir_mode=775,allow_other
            #
    
        # Optional network configuration fields
            # vpc_subnet                : my-subnet
    
        # Optional GPU configuration fields
            # gpu_type               : nvidia-tesla-v100
            # gpu_count              : 8
    
    #  [END cluster_yaml]

Next we will use gcloud to provision the cluster. Within the Cloud Shell, execute the following command to deploy a Slurm cluster using our updated configuration specifications:

    gcloud deployment-manager deployments create on-prem1 --config slurm-cluster.yaml

**Note:** If you're running this codelab in a new project you may be prompted to enable the Deployment Manager API with a message like this:

API [deploymentmanager.googleapis.com] not enabled on project

[956620689341]. Would you like to enable and retry? (y/N)?

If you get this prompt enter **Y** and press **enter** to continue.

<img src="img/d940d0c6ea70c655.png" alt="d940d0c6ea70c655.png"  width="624.00" />

The deployment takes several minutes to complete.  If everything goes according to plan without error, you should see four VM instances: a login node, a controller node, a compute image node corresponding to the **test** partition and a compute image node corresponding to the **smp** partition.  To login into on-prem-login0 VM, you can either click on the **SSH** link under the Connect column or issue this command within the Cloud Shell

    gcloud compute ssh on-prem-login0 --zone=us-central1-b

<img src="img/5cc88d14910b4dd3.png" alt="5cc88d14910b4dd3.png"  width="624.00" />

The initial login into the on-prem cluster will trigger installation and configuration of Slurm. This process takes several minutes, so now is a good time to take a break.

<img src="img/e02ce8a72dd98af7.png" alt="e02ce8a72dd98af7.png"  width="624.00" />

Now that Slurm is installed, log out and log back into the cluster. You first will need to **CTRL+C** out of the process followed by the **exit** command. The Slurm banner should greet you upon successful login. If the Slurm **sinfo** command outputs the partitions as specified  in the **slurm-cluster.yaml** file (with 10 nodes for the **test** partition and 42 nodes for the **smp** partition), our **on-prem** cluster has been successfully deployed. Note that the IPs for the four VM instances are within the 10.10.0.x block, as specified in the **slurm-network** VPC subnet.

<img src="img/5f0be4f346275fe2.png" alt="5f0be4f346275fe2.png"  width="624.00" />


## Deploying the gcp-burst cluster
Duration: 03:00


### **Our goal.**

We want to provision a cluster that represents our cloud resource. This deployment is representative of a cloud resource because it will be in its own network space distinct from the **on-prem** cluster and will have the following subnet ranges 10.20.0.0/16.  The following sections follow the codelab " [Deploy an Auto-Scaling HPC Cluster with Slurm](https://codelabs.developers.google.com/codelabs/hpc-slurm-on-gcp/)" and especially the previous topic **Deploying the on-prem cluster**.

### **Create a new project called gcp-burst**

From the Google Cloud Console, click on **Select a project** → **NEW PROJECT**. Let's name it on-prem and provision it by clicking the **CREATE** button.

<img src="img/9a57ef651ddaa881.png" alt="9a57ef651ddaa881.png"  width="624.00" />

<img src="img/35b981f40870060c.png" alt="35b981f40870060c.png"  width="530.40" />

### **Configure the VPC Network**

Next, we will configure the VPC with the subnet range 10.20.0.0/16. If you have other projects under your account, make sure that the current one selected is called **gcp-burst**. Click on the Google Cloud Console hamburger icon and navigate to the NETWORKING section **VPC network** → **VPC networks**.

<img src="img/ac570dfbe2f90c84.png" alt="ac570dfbe2f90c84.png"  width="624.00" />

Click on **+CREATE VPC NETWORK**, name it *gcp-burst*, set the Subnet creation mode to *custom*, name the subnet as *slurm-network*, set the IP address range to *10.20.0.0/16*, and then click the **CREATE** button.

<img src="img/2175693ecbe3d97e.png" alt="2175693ecbe3d97e.png"  width="624.00" />

<img src="img/227711a8f55becb.png" alt="227711a8f55becb.png"  width="624.00" />

### **Configure the Firewall Rules**

Next we will configure the necessary firewall rules for the on-prem VPC network.

<img src="img/d14f91dd2060a708.png" alt="d14f91dd2060a708.png"  width="624.00" /> <img src="img/33ad0004ebe4de6a.png" alt="33ad0004ebe4de6a.png"  width="624.00" /> <img src="img/33c79c9203e61071.png" alt="33c79c9203e61071.png"  width="624.00" /> <img src="img/f3209f76b40481f0.png" alt="f3209f76b40481f0.png"  width="624.00" /> <img src="img/f66b1efbbb100ebd.png" alt="f66b1efbbb100ebd.png"  width="624.00" /> <img src="img/76813574e5437b08.png" alt="76813574e5437b08.png"  width="624.00" /> <img src="img/848ce8566d2e4ae7.png" alt="848ce8566d2e4ae7.png"  width="624.00" />

### **Configure the NAT Gateway**

Next we will configure the NAT Gateway to provide the gcp-burst VPC network access to the internet. From the Google Cloud Console hamburger menu, navigate to the [NETWORKING] **Network services** → **Cloud NAT** panel. Click on **+CREATE NAT GATEWAY** and fill in the field using the following information:
**Gateway name**  | gcp-burst-us-central1-gateway
--- | ---
**VPC network** | gcp-burst
**Cloud Router** | gcp-burst-us-central1-router

Leave the remaining fields in the default setting and click on the **CREATE** button.

<img src="img/ea6a8859809b49b6.png" alt="ea6a8859809b49b6.png"  width="624.00" /> <img src="img/7ea720249070caea.png" alt="7ea720249070caea.png"  width="624.00" /> <img src="img/d939967f16c582ac.png" alt="d939967f16c582ac.png"  width="624.00" />

### **Provision the gcp-burst cluster**

Next we will deploy the **on-prem** Slurm cluster using the deployment manager scripts as described in the " [Deploy an Auto-Scaling HPC Cluster with Slurm](https://codelabs.developers.google.com/codelabs/hpc-slurm-on-gcp/)" codelab. When deployed, the login node, controller, and image nodes will show up as VM instances under Compute Engine.  Let us start by navigating to the [Compute] **Compute Engine** → **VM instances panel**. 

<img src="img/e467b4b53fb2dfdb.png" alt="e467b4b53fb2dfdb.png"  width="624.00" />

Click on the **Activate Cloud Shell icon** on the upper right side of the toolbar. This will bring up a commandline interface to the **gcp-burst** project.

<img src="img/1e90cdf018a7039.png" alt="1e90cdf018a7039.png"  width="624.00" />

Next, we will clone the deployment-manager files and cd into the directory to modify the deployment configuration YAML file **slurm-cluster.yaml**.  You can use your typical command line editors, such as vi, nano, or emacs or use the Cloud Console Code Editor  <img src="img/373bb0f74110d3dc.png" alt="373bb0f74110d3dc.png"  width="87.75" /> to modify the file.

    git clone https://github.com/SchedMD/slurm-gcp.git gcp-burst
    cd gcp-burst
    vi slurm-cluster.yaml

<img src="img/49e7ea2ffb1b94c2.png" alt="49e7ea2ffb1b94c2.png"  width="624.00" />

Modify the slurm-cluster.yaml file with the following parameters (be sure to uncomment the **vpc_net** and **vpc_subnet** lines):
**cluster_name** | gcp-burst
--- | ---
**zone** | us-central1-b
**vpc_net** | gcp-burst
**vpc_subnet** | slurm-network

Additionally, uncomment the **ompi_version** line and modify the partitions to specify ones for **gpu-k80** and **gpu-v100**.  The configuration snippets are

          partitions :
          - name              : gpu-k80
            machine_type      : n1-standard-16
            max_node_count    : 20
            zone              : us-central1-b

and

        # Additional partition
    
          - name           : gpu-v100
            machine_type   : n1-standard-32
            max_node_count : 10
            zone           : us-central1-b

The full modified **slurm-cluster.yaml** file should look like the following:

    # Copyright 2017 SchedMD LLC.
    # Modified for use with the Slurm Resource Manager.
    #
    # Copyright 2015 Google Inc. All rights reserved.
    #
    # Licensed under the Apache License, Version 2.0 (the "License");
    # you may not use this file except in compliance with the License.
    # You may obtain a copy of the License at
    #
    #     http://www.apache.org/licenses/LICENSE-2.0
    #
    # Unless required by applicable law or agreed to in writing, software
    # distributed under the License is distributed on an "AS IS" BASIS,
    # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    # See the License for the specific language governing permissions and
    # limitations under the License.
    
    # [START cluster_yaml]
    imports:
    - path: slurm.jinja
    
    resources:
    - name: slurm-cluster
      type: slurm.jinja
      properties:
        cluster_name            : gcp-burst
    
        zone                    : us-central1-b
    
      # Optional network configuration fields
      # READ slurm.jinja.schema for prerequisites
        vpc_net                   : gcp-burst
        vpc_subnet                : slurm-network
        # shared_vpc_host_project   : < my-shared-vpc-project-name >
    
        controller_machine_type : n1-standard-2
        # controller_disk_type      : pd-standard
        # controller_disk_size_gb   : 50
        # external_controller_ip    : False
        # controller_labels         :
        #   key1 : value1
        #   key2 : value2
        # controller_service_account: default
        # controller_scopes         :
        # - https://www.googleapis.com/auth/cloud-platform
        # cloudsql                  :
        #   server_ip: <cloudsql ip>
        #   user: slurm
        #   password: verysecure
        #   # Optional
        #   db_name: slurm_accounting
    
        login_machine_type        : n1-standard-2
        # login_disk_type           : pd-standard
        # login_disk_size_gb        : 20
        # external_login_ips        : False
        # login_labels              :
        #   key1 : value1
        #   key2 : value2
        # login_node_count          : 0
        # login_node_service_account: default
        # login_node_scopes         :
        # - https://www.googleapis.com/auth/devstorage.read_only
        # - https://www.googleapis.com/auth/logging.write
    
      # Optional network storage fields
      # network_storage is mounted on all instances
      # login_network_storage is mounted on controller and login instances
        # network_storage           :
        #   - server_ip: <storage host>
        #     remote_mount: /home
        #     local_mount: /home
        #     fs_type: nfs
        # login_network_storage     :
        #   - server_ip: <storage host>
        #     remote_mount: /net_storage
        #     local_mount: /shared
        #     fs_type: nfs
    
        compute_image_machine_type  : n1-standard-2
        # compute_image_disk_type   : pd-standard
        # compute_image_disk_size_gb: 20
        # compute_image_labels      :
        #   key1 : value1
        #   key2 : value2
    
      # Optional compute configuration fields
        # external_compute_ips      : False
        # private_google_access     : True
    
        # controller_secondary_disk         : True
        # controller_secondary_disk_type    : pd-standard
        # controller_secondary_disk_size_gb : 300
    
        # compute_node_service_account : default
        # compute_node_scopes          :
        #   -  https://www.googleapis.com/auth/devstorage.read_only
        #   -  https://www.googleapis.com/auth/logging.write
    
        # Optional timer fields
        # suspend_time              : 300
    
        # slurm_version             : 19.05-latest
        ompi_version              : v3.1.x
    
        partitions :
          - name              : gpu-k80
            machine_type      : n1-standard-16
            max_node_count    : 20
            zone              : us-central1-b
    
        # Optional compute configuration fields
    
            # cpu_platform           : Intel Skylake
            # preemptible_bursting   : False
            # compute_disk_type      : pd-standard
            # compute_disk_size_gb   : 20
            # compute_labels         :
            #   key1 : value1
            #   key2 : value2
            # compute_image_family   : custom-image
    
        # Optional network configuration fields
            # vpc_subnet                : default
    
        # Optional GPU configuration fields
    
            gpu_type               : nvidia-tesla-k80
            gpu_count              : 4
    
    
        # Additional partition
    
          - name           : gpu-v100
            machine_type   : n1-standard-32
            max_node_count : 10
            zone           : us-central1-b
    
        # Optional compute configuration fields
    
            # cpu_platform           : Intel Skylake
            # preemptible_bursting   : False
            # compute_disk_type      : pd-standard
            # compute_disk_size_gb   : 20
            # compute_labels         :
            #   key1 : value1
            #   key2 : value2
            # compute_image_family   : custom-image
            # network_storage        :
            #   - server_ip: none
            #     remote_mount: <gcs bucket name>
            #     local_mount: /data
            #     fs_type: gcsfuse
            #     mount_options: file_mode=664,dir_mode=775,allow_other
            #
    
        # Optional network configuration fields
            # vpc_subnet                : my-subnet
    
        # Optional GPU configuration fields
            gpu_type               : nvidia-tesla-v100
            gpu_count              : 8
    
    #  [END cluster_yaml]

Next we will use gcloud to provision the cluster. Within the Cloud Shell, execute the following command to deploy a Slurm cluster using our updated configuration specifications:

    gcloud deployment-manager deployments create gcp-burst1 --config slurm-cluster.yaml

**Note:** If you're running this codelab in a new project you may be prompted to enable the Deployment Manager API with a message like this:

API [deploymentmanager.googleapis.com] not enabled on project

[956620689341]. Would you like to enable and retry? (y/N)?

If you get this prompt enter **Y** and press **enter** to continue.

<img src="img/257253d3f6d46d46.png" alt="257253d3f6d46d46.png"  width="624.00" />

The deployment takes several minutes to complete.  If everything goes according to plan without error, you should see four VM instances: a login node, a controller node, a compute image node corresponding to the **gpu-k80** partition and a compute image node corresponding to the **gpu-v100** partition.  To login into gcp-burst-login0 VM, you can either click on the **SSH** link under the Connect column or issue this command within the Cloud Shell

    gcloud compute ssh gcp-burst-login0 --zone=us-central1-b

The initial login into the gcp-burst cluster will trigger installation and configuration of Slurm. This process takes several minutes, so now is a good time to take another break.

<img src="img/66c9bc32da796e28.png" alt="66c9bc32da796e28.png"  width="624.00" />

Now that Slurm is installed, log out and log back into the cluster. You first will need to **CTRL+C** out of the process followed by the **exit** command. The Slurm banner should greet you upon successful login. If the Slurm **sinfo** command outputs the partitions as specified  in the **slurm-cluster.yaml** file (with 20 nodes for the **gpu-k80** partition and 10 nodes for the **gpu-v100** partition), our **gcp-burst** cluster has been successfully deployed. Note that the IPs for the four VM instances are within the 10.20.0.x block, as specified earlier in the **slurm-network** VPC subnet of the **gpc-burst** VPC network.


## Setting Up Cross-Project Firewall Rules
Duration: 03:00


### **Our Goal**

Once you have a second project with a second "gcp" Slurm cluster deployed through deployment manager, we can set up the networking in preparation for federating the on-prem and gcp clusters.

We must first **open ports** using Firewall Rules on both projects.

### **Firewall changes for on-prem cluster**

#### **`slurmctl ports:`**

We need to open ports on the on-prem VPC network so that the gcp-burst cluster can communicate with the on-prem cluster's slurmctld (tcp:6817) and slurmdbd (tcp:6819).

From the Google Cloud Console hamburger menu, navigate to [NETWORKING] **VPC network** → **Firewall** panel. Click on **+CREATE FIREWALL RULE** at the top of the page, fill in with following fields and click on **CREATE**:
**Name** | slurm-network
--- | ---
**Network** | on-prem
**Priority** | 1000
**Direction of traffic** | Ingress
**Action to match** | Allow
**Targets** | Specified target tags
**Target tags** | controller
**Source Filter** | IP ranges
**Source IP ranges** | 10.20.0.0/16
**Second source filter** | none
**Specified protocols and ports** | tcp:6817,6819

###  <img src="img/c3a63e32d75027a2.png" alt="c3a63e32d75027a2.png"  width="624.00" /> <img src="img/57dab612dbb2023e.png" alt="57dab612dbb2023e.png"  width="624.00" />

#### **`slurmd ports:`**

From the Google Cloud Console hamburger menu, navigate to [NETWORKING] **VPC network** → **Firewall** panel. Click on **+CREATE FIREWALL RULE** at the top of the page, fill in with following fields and click on **CREATE**:
**Name** | slurmd
--- | ---
**Network** | on-prem
**Priority** | 1000
**Direction of traffic** | Ingress
**Action to match** | Allow
**Targets** | Specified target tags
**Target tags** | compute
**Source Filter** | IP ranges
**Source IP ranges** | 10.20.0.0/16
**Second source filter** | none
**Specified protocols and ports** | tcp:6818

###  <img src="img/dbba76972138a3c9.png" alt="dbba76972138a3c9.png"  width="624.00" /> <img src="img/28a9f3d7b5be7042.png" alt="28a9f3d7b5be7042.png"  width="624.00" />

#### **`srun ports:`**

Optionally, if you require the ability to execute cross-cluster srun jobs, or cross-cluster interactive jobs then ports need to be opened for srun to be able to communicate with the slurmd's on the gcp cluster. Ports also need to be opened for the slurmd's to be able to talk back to the login nodes on the gcp cluster. During these operations srun opens several ephemeral ports for communications. It's recommended to define which ports srun can use when using a firewall. This is done by defining SrunPortRange=<IP Range> in both slurm.conf files.

SrunPortRange=60001-63000

From the Google Cloud Console hamburger menu, navigate to [NETWORKING] **VPC network** → **Firewall** panel. Click on **+CREATE FIREWALL RULE** at the top of the page, fill in with following fields and click on **CREATE**:
**Name** | srun
--- | ---
**Network** | on-prem
**Priority** | 1000
**Direction of traffic** | Ingress
**Action to match** | Allow
**Targets** | Specified target tags
**Target tags** | compute
**Source Filter** | IP ranges
**Source IP ranges** | 10.20.0.0/16
**Second source filter** | none
**Specified protocols and ports** | tcp: 60001-63000

###  <img src="img/f3def33717d1faaa.png" alt="f3def33717d1faaa.png"  width="624.00" /> <img src="img/af8f50d4299ec1be.png" alt="af8f50d4299ec1be.png"  width="624.00" />

### **Firewall changes for gcp-burst cluster**

#### **`slurmctl ports:`**

We need to open ports on the gcp-burst VPC network so that the gcp-burst cluster can communicate with the on-prem cluster's slurmctld (tcp:6817).

From the Google Cloud Console hamburger menu, navigate to [NETWORKING] **VPC network** → **Firewall** panel. Click on **+CREATE FIREWALL RULE** at the top of the page, fill in with following fields and click on **CREATE**:
**Name** | slurm-network
--- | ---
**Network** | gcp-burst
**Priority** | 1000
**Direction of traffic** | Ingress
**Action to match** | Allow
**Targets** | Specified target tags
**Target tags** | controller
**Source Filter** | IP ranges
**Source IP ranges** | 10.10.0.0/16
**Second source filter** | none
**Specified protocols and ports** | tcp: 6817

###  <img src="img/d5f2e27cf679c51e.png" alt="d5f2e27cf679c51e.png"  width="624.00" /> <img src="img/7ff06c59e33a539a.png" alt="7ff06c59e33a539a.png"  width="624.00" />

#### **`slurmd ports:`**

From the Google Cloud Console hamburger menu, navigate to [NETWORKING] **VPC network** → **Firewall** panel. Click on **+CREATE FIREWALL RULE** at the top of the page, fill in with following fields and click on **CREATE**:
**Name** | slurmd
--- | ---
**Network** | gcp-burst
**Priority** | 1000
**Direction of traffic** | Ingress
**Action to match** | Allow
**Targets** | Specified target tags
**Target tags** | compute
**Source Filter** | IP ranges
**Source IP ranges** | 10.20.0.0/16
**Second source filter** | none
**Specified protocols and ports** | tcp:6818

###  <img src="img/63e4bc04b9521dd0.png" alt="63e4bc04b9521dd0.png"  width="624.00" /> <img src="img/6b0f2e64882b8f06.png" alt="6b0f2e64882b8f06.png"  width="624.00" />

#### **`srun ports:`**

Optionally, if you require the ability to execute cross-cluster srun jobs, or cross-cluster interactive jobs then ports need to be opened for srun to be able to communicate with the slurmd's on the gcp cluster. Ports also need to be opened for the slurmd's to be able to talk back to the login nodes on the gcp cluster. During these operations srun opens several ephemeral ports for communications. It's recommended to define which ports srun can use when using a firewall. This is done by defining SrunPortRange=<IP Range> in both slurm.conf files.

SrunPortRange=60001-63000

From the Google Cloud Console hamburger menu, navigate to [NETWORKING] **VPC network** → **Firewall** panel. Click on **+CREATE FIREWALL RULE** at the top of the page, fill in with following fields and click on **CREATE**:
**Name** | srun
--- | ---
**Network** | gcp-burst
**Priority** | 1000
**Direction of traffic** | Ingress
**Action to match** | Allow
**Targets** | Specified target tags
**Target tags** | compute
**Source Filter** | IP ranges
**Source IP ranges** | 10.20.0.0/16
**Second source filter** | none
**Specified protocols and ports** | tcp: 60001-63000

###  <img src="img/e06d16318d5fea42.png" alt="e06d16318d5fea42.png"  width="624.00" /> <img src="img/90eb38be0435033e.png" alt="90eb38be0435033e.png"  width="624.00" />

When all is said and done, here are the summaries of the firewall rules for the on-prem and gcp-burst clusters:

<img src="img/1e20ce3fc0d8c59f.png" alt="1e20ce3fc0d8c59f.png"  width="624.00" /> <img src="img/aab21811fd4ebf7e.png" alt="aab21811fd4ebf7e.png"  width="624.00" />


## Setting Up Cross-Project VPN
Duration: 03:00


The **on-prem** and **gcp-burst** clusters are in their own separate networks. We will set up a VPN to connect the two clusters.  For this set up, it's simplest if you have two Google Cloud Consoles open, one that has the **on-prem** project selected and the other with the **gcp-burst** project selected. The reason why is because some of the settings, such as **Remote peer IP address** and **IKE pre-shared key**, depend on the on-demand assigned/generated values.

Let's create the VPN for the **on-prem** network first (make sure that the Cloud Console has the on-prem project selected). From the Google Cloud Console hamburger menu, navigate to the [NETWORKING] **Hybrid Connectivity** → **VPN** panel.  Click on **Create VPN connection**, select **Classic VPN,** click on **CONTINUE** and fill in the field with the following settings below.  Don't click on the **CREATE** button yet because we need information from the corresponding **gcp-burst** VPN.
**Name** | on-prem-vpn
--- | ---
**Network** | on-prem
**Region** | us-central1
**IP address** | Create IP address → Name: on-prem-vpn-ip → RESERVE(This reserved IP address will be the value for the **gcp-burst** VPN **Remote peer IP address** field)
**Tunnels: Name** | on-prem-tunnel-gcp-burst
**Remote peer IP address** | (Leave blank for now. You will update this field with the IP address that will be reserved for the gcp-burst VPN)
**IKE version** | IKEv2
**IKE pre-shared key** | Generate and copy
**Routing options: Route-based: Remote network IP range** | 10.20.0.0/16

Shown below are the screenshots for the **on-prem** settings (**I've obscured the IKE pre-shared key**)

<img src="img/bc5d19b07217ffd1.png" alt="bc5d19b07217ffd1.png"  width="624.00" /> <img src="img/fb4fec0f3336a49.png" alt="fb4fec0f3336a49.png"  width="624.00" />

Next, let's set up the VPN tunnel on the **gcp-burst** cluster (be sure that the Cloud Console has the gcp-burst project selected). From the Google Cloud Console hamburger menu, navigate to the [NETWORKING] **Hybrid Connectivity** → **VPN** panel.  Click on **Create VPN connection**, select **Classic VPN,** click on **CONTINUE** and fill in the field with the following settings and click on **Create**:
**Name** | gcp-burst-vpn
--- | ---
**Network** | gcp-burst
**Region** | us-central1
**IP address** | Create IP address → Name: gcp-burst-vpn-ip → RESERVE(This reserved IP address will be the value for the **on-prem** VPN **Remote peer IP address** field)
**Tunnels: Name** | gcp-burst-tunnel-on-prem
**Remote peer IP address** | (enter the IP address that was reserved for the on-prem-vpn)
**IKE version** | IKEv2
**IKE pre-shared key** | Generate and copy
**Routing options: Route-based: Remote network IP range** | 10.10.0.0/16

Shown below are the screenshots for the **gcp-burst** VPN settings (I have obscured the **IKE pre-shared key**).

<img src="img/c8c9a542a9b3df3a.png" alt="c8c9a542a9b3df3a.png"  width="624.00" /> <img src="img/a68c1ac92df46f55.png" alt="a68c1ac92df46f55.png"  width="624.00" />

Now that we have the IP address for the **gcp-burst** VPN reserved, copy this and enter into the **Remote peer IP address** field for the **on-prem** VPN settings.  Lastly, the two VPN endpoints need to have the same IKE pre-shared key.  You can either copy the one from the **gcp-burst** endpoint and paste it into the on-prem VPN settings or vice versa.  Finally, click on the **CREATE** button on both Google Cloud Consoles.  If everything goes according to plan, you should see connectivity established at both endpoints.

<img src="img/a8f058ec9b9300e3.png" alt="a8f058ec9b9300e3.png"  width="624.00" /> <img src="img/28dd876dbc239b22.png" alt="28dd876dbc239b22.png"  width="624.00" />


## Configuring Slurm Cluster Accounting
Duration: 09:00


In order to federate between the two clusters we need to configure Slurm cluster accounting on our gcp cluster to report to our on-prem Slurm controller.

For more information about Slurm accounting, see the [Slurm Accounting](https://slurm.schedmd.com/accounting.html) page.

First, we must set both cluster's slurm.conf files to point to the on-prem's controller as the "Accounting Storage Host".

**Log in** to both cluster's login node. In the on-prem cluster **execute** the following command to retrieve the on-prem's controller node IP.

    sudo -i sacctmgr show clusters format=cluster,controlhost,controlport

You should see the following output where **10.10.0.5** is the IP of our on-prem's controller:

      Cluster     ControlHost  ControlPort 
    ---------- --------------- ------------ 
       on-prem       10.10.0.5         6820

Copy this IP address to your clipboard for use in our next step.

On the **gcp-burst** cluster's login node, **open** the slurm.conf using your preferred text editor:

    sudo vi /apps/slurm/current/etc/slurm.conf

**Edit** the AccountStorageHost entry to match:

    AccountingStorageHost=<IP of on-prem's controller instance>

> aside negative
> **Caution**.  If you were paying attention during the **Setting Up Cross-Project Firewall Rules** section, you will recall that we identified the slurmctl port as 6817 but the above command indicates that our **on-prem** Slurm cluster is using port 6820. What is up with that?
> What I believe happened is that the default settings for the Slurm cluster had changed since the introduction of the  [Building Federated HPC Clusters with Slurm](https://codelabs.developers.google.com/codelabs/hpc-slurm-federated-on-gcp) codelab. Recall that we were leveraging the automations scripts from the  [Deploy an Auto-Scaling HPC Cluster with Slurm](https://codelabs.developers.google.com/codelabs/hpc-slurm-on-gcp/) codelab and nowhere did we explicitly configure the slurmctl port. We have two paths in front of us: (1) go back and configure the firewall rules with the new port or (2) update slurm.conf to use the ports specified in the previously configured firewall rules.
> It is simpler to walk the latter path.  Update the slurm.conf on both the **on-prem** and **gcp-burst** clusters with the following parameters
> `SlurmctldPort=6817`
> `SlurmdPort=6818`
> These parameters will be picked up when we restart the Slurm controller daemon later.

Now that we have the gcp cluster pointing to the on-prem cluster we can add the cluster to the on-prem's Slurm DB, and set up user accounts.

On the on-prem cluster's login node, **execute the following command** to add the gcp cluster to the on-prem:

    sudo -i sacctmgr add cluster gcp-burst

Next, run these commands with your cluster name and user to configure your account on the gcp cluster:

    sudo -i sacctmgr add account=default cluster=gcp
    sudo -i sacctmgr add user <user> account=default cluster=gcp

Finally, we must restart the Slurm controller daemon on both clusters. **SSH** into the controller node of both clusters. You may do this either by using the "SSH" button next to the controller node in Google Cloud Console, or by setting up SSH keys from the login1 node by taking advantage of the common /home folder:

    ssh-keygen -q -f ~/.ssh/id_rsa -N ""
    cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
    ssh on-prem-controller

Once logged into both controller nodes, **execute** the following command on both nodes to restart slurmctld:

    sudo systemctl restart slurmctld

Within a few minutes, the two clusters will report in and will appear in the Slurm accounting. Execute the following command to list clusters:

sudo -i sacctmgr show clusters format=cluster,controlhost,controlport

The output should contain both clusters and report IPs for each:

      Cluster     ControlHost  ControlPort 
    ---------- --------------- ------------ 
     gcp-burst       10.20.0.4         6817 
       on-prem       10.10.0.5         6817

If the IPs don't populate immediately please allow a few minutes for the clusters to report in. If you believe there's an issue, please review the [Slurm Troubleshooting Guide](https://slurm.schedmd.com/troubleshoot.html).

Log out of the controller nodes, and back into the login nodes to continue the codelab.


## Creating Slurm Federation of Clusters
Duration: 05:00


Now that we have our Slurm cluster accounting configured, we can create the Slurm federation.

For more information about Slurm federation, see the [Slurm Federation](https://slurm.schedmd.com/federation.html) page.

We will use the name "cloudburst" for our federation. On the on-prem gcp's login1 node **execute** the following command:

    sudo -i sacctmgr add federation starfleet clusters=on-prem,gcp-burst

You've now created a Slurm federation capable of bursting to Google Cloud Platform! Let's view the configration to verify it was created correctly:

    sacctmgr show federation

The output should appear as:

    Federation    Cluster ID             Features     FedState 
    ---------- ---------- -- -------------------- ------------ 
     starfleet  gcp-burst  2                            ACTIVE 
     starfleet    on-prem  1                            ACTIVE 


## Cloud Bursting from on-prem cluster to gcp-burst cluster



Now that we have a federation set up between our on-prem and gcp clusters we can submit jobs that are federated (distributed) across clusters according to whichever cluster is capable of responding to the job request fastest.

### **Checking Cluster Status**

To see the resources available in the federation, run **sinfo** with the --federation flag:

    sinfo --federation

The output should appear as:

    PARTITION CLUSTER  AVAIL  TIMELIMIT  NODES  STATE NODELIST
    test*     on-prem     up   infinite     10  idle~ on-prem-compute-0-[0-9]
    gpu-k80*  gcp-burs    up   infinite     20  idle~ gcp-burst-compute-0-[0-19]
    gpu-v100  gcp-burs    up   infinite     10  idle~ gcp-burst-compute-1-[0-9]
    smp       on-prem     up   infinite     42  idle~ on-prem-compute-1-[0-41]

Let's also check the queues on both of our clusters using **squeue**:

    squeue --federation

You should see the following output:

    JOBID PARTITION     NAME     USER ST       TIME  NODES NODELIST(REASON)

We can see that the queue is empty, and that all nodes are listed as "idle" or "idle~". The "idle" state signifies that the node up online and idle, ready to have jobs allocated to it. The "idle~" state signifies that the node is offline and does not yet exist in the cloud, but that it could be created if necessary to meet demand.

Now that our cluster federation is ready, let's go ahead and submit a job.

### **Submitting Federated Jobs**

First, we can submit a job specifically to a cluster we chose using the -M flag in **sbatch**:

    sbatch -M gcp-burst hostname_batch

> aside negative
> **Note:** If you don't have the hostname_batch script, please refer back to step 6 of the " [Deploy an Auto-Scaling HPC Cluster with Slurm](https://codelabs.developers.google.com/codelabs/hpc-slurm-on-gcp/#5)" codelab. You can find a copy of the script there.

You can now run **squeue** to check the status of the clusters. The -M flag works in squeue as well as sbatch and sinfo. Let's run squeue with the --federation flag, and a few other options:

    [kimberlyyellow_gmail_com@on-prem-login0 ~]$ squeue --federation -O jobid,username,timeused,cluster,numnodes,nodelist
    JOBID               USER                TIME                CLUSTER             NODES               NODELIST            
    134217733           kimberlyyellow_gmail0:28                gcp-burst           2                   gcp-burst-compute-0-
    [kimberlyyellow_gmail_com@on-prem-login0 ~]$ 


You'll notice that the gcp-burst cluster was allocated this job, and there are two nodes allocated to the job.

Now let's check **sinfo** to verify that the gcp cluster is spinning up the necessary nodes to complete the job.

    kimberlyyellow_gmail_com@on-prem-login0 ~]$ sinfo --federation
    PARTITION CLUSTER  AVAIL  TIMELIMIT  NODES  STATE NODELIST
    gpu-k80*  gcp-burs    up   infinite      2   mix# gcp-burst-compute-0-[0-1]
    test*     on-prem     up   infinite     10  idle~ on-prem-compute-0-[0-9]
    gpu-k80*  gcp-burs    up   infinite     18  idle~ gcp-burst-compute-0-[2-19]
    gpu-v100  gcp-burs    up   infinite     10  idle~ gcp-burst-compute-1-[0-9]
    smp       on-prem     up   infinite     42  idle~ on-prem-compute-1-[0-41]
    [kimberlyyellow_gmail_com@on-prem-login0 ~]$ 

We can also allow Slurm's federation to allocate the jobs to whichever cluster is able to respond the fastest by simply submitting the job to the federation we're now part of:

    sbatch hostname_batch

Once the job is submitted check **squeue** to see where the job was allocated:

    [kimberlyyellow_gmail_com@on-prem-login0 ~]$ squeue --federation -O jobid,username,timeused,cluster,numnodes,nodelist
    JOBID               USER                TIME                CLUSTER             NODES               NODELIST            
    67108868            kimberlyyellow_gmail0:12                gcp-burst           2                   gcp-burst-compute-0-
    134217733           kimberlyyellow_gmail0:00                gcp-burst           2                                      

Submit the job once or twice more to see where Slurm places the job. It may allocate jobs to the on-prem cluster the first or second time, but then it will begin distributing the jobs evenly.

Congratulations, you've created a federated Slurm cluster out of two independent Slurm Clusters, and federated jobs between two auto-scaling clusters! You can use this technique in a variety of environments, including between an existing on-premise cluster and a Google Cloud auto-scaled cluster. Furthermore, you could have multiple clusters with each tailored to any given workload and resource profile in a single federation. You could then use [Slurm accounting](https://slurm.schedmd.com/accounting.html) to assign users to cloud-specific partitions, or you might use cluster specification on a per-job basis. All this while transparently allowing users to continue submitting jobs through the same workflow they're used to, logging into the same Slurm cluster.

Try testing some more interesting code like a [Prime Number Generator](https://github.com/GoogleCloudPlatform/deploymentmanager-samples/blob/master/examples/v2/htcondor/applications/primes.c), the [OSU MPI Benchmarks](http://mvapich.cse.ohio-state.edu/benchmarks/), or your own code! To learn more about how to customize the code for your usage, and how to run your workloads most affordably, contact the Google Cloud team today through [Google Cloud's High Performance Computing Solutions website](https://cloud.google.com/solutions/hpc/)!


## Wrapping up and reflecting




