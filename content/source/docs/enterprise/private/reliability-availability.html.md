---
layout: "enterprise2"
page_title: "Private Terraform Enterprise - Reliability & Availability"
sidebar_current: "docs-enterprise2-private-reliability-availability"
---

# Private Terraform Enterprise Reliability & Availability

This section covers details relating to the reliability and availability of
Private Terraform Enterprise (PTFE) installations. This documentation may be
useful to Customers evaluating PTFE or Operators responsible for installing and
maintaining PTFE.

## Components

Terraform Enterprise consists of several distinct components that each play a
role when considering the reliability of the overall system:

- **Application Layer**

  - _TFE Core_ - A Rails application at the center of Terraform Enterprise;
    consists of web frontends and background workers

  - _TFE Services_ - A set of Go services that provide various pieces of key
    functionality for Terraform Enterprise

  - _Terraform Workers_ - A fleet of isolated execution environments that
    perform Terraform Runs on behalf of Terraform Enterprise users

- **Coordination Layer**

  - _Redis_ - Used for Rails caching and coordination between TFE Core's web
    and background workers

  - _RabbitMQ_ Used for Terraform Worker job coordination

- **Storage Layer**

  - _PostgreSQL Database_ - Serves as the primary store of Terraform
    Enterprise's application data

  - _Blob Storage_ - Used for storage of Terraform state files, plan files,
    configuration, and output logs

  - _HashiCorp Vault_ - Used for encryption of sensitive data

  - _Configuration Data_ - The information provided and/or generated at
    install-time (e.g. database credentials, hostname, etc.)

## AMI Architecture

In the AMI Architecture, both the **Application Layer** and the **Coordination
Layer** execute on a single EC2 instance.

_Configuration Data_ is provided via inputs to the [Terraform modules that are
published alongside the AMI][tf-modules]. These inputs are interpolated into an
encrypted file on S3. The EC2 instance is granted access to download this file
via its IAM instance profile and configured to do so on boot via its User Data
script.

This setup allows the instance to automatically reconfigure itself on each
boot, making for a consistent story for upgrades and recovery. The instance is
launched in an Auto Scaling Group of size one, which facilitates automatic
re-launch in the case of instance loss.

The **Storage Layer** is delegated to Amazon services and inherits their
respective reliability and availability properties:

- _PostgreSQL Database_ - Amazon RDS is configured by default to use a
  [Multi-AZ][multi-az] deployment, which provides high availability and
  automated failover of the database instance. Nightly database snapshots are
  automatically configured and retained for 31 days. The [Amazon RDS
  Documentation][rds-docs] has much more information about the reliability and
  availability of this service.

- _Blob Storage_ - Amazon S3 is used for all blob storage. The [Amazon S3
  Documentation][s3-docs] has much more information about the reliability and
  availability of this service.

- _HashiCorp Vault_ - Consul is run on the PTFE EC2 instance and stores all
  Vault data. Consul data is backed up hourly to S3 alongside of the
  _Configuration Data_ and is set to automatically restore on boot.

[tf-modules]: https://github.com/hashicorp/terraform-enterprise-modules
[multi-az]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html
[rds-docs]: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html
[s3-docs]: https://aws.amazon.com/documentation/s3/

### Availability During Upgrades

Upgrades for the AMI Architecture are delivered as new AMIs. Switching AMI IDs
within the Terraform config that manages your PTFE installation will cause the
plan to include replacement of the Launch Configuration and Auto Scaling Group
associate with your Terraform Enterprise instance.

Applying this plan will result in the instance being relaunched. It will
automatically recover the latest backup of _Configuration Data_ from S3 and
resume normal operations.

In normal conditions, Terraform Enterprise will be unavailable for less than
five minutes as the old instance terminates and the new instance launches and
boots the application.

### Recovery From Failures

The boot behavior and Auto Scaling Group configuration described above means
that the system can automatically recover from any failure that results in loss
of the instance. This recovery generally completes in less than five minutes.

The Multi-AZ setup used for RDS protects against failures that affect the
Database Instance, and the nightly automated RDS Snapshots provide coverage
against data corruption.

The redundancy guarantees of Amazon S3 serve to protect the files that PTFE
stores there.

## Installer Architecture (Beta)

~> **Note**: The PTFE Installer architecture is currently in beta and is being
actively developed. The section below is subject to change as work continues.

The PTFE Installer architecture has several supported modes of operation, each
of which has different implications for reliability and availability.

The operational mode is selected at install time and can not be changed
once the install is running.

### Demo mode

In the Demo mode of the Installer Architecture, the **Application Layer**,
**Coordination Layer**, and **Storage Layer** execute on a linux instance.

All data is stored within Docker volumes on the instance.

The builtin Snapshot mechanism can be used to package up all data and store it
off the instance. The builtin Restore mechanism can then be used to pull the
data back in and restore operations. The Snapshot and Restore functionality is
available to be configured and automated from the Admin Console.

### Mounted Volumes

