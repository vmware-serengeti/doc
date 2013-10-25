# Serengeti Troubleshooting Notes
The Serengeti CLI reports invalid parameters and unsupported cluster specifications. 

The Serengeti Web service layer stores the user input in a meta-database, which is PostgreSQL. The user input is verified in the Web service layer again, to make sure the user input is consistent at the system level. When an error occurs, users can check the Serengeti Web Service log in the /opt/serengeti/logs/serengeti*.log file.

The Serengeti provision engine validates vSphere resources, places virtual machine at vSphere, and leverages Chef to configure Hadoop cluster.  

Because the cluster provisioning and configuration takes some time to finish, Serengeti launches a backend process for each task. Users can check the progress or find error information at /opt/serengeti/logs/ironfan*.log

##  Related Documentation
See the VMware vSphere Big Data Extensions Administrator's and User's Guide for additional troubleshooting tips and solutions.

##  CLI setup error for NoClassDefFoundError
After download remote CLI and unzip the file, user should not change the file structure, since serengeti-*.jar file defined dependency for jars in cli/lib.

If you find following error, please make sure the serengeti-cli-*.jar file is located in the unzip directory, and the "unzip dir"/lib contains the original jar files.

    [serengeti@10 ~]$ java -jar serengeti-cli-*.jar
    Exception in thread "main" java.lang.NoClassDefFoundError: org/springframework/shell/Bootstrap
    Caused by: java.lang.ClassNotFoundException: org.springframework.shell.Bootstrap
            at java.net.URLClassLoader$1.run(URLClassLoader.java:202)
            at java.security.AccessController.doPrivileged(Native Method)
            at java.net.URLClassLoader.findClass(URLClassLoader.java:190)
            at java.lang.ClassLoader.loadClass(ClassLoader.java:306)
            at sun.misc.Launcher$AppClassLoader.loadClass(Launcher.java:301)
            at java.lang.ClassLoader.loadClass(ClassLoader.java:247)
    Could not find the main class: org.springframework.shell.Bootstrap. Program will exit.

