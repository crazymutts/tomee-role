TomEE Ansible Role
==================

Helps to manage the lifecycle and ansible-enabled configuration of 
Apache TomEE on a Linux server. This role encourages multiple instances 
(one shared TomEE installation with multiple jvms). 
   * Create one or more TomEE binary installations
   * Each TomEE binary installation can have one or more JVMs
   * Each JVM can use a different JRE if desired/needed
   * Add TomEE instances (by dirname/instancename) to systemd
   * The TomEE user can use systemctl to start, stop, and restart the JVMs
     via either sudo or polkit (when available)
     For example:  systemctl restart tomee8@jvm01.service
     ("tomee8" is the TomEE binary installation, jvm01 is one of the JVMs)

Future features that would be nice:
   * selinux awareness
   * while polkit is super awesome on RedHat-like systems, flag for sudoers only
   * ansible control of connection pools

Test results from November 2022:
~~~
ubuntu 22.04 - works
ubuntu 20.04 - works
fedora 36 - works
centos stream 9 - works
rocky 9 - works
debian 11 - DOES NOT WORK - cannot test (presence of selinux, ufw, etc.)
~~~

Requirements
------------

THIS GOES IN A SHARED ROLES AREA IF YOU WANT TO USE A DEEP-PLAYBOOK 
DIRECTORY STRUCTURE.  EXAMPLES: /usr/share/ansible/roles or
~/.ansible/roles

Ensure java JRE is installed and you know the JRE_HOME value for it
If running Debian/Ubuntu, ensure the "acl" package is installed
Ensure SELinux is off
Configure the system's firewall for the TomEE port(s)

  SPECIAL NOTE FOR UBUNTU/DEBIAN ONLY
  * for some ansible functions (file copying, specifically) to work,
    the "acl" package MUST be installed! This is not just for the 
    TomEE role--it's more of an ansible thing

  JAVA
  * know it's JRE_HOME path (hint: look at /usr/lib/jvm directory
    and pick something from there)
    - Package examples:
          Debian/Ubuntu: openjdk-17-jre-headless
          RedHat like:   java-17-openjdk-headless
    - JRE_HOME examples:
          Debian/Ubuntu: /usr/lib/jvm/java-17-openjdk-amd64
          RedHat like:   /usr/lib/jvm/jre-17

  FIREWALL (firewalld in RedHat like, ufw in Debian/Ubuntu)
  * if a firewall is running, you MUST configure whatever port

  TOMEE BINARY .TAR.GZ
  * You MUST put a TomEE .tar.gz file in the role's tomee/files
    subdirectory and put that filename in the defaults/main.yml 
    file

  SELINUX
  * SELinux should be off or permissive
    - to do SELinux with ansible, recommend loading:
         Debian/Ubuntu: selinux-utils, 
                        python3-selinux, & 
                        selinux-policy-default
         RedHat like:   libselinux-utils
    - if you want to run TomEE with SELinux enabled, please enhance 
      the role to support SELinux

Role Variables
--------------

        jvm_name: "myinstance"
        # =====  !!  PICK ONLY ONE JRE_HOME  !!  =====
        # =====                                  =====
        # ===== Ansible will NOT check to ensure =====
        # ===== it is on the system. You have to =====
        # ===== set it correctly yourself.       =====
        # ===== For RHEL/Rocky/Fedora like Linux =====
        jre_home: "/usr/lib/jvm/jre-17"
        #jre_home: "/usr/lib/jvm/jre-11"
        # =====   For Ubuntu/Debian like Linux   =====
        #jre_home: "/usr/lib/jvm/java-17-openjdk-amd64"
        #jre_home: "/usr/lib/jvm/java-11-openjdk-amd64"
        # ============================================

        # set java memory and other misc. java settings in catalina_opts
        catalina_opts: "-Xms128m -Xmx1024m"

        # The "SHUTDOWN" port, default 8005
        shutdown_port: 8005

        # Various network ports - must be unique for each role instance!
        http_port: 8080
        https_port: 8443
        ajp_port: 8009

        http_timeout: 20000
        https_max_threads: 150
        default_host: localhost
        tomcat_thread_pool: 150
        tomcat_min_spare_threads: 4

In the defaults/main.yml file, the following are defined as well. These 
should undoubtedly be set only once on the server, so look them over, 
set as desired, but do not use them in your role instance definitions.

        tomee_binary: "apache-tomee-8.0.12-webprofile.tar.gz"
        tomee_dirname: "tomee8"
        tomee_install_dir: "/opt/apache"

        catalina_home: "{{ tomee_install_dir }}/{{ tomee_dirname }}"
        tomee_owner: "tomcat"
        tomee_group: "tomcat"
        catalina_base: "{{ tomee_install_dir }}/{{ jvm_name }}/jvms"