In the Mounted Volumes mode of the Installer Architecture, the
**Application Layer**, **Coordination Layer**, and **Storage Layer** execute on
a Linux instance.

_Configuration Data_ for the installation is stored in Docker volumes on the
instance.

Both the _PostgreSQL Database_ and _Blob Storage_ use mounted volumes for their
data. Backup and restore of that volume is the responsibility of the user, the system
does not manage that.

The builtin Snapshot mechanism can be used to package up the _Configuration
Data_ and store it off the instance. The builtin Restore mechanism can then be
used to pull the configuration data back in and restore operations.  The
Snapshot and Restore functionality is available to be configured and automated
from the Admin Console.

### External Services

In the External Services mode of the Installer Architecture, the
**Application Layer** and **Coordination Layer** execute on a Linux instance.
The **Storage Layer** is configured to use as external services in the form an
a PostgreSQL server and an S3-compatible service.

The maintainance of PostgreSQL and S3 are handled by the user, which includes backing
up and restoring if necessary.

_Configuration Data_ for the installation is stored in Docker volumes on the
instance.

The builtin Snapshot mechanism can be used to package up the _Configuration
Data_ and store it off the instance. The builtin Restore mechanism can then be
used to pull the configuration data back in and restore operations.  The
Snapshot and Restore functionality is available to be configured and automated
from the Admin Console.

### Availability During Upgrades

Upgrades for the Installer Architecture utilize the Installer Admin Console.
Once an upgrade has been been detected (either online or airgap), the new code
is imported and once ready, all services on the instance are restarted running
the new code. The expected downtime is between 30 seconds and 5 minutes,
depending on if database updates have to be applied.

Only application services are changed during the upgrade; data is not backed up
or restored. The only data changes that may occur during are the application of
migrations the new version might apply to the _PostgreSQL Database_.

When an upgrade is ready to start the new code, the system waits for all terrform runs
to finish before continuing. Once the new code has started, the queue of runs is
continued in the same order.

### Recovery From Failures

#### Demo mode

If the instance running Terraform Enterprise is lost, the only recovery mechanism
in demo mode is to create a new instance and use the builtin Restore mechanism to
recreate it from a previous snapshot.

The frequency of automated snapshots can be configured such that worst-case
data loss can be as low as 1 hour.

#### Mounted Disk

If the instance running Terraform Enterprise is lost, the presumption is that the
volume storing the data is not lost. Because only configuration data is stored
on the instance, we recommend using a system snapshot mechanism to provide fast
recovery.

The procedure here would be as follows:

- Be sure that the mounted volume being used is automatically mounted
  automatically on start.  For instance with AWS, this means having it attached
  to the EC2 instance as well as being mounted to a path in Linux.

- Perform the initial installation of the product, including entering the
  license and doing initial setup.

- From the admin UI, click the **Stop** button. This will stop all of the
  services the application runs.

- Take a snapshot of the instance. For instance, in AWS this would mean shutting
  down the EC2 instance and then triggering an AMI to be created from the
  instance. The exact mechanism used is specific to the virtualization
  environment that the product runs in.

- Configure the virtualization to run the new snapshot and restart a new
  instance should it stop. In AWS, this is done using an Autoscaling Group.
  The environment can monitor the availability of port 443 for instance health
  as well.

The new instance now has the software as well as configuration on it by default
and can access the data stored on the mounted disk. Using these steps, the MTTR
of a lost instance is almost as fast as the virtualization environment can start
a new instance, typically less than a minute.

~> **NOTE:** Because the software is being restored from a snapshot, it's
important that this process be repeated when the Terraform Enterprise software
is updated so that any restored instance due to loss has the newest version.
This is important because the data stored on the mounted disk is versioned
along with the software.

#### External Services

If the instance running Terraform Enterprise is lost, the utilization of
external servies means no state data is lost. Because only configuration data is
stored on the instance, we recommend using a system snapshot mechanism to provide
fast recovery.

The procedure here would be as follows:

- Perform the initial installation of the product, including entering the
  license and doing initial setup.

- From the admin UI, click the **Stop** button. This will stop all of the
  services the application runs.

- Take a snapshot of the instance. For instance, in AWS this would mean shutting
  down the EC2 instance and then triggering an AMI to be created from the
  instance. The exact mechanism used is specific to the virtualization
  environment that the product runs in.

- Configure the virtualization to run the new snapshot and restart a new
  instance should it stop. In AWS, this is done using an Autoscaling Group.
  The environment can monitor the availability of port 443 for instance health
  as well.

The new instance now has the software as well as configuration on it by default
and can access the data stored on the mounted disk. Using these steps, the MTTR
of a lost instance is almost as fast as the virtualization environment can start
a new instance, typically less than a minute.

~> **NOTE:** Because the software being restored is preinstalled on the
virtualization specific snapshot, it's important that a new snapshot is taken 
when the Terraform Enterprise software is updated so that
any restored instance has the newest version. This is important
because the data stored in the external services is versioned along with the
software.