##  CLI parameter error
Check the error message and fix the invalid parameter.
![(attachements/image001.jpg)](https://github.com/vmware-serengeti/doc/raw/master/wiki/attachements/image001.jpg)
##  CLI command warning
If the cluster specification contains un-suggested definition, the CLI displays a warning message and waits for user input before continuing or interrupting the task.

If user chooses to continue, Serengeti will finish the Hadoop creation process.
![(attachements/image002.jpg)](https://github.com/vmware-serengeti/doc/raw/master/wiki/attachements/image002.jpg)
##  Where to find CLI logs
At current CLI execution directory, you can find file cli-debug.log, which is the CLI log file.
##  Where to find Serengeti server logs
Serengeti administrator can check server logs at Serengeti server VM. The log file /opt/serengeti/logs/serengeti*.log contains Web Service component logs. 

For each long run task, Serengeti starts one backend task. The task log files are located in /opt/serengeti/logs/ironfan*.log.

If the cluster task failed, the error message also shows where to find the backend task log.
##  Where to find Serengeti deployment logs
After deploy Serengeti, you may check the first boot error and log files /opt/serengeti/logs/serengeti-firstboot.err and /opt/serengeti/logs/serengeti-firstboot.log.
##  Bootstrap failure for 401 unauthorized error
Cluster creates or resume failed for some VM’s “Bootstrap Failed”. Following output is a sample.

    FAILED 100%

    node group: master,  instance number: 1
    roles:[hadoop_namenode, hadoop_jobtracker]
      NAME                   IP            STATUS            TASK
      -----------------------------------------------------------
      cluster_conf-master-0  X.X.X.X  Bootstrap Failed
    
    node group: worker,  instance number: 3
    roles:[hadoop_datanode, hadoop_tasktracker]
      NAME                   IP            STATUS         TASK
      --------------------------------------------------------
      cluster_conf-worker-1  X.X.X.X  Bootstrap Failed
      cluster_conf-worker-2  X.X.X.X  Bootstrap Failed
      cluster_conf-worker-0  X.X.X.X  Service Ready
    
    node group: client,  instance number: 1
    roles:[hive, hadoop_client, pig]
      NAME                   IP            STATUS         TASK
      --------------------------------------------------------
      cluster_conf-client-0  X.X.X.X  Service Ready

cluster cluster_conf resume failed: Bootstrapping VM failed. you can get task failure details from serengeti server log at: /opt/serengeti/logs/task/22

In the log file, you may find that the bootstrap failed for 401 unauthorized errors.

One reason for this issue is the skewed clocks. You may check the time setting at the failed VM and Serengeti Server. If the time difference is big, it’s the problem.

To fix this issue, you need to correct the clock with ntp update, i.e. all ESXi hosts sync to the same NTP server. So far, the provisioned VM will sync to their ESXi hosts automatically.

After the clock is corrected, you may run cluster create resume to finish the cluster provision.
##  Cluster creation failed for resource pool, datastore, or network is not added.
Add a vSphere resource pool, datastore, or network into the Serengeti system before you create a cluster.
![(attachements/image003.jpg)](https://github.com/vmware-serengeti/doc/raw/master/wiki/attachements/image003.jpg)
##  Cluster creation failed for parameter error
The error message points out the invalid parameter. The following example illustrates a message returned when a resource pool name specified in the command is not defined in the Serengeti system.
![(attachements/image004.jpg)](https://github.com/vmware-serengeti/doc/raw/master/wiki/attachements/image004.jpg)
##  Cluster creation failed for cluster spec validation error
Serengeti supports an advanced feature that customizes cluster definition with "--specFile" in cluster creation command. The spec file is the customized cluster definition. The cluster definition should match following rules.

  - Roles in each node group should not be empty.
  -	Supported Hadoop roles can be listed using "distro list" command. Unsupported Hadoop roles should not appear in the cluster spec file.
  -	Users do not have to write all the required roles for one Hadoop cluster. The required roles will added into the cluster spec with default configuration.
  -	The hadoop_namenode and hadoop_jobtracker are master roles. The total instance number must be 1 and only 1 for the distros based on hadoop 1.x.
  -	Node group instance numbers must not be negative numbers.

The CLI displays the following error message, when these rules are not followed

![(attachements/image005.jpg)](https://github.com/vmware-serengeti/doc/raw/master/wiki/attachements/image005.jpg)

You can find detailed information in the /opt/serengeti/logs/serengeti.log file.
##  How to get final cluster spec
Serengeti has a default cluster spec, with three node groups. 
-  Master group, which runs Hadoop namenode and Hadoop jobtracker and contains 1 node. 
-  Worker group, which runs Hadoop datanode, and Hadoop tasktracker and contains 3 nodes. 
-  Client group, which runs Hadoop client, pig, hive and hive_server and contains 1 node.

If you change the default value, through the command line or if you customize cluster spec file, you can export the cluster spec to verify if you get a cluster as your expectation.
##  The JSON file located at Serengeti Server conf directory is not sample spec
The file /opt/serengeti/conf/template-cluster-spec.json at Serengeti server VM is not a sample spec file. User should not use this file as a specification file to create your own cluster. 

User should find the sample specification at downloaded Serengeti CLI package.
##  Cluster creation/resume/resize failed before 50% progress, and show error message "Can not alloc resource for vm."
The operation might have failed because the system is out of resources or because the vSphere resource name specified is wrong.

Use vSphere Client to verify that all virtual machines that you created have Hadoop installed.

At the present time, it is not possible to release vSphere system resources by using Serengeti, so you must either rename the resources through the vSphere Client, or delete the problem resources through Serengeti.
##  Virtual machine cannot get ip address.
In the log file /opt/senrengeti/logs/serengeti.log, if the failed virtual machine cannot get IP address, the reason might be a network configuration error.
-  Verify that the vSphere port group still has enough port for new virtual machine.
-	If the network is using static IP address check that the IP address range is not duplicated with other virtual machine.
-	If the network is using DHCP, check that the DHCP still has an IP address to allocate for the new virtual machine. 
-	After you have corrected the error, resume the cluster creation or rerun the previous command if it's not cluster creation.
##  vCenter Server connection failure
If the CLI shows an error message such as "VC login error massages: vSphere login failed. Reason: 5 connections fail to login.". Please check the vCenter Server status or network to make sure vCenter Server is reachable.

##  How to change the log level at Serengeti
The Serengeti system uses log4j to log out debug or error messages. You can change the /opt/serengeti/conf/log4j.properties file to customize the log level. The backend task will also consumes this log level.
##  Cluster task failed after 50%, with distro downloading failure
If cluster creation fails with bootstrap failure and for the failed virtual machine, the downloading distro failed. The probably reason is that the distro server is down. 

You can view error messages at /opt/Serengeti/logs/ironfan.log. For example:

![(attachements/image009.jpg)](https://github.com/vmware-serengeti/doc/raw/master/wiki/attachements/image009.jpg)

To fix this problem, you can reset the Serengeti services by running serengeti-stop-services.sh and serengeti-start-services.sh. 

NOTE: make sure the script is running correctly. If any process is not stopped successfully, you must stop that service manually before you run the serengeti-start-services.sh script to start services. You can then resume the cluster creation or rerun the previous cluster creation command.
##  Cluster task failed after 50%, with failed to startup Hadoop service.
If a cluster fails with a bootstrap failure and for failed virtual machine, the Hadoop service has not started. The error probably is because user defined too small virtual machine box. The CPU or memory is not enough to startup the service. Check your cluster spec file.

In this case, you can resize the cluster with a larger CPU/memory configuration.
##  Cluster task failure after 50%, with bootstrap failure.
If the reason for the virtual machine bootstrap failure is unclear, the Chef server might be down, because Serengeti uses Chef to bootstrap the cluster.

For this type of error, reset the Serengeti services through serengeti-stop-services.sh and serengeti-start-services.sh and then resume the cluster creation process.
##	When should you use cluster resume.
If cluster creation fails and you fix the error, you can run the "cluster create --name <name> --resume" command to resume the cluster creation. 

But if Serengeti contains some wrong resource information for vCenter Server, the better solution is to remove the cluster, and recreate vSphere resources, recreate cluster. 
##	Hadoop TESTDFSIO failed for IOException.
For example, there might be an error when running command:

    hadoop jar hadoop-test-*.jar TestDFSIO -write -nrFiles 10 -fileSize 1000

get following exceptions:
    
    12/05/22 06:24:07 INFO mapred.JobClient: map 70% reduce 20%
    12/05/22 06:24:17 INFO mapred.JobClient: map 70% reduce 23%
    12/05/22 06:24:38 INFO mapred.JobClient: map 80% reduce 23%
    12/05/22 06:24:40 INFO mapred.JobClient: Task Id : 
    attempt_201205220612_0001_m_000008_0, Status : FAILED
    org.apache.hadoop.ipc.RemoteException: java.io.IOException: File 
    /benchmarks/TestDFSIO/io_data/test_io_8 could only be replicated to 0 nodes, 
    instead of 1 at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.getAdditionalBlock
    (FSNamesystem.java:1556)
    at org.apache.hadoop.hdfs.server.namenode.NameNode.addBlock
    (NameNode.java:696)
    at sun.reflect.GeneratedMethodAccessor7.invoke(Unknown Source)
    at sun.reflect.DelegatingMethodAccessorImpl.invoke(Unknown Source)
    at java.lang.reflect.Method.invoke(Unknown Source)
    at org.apache.hadoop.ipc.RPC$Server.call(RPC.java:563)
    at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:1388)
    at org.apache.hadoop.ipc.Server$Handler$1.run(Server.java:1384)
    at java.security.AccessController.doPrivileged(Native Method)
    at javax.security.auth.Subject.doAs(Unknown Source)
    at org.apache.hadoop.security.UserGroupInformation.doAs
    (UserGroupInformation.java:1093)
    at org.apache.hadoop.ipc.Server$Handler.run(Server.java:1382)
    at org.apache.hadoop.ipc.Client.call(Client.java:1066)

The error occurs because the datanode has too small disk size. When running Hadoop job, not enough disk is available for the data.

There is no command to resize the virtual machine disk size directly. For such error, you must delete current cluster, and recreate cluster with larger disk configuration. See the Hadoop documentation for information about changing disk size in the physical environment.

http://hadoop.apache.org/common/docs/stable

http://wiki.apache.org/hadoop/

http://wiki.apache.org/hadoop/TroubleShooting
##	Pig or Hive runtime error
Useful links:
https://cwiki.apache.org/confluence/display/PIG/Index%3bjsessionid=F1A021E3B0499C050216372F1926CD6B

http://docs.1h.com/Hive_Troubleshooting

## No disk space left on Serengeti Server
This issue is described on [SERENGETI-531](https://issuetracker.springsource.com/browse/SERENGETI-531). Solution:
* Reboot and login to Serengeti Server with user serengeti
* # clean up Serengeti logs and temp files
* $ rm -rf /opt/serengeti/logs/task/* /opt/serengeti/logs/*.log /opt/serengeti/tomcat6/logs/*.log
* # compact couchdb file used by Chef Server
* $ curl -H "Content-Type: application/json" -X POST http://localhost:5984/chef/_compact
* There should be more disk space on Serengeti Server now. Let's restart Serengeti services:
* $ serengeti-stop-services.sh; serengeti-start-services.sh

## IP will change unexpectedly if the IP allocation by DHCP on serengeti
Restart serengeti server, update yum repo file and manifest file related to ip of web server.
