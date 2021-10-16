# Practical Guide on Writing a SELinux Module
This guide will show users how to write a basic SELinux policy module for the program htop (website: https://htop.dev), which is an interactive process manager that runs in a TTY or a terminal emulator. This guide was written using Fedora Linux 33, although it should work on any newer version of Fedora. Other Linux distributions that use SELinux should work as well, although the filesystem labels may be different and the final output modules shown here will not be the same for you.

## Prerequisites
1. On Fedora, you will need to install the following packages that provide us the `checkmodule` command, used for compiling SELinux modules, and the `semanage` command, used for labeling, respectively:

    * checkpolicy
    * policycoreutils-python-utils
    
    `$ sudo dnf install checkpolicy policycoreutils-python-utils`

2. Finally, this tutorial uses files contained in this directory, so make sure you clone the repository:
    
    `$ git clone https://github.com/noatsecure/tutorial-selinux.git`

3. Since we will be writing the SELinux module for htop, it should be installed as well:

    `$ sudo dnf install htop`

4. Optionally (and recommended) you can install the following package to gain access to the `seinfo` command, which is very useful:

    * setools-console

    `$ sudo dnf install setools-console`
    
## Step 1: Preparing *htop.te*
First, let's rename the *template.te* file to *htop.te*. This is required because the module name located on the first line of the file needs to match the filename, otherwise you will get an error. Try it if you don't believe me.

Next, open the *htop.te* file in your favorite text editor, and you will see:

```
module NAME 1.0.0;
########################################
require {
    attribute application_domain_type, application_exec_type, entry_type, exec_type, file_type, non_auth_file_type, non_security_file_type;
    class process { transition };
    role unconfined_r;
    type unconfined_t;
}
########################################
type NAME_t, application_domain_type;
type NAME_exec_t, application_exec_type, entry_type, exec_type, file_type, non_auth_file_type, non_security_file_type;
role unconfined_r types NAME_t;
type_transition unconfined_t NAME_exec_t: process NAME_t;
########################################
```
Perform a find and replace of all instances of `NAME` to `htop`.
   
<sub>***Note:** If a module with this same name (htop) already exists, this new policy will overwrite it. Be careful when naming your modules.*</sub>

```
module htop 1.0.0;
########################################
require {
    attribute application_domain_type, application_exec_type, entry_type, exec_type, file_type, non_auth_file_type, non_security_file_type;
    class process { transition };
    role unconfined_r;
    type unconfined_t;
}
########################################
type htop_t, application_domain_type;
type htop_exec_t, application_exec_type, entry_type, exec_type, file_type, non_auth_file_type, non_security_file_type;
role unconfined_r types htop_t;
type_transition unconfined_t htop_exec_t: process htop_t;
########################################
```

As a quick explanation of this file:

   * The first line describes the name and version of the SELinux policy module
   * Line 2 is a comment since it starts with `#`
   * The 'require' section is specifying all of the attributes, classes, roles, and types that are used in the policy module, similar to importing modules with Python. See **Step 3: Labeling** below for more information
   * `type htop_t, application_domain_type;` line defines a new type that will be used for the htop process with the 'application_domain_type' attribute
   * `type htop_exec_t, application_exec_type, entry_type, exec_type, file_type, non_auth_file_type, non_security_file_type;` defines another new type we will use to label the htop binary in /usr/bin
   * `role unconfined_r types htop_t;` defines the role 'htop_t' will inherit
   * `type_transition unconfined_t htop_exec_t: process htop_t;` allows our user, labeled as type 'unconfined_t' by default in Fedora, to start the 'htop_t' process via executing a file of type 'htop_exec_t'

## Step 2: Installing *htop.te* as a SELinux Policy Module
Let's now compile and install this module. You can use the supplied *semodcompile* shell script to do so:

   `$ sudo ./semodcompile htop.te`
  
<sub>***Note:** Compiling the module does not require root privileges but installing it does.*</sub>

Once the module has been compiled and installed, we now need to label the htop binary located in /usr/bin.  But what is a label?

## Step 3: Labeling
SELinux uses labels to define permissions, and you can view the label of any file and directory on the system by running the command:

   `$ ls -lZ /path/to/file_or_directory`

For example, the SELinux relevant information from the output of `ls -lZ /usr/bin/ls` will show us  `system_u:object_r:bin_t:s0`.
   * **system_u**: The user for this binary file is system_u
   * **object_r**: The role for this binary file is object_r 
   * **bin_t**:    The type for this binary file is bin_t

Documentation regarding users, roles, and types can be found in the quickstart guide in the Gentoo wiki ([link](https://wiki.gentoo.org/wiki/SELinux/Quick_introduction#Access_controls); [archive](https://web.archive.org/web/20210119022805/https://wiki.gentoo.org/wiki/SELinux/Quick_introduction)). You can see that the *htop.te* file above defines two new types at the following lines:

```
type htop_t, application_domain_type;
type htop_exec_t, application_exec_type, entry_type, exec_type, file_type, non_auth_file_type, non_security_file_type;
```

Here, `htop_exec_t` has the attributes `application_exec_type`, `file_type`, and `exec_type` among others, meaning this type is made for an executable file or binary. And the `htop_t` type will be used for the htop process when the program is running. For a more detailed explanation of SELinux attributes, see the Gentoo Wiki's article ([link](https://wiki.gentoo.org/wiki/SELinux/Tutorials/How_SELinux_controls_file_and_directory_accesses#How_does_SELinux_know_the_labels_of_files.3F); [archive](https://web.archive.org/web/*/https://wiki.gentoo.org/wiki/SELinux/Tutorials/How_SELinux_controls_file_and_directory_accesses)). You can run the shell script *label.sh* that will automatically perform the labeling, but also make sure to take a look at the command used in the script too for learning purposes.

Although you've now labelled /usr/bin/htop, it hasn't been applied yet. To do so, run the command:

   `$ sudo restorecon -Rv /usr/bin/htop`
   
And you should get the output message:

   `Relabeled /usr/bin/htop from system_u:object_r:bin_t:s0 to system_u:object_r:htop_exec_t:s0`

## Step 4: Starting htop
Now we can start generating permissions for htop. First, let's start the program by opening up a terminal emulator and running the command:

   `$ htop`

And we get the output:

   `bash: /usr/bin/htop: Permission denied`

This happens because *htop.te* has no permssions yet, so everything is denied by default. We can see these denials using the journalctl command:

   `$ sudo journalctl -b0 -g htop`

You'll see a AVC denials, but we can't do anything with this output yet. In order to generate rules that we can put into our *htop.te* SELinux policy module file, pipe the output of the journalctl command into audit2allow like so:

   `$ sudo journalctl -b0 -g htop | audit2allow -m x`

Here's a break down of these commands:
   
   | Command | Flag | Description |
   | --- | --- | --- |
   | journalctl | -b0 | Search events from only the last boot (current session) |
   | journalctl | -g htop | Grep for the term 'htop' |
   | audit2allow | -m x | Generate a SELinux policy module with the name 'x' |
   
   <sub>***Note:** With the audit2allow command, the module name for the '-m' flag can be anything at all. This is because we won't use the generated module it gives us, we're going to only be copying the rules.*</sub>

The output of the above command will show something like this:

```
module x 1.0;

require {
   type unconfined_t;
   type htop_t;
   class process transition;
}
 
#============= unconfined_t ==============
allow unconfined_t htop_t:process transition;
```
   
This output provides us with a standalone module we can use in addition to the *htop.te* one. This is great for making exceptions to SELinux policy modules that are already written and loaded, but we currently have no rules in our *htop.te* file. So instead, you could just copy everything starting from `require {` and place that into the *htop.te* file. But for now, let's just clear up a few things about this output:

   1. The `require {}` section specifies all of the attributes, roles, types, and classes used in our module. It's similar to importing modules in Python or libraries in C.
   2. By default, Fedora sets the user's type as 'unconfined_t'. The line `allow unconfined_t htop_t:process transition;` is allowing our user of type 'unconfined_t' to transition to the 'htop_t' type so we can start htop.

SELinux denials are logged as they happen. This means that if you want to generate all of the required permissions for a given program, you'll have to keep launching the program over and over and over until all rules have been generated. However, there is a better way.

## Step 5: setenforce
It's much easier to generate all of the rules a program needs at one time rather than one by one. To do so, first change SELinux from enforcing mode to permissive mode:

    $ sudo setenforce 0

**Note: THIS IS DANGEROUS! Setting SELinux to permissive means it will log denials but not enforce them. You always need to remember to either run `$ sudo setenforce 1` after you're done or to reboot which will reset SELinux back to enforcing mode.**

Now run htop again and it will start up perfectly fine since SELinux is now only logging and not enforcing any permissions. Go ahead and press the F5 button to change htop into tree view too, which will create a htop configuration file at $HOME/.config/htop.


## Step 6: Adding Initial Rules
Okay, now let's generate our SELinux rules again:

    $ sudo journalctl -b0 -g htop | audit2allow -m x

And the output will be much longer than last time:

<sub>***Note:** Your output may not be exactly the same, which shouldn't be an issue. If you have any problems, then copy my output.*</sub>

```
module htop 1.0;

require {
    type var_run_t;
    type locale_t;
    type system_dbusd_t;
    type kernel_t;
    type policykit_t;
    type user_devpts_t;
    type passwd_file_t;
    type systemd_resolved_t;
    type sysfs_t;
    type local_login_t;
    type chronyd_t;
    type devicekit_power_t;
    type unconfined_dbusd_t;
    type sssd_var_lib_t;
    type init_t;
    type ld_so_cache_t;
    type sshd_t;
    type udev_t;
    type htop_t;
    type unconfined_t;
    type systemd_userdbd_t;
    type config_home_t;
    type syslogd_t;
    type sysctl_kernel_t;
    type usr_t;
    type systemd_logind_t;
    type firewalld_t;
    type ld_so_t;
    type rtkit_daemon_t;
    type proc_t;
    type auditd_t;
    type tty_device_t;
    type htop_exec_t;
    type lib_t;
    type NetworkManager_t;
    type xserver_t;
    type user_tty_device_t;
    type etc_t;
    class process { noatsecure rlimitinh siginh transition };
    class file { create entrypoint execute getattr map open read write };
    class chr_file { append getattr ioctl read write };
    class fd use;
    class dir { add_name create getattr open read search write };
    class lnk_file read;
    class unix_stream_socket { connect create };
    class sock_file write;
}

#============= htop_t ==============
allow htop_t NetworkManager_t:dir { getattr open read search };
allow htop_t NetworkManager_t:file { getattr open read };
allow htop_t NetworkManager_t:lnk_file read;
allow htop_t auditd_t:dir { getattr open read search };
allow htop_t auditd_t:file { getattr open read };
allow htop_t auditd_t:lnk_file read;
allow htop_t chronyd_t:dir { getattr open read search };
allow htop_t chronyd_t:file { getattr open read };
allow htop_t chronyd_t:lnk_file read;
allow htop_t config_home_t:dir { add_name create write };
allow htop_t config_home_t:file { create getattr open write };
allow htop_t devicekit_power_t:dir { getattr open read search };
allow htop_t devicekit_power_t:file { getattr open read };
allow htop_t devicekit_power_t:lnk_file read;
allow htop_t etc_t:file { getattr open read };
allow htop_t firewalld_t:dir { getattr open read search };
allow htop_t firewalld_t:file { getattr open read };
allow htop_t firewalld_t:lnk_file read;
allow htop_t htop_exec_t:file { entrypoint execute map read };
allow htop_t init_t:dir { getattr open read search };
allow htop_t init_t:file { getattr open read };
allow htop_t init_t:lnk_file read;
allow htop_t kernel_t:dir { getattr open read search };
allow htop_t kernel_t:file { getattr open read };
allow htop_t ld_so_cache_t:file { getattr map open read };
allow htop_t ld_so_t:file { execute map read };
allow htop_t lib_t:file { execute getattr map open read };
allow htop_t lib_t:lnk_file read;
allow htop_t local_login_t:dir { getattr open read search };
allow htop_t local_login_t:file { getattr open read };
allow htop_t local_login_t:lnk_file read;
allow htop_t locale_t:file { getattr map open read };
allow htop_t locale_t:lnk_file read;
allow htop_t passwd_file_t:file { getattr open read };
allow htop_t policykit_t:dir { getattr open read search };
allow htop_t policykit_t:file { getattr open read };
allow htop_t policykit_t:lnk_file read;
allow htop_t proc_t:dir read;
allow htop_t proc_t:file { getattr open read };
allow htop_t proc_t:lnk_file read;
allow htop_t rtkit_daemon_t:dir { getattr open read search };
allow htop_t rtkit_daemon_t:file { getattr open read };
allow htop_t rtkit_daemon_t:lnk_file read;
allow htop_t self:dir { getattr open read search };
allow htop_t self:file { getattr open read };
allow htop_t self:lnk_file read;
allow htop_t self:unix_stream_socket { connect create };
allow htop_t sshd_t:dir { getattr open read search };
allow htop_t sshd_t:file { getattr open read };
allow htop_t sshd_t:lnk_file read;
allow htop_t sssd_var_lib_t:sock_file write;
allow htop_t sysctl_kernel_t:dir search;
allow htop_t sysctl_kernel_t:file { getattr open read };
allow htop_t sysfs_t:file { getattr open read };
allow htop_t sysfs_t:lnk_file read;
allow htop_t syslogd_t:dir { getattr open read search };
allow htop_t syslogd_t:file { getattr open read };
allow htop_t syslogd_t:lnk_file read;
allow htop_t system_dbusd_t:dir { getattr open read search };
allow htop_t system_dbusd_t:file { getattr open read };
allow htop_t system_dbusd_t:lnk_file read;
allow htop_t systemd_logind_t:dir { getattr open read search };
allow htop_t systemd_logind_t:file { getattr open read };
allow htop_t systemd_logind_t:lnk_file read;
allow htop_t systemd_resolved_t:dir { getattr open read search };
allow htop_t systemd_resolved_t:file { getattr open read };
allow htop_t systemd_resolved_t:lnk_file read;
allow htop_t systemd_userdbd_t:dir { getattr open read search };
allow htop_t systemd_userdbd_t:file { getattr open read };
allow htop_t systemd_userdbd_t:lnk_file read;
allow htop_t tty_device_t:chr_file getattr;
allow htop_t udev_t:dir { getattr open read search };
allow htop_t udev_t:file { getattr open read };
allow htop_t udev_t:lnk_file read;
allow htop_t unconfined_dbusd_t:dir { getattr open read search };
allow htop_t unconfined_dbusd_t:file { getattr open read };
allow htop_t unconfined_dbusd_t:lnk_file read;
allow htop_t unconfined_t:dir { getattr open read search };
allow htop_t unconfined_t:fd use;
allow htop_t unconfined_t:file { getattr open read };
allow htop_t unconfined_t:lnk_file read;
allow htop_t user_devpts_t:chr_file { append getattr ioctl read write };
allow htop_t user_tty_device_t:chr_file getattr;
allow htop_t usr_t:file { getattr open read };
allow htop_t var_run_t:lnk_file read;
allow htop_t xserver_t:dir { getattr open read search };
allow htop_t xserver_t:file { getattr open read };
allow htop_t xserver_t:lnk_file read;

#============= unconfined_t ==============
allow unconfined_t htop_t:process { noatsecure rlimitinh siginh transition };
```

Let's go ahead and place this into our *htop.te* file so it looks like so:

```
module htop 1.0.0;
########################################
require {
    attribute application_domain_type, application_exec_type, entry_type, exec_type, file_type, non_auth_file_type, non_security_file_type;
    class process { transition };
    role unconfined_r;
    type unconfined_t;
}
########################################
type htop_t, application_domain_type;
type htop_exec_t, application_exec_type, entry_type, exec_type, file_type, non_auth_file_type, non_security_file_type;
role unconfined_r types htop_t;
type_transition unconfined_t htop_exec_t: process htop_t;
########################################
require {
    type var_run_t;
    type locale_t;
    type system_dbusd_t;
    type kernel_t;
    type policykit_t;
    type user_devpts_t;
    type passwd_file_t;
    type systemd_resolved_t;
    type sysfs_t;
    type local_login_t;
    type chronyd_t;
    type devicekit_power_t;
    type unconfined_dbusd_t;
    type sssd_var_lib_t;
    type init_t;
    type ld_so_cache_t;
    type sshd_t;
    type udev_t;
    type htop_t;
    type unconfined_t;
    type systemd_userdbd_t;
    type config_home_t;
    type syslogd_t;
    type sysctl_kernel_t;
    type usr_t;
    type systemd_logind_t;
    type firewalld_t;
    type ld_so_t;
    type rtkit_daemon_t;
    type proc_t;
    type auditd_t;
    type tty_device_t;
    type htop_exec_t;
    type lib_t;
    type NetworkManager_t;
    type xserver_t;
    type user_tty_device_t;
    type etc_t;
    class process { noatsecure rlimitinh siginh transition };
    class file { create entrypoint execute getattr map open read write };
    class chr_file { append getattr ioctl read write };
    class fd use;
    class dir { add_name create getattr open read search write };
    class lnk_file read;
    class unix_stream_socket { connect create };
    class sock_file write;
}

#============= htop_t ==============
allow htop_t NetworkManager_t:dir { getattr open read search };
allow htop_t NetworkManager_t:file { getattr open read };
allow htop_t NetworkManager_t:lnk_file read;
allow htop_t auditd_t:dir { getattr open read search };
allow htop_t auditd_t:file { getattr open read };
allow htop_t auditd_t:lnk_file read;
allow htop_t chronyd_t:dir { getattr open read search };
allow htop_t chronyd_t:file { getattr open read };
allow htop_t chronyd_t:lnk_file read;
allow htop_t config_home_t:dir { add_name create write };
allow htop_t config_home_t:file { create getattr open write };
allow htop_t devicekit_power_t:dir { getattr open read search };
allow htop_t devicekit_power_t:file { getattr open read };
allow htop_t devicekit_power_t:lnk_file read;
allow htop_t etc_t:file { getattr open read };
allow htop_t firewalld_t:dir { getattr open read search };
allow htop_t firewalld_t:file { getattr open read };
allow htop_t firewalld_t:lnk_file read;
allow htop_t htop_exec_t:file { entrypoint execute map read };
allow htop_t init_t:dir { getattr open read search };
allow htop_t init_t:file { getattr open read };
allow htop_t init_t:lnk_file read;
allow htop_t kernel_t:dir { getattr open read search };
allow htop_t kernel_t:file { getattr open read };
allow htop_t ld_so_cache_t:file { getattr map open read };
allow htop_t ld_so_t:file { execute map read };
allow htop_t lib_t:file { execute getattr map open read };
allow htop_t lib_t:lnk_file read;
allow htop_t local_login_t:dir { getattr open read search };
allow htop_t local_login_t:file { getattr open read };
allow htop_t local_login_t:lnk_file read;
allow htop_t locale_t:file { getattr map open read };
allow htop_t locale_t:lnk_file read;
allow htop_t passwd_file_t:file { getattr open read };
allow htop_t policykit_t:dir { getattr open read search };
allow htop_t policykit_t:file { getattr open read };
allow htop_t policykit_t:lnk_file read;
allow htop_t proc_t:dir read;
allow htop_t proc_t:file { getattr open read };
allow htop_t proc_t:lnk_file read;
allow htop_t rtkit_daemon_t:dir { getattr open read search };
allow htop_t rtkit_daemon_t:file { getattr open read };
allow htop_t rtkit_daemon_t:lnk_file read;
allow htop_t self:dir { getattr open read search };
allow htop_t self:file { getattr open read };
allow htop_t self:lnk_file read;
allow htop_t self:unix_stream_socket { connect create };
allow htop_t sshd_t:dir { getattr open read search };
allow htop_t sshd_t:file { getattr open read };
allow htop_t sshd_t:lnk_file read;
allow htop_t sssd_var_lib_t:sock_file write;
allow htop_t sysctl_kernel_t:dir search;
allow htop_t sysctl_kernel_t:file { getattr open read };
allow htop_t sysfs_t:file { getattr open read };
allow htop_t sysfs_t:lnk_file read;
allow htop_t syslogd_t:dir { getattr open read search };
allow htop_t syslogd_t:file { getattr open read };
allow htop_t syslogd_t:lnk_file read;
allow htop_t system_dbusd_t:dir { getattr open read search };
allow htop_t system_dbusd_t:file { getattr open read };
allow htop_t system_dbusd_t:lnk_file read;
allow htop_t systemd_logind_t:dir { getattr open read search };
allow htop_t systemd_logind_t:file { getattr open read };
allow htop_t systemd_logind_t:lnk_file read;
allow htop_t systemd_resolved_t:dir { getattr open read search };
allow htop_t systemd_resolved_t:file { getattr open read };
allow htop_t systemd_resolved_t:lnk_file read;
allow htop_t systemd_userdbd_t:dir { getattr open read search };
allow htop_t systemd_userdbd_t:file { getattr open read };
allow htop_t systemd_userdbd_t:lnk_file read;
allow htop_t tty_device_t:chr_file getattr;
allow htop_t udev_t:dir { getattr open read search };
allow htop_t udev_t:file { getattr open read };
allow htop_t udev_t:lnk_file read;
allow htop_t unconfined_dbusd_t:dir { getattr open read search };
allow htop_t unconfined_dbusd_t:file { getattr open read };
allow htop_t unconfined_dbusd_t:lnk_file read;
allow htop_t unconfined_t:dir { getattr open read search };
allow htop_t unconfined_t:fd use;
allow htop_t unconfined_t:file { getattr open read };
allow htop_t unconfined_t:lnk_file read;
allow htop_t user_devpts_t:chr_file { append getattr ioctl read write };
allow htop_t user_tty_device_t:chr_file getattr;
allow htop_t usr_t:file { getattr open read };
allow htop_t var_run_t:lnk_file read;
allow htop_t xserver_t:dir { getattr open read search };
allow htop_t xserver_t:file { getattr open read };
allow htop_t xserver_t:lnk_file read;

#============= unconfined_t ==============
allow unconfined_t htop_t:process { noatsecure rlimitinh siginh transition };
```
    
<sub>***Note:** You can have multiple `require` sections in a given SELinux policy module file.*</sub>

Now recompile the *htop.te* file using the same command we used in step 2:

    $ sudo ./semodcompile htop.te

## Step 7: Adding Missing Rules
Let's try to launch htop again:

    $ htop
    
Unfortunately, the following error appears:

    htop: error while loading shared libraries: libncursesw.so.6: cannot open shared object file: Permission denied
    
We can also check the logs for any new AVC errors within the past minute:

    $ sudo journalctl --since "1 minute ago" -g htop | audit2allow -m x
    
And there's `Nothing to do`. So how do we solve this error? There are certain AVC denials for programs and system libraries that are not shown by default (see: [link](https://docs.fedoraproject.org/en-US/Fedora/13/html/Security-Enhanced_Linux/sect-Security-Enhanced_Linux-Fixing_Problems-Possible_Causes_of_Silent_Denials.html); [archive](https://web.archive.org/web/20160927042741/https://docs.fedoraproject.org/en-US/Fedora/13/html/Security-Enhanced_Linux/sect-Security-Enhanced_Linux-Fixing_Problems-Possible_Causes_of_Silent_Denials.html)). To show these AVC denials, run the command:

    $ sudo semodule -DB

Re-run `$ htop` and view the logs again using `$ sudo journalctl --since "1 minute ago" -g htop | audit2allow -m x`. Let's take the following rules from the output and add them to *htop.te*:

```
allow htop_t config_home_t:dir search;
allow htop_t config_home_t:file read;
allow htop_t device_t:dir search;
allow htop_t devpts_t:dir search;
allow htop_t etc_t:dir { getattr search };
allow htop_t home_root_t:dir search;
allow htop_t lib_t:dir search;
allow htop_t locale_t:dir search;
allow htop_t proc_t:dir { getattr open search };
allow htop_t root_t:dir search;
allow htop_t sssd_public_t:dir search;
allow htop_t sssd_var_lib_t:dir search;
allow htop_t sysctl_t:dir search;
allow htop_t sysfs_t:dir search;
allow htop_t user_home_dir_t:dir search;
allow htop_t user_home_t:dir search;
allow htop_t usr_t:dir { getattr search };
allow htop_t var_lib_t:dir search;
allow htop_t var_run_t:dir search;
allow htop_t var_t:dir search;
```

You also will need to add the following entries to the `require` section:

```
type device_t;
type devpts_t;
type home_root_t;
type root_t;
type sssd_public_t;
type sysctl_t;
type user_home_dir_t;
type user_home_t;
type var_lib_t;
type var_t;
```
    
Your *htop.te* file should now look like this:

```
module htop 1.0.0;
########################################
require {
    attribute application_domain_type, application_exec_type, entry_type, exec_type, file_type, non_auth_file_type, non_security_file_type;
    class process { transition };
    role unconfined_r;
    type unconfined_t;
}
########################################
type htop_t, application_domain_type;
type htop_exec_t, application_exec_type, entry_type, exec_type, file_type, non_auth_file_type, non_security_file_type;
role unconfined_r types htop_t;
type_transition unconfined_t htop_exec_t: process htop_t;
########################################
require {
    type var_run_t;
    type locale_t;
    type system_dbusd_t;
    type kernel_t;
    type policykit_t;
    type user_devpts_t;
    type passwd_file_t;
    type systemd_resolved_t;
    type sysfs_t;
    type local_login_t;
    type chronyd_t;
    type devicekit_power_t;
    type unconfined_dbusd_t;
    type sssd_var_lib_t;
    type init_t;
    type ld_so_cache_t;
    type sshd_t;
    type udev_t;
    type htop_t;
    type unconfined_t;
    type systemd_userdbd_t;
    type config_home_t;
    type syslogd_t;
    type sysctl_kernel_t;
    type usr_t;
    type systemd_logind_t;
    type firewalld_t;
    type ld_so_t;
    type rtkit_daemon_t;
    type proc_t;
    type auditd_t;
    type tty_device_t;
    type htop_exec_t;
    type lib_t;
    type NetworkManager_t;
    type xserver_t;
    type user_tty_device_t;
    type etc_t;
    type device_t;
    type devpts_t;
    type home_root_t;
    type root_t;
    type sssd_public_t;
    type sysctl_t;
    type user_home_dir_t;
    type user_home_t;
    type var_lib_t;
    type var_t;
    class process { noatsecure rlimitinh siginh transition };
    class file { create entrypoint execute getattr map open read write };
    class chr_file { append getattr ioctl read write };
    class fd use;
    class dir { add_name create getattr open read search write };
    class lnk_file read;
    class unix_stream_socket { connect create };
    class sock_file write;
}

#============= htop_t ==============
allow htop_t NetworkManager_t:dir { getattr open read search };
allow htop_t NetworkManager_t:file { getattr open read };
allow htop_t NetworkManager_t:lnk_file read;
allow htop_t auditd_t:dir { getattr open read search };
allow htop_t auditd_t:file { getattr open read };
allow htop_t auditd_t:lnk_file read;
allow htop_t chronyd_t:dir { getattr open read search };
allow htop_t chronyd_t:file { getattr open read };
allow htop_t chronyd_t:lnk_file read;
allow htop_t config_home_t:dir { add_name create write };
allow htop_t config_home_t:file { create getattr open write };
allow htop_t devicekit_power_t:dir { getattr open read search };
allow htop_t devicekit_power_t:file { getattr open read };
allow htop_t devicekit_power_t:lnk_file read;
allow htop_t etc_t:file { getattr open read };
allow htop_t firewalld_t:dir { getattr open read search };
allow htop_t firewalld_t:file { getattr open read };
allow htop_t firewalld_t:lnk_file read;
allow htop_t htop_exec_t:file { entrypoint execute map read };
allow htop_t init_t:dir { getattr open read search };
allow htop_t init_t:file { getattr open read };
allow htop_t init_t:lnk_file read;
allow htop_t kernel_t:dir { getattr open read search };
allow htop_t kernel_t:file { getattr open read };
allow htop_t ld_so_cache_t:file { getattr map open read };
allow htop_t ld_so_t:file { execute map read };
allow htop_t lib_t:file { execute getattr map open read };
allow htop_t lib_t:lnk_file read;
allow htop_t local_login_t:dir { getattr open read search };
allow htop_t local_login_t:file { getattr open read };
allow htop_t local_login_t:lnk_file read;
allow htop_t locale_t:file { getattr map open read };
allow htop_t locale_t:lnk_file read;
allow htop_t passwd_file_t:file { getattr open read };
allow htop_t policykit_t:dir { getattr open read search };
allow htop_t policykit_t:file { getattr open read };
allow htop_t policykit_t:lnk_file read;
allow htop_t proc_t:dir read;
allow htop_t proc_t:file { getattr open read };
allow htop_t proc_t:lnk_file read;
allow htop_t rtkit_daemon_t:dir { getattr open read search };
allow htop_t rtkit_daemon_t:file { getattr open read };
allow htop_t rtkit_daemon_t:lnk_file read;
allow htop_t self:dir { getattr open read search };
allow htop_t self:file { getattr open read };
allow htop_t self:lnk_file read;
allow htop_t self:unix_stream_socket { connect create };
allow htop_t sshd_t:dir { getattr open read search };
allow htop_t sshd_t:file { getattr open read };
allow htop_t sshd_t:lnk_file read;
allow htop_t sssd_var_lib_t:sock_file write;
allow htop_t sysctl_kernel_t:dir search;
allow htop_t sysctl_kernel_t:file { getattr open read };
allow htop_t sysfs_t:file { getattr open read };
allow htop_t sysfs_t:lnk_file read;
allow htop_t syslogd_t:dir { getattr open read search };
allow htop_t syslogd_t:file { getattr open read };
allow htop_t syslogd_t:lnk_file read;
allow htop_t system_dbusd_t:dir { getattr open read search };
allow htop_t system_dbusd_t:file { getattr open read };
allow htop_t system_dbusd_t:lnk_file read;
allow htop_t systemd_logind_t:dir { getattr open read search };
allow htop_t systemd_logind_t:file { getattr open read };
allow htop_t systemd_logind_t:lnk_file read;
allow htop_t systemd_resolved_t:dir { getattr open read search };
allow htop_t systemd_resolved_t:file { getattr open read };
allow htop_t systemd_resolved_t:lnk_file read;
allow htop_t systemd_userdbd_t:dir { getattr open read search };
allow htop_t systemd_userdbd_t:file { getattr open read };
allow htop_t systemd_userdbd_t:lnk_file read;
allow htop_t tty_device_t:chr_file getattr;
allow htop_t udev_t:dir { getattr open read search };
allow htop_t udev_t:file { getattr open read };
allow htop_t udev_t:lnk_file read;
allow htop_t unconfined_dbusd_t:dir { getattr open read search };
allow htop_t unconfined_dbusd_t:file { getattr open read };
allow htop_t unconfined_dbusd_t:lnk_file read;
allow htop_t unconfined_t:dir { getattr open read search };
allow htop_t unconfined_t:fd use;
allow htop_t unconfined_t:file { getattr open read };
allow htop_t unconfined_t:lnk_file read;
allow htop_t user_devpts_t:chr_file { append getattr ioctl read write };
allow htop_t user_tty_device_t:chr_file getattr;
allow htop_t usr_t:file { getattr open read };
allow htop_t var_run_t:lnk_file read;
allow htop_t xserver_t:dir { getattr open read search };
allow htop_t xserver_t:file { getattr open read };
allow htop_t xserver_t:lnk_file read;
#
#######################################
# Below are the new rules we just added
#######################################
allow htop_t config_home_t:dir search;
allow htop_t config_home_t:file read;
allow htop_t device_t:dir search;
allow htop_t devpts_t:dir search;
allow htop_t etc_t:dir { getattr search };
allow htop_t home_root_t:dir search;
allow htop_t lib_t:dir search;
allow htop_t locale_t:dir search;
allow htop_t proc_t:dir { getattr open search };
allow htop_t root_t:dir search;
allow htop_t sssd_public_t:dir search;
allow htop_t sssd_var_lib_t:dir search;
allow htop_t sysctl_t:dir search;
allow htop_t sysfs_t:dir search;
allow htop_t user_home_dir_t:dir search;
allow htop_t user_home_t:dir search;
allow htop_t usr_t:dir { getattr search };
allow htop_t var_lib_t:dir search;
allow htop_t var_run_t:dir search;
allow htop_t var_t:dir search;

#============= unconfined_t ==============
allow unconfined_t htop_t:process { noatsecure rlimitinh siginh transition };
```

Recompile the *htop.te* module (`$ sudo ./semodcompile htop.te`) and execute the htop command in your terminal emulator again:

    $ htop
    
Success, it finally works. Now that we have htop working, let's make sure to change SELinux back into enforcing mode:

    $ sudo setenforce 1
    
At this point, we have a lot of allow statements in our *htop.te*, most of which are not needed. Tightening SELinux policy modules is part experience and part trial and error; taking htop for instance, ask yourself if htop really needs access to any systemd or NetworkManager related file or directory. When you want to test, simply comment out the lines using `#`, recompile the module, and try launching it again. If it launches and all features, at least the ones you use, are working, then those permissions are not necessary and should be removed.

The *htop.te* file located in this repository has the minimal set of rules required to run htop if you want to check your *htop.te* against mine. Additionally, there is no harm in having multiple `require {}` sections or even having a long list of users, roles, classes, and types in a given `require {}` section, but to clean up the SELinux policy module I also provide a Python script, *semodgenreq* that outputs a list of only the users, roles, classes, and types allowed in your policy. 

## Summary
To write a SELinux policy module for a given program, you must first define new types for both the process and executable binary of the program, and additionally, define a type_transition so that your user has permission to launch the program's executable binary. Up to this point, all of this was provided in the *template.te* file that you can use to generate more SELinux policy modules.

Next, we compiled the *htop.te* module and defined a label for the program's binary. We used the `restorecon` command to apply the label, and after this, we temporarily set SELinux to permissive mode so that we could launch the program and use its features. This generated logs in `journalctl` that we piped to `audit2allow`, giving us the allowances that we placed in *htop.te*. After running `$ sudo semodule -DB` to show all AVC denials, we added the last set of allowances required to launch htop, and now the final steps were to remove extraneous allowances in order to properly tighten the module.

Overall, this guide glosses over important information such as understanding type_transitions, attributes, roles, types, and users among other things, which you should to review. However, you should now have a basic understanding of how to write a SELinux policy for a desktop program, which I assure you is easier than it first appears. Good luck.