Not defined are the xml template variables that can contain the filename 
of a .j2 template for TomEE/Tomcat configuration files.

        server_xml_template
        tomcat_users_xml_template

If not defined in the play, a default template included with the role 
will be used. If defined in the play, a template MUST be put into 
ansible's normal templates subdirectory or wherever ansible will look 
for a .j2 template.

In an earlier pre-release version, there was a variable named "state" 
that was deprecated in favor of installing the necessary configurations 
for systemd and polkit to control the jvms instead. This way, ansible 
can control starting, stopping and restarting the jvms via the normal 
builtin systemd capabilities.

Dependencies
------------

No other roles are required. 

There are OS dependencies.
  * Ansible itself needs the acl package in Ubuntu/Debian
  * An unprivileged userid and group to run TomEE must have been created
  * For now, SELinux should be off (or at most, permissive)
  * A JRE (or JDK) must have been installed 
  * polkit must have been installed for a normal user to control systemctl

Example Playbook
----------------
	---
	- hosts: ghost
	  become: true
	  gather_facts: yes

	# NOTES ---- NOTES ---- NOTES ---- NOTES ---- NOTES ---- NOTES ---- NOTES
	#  - name: Ensure acl is isntalled on Ubuntu/Debian systems (for ansible)
	#  - name: Ensure SELinux is off (or at most, permissive)

	  tasks:

	# #####################################################################
	# Required system IDs
	# #####################################################################

	  - name: create tomee group
	    group:
	      name: tomcat
	      state: present

	  - name: create tomee user
	    user:
	      name: tomcat
	      state: present
	      create_home: yes
	      group: tomcat
	      shell: /bin/bash
	      comment: "TomEE user"

	# #####################################################################
	# PREREQUISITE FOR REDHAT-LIKE: polkit / polkitd (so tomee_user can 
	# use systemctl to control starting, stopping, and restarting the JVMs
	# #####################################################################

	  - name: ensure polkit is installed (via dnf)
	    dnf:
	      pkg: polkit
	    when: ansible_os_family == "Rocky" or ansible_os_family == "RedHat"

	# #####################################################################
	# Java JRE
	# #####################################################################

	  - name: ensure JRE is installed (via apt)
	    apt:
	      name: openjdk-17-jre-headless
	      update_cache: yes
	    when: ansible_os_family == "Debian" or ansible_os_family == "Ubuntu"

	  - name: ensure JRE is installed (via dnf)
	    dnf:
	      pkg: java-17-openjdk-headless
	    when: ansible_os_family == "Rocky" or ansible_os_family == "RedHat"

	# #####################################################################
	# Firewall ports, if needed
	# #####################################################################

	  - name: gather service facts
	    ansible.builtin.service_facts:

	  - name: if Debian/Ubuntu and ufw is running, open port(s)
	    ufw:
	      rule: allow
	      proto: tcp
	      port: "{{ item }}"
	    loop:
	      - "8081"
	      - "8082"
	      - "8083"
	    when: ( ansible_facts.services["ufw"] is defined and ansible_facts.services["ufw"].state == "running" ) or ( ansible_facts.services["ufw.service"] is defined and ansible_facts.services["ufw.service"].state == "running" )

	  - name: Open firewall port if firewalld is running
	    firewalld:
	      port: "{{ item }}"
	      permanent: yes
	      state: enabled
	      immediate: yes
	    loop:
	      - "8081/tcp"
	      - "8082/tcp"
	      - "8083/tcp"
	    when: ( ansible_facts.services["firewalld.service"] is defined and ansible_facts.services["firewalld.service"].state == "running" )

	# #####################################################################
	# TomEE instances
	# #####################################################################

	# NOTE: using include_role instead of a "roles:" invocation
	# effectively "resets" the default values (most specifically
	# for the template overrides such as server_xml_template)

	  - name: TomEE instance 1 jvm 1
	    include_role:
	      name: tomee
	    vars:
	      tomee_binary: "apache-tomee-8.0.12-webprofile.tar.gz"
	      tomee_dirname: "tomee8"
	      tomee_install_dir: "/opt/apache"
	      jvm_name: "jvm01"
	      # =====  !!  PICK ONLY ONE JRE_HOME  !!  =====
	      # =====                                  =====
	      # ===== Ansible will NOT check to ensure =====
	      # ===== it is on the system. You have to =====
	      # ===== set it correctly yourself.       =====
	      # ===== For RHEL/Rocky/Fedora like Linux =====
	      jre_home: "/usr/lib/jvm/jre"
	      #jre_home: "/usr/lib/jvm/jre-17"
	      #jre_home: "/usr/lib/jvm/jre-11"
	      # =====   For Ubuntu/Debian like Linux   =====
	      #jre_home: "/usr/lib/jvm/java-17-openjdk-amd64"
	      #jre_home: "/usr/lib/jvm/java-11-openjdk-amd64"
	      # ============================================
	      catalina_opts: "-Xms256m -Xmx1024m"
	      http_port: 8081
	      https_port: 8444
	      https_max_threads: 100
	      ajp_port: 8010
	      shutdown_port: 8006
	      default_host: localhost
	      #server_xml_template: "citmvmqsd04-server.xml.j2"
	      #tomcat_users_xml_template: "citmvmqsd04-tomcat-users.xml.j2"

	  - name: TomEE instande 1 jvm 2
	    include_role:
	      name: tomee
	    vars:
	      tomee_binary: "apache-tomee-8.0.12-webprofile.tar.gz"
	      tomee_dirname: "tomee8"
	      tomee_install_dir: "/opt/apache"
	      jvm_name: "jvm02"
	      # =====  !!  PICK ONLY ONE JRE_HOME  !!  =====
	      # =====                                  =====
	      # ===== Ansible will NOT check to ensure =====
	      # ===== it is on the system. You have to =====
	      # ===== set it correctly yourself.       =====
	      # ===== For RHEL/Rocky/Fedora like Linux =====
	      jre_home: "/usr/lib/jvm/jre"
	      #jre_home: "/usr/lib/jvm/jre-17"
	      #jre_home: "/usr/lib/jvm/jre-11"
	      # =====   For Ubuntu/Debian like Linux   =====
	      #jre_home: "/usr/lib/jvm/java-17-openjdk-amd64"
	      #jre_home: "/usr/lib/jvm/java-11-openjdk-amd64"
	      # ============================================
	      catalina_opts: "-Xms768m -Xmx8192m"
	      http_port: 8082
	      https_port: 8445
	      https_max_threads: 125
	      ajp_port: 8011
	      shutdown_port: 8007
	      default_host: localhost

	  - name: TomEE instance 2 jvm 1
	    include_role:
	      name: tomee
	    vars:
	      tomee_binary: "apache-tomee-8.0.13-webprofile.tar.gz"
	      tomee_dirname: "tomee8013"
	      tomee_install_dir: "/opt/apache"
	      jvm_name: "i2jvm01"
	      # =====  !!  PICK ONLY ONE JRE_HOME  !!  =====
	      # =====                                  =====
	      # ===== Ansible will NOT check to ensure =====
	      # ===== it is on the system. You have to =====
	      # ===== set it correctly yourself.       =====
	      # ===== For RHEL/Rocky/Fedora like Linux =====
	      jre_home: "/usr/lib/jvm/jre"
	      #jre_home: "/usr/lib/jvm/jre-17"
	      #jre_home: "/usr/lib/jvm/jre-11"
	      # =====   For Ubuntu/Debian like Linux   =====
	      #jre_home: "/usr/lib/jvm/java-17-openjdk-amd64"
	      #jre_home: "/usr/lib/jvm/java-11-openjdk-amd64"
	      # ============================================
	      catalina_opts: "-Xms1024m -Xmx2048m"
	      http_port: 8083
	      https_port: 8446
	      https_max_threads: 125
	      ajp_port: 8012
	      shutdown_port: 8008
	      default_host: localhost

