# Proxmox Backup Server

While the Proxmox Virtual Environment includes basic backup abilities,
adding Proxmox Backup Server (PBS) provides a much more robust backup
solution, and provides many additional functions. We will not be
covering these in-depth in this document, but will walk through the
initial setup, which should provide a solid foundation to build on.

- Documentation for the latest version is [available
  online](https://pbs.proxmox.com/docs/introduction.html)
- Documentation for the installed version is available from the web
  interface of the server
- Proxmox has an active and helpful [community
  forum](https://forum.proxmox.com/)
- Proxmox also provides paid support if desired

## Planning and Preparation

There are a few things to consider before getting started.

### Bare Metal vs. Virtual

Proxmox recommends to always install PBS onto bare metal. The reason for
this is simple enough: If the VM host PBS is installed on fails, you
will also lose access to your backups, making recovery much more
complicated than it needs to be.

However, there are some very real performance benefits to running PBS
from within you hosting environment. Given the hardware budget, you
might consider doing both! Install PBS inside the VM environment and use
that for Near-line, fast backups, then sync those backups (or set up a
completely separate schedule) to a bare metal PBS install for
longer-term or even off site backups.

- **Bare Metal Considerations**
  - The faster the storage the better. SSD or NvME RAID setup is ideal.
  - Consider placing PBS onto a separate/isolated network. This can help
    with performance, but provides isolation/segregation as well.
  - PBS has support for Tape backup, including auto-changers, and
    printing barcodes, etc. WANSOL has not had any opportunity to test
    this yet, though.
- **Virtual Environment Considerations**
  - There are two options for this arrangement also:
    - Install directly on one of your Nodes - This makes assigning
      dedicated storage simpler, but means you can't really backup the
      backup server itself, or migrate it to a different node, etc. if
      the need arises. This would be a good option for a small, 2 node
      cluster though.
    - Create a virtual machine inside of Proxmox, and install PBS to
      this VM. This allows more flexibility and would likely be the
      better option for larger clusters, but possibly adds some overhead
      to connecting storage.
  - Still recommend the fastest storage you can get.
  - Separate storage from your primary VM storage is also highly
    recommended.

### Install on one of your VM Hosts

Installation to a virtual machine is essentially the same as installing
onto bare metal, so this section is specific to installing PBS directly
onto one of your VM nodes, alongside PVE.

Log in as root to the console (SSH) of the node you want to install PBS
onto.

- First you need to add the encryption keys for the PBS APT
  repositories:\
      `wget https://enterprise.proxmox.com/debian/proxmox-release-bookworm.gpg -O /etc/apt/trusted.gpg.d/proxmox-release-bookworm.gpg`
- Next you add the PBS repository to the APT source list:\
      `nano /etc/apt/sources.list`

  \
  then add the following lines:

<!-- -->

    # Proxmox Backup Server pbs-no-subscription repository provided by proxmox.com
    deb http://download.proxmox.com/debian/pbs bookworm pbs-no-subscription

- Save the file (CTRL+O) and exit Nano (CTRL+X).
- `apt update` to refresh the list of available software/updates
- then `apt install proxmox-backup-server` to actually install the
  server.
- When installation is complete, you can reach the PBS web interface at
  <https://ip.address.of.node:8007/> and log in using the same root
  account already configured on the node.

### Bare Metal Installation

Installation on bare metal or a virtual machine is very much like
installation of any other operating system. Highlights are below:

- [Download](https://www.proxmox.com/en/downloads) and prepare the
  installation media
- Boot from installation media
- It is recommended to only configure storage for the boot disk during
  installation. Data storage can be configured after installation
- Be sure to set a strong root password and to enter a valid email
  address for system notifications
- Network config during installation is for the management interface.
  Additional networks can be configured after installation

## First Access

Once installation is complete, access the Proxmox Backup Server web
interface at <https://ip.of.the.server:8007>. Continue past the warning
about the self signed certificate, and sign in using the root account
credentials you created during the installation.

### Configuration Section

For your first stop, Highlight "Configuration" in the left sidebar, then
the "Network/Time" tab. Take a moment to set the correct time and time
zone as needed, and to double check DNS and other network settings
before moving on.

### Updates

First things first, get the server up to date. If paid support was
purchased, now is the time to enter the license information. From the
web interface, go to "Configuration -\> Subscription", then "Upload
Subscription Key". Simple enough!

If a support license was not purchased, you need to point the server to
the free repositories before checking for and installing updates.

Start in the web interface, highlight "Administration" in the left
sidebar, then go to "Repositories" You should see a list of the
configured repositories similar to the image below.

![Proxmox repositories
screen](/img/screenshot_2025-02-09_125057.png)

Click the "Add" button, and choose "No-Subscription" from the dropdown
list. Then click "Add" again. At this point you should see both the
enterprise and no-subscription repositories listed. Highlight the
enterprise repository and click the "Disable" button.

Now you are all set! Go to "Updates" from the top options, click
"Refresh" and the server will check for, then present a list of
available updates. Click the "Upgrade" button and a separate window will
open and all available updates will be installed. When updates are
complete, you can either enter the "exit" command then close the window,
or simply close the window.

If there was a newer kernel included in the updates, you should consider
rebooting the system to apply the updated kernel. (The reboot button can
be found under "Administration -\> Server Status -\>Reboot".)

### Storage/ZFS Setup

In this simple example, we will use the PBS web interface to setup a 3
drive ZFS RAID array for backup storage. Start by looking at information
about all of the disks installed on the server. Highlight Storage/Disks
in the left sidebar, then choose "Disks" You should see information
similar to the image below:

![Proxmox disk information
screen](/img/screenshot_2025-02-09_131733.png)

In this example the first disk listed (`/dev/nvme0n1`) is the OS drive,
and the other three drives are the ones you will be using to setup our
ZFS pool on. Since these were previously used drives placed into this
test system they already have partitions and data on them. So first you
need to wipe the drives and initialize them to make them available for
use in your pool.

Highlight one of the drives [^1] and click "Wipe Disk". Repeat the same
process for each drive you will be using in the ZFS pool. Each drive
should now appear with no partitions listed, and "no" in the Usage
column of the drive table. [^2]

If the drives are brand new out of the box, you likely will just need to
highlight the drive and click the "Initialize Disk with GPT" button.

Now that all of the drives are clear, go over to to "ZFS". There should
be nothing in this list right now. Click on "Create: ZFS" and you should
see a dialog similar to the image below. [^3]

![Proxmox ZFS pool creation
dialog](/img/screenshot_2025-02-09_133333.png)

- Enter a name for the ZFS pool. In this example, enter `zbackups`
- Select the drives to be used by placing a checkmark in the box next to
  each listing.
- Choose the RAID level.
  - This is a big topic, and beyond the scope of this document, but generally speaking, if you have 6 or fewer drives, you probably want to choose one of the RAIDZ options. If you have a higher number of drives, you probably want one of the "dRAID" options.
   - There are as many ways of configuring drives for ZFS as there are drives. For more complex setups, you will need to use the command line for setup. The ProxMox GUI only supports some basic, most common options.
   - RAIDZ - Similar to RAID5 - uses one drive for parity
   - RAIDZ2 - Uses 2 drives for parity
   - RAIDZ3 - Uses 3 drives for parity
   - dRAID levels - distributes the parity information across all available drives. The higher levels allow for a higher number of drives to fail simultaneously without data loss.

You will choose `ZRAID` for this example. Do not worry about defining
the "order" for the drives. This will be automatically determined in
this case. [^4]

When you are happy with the settings, Click "Create" and the ZFS pool
will be created and should be visible Under Datastore in the left
sidebar.

#### Namespaces

A namespace is essentially just a subdirectory created in a datastore.
PBS supports a maximum depth of 8 levels of namespaces [^5]. Permissions
can be configured per namespace, and are especially handy for preventing
naming conflicts when one datastore is housing backups for multiple
virtual environments.

Now that you have a datastore, go ahead and create a namespace under it
which will be used later to house backups to be synced to a remote
Proxmox Backup Server.

- Highlight your "zbackups" datastore in the left sidebar, then choose
  "Content".
- Top left, click "Add Namespace" and a new dialog will open:\
  ![Create Namespace
  Dialog](/img/screenshot_2025-03-07_135457.png)
  - **Parent Namespace** - In this case, your only option is "root", but
    if there are already other namespaces created, you will be able to
    select one here
  - **Namespace Name** - Keep thing simple and call this one "remotes"
    [^6]
  - Click Create and you should see the new namespace listed in the
    contents list.
  - \*Repeat the process to create a namespace called "backups"

#### Pruning and Garbage collection

Prune jobs essentially determine the retention policy for backups stored
in a namespace or datastore. It tells the server how many copies of each
backup to keep and for how long. When a backup passes this retention
period, it is marked for deletion

Garbage Collection is simply the process of going through and actually
deleting those backups marked for deletion.

It is time to set up a simple schedule for each of these now.

##### Prune Job

Prune Jobs are very quick to complete [^7] and do not require many
resources, so they can be run relatively often. So feel free to schedule
them for any interval that makes sense for the environment.

- Highlight the "zbackups" datastore in the left sidebar, then choose
  "Prune & GC Jobs"
- You should see a default schedule listed under both the Garbage
  Collect Jobs and Prune Jobs sections.
- Highlight the entry under Prune Jobs, then click the "Edit" button. A
  new dialog will open:\
  ![PBS Prune Job
  Dialog](/img/screenshot_2025-03-07_141624.png)
  - **Prune Schedule** - Leave this at daily for now. There are several
    example schedules you can choose from in the drop down, or you can
    just enter in your own schedule. (Search documentation for "Calendar
    Events" for complete information on formatting custom schedules.)
  - **Namespace** - Which namespace this prune job applies to. This
    allows you to set different retention policies for backups stored in
    different name spaces.
  - **Max Depth** - How many child namespaces this job applies to. In
    the screenshot, the job will only apply to backups stored in the
    root of the datastore, and not to any deeper namespaces.
  - **Keep Fields** - These are counted from smallest to largest, and
    are additive. In the screenshot above we are keeping the 3 most
    recent backups regardless of their interval, **then** then last
    backup on each of the next 4 days, **then** the 3 most recent
    backups for the next 3 previous weeks, and then the most recent
    backup from the next 5 previous months. Assuming one backup per day,
    this should work out to a total of 6 months worth of backups.
  - **Comment** - Exactly what it says.
- Click OK and you should see your changes reflected in the listing of
  prune jobs.

##### Garbage Collect Job

Garbage collection can take considerably longer than prune jobs [^8] so
should be scheduled a bit more conservatively.

- Highlight the default garbage collection job and click "Edit" a new
  dialog will open with a single field: The schedule. Enter `2:30` here
  and click OK. This will schedule Garbage Collection for 2:30am each
  night.

#### Verification Jobs

Verification jobs are an important part of managing backups on PBS.
Essentially, the first time a verification job is run on any backup, a
hash is created and stored. All later verification jobs re-run that hash
process, and compares the result to the original. This verifies that the
backup has not changed since it was first written. **This is
NOT a substitute for regular recovery testing!!** This
process helps to detect things like disk errors, bit-rot, or even
ransomware attacks. It doesn't mean the backup will be able to be
restored successfully, it just means the backup has not been altered
since it was created.

Verification can take a long time. This is highly dependent on the
number of backups, storage speed, and CPU speed; but no matter what, it
is much more resource intensive than either prune or garbage collection
jobs. Schedule accordingly.

- Highlight the zbackups datastore in the left sidebar, then choose
  Verify Jobs. The list will be empty.
- Click "Add" and a new dialog will open:\
  ![PBS Verification Job
  Dialog](/img/screenshot_2025-03-07_153138.png)
  - **Schedule** - Type `sat 4:15` here. This will schedule the job to
    run at 4:15am every Saturday
  - **Namespace** - We will leave this at "root"
  - **Skip Verified** - When checked, will skip over backups which have
    been recently verified
  - **Max Depth** - Whether to verify deeper namespaces
  - \*\* Re-Verify After\*\* - How often to reverify backups which were
    previously verified.
  - Click Add when you are satisfied with the settings, and the job
    should be listed in the main window now.

### Networking

Networking in PBS is configured nearly identically to how Proxmox
Virtual Environment handles it. Please refer to [the Proxmox Virtual
Environment Setup](/img/proxmoxsetup#networking) page for
more information.

- Consider separate networks for management and data/backups. In
  addition to better performance, it is always a good idea to segregate
  the management interface.
- Only the data network needs to be able to communicate with the VM
  hosts.

### Users, Realms, and Permissions

PBS has a robust and detailed permissions system. We will barely scratch
the surface in this document. It is **highly** recommended to refer to
the documentation for complete descriptions of the permissions systems
in PBS. User management in PBS is extremely similar to that in Proxmox
Virtual Environment, if a bit simplified, so If you are already familiar
with users there you should find PBS extremely simple to get going.

#### Realms

A realm is basically just a source for user authentication. By default,
PBS has two realms available:

- **Linux PAM Standard Authentication**
  - This is user authentication at the underlying operating system
    level. By default, the `root` account is the only member of this
    realm. If you prefer, you can connect to the Proxmox node via SSH or
    the console and create new users on the OS, assign them to required
    groups and those users will be able to use those credentials to
    manage Proxmox. This is usually only used in locations which already
    have a large Linux infrastructure, and allows Proxmox to tie into
    that existing infrastructure.
- **Proxmox Backup Authentication**
  - This is user authentication specific to Proxmox. These users may (or
    may not) have permissions to almost any portion of Proxmox, but will
    not have access to any of the underlying operating system, so will
    not be able to connect via SSH or install updates, for example. This
    is more or less the default way to add local users to Proxmox. This
    realm also has built-in support for Time-based One Time Passwords
    (TOTP), WebAuthN and YubiKeys as MFA methods as well as being able
    to create a set of recovery keys to use in case of issues with any
    of the MFA methods.

PBS ships with support for Active Directory, Open LDAP, and OpenID
Connect as additional realms. Setting up these realms is identical to
setting them up in PVE so refer to the [Proxmox Virtual Environment
Setup](README.md#users_realms_and_roles) page for
more details.

Note that any/all of these realms can be used side by side, though it is
helpful if you try to avoid duplicate usernames in different realms.

### Users and Access Tokens

Using the `root` account for everything is generally considered a Bad
Idea(tm) and is to be avoided.

The simplest definition of access tokens is probably "A single
task/purpose account". Any user can create an access token and assign
specific permissions to that token (though never with more permissions
than the account used to create the token). Generally speaking, an
access token is considered more secure and the preferred option for most
automated tasks. Think things like remote PBS server backup syncs,
scripted jobs, etc.

Start off by creating an admin user account. You will create and use
access tokens later on to connect PBS to PVE and for sending backups to
a remote PBS server.

By default, all new users and access tokens have no permissions at all,
to any part of the server.[^9] So you will also need to configure some
permissions for this new account. This example will not set up MFA on
this account, but MFA in PBS is set up identically to in PVE, so refer
to the [Proxmox Virtual Environment
Setup](README.md#users_realms_and_roles) page for
more details.

- Highlight Configuration -\> Access Control in the left sidebar, and
  choose User Management. The only user at this stage will be the root
  account created during installation.
- Click "Add" above the user list and a new dialog will open:\
  ![PBS new user
  dialog](/img/screenshot_2025-03-08_135048.png)
  - Most fields are self explanatory.
  - **Realm** - For this example, you will leave it set to "Proxmox
    Backup authentication". If you have added another realm (such as
    Active Directory) you can choose it here.
  - **Expire** - The date the user account will be automatically
    disabled.
- When everything is entered correctly, click "Add" and the new user
  should be shown in the user list.

#### Permissions

As mentioned, by default new users do not have permission to even log in
yet, much less see or edit anything in PBS. Search the documentation for
"Access Control" for a full description of privileges, roles and
permissions. In this example, you are creating an administrator account,
so setting up permissions is going to be extremely simple.

- Choose the "Permissions" tab. There should not be any listings at this
  point.
- Click "Add -\> User Permission" and a new dialog will open:\
  ![PBS User permission
  dialog](/img/screenshot_2025-03-08_140920.png)
  - **Path** - Since this is an administrator account, we will choose
    the root path `/`
  - **User** - Choose the new `pbsadmin` account we just created.
  - **Role** - Choose "Admin" from the drop down.
  - **Propagate** - In this case, leave it checked. Determines whether
    the assigned permissions should apply to all lower namespaces in the
    path, or only to the specific level chosen. This means one user can
    have different permissions in different paths of the system.
- When all the fields are set correctly, click "Add" and the permission
  should be listed in the main window.

#### Access Token

We are nearly there! Next, we are going to set up an access token which
we will use to connect PVE to PBS. You can also use a regular user
account for this but an access token is going to be much safer. (Not
only will the access token only have the minimum permissions needed to
create backups, but even if the PVE cluster were compromised somehow,
**and** a flaw was found which allowed the stored credentials to be
discovered, the access token would still not have permission to actually
log into the PBS and cause further damage. )

- Still under Access Control, choose the "API Token" tab at the top of
  the main window. There won't be any tokens listed yet.
- Click "Add" and a new dialog will open:\
  ![PBS create access token
  dialog](/img/screenshot_2025-03-08_151850.png)
  - **User** - The user account the token will be tied to. [^10]
  - **Token Name** - Can be anything, no spaces allowed
- When all the fields are entered correctly, click "Add" and a new
  dialog will open:\
  ![PBS Token Secret
  Dialog](/img/screenshot_2025-03-08_152114.png){.align-center
  width="400" query="?direct&400"}
- **IMPORTANT!** - Record this information. This will the
  the **only** chance you have to view the secret for this token. You
  will need this information to add PBS to PVE.

Now you need to add permissions to this token. Still in Access Control,
click the Permissions tab.

- Click "Add -\> API Token Permission" and a new dialog will open. This
  is the same dialog encountered when you set up the admin account
  permissions.
- Choose `/datastore/backups` for the path [^11]
- Choose "DatastoreBackup" For the role. [^12]
- Leave Propagate checked. If you do not want this token to have access
  to lower level namespaces remove the check.
- When everything looks right, click "Add" and the new permission should
  be shown in the list.

## Connect PBS to PVE

You are finally all set to connect PBS to the virtual environment and
start creating backups!

- **In the Proxmox Backup Server interface**
  - Highlight Dashboard in the left sidebar
  - Click "Show Fingerprint" in the title bar of the server summary
    widget. A new Dialog will open:\
    ![PBS Fingerprint
    dialog](/img/screenshot_2025-03-08_155834.png){.align-center
    width="400" query="?direct&400"}
  - Copy this fingerprint. You **did** copy the API token ID and secret
    from the previous step, right? You'll need all of these to set up
    the connection the PVE \* Also, make note of the datastore and
    namespace you will want backups to be stored in. In this example the
    datastore will be ''zbackup'' and you will use the backups
    namespace.
- **In the Proxmox Virtual Environment interface**
  - Go to Datastore -\> Storage
  - Click "Add -\> Proxmox Backup Server. A new dialog will open:\
    ![PVE Add PBS
    dialog](/img/screenshot_2025-03-08_160511.png)
    - **ID** - The name PVE will call
    this storage location. No spaces allowed.
    - **Server** - FQDN or IP
    of the PBS server
    - **Username** - This will be the token ID you
    created earlier.
    - **Password** - The Token Secret you recorded
    earlier.
    - **Nodes** - Whether to add this storage to all nodes, or
    specific nodes in the cluster.
    - **Datastore** - `zbackup` in this
    case
    - **Namespace** - In this example, will be set as `backups`
    - **Fingerprint** - The fingerprint of PBS, copied earlier.
    - **Backup Retention Tab** - You will leave this blank for this
    example. This means backups to this storage location will use the
    retention policy that is configured in PBS
    - **Encryption Tab** - For this example, you will not enable encryption. You will be
    setting up an encrypted storage location later in this document.
    - When everything looks correct, click "Add" and the new storage
    location should be shown in the list.

#### Make a Test Backup

Now that PBS is set up as a valid storage destination, backups can be
created there using any of the available methods in PVE. Make a quick
test backup to make sure things are working.

- **In PVE interface**
  - Choose one of the virtual machines from the left sidebar, then
    highlight "Backup" in the next sidebar.
  - At the top of the list of backups, choose the PBS server you just
    added from the "Storage" drop down. The list will be empty right
    now.
  - Click the "Backup Now" button. A dialog will open with some options
    for the backup. Don't make any changes to these right now. Click the
    "Backup" button, and a new dialog will open showing the progress of
    the backup job. Feel free to watch and wait, or to close the dialog
    window and the job will continue in the background. Once the backup
    job completes, the backup will show in the list. [^13]
- You can also go to "Datacenter -\> Backups" to set up scheduled backup
  jobs, choosing PBS as the destination storage.

#### Restoration

One feature you gain by creating backups to PBS is that you are able to
do file level restores directly from inside PVE. Other restorations are
also handled directly from PVE, so refer to the [Proxmox Virtual
Environment Setup](README.md#built-in_backups) page
for details if needed.

A walk through restoring a single file or directory from the backup we
just created.

- **In the PVE interface**
  - Select the VM you just created the test backup of, and choose the
    Backup section
  - Highlight the test backup in the list of available backups, then
    click the "File Restore" button. A new Dialog will open:\
    ![PVE file restore
    dialog](/img/screenshot_2025-03-08_164818.png)
  - A tree of the drives and partitions included in the backup will be
    shown. Browse to the location of the file or directory you need to
    restore and highlight it.
  - If a directory is highlighted, the button in the dialog will be
    "Download as:" and give the option to download a .zip or tar.gz file
    of the selected directory. This will be downloaded to the normal
    directory you have your browser configured to save downloads in.
  - If a single file is selected, the button will just read "Download"
    and the file will be downloaded directly via your browser.
  - You can then use whatever method works best to transfer the file
    where it is needed.

## Remote Sync

With PBS, backups can be duplicated (synced) to a remote PBS system,
allowing for secure offsite backups. By default, remote backups are
**pulled** into the remote server, not **pushed** from the local system.
More recent versions of PBS allow you to choose whether to push or pull
backups, but the recommendation is still to pull.

By pulling backup, there are no credentials or other configuration for
the remote PBS stored on the local PBS instance. All permissions for the
remote PBS instance are controlled on the local instance, allowing for a
cleaner separation if a remote instance becomes untrustworthy for any
reason.

A couple of things to consider:

- TCP Port 8007 has to be open between the servers.
- All communication between servers is already encrypted, but if
  desired, a VPN link between the sites can also be used, allowing PBS
  to remain isolated from the internet.
- Sync jobs can be rate limited to prevent bandwidth hogging.
- It is recommended to encrypt all backups which will be synced
  off-site. The remote PBS will never have any encryption keys or other
  information, meaning you don't have to have full trust in the remote
  server.

#### Encrypted Backups

First you need to create an encrypted backup you can sync.

- **On your new PBS**, you already created a new namespace called
  `remotes`. This is where you will be storing the backups you want to
  sync with a remote PBS instance.
- On the local PBS, create a new Access Token to use for the encrypted
  backup and sync jobs [^14]. The process is identical to when you set
  up the access token above. The only difference will be that you need
  to set the permissions for this token to point **only** to the
  `remotes` namespace, and not to `backups`.
  - Be sure to record the Token ID and Secret. You will need them to set
    things up in PVE and the remote PBS system.
  - Set permissions for the token exactly as you set them for the first
    token you created, just pointed at the `remotes` namespace instead
    of `backups`.
  - You will also need the PBS thumbprint again for both PVE and the
    remote PBS.
- **On the PVE you will be backing up**
  - The basic process is identical to when you set up PBS as a storage
    location above. "Datacenter -\>Storage -\> Add -\> Proxmox Backup
    Server" The dialog should be familiar now, go ahead and fill out the
    General tab, be sure to add `remotes` as the namespace:\
    ![PVE add storage
    dialog](/img/screenshot_2025-03-09_125502.png)
  - **Encryption Tab** - Go ahead and choose "Auto-generate a client
    encryption key"\
    ![PVE storage
    dialog](/img/screenshot_2025-03-09_125737.png)
  - Now when you click "Add" a new dialog will open:\
    ![PVE encryption info
    dialog](/img/screenshot_2025-03-09_130021.png)
  - **[IMPORTANT!]{.underline}** - This is your **ONLY** chance to
    record this encryption key. It is unrecoverable from the system. If
    you lose the key, any backups made with that key will be
    unrecoverable.
  - It is highly recommended to utilize **all three** options presented
    to you.
    - Copy the key and store it in a password manager
    - Download the key as a text file and save it to a securely stored
      thumb drive
    - Print the key, laminate the print and store it in a fireproof safe
- Create a backup to this new storage location. You can go ahead and set
  up a schedule, or make an on-demand backup just for testing.

Once your test backup has completed, you are ready to set up a link to a
remote PBS and configure a sync job.

#### Connecting to a Remote PBS

Now you have all the pieces in place to create the connection to a
remote PBS and to configure the sync job. All of the set up has already
been completed on the local PBS instance.

- Access Token credentials
- Local PBS fingerprint
- An encrypted backup to sync

All of the following set up is to be done on the **remote** PBS
instance:

- Log in with an admin account, or another account with permission to
  create and manage remotes.
- Go to "Configuration -\> Remotes" then click "Add" a new dialog will
  open:\
  ![PBS add remote
  dialog](/img/screenshot_2025-03-09_140949.png)
  - This dialog is very similar when you added your local PBS to PVE.
    Fill out the fields and click "Add" The new remote PBS should be
    shown in the listing now.
  - Now to set up the sync schedule. Highlight the datastore the remote
    backups will be synced to in the left sidebar, then choose "Sync
    Jobs" Any configured sync jobs will be listed here.
  - Click "Add -\> Pull Sync Job". A new dialog will open:\
    ![PBS Sync job
    dialog](/img/screenshot_2025-03-09_141721.png)
    - **Local Namespace** - Destination the remote backups will be saved
      to
    - **Local Owner** - Local account which will be the "owner" of the
      local copies. This account or token needs to have at least
      "DatastoreBackup" permissions to the path the remote backups will
      be synced to.
    - **Sync Schedule** - There are several example schedules in the
      dropdown, or you can enter your own custom schedule
    - **Rate Limit** - Limits network bandwidth to prevent the sync from
      affecting other clients on the network
    - **Source Remote** - The remote PBS just added in previous steps.
    - **Source Datastore** - Datastore remote backups are stored on
    - **Source Namespace** - Namespace remote backups are stored on
    - **Max depth** - Whether to include deeper namespaces
    - **Remove Vanished** - If checked, when a backup is purged from the
      source, it will also be deleted locally
    - **Re-Sync corrupt snapshots** [^15] - Will re download a backup if
      local verification check fails.
    - Click "Add when all of the fields are entered correctly. The new
      job will be shown in the list.

#### Test Sync

Still on the remote PBS, highlight the sync job you just created, then
click "Run Now" A new dialog will open, showing the status of the job.\
![PBS Sync job status
dialog](/img/screenshot_2025-03-09_143142.png)

If things are all set up correctly, the job should process successfully.

If needed, remember to schedule prune, garbage collection, and
verification job schedules for these remote backups.

## Conclusion

That's it! You should have a nice solid foundation to build additional
users, tokens, datastores, namespaces and create backups to them.

[^1]: !PAY ATTENTION! - be sure you are picking the correct drive. This
    operation will completely wipe out any data on the drive!

[^2]: Some drives previously used by Windows require a second "Wipe" to
    remove all of the partitions

[^3]: Screenshot is actually from a Proxmox Virtual Environment, but is
    nearly identical to PBS dialog

[^4]: Essentially, the order determines where the parity information is
    stored. On a RAIDZ selection, the first drive holds parity
    information and the remaining drives hold the data stripe. in
    RAIDZ2, the first 2 disks will hold parity information and the
    remaining drives will hold the data stripe. So if you need to
    specify which drive(s) will hold parity information for any reason,
    the Order field allows you to define that.

[^5]: where root '/' is counted as the first level

[^6]: Like all Linux systems, capitalization matters. `Remotes` is
    different than `remotes`.

[^7]: From experience, it takes less than one second to process prune
    jobs for about 25 different VMs

[^8]: From experience, roughly 20 minutes for those same 25 VMs
    mentioned in the prune job section above

[^9]: Technically, they are assigned the "NoAccess" role.

[^10]: Only available if logged in as root

[^11]: Search for "Objects and Paths" in the documentation for a full
    explanation

[^12]: This is the minimum role required to allow creating backups. This
    does **not** grant permissions to edit or delete backups.

[^13]: Note that this first backup will take a bit more time to
    complete. Future backups will only send over changed blocks, so will
    be considerably faster to complete

[^14]: These can be separate users if preferred.

[^15]: Visible when "Advanced" is checked
