# Proxmox Virtual Environment (PVE): Initial Setup

Proxmox Virtual Environment is an open source virtualization platform
with built in, masterless clustering, replication, migration, software
defined networking and storage, and much more.

This document walks through the initial installation and configuration
of the system to get it into a working state, give you a solid
foundation to build from, and should cover enough of the most common
options to fit clients' needs.

### Documentation and Support

- [Documentation](https://pve.proxmox.com/pve-docs) for the latest
  version of Proxmox is available online.
- Documentation for the installed version is included on the server and
  available from a button at the top-right of the web interface
- The Proxmox community has a robust and helpful
  [forum](https://forum.proxmox.com/)
- Proxmox offers [paid
  support](https://proxmox.com/en/products/proxmox-virtual-environment/pricing)
  and implementation services as well.

## Before You Install

Like any other project, planning ahead will go a long way toward a
smooth implementation. Most of the considerations for Proxmox will be
the same as any other virtualization system, so this document will
mostly mention those bits specific to Proxmox.

### Clustering

Proxmox has built-in masterless clustering. This means you can have
multiple VM host machines (nodes) clustered together, and any of the
nodes in that cluster will provide full management access to any other
node in the cluster. This eliminates a single point of failure. It also
allows things like migrating virtual machines between nodes or storage
locations, high availability failover of VMs, and storage replication.

If clustering is going to be used, here are a couple of things to take
into consideration:

- Cluster network
  - It is highly recommended to implement a separate network
    specifically for clustering communications. The clustering protocol
    used by Proxmox does not need much bandwidth, but is highly
    sensitive to latency and jitter.
  - At a minimum, never use the same network for storage access and
    clustering, except possibly as a failover.

### Storage

Proxmox supports many different types of local, remote, shared, and
distributed storage. Which type to use will depend on the client's needs
and the specific task the storage will be used for. This document will
cover using ZFS as local storage, and setting up a Windows (CIFS)
network share which can be used as simple shared storage.

As a general rule, the server hardware should be setup with a single
drive (or RAID 1 if you prefer) as the boot drive, and a separate drive,
device, or set of drives for virtual machine storage. The example in
this document will be using a local ZFS pool for VM storage. With ZFS,
the operating system should have full direct access to the physical
drives. So any hardware RAID controllers should be in IT or JBOD mode
for these drives. If the firmware of the controller doesn't support
drive passthrough, then you can set up each drive as a standalone (or
RAID 0) device. If there are no other options, ZFS will still work when
a RAID array is setup using the hardware controller, but some of the
data protections of ZFS will be reduced.

### Network Design

At the very simplest, Proxmox can use a single network for all
functions. It is probably a good idea to at least create separate
management and data networks. If you are going to create a cluster, I
would highly recommend setting up a separate network specifically for
cluster communication. Depending on the client's needs, you may also
setup multiple data networks (data and voice vlans for example). Knowing
how you will need to configure networking ahead of time always
simplifies things!

## Installation

The actual [installation of
Proxmox](https://phoenixnap.com/kb/install-proxmox) is very simple, and
not really any different from installing any other operating system.

- [Download the iso](https://proxmox.com/en/downloads) and prepare the
  installation media
- Boot from the installation media
- The installer will ask a few questions about language, location, time
  zone, etc.
- It will ask about which drive to install to. I would **only**
  configure the OS drive, and not try to do any configuration of other
  storage yet.
- Set a network address
- Create a root password.
- Once installation is complete, the system will reboot and from that
  point on, most configuration can be handled via the web interface

## First Access

Once installation is complete, access the Proxmox web interface at
<https://ip.of.the.server:8006>. Continue past the warning about the
self signed certificate, and sign in using the root account credentials
you created during the installation.

### Updates

First things first, lets get the server up to date. If paid support was
purchased, now is the time to enter the license information. From the
web interface, highlight the node in the left sidebar, then choose
"Subscription" then "Upload Subscription Key". Simple enough!

If a support license was **not** purchased, we need to point the server
to the free repositories before checking for and installing updates.

Again, we will start in the web interface, highlight the server in the
left sidebar, then go to "Updates -\> Repositories" You should see a
list of the configured repositories similar to the image below.

![Proxmox repositories
screen](/img/screenshot_2025-02-09_125057.png)

Click the "Add" button, and choose "No-Subscription" from the dropdown
list. Then click "Add" again. At this point you should see both the
enterprise and no-subscription repositories listed. Highlight the
enterprise repository and click the "Disable" button.

Now we are all set! Go to "Updates" from the sidebar, click "Refresh"
and the server will check for, then present a list of available updates.
Click the "Upgrade" button and a separate window will open and all
available updates will be installed. When updates are complete, you can
either enter the "exit" command then close the window, or simply close
the window.

If there was a newer kernel included in the updates, I would recommend
rebooting the system to apply the updated kernel. (The reboot button
should be top-right of the window.)

### Storage

By default, a new Proxmox node will have 2 storage locations created:

- **"local"** - Is actually just a path on the root drive: `/var/lib/vz`
    - This path is configured to store backups, iso images and Linux container templates.
- **"local-lvm"**  - This is a Logical Volume Manager partition comprising roughly half the available space on the root drive.
    - This path is configured to store VM disks and Linux Container Volumes.

On a stand alone node, these storage locations should be more than
adequate for running and managing a handful of VMs. Even with a stand
alone node though, it is recommended to use separate storage for VM and
other data storage. This setup will enable you to reinstall the Proxmox
OS without losing any of the virtual machine or other data in the
process.

#### ZFS Setup

In this simple example, we will use the Proxmox web interface to setup a
3 drive ZFS RAID array for VM storage. Let's start by looking at
information about all of the disks installed on the server. Highlight
the node in the left sidebar, then choose "Disks" You should see
information similar to the image below:

![Proxmox disk information
screen](/img/screenshot_2025-02-09_131733.png)

In this example the first disk listed (`/dev/nvme0n1`) is the OS drive,
and the other three drives are the ones we will be using to setup our
ZFS pool on. Since these were previously used drives placed into this
test system they already have partitions and data on them. So first we
need to wipe the drives and initialize them to make them available for
use in our pool.

Highlight one of the drives [^1] and click "Wipe Disk". Repeat the same
process for each drive you will be using in the ZFS pool. Each drive
should now appear with no partitions listed, and "no" in the Usage
column of the drive table. [^2]

If the drives are brand new out of the box, you likely will just need to
highlight the drive and click the "Initialize Disk with GPT" button.

Now that all of the drives are clear, go to "Disks → ZFS". There should
be nothing in this list right now. Click on "Create: ZFS" and you should
see a dialog similar to the image below.

![Proxmox ZFS pool creation
dialog](/img/screenshot_2025-02-09_133333.png)

- Enter a name for the ZFS pool. In this example, we will enter
  `ZFS-Data`
- Select the drives to be used by placing a checkmark in the box next to
  each listing.
- Choose the RAID level.
  - This is a big topic, and beyond the scope of this document, but generally speaking, if you have 6 or fewer drives, you probably want to choose one of the RAIDZ options. If you have a higher number of drives, you probably want one of the "dRAID" options.
   - There are as many ways of configuring drives for ZFS as there are drives. For more complex setups, you will need to use the command line for setup. The ProxMox GUI only supports some basic options.
   - RAIDZ - Similar to RAID5 - uses one drive for parity
   - RAIDZ2 - Uses 2 drives for parity
   - RAIDZ3 - Uses 3 drives for parity
   - dRAID levels - distributes the parity information across all available drives. the higher levels allow for a higher number of drives to fail simultaneously without data loss.

We will choose `ZRAID` for this example. We will not worry about
defining the "order" for the drives. This will be automatically
determined in this case. [^3]

When you are happy with the settings, Click "Create" and the ZFS pool
will be created and should be visible in the left sidebar: ![Proxmox
storage
sidebar](/img/screenshot_2025-02-09_135004.png)

#### Windows (CIFS) Share

A Windows network share likely doesn't have the performance you would
want to be able to run virtual machines off of, but they sure can be a
handy place to store backups or iso files. Let's get one added! [^4]

- In the left sidebar, highlight "Datacenter" then choose "Storage". You
  should see a list of the existing storage locations configured so far.
- Click "Add → SMB/CIFS" and a dialog window will open up: ![Proxmox SMB
  connect
  dialog](/img/screenshot_2025-02-09_141203.png)
- Fill out the fields as needed:
  - **ID**  = This can be anything, it will be the name the storage location is listed as in Proxmox.
  - **Nodes**  = Whether this storage should be configured on all nodes in the cluster, or only specific ones.
  - **Server**  = IP or host name of the server the share is located on.
  - **Enable**  = Should the storage be enabled?
  - **Username**  = Username used for authentication, if needed.
  - **Content**  = What types of data are allowed to be stored in this location. Highlight the types you want allowed. In this example we will choose ISO Image, VZDump backup file, and Snippets
  - **Password**  = The password used for authentication, if needed.
  - **Domain**  = The domain of the credentials used for authentication, if needed.
  - **Share**  = If the above info is correct, this should be a drop-down list of available network shares on the server.
  - **Subdirectory**  = Path below the specified share, if needed.
  - Backup Retention Tab:
     - You can define how many backups to keep in two places. Either defined here, at the storage level, or in the configuration of the backup itself. In this example, we will leave the storage settings at default (Keep all backups) so there is no additional configuration required here.

When everything looks right, Click "Add" and the storage will be added
to the list in the storage window and be visible in the left sidebar as
well.

### Networking

The test server which was set up for this documentation only has a
single port on the NIC, so all screenshots will be coming from a live
server with multiple ethernet ports available. Some screenshots may not
exactly match the descriptions.

Proxmox supports both standard Linux Bridges, Bonds and VLANs, and Open Virtual Switch (OVS) Bridges, Bonds and VLANs. This document will onl cover the default Linux networking setup. 

Also, Proxmox includes Software Defined Network capabilities which will
also not be covered here. [^5]

#### Overview

After installation, Proxmox is configured with a single Linux Bridge,
with one interface assigned to it and IP information for that interface
setup. Essentially this is the management network. Since we don't want
to do **everything** on that management network, we will create a new
Bridge and separate VLANs for data, and create another network we will
use later for clustering. When we are done, the result should look
similar to the following:

![Proxmox network information
screen](/img/screenshot_2025-02-09_160457.png)

Start by highlighting the node in the left sidebar, then choosing
"System -\> Network". You should see a list of the physical network
ports, and the default bridge and it's IP configuration.

Be sure you have all of your network switches and routers configured as
needed with appropriate VLANs and other settings. It may be a good idea
to perform this network configuration on a currently unused network
port, allowing you to remain connected to the default [^6] network
connection while testing all of the new configurations. That is what
will be shown in this document. [^7]

#### Linux Bridge

The simplest definition of a Linux bridge is a virtual device which can
forward traffic between different network segments. Both physical
devices (Ethernet ports, for example) and virtual devices (VLANs) can be
attached to a bridge.

So to get started, click on "Create -\> Linux Bridge". A dialog window
similar to the image below will open up.

![Proxmox bridge create
dialog](/img/screenshot_2025-02-09_152247.png)

- **Name** = Can be anything, but in this case we will use the default
  naming (`vmbr2`).
- **Autostart** = Whether the bridge should be activated when the host
  system boots up.
- **IPv4/CIDR** = IPv4 address to assign to the bridge. Since we will be
  creating separate VLANs to attach to this bridge, we do not need any
  direct IP accessibility to the bridge itself. We also will not need to
  enter anything into the \*\* gateway\*\* or any of the **IPv6**
  fields.
- **Bridge Ports** = List of physical network port(s) to connect to the
  bridge, In this example we will add `eno3` as the only port.
- **VLAN Aware** = Whether the bridge knows about VLANs or not. We will
  make sure this is checked in this case.
- **Comment** = Useful for noting the purpose of the bridge.
- And when "Advanced" is checked, you can also set the MTU and allowed
  VLANs for the bridge. We left these at defaults in this case.

When everything looks correct, click "Create" and you should see the new
bridge listed as well as some information in the lower part of the
window about the pending changes.

#### VLANs

Now we will create VLANs and attach them to the bridge. Click "Create
-\> Linux VLAN" and a dialog windows will open.

![Proxmox VLAN create
dialog](/img/screenshot_2025-02-09_153825.png)

- **Name** = Can be anything **but** if you define the name similar to
  `vmbrx.yy`[^8], some of the other fields will be filled out for you.
  So in this example, it will be named `vmbr2.15` since we are adding
  VLAN 15 to vmbr2
- **Autostart** - Whether the VLAN should be activated when the system
  boots.
- **IPv4 Fields** = IP Addressing information. In this example, we set
  the IP Address, but left the gateway blank [^9], since there is
  already a default gateway defined on the default/management bridge.
- **VLAN raw device** = Automatically filled in when we named the VLAN.
  If set manually, should be the bridge or port you want to connect the
  VLAN to.
- **VLAN Tag** = The VLAN ID number. Also filled in automatically based
  on naming.
- **IPv6 Fields** = IP Addressing information.

When everything looks correct, click on "Create" and you should see the
new VLAN listed, and the "Pending Changes" section will be updated.

You can repeat the same process for any additional VLANs or bridges you
may need to create to suit the client's environment. Go ahead and create
a new bridge, assign a network port to it, and assign IP Addressing to
be used later for the clustering network.

When everything is configured to your liking, click "Apply
Configuration" for the new network settings to be applied, then test
everything and make changes if needed.

### Clustering

Clustering in Proxmox is essentially a group of several physical host
servers (nodes) which share management and configuration. Proxmox uses
masterless clustering. This means you can connect to **any** node in the
cluster and have full management access to any other node in the
cluster.

Clustering means you only have one place to configure users and groups
and permissions. You can also assign shared shared storage to all (or
specific) nodes in a cluster, shared firewalling, Software defined
networking, backups, migrate virtual machines between different nodes
and storage locations, set up storage replication and high
availability...

A couple of quick notes on clustering:

- It is **highly** recommended to use a separate network for clustering
  traffic. This traffic is not bandwidth intensive, but it sensitive to
  latency and jitter. At the minimum, do not put the cluster traffic on
  the same network as storage, except as a failover.
- Clustering in Proxmox uses a quorum to determine the health of the
  cluster before performing an action. Essentially, each host in a
  cluster gets one vote on whether it can see all of the other hosts. As
  long as half the available votes +1 are cast, action may proceed. So
  for example, if there are 5 hosts in a cluster, a minimum of 3 hosts
  must be available. Search for "cluster manager" in the documentation
  for full details.
- Ideally, there should be an odd number of nodes in a cluster. If there
  is going to be an even number of nodes, it is highly recommended to
  use an external device to cast a "tie-breaker" vote. Search for
  "external vote support" in the documentation for details.

Before creating or joining a cluster, you need to make sure the node has
its' final host name and management IP address configured properly. You
will **not** be able to change these after the cluster has been
created/joined.

#### Create a New Cluster

Creating a new Cluster is very simple.

- Start by highlighting "Datacenter" in the left sidebar, then select
  "Cluster"
- Click "Create Cluster" and a new dialog windows will open:\
  ![Proxmox create cluster
  dialog](/img/screenshot_2025-02-09_172242.png)
- Give the cluster a name
- Choose a network link or links for the cluster to communicate over. If
  you add muliple links, lower numbered links get highest priority and
  communication will fail over to the next highest numbered link in case
  of failure.

When everything looks good, click "Create" and you should see your new
cluster listed with one node connected to it.

#### Joining an Existing Cluster

Joining an existing cluster is also very simple. First we need to get
the join information from one of the nodes already in the cluster.

- On the node already a member of the cluster, go to Datacenter -\>
  Cluster, then click "Join Information" A dialog windows will open\
  ![Proxmox join cluster
  dialog](/img/screenshot_2025-02-09_173511.png)
- Click the "Copy Information" button to place the join string into the
  clipboard.
- Now on the node you need to join to the cluster, Start at Datacenter
  -\> Cluster and click "Join Cluster" a dialog windows will open\
  ![Proxmox cluster join
  dialog](/img/screenshot_2025-02-09_173806.png)
- Paste in the join string you copied from the other node and click
  "Join"
- You will be prompted for the `root` account password of the node which
  is already part of the cluster, and then the new node will be joined
  to the cluster.
- You should be able to see all other nodes in the cluster in the left
  sidebar now.

#### Hosts File

If you are using DNS names to identify your nodes, it is a good idea to
create entries for all of the nodes [^10] in each node's ''hosts'' file.

- Highlight the Node in the left sidebar, then go to System -\> Hosts
- Make any edits needed in the main window, then click "Save" when you
  are done.\
  ![proxmox hosts file
  edit](/img/screenshot_2025-03-24_181029.png)

### Users, Realms, and Roles

Proxmox has a robust and detailed permissions system. We will barely
scratch the surface in this document. I would **highly** recommend
referring to the documentation for complete descriptions of the
permissions systems in Proxmox.

#### Realms

A realm is basically just a source for user authentication. By default,
Proxmox has two realms available:

- **Linux PAM Standard Authentication**
  - This is user authentication at the underlying operating system
    level. By default, the `root` account is the only member of this
    realm. If you prefer, you can connect to the Proxmox node via SSH or
    the console and create new users on the OS, assign them to required
    groups and those users will be able to use those credentials to
    manage Proxmox. This is usually only used in locations which already
    have a large Linux infrastructure, and allows Proxmox to tie into
    that existing infrastructure.
- **Proxmox VE Authentication**
  - This is user authentication specific to Proxmox. These users may (or
    may not) have permissions to almost any portion of Proxmox, but will
    not have access to any of the underlying operating system, so will
    not be able to connect via SSH or install updates, for example. This
    is more or less the default way to add local users to Proxmox. This
    realm also has built-in support for Time-based One Time Passwords
    (TOTP), WebAuthN and YubiKeys as MFA methods as well as being able
    to create a set of recovery keys to use in case of issues with any
    of the MFA methods.

Proxmox ships with support for Active Directory, Open LDAP, and OpenID
Connect as additional realms.

Note that any/all of these realms can be used side by side, though it is
helpful if you try to avoid duplicate usernames in different realms.

#### Roles

Roles are essentially a defined set of permissions. You can see the
included roles at Datacenter -\> Permissions -\> Roles. If needed, you
can create new roles or edit the permissions of existing roles. This
isn't often needed, but the ability is there.

#### Groups

While roles are sets of permissions, groups are sets of users. We can
then assign roles to that set of users.

#### Creating a new Admin User

We are going to create a new user in the Proxmox VE Realm, with full
administrator permissions.

- First we need to create a new group, so start at Datacenter -\> Groups
- Click "Create" and a new Dialog will open\
  ![Proxmox New Group
  Dialog](/img/screenshot_2025-02-10_155630.png)
- Name this new Group "Admins", enter a comment if you would like, and
  click "Create" The new group should appear in the list.
- Now let's set the permissions this group will have. Go To Datacenter
  -\> Permissions and click "Add -\> Group Permission" a new dialog will
  open\
  ![Proxmox new permission
  dialog](/img/screenshot_2025-02-10_160134.png)
  - **Path** = We will set this as ''/'' this is the
  root path. Click the "help" button to open the documentation to the
  permissions and paths section.
  - **Group** = We will choose the "Admins" group we just created.
  - **Role** = We will choose Administrator.
  - **Propagate** = Leave this checked. This determines whether the
    group permissions will apply to all lower levels of the path chosen.
- When you are happy with the settings, click the "Add" button, and the
  new permission will show in the list.
- Now we are ready to create the new user. Go to Datacenter -\> Users.
  Click the "Add button and a new dialog will open\
  ![Proxmox new user
  dialog](/img/screenshot_2025-02-10_161111.png) \* Fill out the fields, be sure to
  choose "Proxmox VE Authentication" as the realm and out new "Admins"
  group. If this user will only need temporary access, you can set a
  date for the account to expire as well. When you are happy with your
  entries, click "Add" and the new user will show in the list.

#### TOTP

- Now let's set up MFA for this user. Go to Datacenter -\> Permissions
  -\> Two Factor, then click "Add -\> TOTP". A new dialog will open\
  ![Proxmox new totp
  dialog](/img/screenshot_2025-02-10_161930.png)
- Select our new user from the dropdown, and enter a description.
- Use any compliant TOTP app (Google Auth, Microsoft Authenticator,
  Authy, etc.) to scan the provided QR Code, or paste the secret into
  the app as needed.
- You (or the user) will need to enter the correct code into the "Verify
  Code" field before you can click the "Add" button.
- We can also create a set of recovery keys to be used in case the
  primary MFA method fails for whatever reason. So we will start at
  "Datacenter -\> Permissions -\> Two Factor" Then click "Add -\>
  Recovery Keys" and a new dialog will open
  - Choose a user from the drop down, and click "Add" and a list of 10
    recovery keys will be displayed. Be certain to record these keys
    elsewhere, as this is the **only** time they will be displayed.

#### Active Directory Realm

If there will be several clusters inside the same Active Directory
reach, it may be advantageous to have users authenticate with their
domain credentials. Proxmox ships with an AD Realm. Proxmox also
includes an optional sync utility, which can be used to regularly
synchronize specific AD users or groups and automatically create the
associated users in Proxmox. We will not be setting up the directory
sync in this example. [^11]

So we will set up the AD realm, but still manually create the users in
Proxmox who will authenticate using that realm.

- Start at "Datacenter -\> Permissions -\> Realms". You should see the
  two default realms listed.
- Click "Add -\> Active Directory Server" A new dialog will open.\
  ![Proxmox AD Realm
  dialog](/img/screenshot_2025-02-11_110612.png)
  - **Realm** - The name Proxmox will use to identify the realm. Cannot
    include any spaces. [^12]
  - **Domain** - The domain name you are connecting to
  - **Server and Fallback Server** - IP Addresses of the primary and
    secondary domain controllers.
  - **Mode** - For most modern AD domains, you will need to choose
    "LDAPS" for the mode.
  - **Require TFA** - You can choose to require two factor
    authentication for this realm. Note that an administrator will need
    to work with the user to configure TOTP or Yubikey **before** the
    new user will be able to log in.
  - **all other fields** - can likely be left at defaults unless you
    know better.
  - **Sync Options Tab** - Contains all of the options for configuring
    user sync with the realm. We will not be setting this up for this
    document, but the Proxmox documentation covers it very well.
  - When all the fields look correct, click the "Add" button, and the
    connection will be tested, and if everything works, the realm will
    appear in the list.

Now lets create a Proxmox user who will log in using their Active
Directory credentials. Note that this user should already exist in
Active Directory.

* "Datacenter -> Permissions -> Users -> Add" You have seen this dialog before!
  * **Username** - Must match the username in AD (depending on the realm, it may or may not be case sensitive.)
  * **Realm** - Choose the new AD realm we just created
  * **Group** - The group you want to assign this user to. Right now, we only have the one "Administrators" set up, but if you have other groups configured you can pick the appropriate one for this user. 
  * Once everything looks correct, click "Add" and the user will show up in the list. 
  * Let's test the new user. The simplest way will be to open Proxmox in either a different browser, or in private or incognito mode. \\  ![Proxmox login dialog](/img/screenshot_2025-02-11_113042.png)
  * Just be sure to select Your Active Directory Realm from the drop down, and enter the user credentials. If you set everything up right, you should be logged in.

## Your First Virtual Machine

Quite honestly, creating a new VM in Proxmox is at least as simple as it
is in ESXi or most other virtualization platforms. But we need a VM in
order to talk about backups, so let's build one real quick.

- Top-right of the page, click "Create VM" a new Dialog will open.\
  ![Proxmox new VM
  dialog](/img/screenshot_2025-02-11_145214.png)
- This is a wizard which walks you through all of the settings for a new
  VM. Most of these are very common and self-explanatory, so we will
  only mention a couple of points.
  - **VM ID** - Proxmox requires all virtual machines (and containers)
    to have an ID number between 100 and 999999999. This ID has to be
    unique and you will see it referenced in many different functions.
    Like IP Addresses, you should consider designing a standard
    convention to follow.
  - **System tab -\> BIOS** - SeaBIOS is the default, Choose OVMF if
    UEFI is a requirement.
  - **System Tab -\> Qemu Agent** - It is suggested to always install
    the qemu guest agent tools (Similar to VMware tools on that
    platform.)
  - **CPU tab -\> Type** - Especially if your cluster has different
    hardware on different nodes, it is recommended to choose "host" for
    the CPU type. This sets the VM CPU to match the CPU available on the
    host. This avoids having to virtualize any functions, and makes the
    VM a bit more portable.
  - **Network tab -\> Model** - You will get best performance choosing
    "VirtIO (paravirtualized)"

## Built-in Backups

In Proxmox, backups are also integrated into the core management
interface. This is a bare-bones, basic backup, but it works well. For a
more robust backup solution see [Proxmox Backup
Server](PBS%20Initial%20Setup.md).

For this document, we will create a simple backup schedule which saves
backups to the Windows Share we set up earlier. Along the way we will
cover retention and restoration as well. All of these pieces will apply
to [Proxmox Backup Server](PBS%20Initial%20Setup.md) once it is set up
also.

#### On Demand Backup

You can create a backup, or restore a virtual machine from backup at any
time very simply. From the left sidebar, highlight the virtual machine,
then choose "Backup".

![Proxmox VM backup
window](/img/screenshot_2025-02-14_154258.png)

Across the top of the main windows are several action buttons, and a
drop-down to choose a storage location. The main window will show a list
of all backups stored in the selected storage location. Go ahead and
choose the Windows Network share we created earlier in this document.
Now click the "Backup Now" button and a dialog will open up showing a
few options. In this case, we are going to leave all of the defaults in
place and click "Backup" to start the backup process. A status window
will open giving details of the process. You can close this window if
you prefer and the backup job will continue in the background.

#### Restore

As you might have guessed, you can also restore a specific virtual
machine in the same place. Choose a storage location, highlight one of
the available backups, and click the "Restore" button.

- **Warning** - Restoring a backup from this location will overwrite the
  existing virtual machine.
- If you need to restore a backup to a different location or without
  overwriting the existing VM, you need to do it from the Storage
  browser and change the VM ID in the restore dialog before continuing.
- There is also a handy button to show you which virtual machines are
  not a part of any backup jobs.

### Scheduling Backups

Backup Jobs can also be scheduled and managed at the Datacenter level.
Highlight Datacenter in the left sidebar, and choose "Backups". Since
this is our first schedule, click the "Add" button up top and a new
dialog will open

![Proxmox backup schedule
dialog](/img/screenshot_2025-02-14_163115.png)

- **Storage** - Choose the Windows share we configured earlier.
- **Schedule** - There are several pre-built schedules you can choose
  from, or you can type a schedule directly into the dialog. I'd suggest
  reviewing the documentation for the syntax, and the main window has a
  handy Schedule Simulator you can use to simulate the results of a
  schedule and is handy for learning the syntax and ensuring what you
  enter gives the results you expect.
- **Send email to:** - enter the address to send job notifications
  to for this schedule.
- Then in the lower section of the dialog place a checkmark next to the
  virtual machines you want to be included in this job.
- **Retention Tab** - You can set a retention policy here in the
  schedule if you would like.
  - Generally speaking, and especially in the case of the Proxmox Backup
    Server, it is advised to set a retention policy at the storage
    level.
  - Since we did not set a retention policy when we set up the Windows
    share location, we can go ahead and set the retention policy as part
    of this job.
- Once everything looks correct, click the "Create" button, and the new
  backup job will appear in the list.

## Conclusion

That's it! We have a working Proxmox installation, a good foundation to
build from, a running VM to play with, and a working backup schedule. Be
sure to continue by reading the page on setting up [Proxmox Backup
Server](PBS%20Initial%20Setup.md)

[^1]: !PAY ATTENTION! - be sure you are picking the correct drive. This
    operation will completely wipe out any data on the drive!

[^2]: Some drives previously used by Windows require a second "Wipe" to
    remove all of the partitions

[^3]: Essentially, the order determines where the parity information is
    stored. On a RAIDZ selection, the first drive holds parity
    information and the remaining drives hold the data stripe. in
    RAIDZ2, the first 2 disks will hold parity information and the
    remaining drives will hold the data stripe. So if you need to
    specify which drive(s) will hold parity information for any reason,
    the Order field allows you to define that.

[^4]: We will assume the network share is already created in Windows and
    any permissions set as needed.

[^5]: I haven't tested any of the SDN abilities yet, so look forward to
    this document being updated as we have a need or chance to delve
    into SDN.

[^6]: already working

[^7]: Also note that the system as a whole should only have one Default
    Gateway defined. By default this is set during installation, and is
    on the management network.

[^8]: Where x is the bridge number and yy is the VLAN number

[^9]: screenshot shows a gateway, but was not needed

[^10]: and any other high priority devices

[^11]: The idea of AD causing a new user to be created in the virtual
    environment just gives me a case of the heebie-jeevies, even with
    good AD monitoring and reporting...

[^12]: Despite the screenshot!