When the playbook is created and run, it sets up a serviceable TomEE
installation.

It is the ansible playbook's responsibility to meet all prerequisite 
requirements (ID & group, packages, java, etc.), do the firewall if a 
firewall is running, and then after invoking the tomee role to install 
and set things up, enabling the systemd service (if auto-start upon 
reboot is required) and starting and stopping any JVM(s).

If the prerequisites are met (and they should be done before invoking
the TomEE role, each invocation of the role will:

1. Create the binary installation directory (catalina_home) and the 
   jvm instance directory (catalina_base) underneath catalina_home
~~~
	/
	|--- tomee_install_dir (i.e., /opt/apache)
	  |
	  |-------- catalina home (i.e., /opt/apache/tomee8)
	  |    |
	  |    |------- bin  (binaries/scripts/etc.)
	  |    |------- conf (distribution-provided conf files)
	  |    |------- lib  (all the libraries)
	  |    |------- logs (empty)
	  |    |------- temp (empty)
	  |    |------- webapps (distribution-provided apps)
	  |    |------- work (empty)
	  |    |
	  |    |------- jvms  (/opt/apache/tomee8/jvms)
	  |        |
	  |        |---------- jvm01 (catalina_base: /opt/apache/tomee8/jvms/jvm01)
	  |        |    |
	  |        |    |------ bin (setenv.sh & tomcat-juli.jar)
	  |        |    |------ conf (our configuration)
	  |        |    |------ lib (empty)
	  |        |    |------ logs (logs)
	  |        |    |------ temp (use as you see fit)
	  |        |    |------ webapps (ROOT & docs. The managers are linked)
	  |        |    |------ work (use as you see fit)
	  |        |
	  |        |---------- jvm02 (catalina_base: /opt/apache/tomee8/jvms/jvm02)
	  |             |
	  |             |------ bin
	  |             |------  conf
	  |             |------- lib
	  |             |------- logs
	  |             |------- temp
	  |             |------- webapps
	  |             |------- work
	  |
	  |-------- another catalina home (i.e., /opt/apache/tomee8013)
	       |
	       |------- bin  (binaries/scripts/etc.)
	       |------- conf (distribution-provided conf files)
	       |------- lib  (all the libraries)
	       |------- logs (empty)
	       |------- temp (empty)
	       |------- webapps (distribution-provided apps)
	       |------- work (empty)
	       |
	       |------- jvms  (/opt/apache/tomee8013/jvms)
	           |
	           |---- i2jvm01 (catalina_base: /opt/apache/tomee8013/jvms/i2jvm01)
	             |
	             |------ bin (setenv.sh & tomcat-juli.jar)
	             |------ conf (our configuration)
	             |------ lib (empty)
	             |------ logs (logs)
	             |------ temp (use as you see fit)
	             |------ webapps (ROOT & docs. The managers are linked)
	             |------ work (use as you see fit)
~~~
2. Check to see if TomEE is installed already by looking in the lib
   subdirectory of each catalina_home for a tomee-common-*.jar file 
   and set a flag (called tomee_is_installed) as true or false

3. If it has been determined that TomEE is not installed, copy the 
   TomEE installation .tar.gz from the ansible host's files 
   directory to the target and install it (under catalina_home)

4. Create the jvm instance directory structure (catalina_base)

5. Populate the jvm instance with the minimum required jvm 
   instance configuration (/bin/setenv.sh, /bin/tomcat-juli.jar, 
   a complete copy of the contents of the TomEE installation's conf 
   directory)

6. Customize conf/server.xml and conf/tomcat-users.xml

7. Enable the jvm instance to run the Tomcat manager application 
   (via http, go to the server URL with the http_port followed by 
   /manager to access it, http://tomee.domain.com:8081/manager is
   an example)

8. For each catalina_base, which are separate binary installations 
   of TomEE, make a systemd unit file in /etc/systemd/system called 
   <tomee_dir>@.service so that each jvm is enabled as 
   <tomee_dir>@<jvm_name>.service. This makes it so something like:
             systemctl stop tomee8@jvm01.service
   will stop that JVM. The example given in this README makes three 
   JVMs:
                  tomee8@jvm01.service
                  tomee8@jvm02.service
                  tomee8013@i2jvm01.service
   and each is controlled individually using the normal systemd 
   systemctl mechanism. Think of it this way:
     * Each is a JVM
     * BEFORE the at-sign (@) is the TomEE binary installation
     * AFTER the at-sign (@) is the JVM name
   It is structured this way so that each TomEE binary installation 
   can have JVMs of the same name if needed. It is imagined that 
   marketing@online.service, marketing@printads.service, and 
   marketing@socialmedia.service could be the names of three 
   separate JVMs all using the marketing TomEE binary installation. 
   Each would have their own configurations and each can be 
   controlled separately with systemctl (so it's super easy to 
   do with Ansible).

9. The tomee_owner and tomee_group is granted the ability to control 
   the JVMs via systemctl. This is done with a polkit rule dropped 
   into /etc/polkit-1/rules.d on systemds with a recent enough polkit 
   installation (such as RedHat-like systems) or a sudo rule if not.

Again, the ansible playbook should enable the systemd service(s) and 
start the JVM(s) if required.

License
-------

BSD

Author Information
------------------

Role created as a learning exercise by Doug S. Started in Q3 of 2022.
Despite this section being all about contact information, please do not 
contact me.
