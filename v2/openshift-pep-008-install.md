PEP: 008  
Title: OpenShift Installation Tool  
Status: accepted  
Author: N. Harrison Ripps <hripps@redhat.com>  
Arch Priority: High  
Complexity: 40  
Affected Components: admin_tools  
Affected Teams: UI (1), Origin (1)  
User Impact: High  
Epic: [Origin v3 Installation Wizard](https://trello.com/c/H01J2eo4/30-origin-v3-installation-wizard-design)

## Abstract
This document describes an improved installation system for OpenShift. The proposed system will use a CLI-based installer and [Workflow](#installer-workflows) model to deploy OpenShift onto one or more hosts. As an example of this, one Workflow will target the the VM that is currently used to deliver an all-in-one OpenShift Origin instance. This Workflow will enable the Origin VM to selectively serve one role (Broker, Node, or Message Queue Server) in a multi-instance deployment.

## Motivation
This PEP is the continuation of an effort that began with a [call to the OpenShift Origin Community](https://www.openshift.com/blogs/help-us-fix-the-openshift-origin-unboxing-experience) to help improve the system's installation process. Around one hundred community members [participated in a survey](https://www.openshift.com/blogs/survey-results-the-openshift-origin-unboxing-experience) to help us understand what their most important use cases were, and which installation tools would be most valuable to them. Based on the survey results, it seems clear that users need more general deployment capabilities beyond the all-in-one virtual machine where OpenShift Origin is showcased.

After an initial peer review, this PEP has been revised to clarify the standalone nature of this installer. Through the instrumentation of Workflows and Tasks, the first release of the installer will include support for the most popular install methods from the Origin survey. However, the installer is designed to be extensible enough to support other scenarios.

## Specification
In broad terms, the design proposal for this installation tool is as follows:

* The installer will consist of a text-based wizard that guides a user to:
	1. Define their OpenShift configuration
	2. Select an installation Workflow
	3. Execute the workflow
* The initial Workflows provided with the installer will include:
    * [Origin VM Role assignment](#workflow-use-an-origin-vm-in-a-distributed-deployment) - Specific to the Origin VM, the VM takes on one Role in a multi-VM Origin deployment (Broker, Node or Message Queue Server)
    * [Remote system Role assignment](#workflow-install-a-role-on-a-remote-system) - the installer connects to a [compatible host](#target-system-requirements) via SSH and configures it for a given Role.
    * [Local system Role assignment](#workflow-install-a-role-on-the-local-system) - the installer configures the local host for a given Role.
    * [Puppet script templates](#workflow-download-puppet-templates) - Users can also copy the puppet scripts from the VM to modify and run on their own.

The Roles referenced here are defined in [Simplified Deployment Roles](#simplified-deployment-roles).

### The Text-Based Installer
The Origin VM completes its boot-up procedure by invoking [oo-login#login()](https://github.com/openshift/puppet-openshift_origin/blob/master/templates/custom_shell/oo-login). This is the method that presents the user with some summary information about the running system:

    OpenShift Installer
    ----------------

    OpenShift console: https://broker.openshift.local/console
    Console login:     admin
    Console password:  admin

    IP address: X.X.X.X
    SSH user:   openshift
    Password:   openshift

    Press any key for root console

This utility will be replaced with a general-purpose package that can provide the additional user interaction that is described in this document. The expanded utility will be called `oo-install` and will function both within the context of the Origin VM and in a standalone capacity for use with other OpenShift distros.

The new installer will begin with an intro screen that summarizes the user's options:

    OpenShift Installer
    ----------------

    Welcome to OpenShift.

    This VM is currently running a complete OpenShift installation.
    You can connect to it by pointing your browser to:

    https://broker.openshift.local/console

    To do more with this VM, select from the following options:

    <1> Use this VM in a multi-instance deployment
    <2> Install OpenShift on another system
    <3> Download Puppet templates
    <4> See login information for this Origin VM
    <5> Exit the installer

The portion of text starting with "This VM is currently running..." and ending with "...select from the following options:" will be read directly from a startup message file that can be modified for use in different contexts (standalone installer vs. packaged with Origin VM).

The options presented to the user will come from the Workflows that register themselves with the installer at startup. This is described in greater detail in the [Installer Workflows](#installer-workflows) section.

#### Recording the System Configuration
When invoked, the oo-install utility will look for a configuration file at `~/.openshift/oo-install-cfg.yml`. Users can manually specify a config file location by passing an argument to oo-install. As users work with the utility, general information about the Origin system and specific information about the configuration choices the user is making will be recorded here. The Origin VM will be shipped with a default configuration file that describes the all-in-one Origin system running on the VM itself.

- - -

**NOTE**:  
The installer will also keep a logfile at `~/.openshift/oo-install-cfg.log`. This will be overwritten every time the installer is invoked.

- - -

##### Default Configuration
The default configuration file for the OpenShift VM will look something like this:

    ---
    Name: OpenShift Installer Configuration
    Vendor: OpenShift Origin Community
    Description: This is the configuration file for the OpenShift Installer.
    Version: 0.0.1
    Deployment:
      Hosts:
        - host: broker.mydomain.com
          roles:
          - broker
          - mqserver
          - dbserver
          ssh_host: broker
          user: root
        - host: node1.mydomain.com
          roles:
          - node
          ssh_host: node1
          user: root
      DNS:
        app_domain: example.com
    Subscription:
      type: none

The configuration file will always contain a Deployment section. This section is intended to store the complete OpenShift deployment in a way that can be used by any Workflow. The file will also start with a default Subscription section with type 'none' (more on this below, see [Subscriptions](#subscriptions). When a user begins working with a given Workflow, the answers to any Workflow-specific questions will be stored in the Workflows section in a subgroup that is keyed to [Installer Workflow](#installer-workflows) IDs.

    Workflows:
      <Workflow ID 1>:
        <Question ID 1>: Answer value 1
        <Question ID 2>: Answer value 2
      ...
      <Workflow ID N>:
        <Question ID N>: Answer value N

- - -

**Limitations**:  
For the first iteration of the installer, multiple hosts will only be supported for the Node role.

- - -

#### Installer Workflows
Each installation option presented by the installer will come from a different `Installer::Workflow` object defined in the installer codebase. The function of each Workflow is to do the following:

* Provide an installation option on the text-based installer main page
* Post questions to the user that are relevant to the Workflow and validate the answers to those questions
* Summarize the user's responses
* Trigger an automated installation based on the user's responses
* Report on the outcome of the installation

The Workflows will be instantiated from a config file located at `<gem_root>/conf/workflows.yml`. Any additional files needed by a workflow can either be provided as URLs (including git://) or in the workflow directory `<gem_root>/workflows/<workflow_id>`. As the user answers the questions for a given Workflow, their responses will be recorded in the `oo-install-cfg.yml` file in a section that is keyed to that Workflow.

The format for every workflow defined in `workflows.yml` is as follows:

    Name: The name of the workflow
    Type: A single value or list of workflow types
    Description: The text displayed by the installer in the workflow list
    ID: A distinct alphanumeric ID for the workflow, used in oo-install-cfg.yml
    WorkflowDir: Defaults to <gem_root>/workflows/<workflow_id>, but can be overriden to <gem_root>/workflows/<workflow_dir>
    SkipDeploymentCheck: Default is "N", specifies whether or not to ask the standard Deployment settings questions at the start of the workflow.
    Questions:
      - ID: The unique variable name to associate the response with
        Text: The text of the question to ask
        AnswerType: An answer type, see the section below for details.
      - Question 2...
      - Question 3...
    Executable: The full command to be executed to complete the Workflow.
    RemoteDeployment: "Y" or "N"; indicates whether the Executable is run locally or on the target system
    SubscriptionCheck: "Y" or "N"; indicates whether the workflow will utilize any of yum, Red Hat Network, or Red Hat Subscription Manager to acquire necessary packages
    NonDeployment: "Y" or "N"; used for information workflows that do not actually perform installations
    RequiredUtilities:
      - The command name of a utility (like 'puppet' or 'unzip') that should be checked for on the target host
      - Utility 2
      - Utility 3

##### Type
The installer will support three installation types: 'origin', 'origin_vm', and 'ose'. This determines which contexts this workflow will be listed with. For example, if a workflow's type is 'ose', then it will only appear in the menu of installation options when oo-install is invoked with the '-e' flag (for OpenShift Enterprise).

Type can either be a single value or a reference to a list of values to indicate which contexts will include this workflow.

##### SkipDeploymentCheck
By default, before the Workflow's questions are asked, the Installer always asks the user if they want to review and modify the settings from the [Deployment section](#default-configuration) of the configuration file. If the user wants to review the settings, they will see a series of screens that list the current info and give the user the opportunity to change that info. This is a valuable first step for most Workflows. However, for Workflows that only provide information to the user without doing any installation work, this flag can be set to skip the Deployment questions altogether.

##### Questions
After asking the user to verify the Deployment, the installer starts iterating through the Workflow questions. When an answer already exists in the `oo-install-cfg.yml` file, the installer will present the question using Highline's [answer_or_default()](http://highline.rubyforge.org/doc/classes/HighLine/Question.html#M000030) method. All answers are validated according to the AnswerType setting. A few special AnswerTypes will trigger specific test behaviors as well:

* **_role_** causes the system to offer the [Simplified Deployment Roles](#simplified-deployment-roles) as options.
* **_rolehost_** - This question must follow the **_role_** question; it presents a list of the hosts for the indicated role in the `oo-install-cfg.yml` file or automatically selects the host if only one is defined for that role.

##### Executable
In the "Executable" string, the keyword "`<workflow_path>`" will be automatically expanded to the full path of "`<gem_root>/workflows/<workflow_id>`". The answers to workflow questions can also be included using the notation "`<q:question_variable>`". For example, if one question had a Variable "openshift_role", its value could be used in the executable like this:

    <workflow_path>/installerific -role <q:openshift_role>

See ExecuteOnTarget for information on how paths are handled on a remote target system.

##### RemoteDeployment
When RemoteDeployment is "Y", the installer will implicitly copy the oo-install-cfg.yml file and any workflow-specific files to the remote host. They will be copied to the `$HOME/.openshift` directory and the `$HOME/.openshift/installer/<workflow_id>` directories respectively, and the installer will implicitly modify the Executable string accordingly on the remote system.

This flag also determines whether or not the [Deployment Check](#skipdeploymentcheck) will attempt to connect to target systems via SSH. A value of 'N' indicates that this Workflow is local only.

##### SubscriptionCheck
When set to "Y", the installer will ensure that the user has provided valid subscription information, either through the [configuration file](#default-configuration) file or through direct input. Read the section on Subscriptions for more information.

##### NonDeployment
When set to "Y", this overrides and forces SkipDeploymentCheck to "Y" and RemoteDeployment and SubscriptionCheck to "N". The purpose of a non-deployment workflow is strictly informational; the indicated executable is run and then the workflow is complete. This is useful for running bash scripts that summarize information about the local system or the installer configuration.

#### Installation Methodology
The specific installation methodology used by a given Workflow is entirely at the discretion of the Workflow's author. As long as the Workflow's "Executable" string results in the completion of the installation task using values from `oo-install-cfg.yml` (or exits with a non-zero error code in the event of a problem), the installer will function correctly. Three of the [Provided Workflows](#provided-workflows) will use [Puppet](http://puppetlabs.com/) and [hiera](http://docs.puppetlabs.com/hiera/1/puppet.html) to perform installations, but these tools only represent one method of extracting values from the config file and using them complete a Workflow.

#### Unattended Installations
The installer supports unattended installation for a given Workflow assuming the following requirements are met:

* The Deployment section of the `oo-install-cfg.yml` file is complete
* All of the questions for a given Workflow have answers in the config file as well
* None of the necessary SSH sessions require password authentication (with a caveat; see below)

To perform an unattended installation, the user invokes the installer with an additional argument:

    oo-install -w <workflow ID>

### Provided Workflows
This section contains the workflows that will be provided with the initial release of the installer.

Two of the workflows defined here are specific the OpenShift Origin VM. However, the remote system installation is intended to develop into a general purpose remote deployment option that is usable for Origin or Enterprise on Fedora, RHEL or any other yum- and RPM-friendly host.

#### Workflow: Use an Origin VM in a Distributed Deployment
The goal of this Workflow is to make it possible for a user to set up an entire distributed, multi-instance OpenShift system using multiple pre-built OpenShift VMs. By default each VM runs its own complete system, but this installation path turns off services and configures the remaining services to interact with other hosts. Whether the other hosts are Origin VMs or some other system instances is not important as long as they all meet the [Target System Requirements](#target-system-requirements).

    OpenShift Installer: Multi-Instance Deployment
    ----------------------------------------------

    What role should this VM fill in the Origin system?    
    <1> Database server	
    <2> Message queueing server
    <3> Broker
    <4> Node

    <esc> - Go to main menu

Once a user selects the role for this VM, the installer calls the Workflow Executable to configure the current Origin VM instance.

#### Workflow: Install a Role on a Remote System
This Workflow causes the Installer to copy data over to a target host and then configure that host to act as one of the [Simplified Deployment Roles](#simplified-deployment-roles) as specified in the [Deployment section](#default-configuration) of the installer configuration.

    OpenShift Installer: Remote System Setup

    Which role do you want to deploy?  
    <1> Database server  
    <2> Message queueing server  
    <3> Broker  
    <4> Node  

    <return> - Continue  
    <esc> - Go to main menu  

Once a role is selected, the connection details are read from the Deployment settings. If the Node role is selected and more than one Node is defined, the wizard will ask which instance to deploy.

When the target host is identified, the installer will attempt to connect to the remote system via SSH. Because this Workflow starts with confirmation of the Deployment settings, the installer can assume at this point that the target host meets the [Target System Requirements](#target-system-requirements).

#### Workflow: Install a Role on the Local System
This Workflow is almost identical to the [remote system installation](#workflow-install-a-role-on-a-remote-system), however, it does not require SSH access to any other system. The user indicates the Role that the current system should assume, and the [Workflow Executable](#executable) performs the necessary work on the local system to establish it in the overall OpenShift system.

    OpenShift Installer: Local System Setup

    Which role do you want to deploy?  
    <1> Database server  
    <2> Message queueing server  
    <3> Broker  
    <4> Node  

    <return> - Continue  
    <esc> - Go to main menu  

This Workflow is intended for networks where remote, SSH-based deployments from a single host are not possible. This scenario benefits from the portability of the `oo-install-cfg.yml` file; by copying this into each participating system, a user can bring up each piece of the OpenShift deployment without having to re-define the Role definitions on each host.

#### Workflow: Download Puppet Templates
This workflow exists simply to call attention to the various ways that users can gain access to the Puppet templates that are used by the installer. For OpenShift Origin, the output of this workflow may look like this:

    OpenShift Installer: Puppet Templates Download
    ----------------------------------------------

    The templates that are packaged with this Origin VM are located at:

        /root/puppet_templates.zip

    The latest versions of the puppet templates are always available on GitHub:

        https://github.com/openshift/puppet-openshift_origin

    <esc> - Go to main menu

The only technical requirement for Origin is that the VM packager will need to be instrumented to create and place the zip file referenced in the installer.

#### Workflow: Display VM Login Info
_(Origin VM only)_ This workflow simply displays the login information that the `oo-login` utility displayed by default.

    OpenShift Origin VM Login Details
    ---------------------------------

    To connect to this Origin VM, use the following information:

    OpenShift console: https://broker.openshift.local/console
    Console login:     admin
    Console password:  admin

    IP address: X.X.X.X
    SSH user:   openshift
    Password:   openshift

### Subscriptions
On Fedora-based systems, Installation Workflows can focus on using `yum` as their primary source for software package downloads. However, the situation is more complex on RHEL. Enterprise systems may still be configured to use `yum` directly, but there are other ways of establishing software channels that Installation Workflows must be aware of.

When a [Workflow configuration](#installer-workflows) indicates that a subscription check should be performed, this cues the installer to ask questions that will help to establish one of the four possible software channel sources:

* none - The Workflow can assume that all necessary packages are already available on the target system.
* yum - The Workflow will use `yum` to install packages, and will create new repo files under `/etc/yum.repos.d/` if necessary.
* rhn - The Workflow will use Red Hat Network to add necessary software channels.
* rhsm - The Workflow will use Red Hat Subscription Manager to add necessary software channels.

#### Getting and Storing Subscription Information
The "none" and "yum" subscription methods do not require the installer to store particularly sensitive data. However, connecting to RHN or the Red Hat Subscription manager will require the user to provide a username and password. The installer will provide the user with a few different ways of handling this. However, it is first important to understand the difference between _temporary_ and _configured_ subscription settings.

#### Temporary and Configured Subscription Settings
Storing subscription info in the `oo-install` [configuration file](#default-configuration) is convenient but inherently insecure. Taking subscription info at runtime is more secure, but limits the feasibility of unattended deployments. The approach taken by the installer will be to support _both_.

Supporting both methods means that the installer will offer to store subscription information, but will also accept values at runtime that will *replace* the configured values. The values supplied at runtime can either be provided as arguments to the `oo-install` command, or can be provided during the course of Workflow configuration.

#### Command-Line Options
The installer will support three command-line options relevant to subscription info:

* `-s --subscription-type [TYPE]` - 'none', 'yum', 'rhn' or 'rhsm'
* `-u --username USERNAME` - Red Hat Login username
* `-p --password PASSWORD` - Red Hat Login password

On the subject of [unattended installations](#unattended-installations): Other than the 'none' subscription type, each type will require additional information to work correctly. However, because this information is not as sensitive, it should be added to the installer configuration file.

#### Runtime Evaluation
When the user is manually stepping through a Workflow, the Subscription information is compiled and evaluated in this order:

1. Configuration file
2. Command line
3. User modifications at runtime

In other words, at runtime, config file subscription info can be superseded by command line arguments, and the user can trump both while configuring the selected Workflow.

Additionally, the installer will give the user the opportunity to save their runtime configurations back to the config file, _or to only use them for this run of the installer_. This behavior supports the need for security in some deployments.

When the installer is running unattended, the evaluation order is almost the same, but there is no opportunity for runtime modifications:

1. Configuration file
2. Command line

This does not guarantee that a given Workflow can run unattended; it is the Workflow's "responsibility" to provide questions for capturing and storing other necessary info for unattended operation.

#### Passing Subscription Settings to Workflow Executables
The runtime subscription settings can include a mixture of information that isn't necessarily stored in the installer configuration file. For this reason, a workflow executable can't inspect the 'Subscription' section of the installer configuration file (if the section is even there) to get at the data that it needs.

For this reason, subscription information is passed to the workflow executable as a set of environment variables. All environment variables set by the installer are prefixed with 'OO_INSTALL_' so they can be grepped out of the complete environment.

The installer will pass the following environment variables:

* OO_INSTALL_SUBSCRIPTION_TYPE - 'none', 'yum', 'rhn', or 'rhsm'
* OO_INSTALL_RH_USERNAME - Red Hat Login username for 'rhn' and 'rhsm' subscriptions
* OO_INSTALL_RH_PASSWORD - Red Hat Login password for 'rhn' and 'rhsm' subscriptions
* OO_INSTALL_REPOS_BASE - For 'yum'; the base URL for the OpenShift repositories
* OO_INSTALL_RHEL_REPO - For 'yum'; the URL for a RHEL repository
* OO_INSTALL_JBOSS_REPO_BASE - For 'yum'; the base URL for a JBoss repository
* OO_INSTALL_RHEL_OPTIONAL_REPO - For 'yum'; the URL for an unsupported RHEL "optional" repo
* OO_INSTALL_SM_REG_POOL - For 'rhsm'; pool ID for OpenShift subscription
* OO_INSTALL_SM_REG_POOL_RHEL - For 'rhsm'; pool ID for RHEL subscription
* OO_INSTALL_RHN_REG_ACTKEY - For 'rhn'; channel activation key

In practice, not all environment variables may be necessary for the Workflow. That is up to the workflow to decide. The installer passes along all of these values that have been set at runtime.

## Backwards Compatibility
This PEP describes functionality that was not previously available in OpenShift. From that perspective, backwards compatibility is not a factor in this design. However, anyone wishing to create a [Workflow](#installer-workflows) that leverages previously existing scripts will need to adjust them to read system configuration information from the installer's [configuration file](#recording-the-system-configuration) file.

## Rationale
The follow sections explain design considerations that were necessary to define reasonable bounds for this PEP.

### Text-Based Installer
The choice to use a text-based installer for this installer was two-fold. First off, per the [survey](https://www.openshift.com/blogs/survey-results-the-openshift-origin-unboxing-experience), the option of a more graphical installer was low in popularity. Second, being command-line invokable, the installer lends itself to use with [unattended installations](#unattended-installations).

### Workflows
The Workflow concept enables this OpenShift installer to be easily reconfigured different deployment scenarios. By providing different `workflow.yml` files, oo-install RPMs can be customized to different distributions and environments.

### Simplified Deployment Roles
In order to deliver a basic, functional deployment on one or more target hosts, the Workflows that initially ship with the installer will split all of the components of OpenShift into four basic roles, using the specific software as indicated:

* Role: Broker
    * Broker RPM
    * MCollective
    * BIND DNS Server
* Role: Node
    * Node RPM
    * MCollective
* Role: MQServer
    * ActiveMQ
    * MCollective
* Role: DBServer
    * MongoDB

This division of labor will not satisfy every deployment. The intent is to provide the administrator with a working OpenShift system that can be incrementally adjusted to suit the specific needs of a given environment.

### Target System Requirements
The following expectations are tested by the installer for every target system, with respect to relevant [workflow settings](#remotedeployment).

1. **For remote deployments only**, the target system must be running and accessible via SSH with the specified user and ssh port. If the SSH session requires password authentication, the installer will ask the user for the password and pass it through to the SSH authentication but will not record it in the `oo-install-cfg.yml` file. See [Unattended Installations](#unattended-installations) for more comments on this.
2. The installing user is root, or has sudo access.
3. The system has `yum` and can install packages from RPMs. The installer will perform tests to make this determination, leveraging the compatibility tool described in [this user story](https://trello.com/c/sYLNUE8d) when it becomes available.
