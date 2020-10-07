# RMTLOG

**z/OS Remote Logging Facility v1.1**

Original Author: John C. Miller, john@jmit.com 09/12/2016

## Overview

RMTLOG is a z/OS started task that transmits z/OS console hardcopy messages to an external RFC 3164/5424 compliant Syslog appliance or server.  Its purpose is to enhance z/OS security and auditability by establishing near real time logging to an external syslog server. Such a remote syslog appliance/server can be any device that is compliant with RFC 3164 and/or RFC 5424.  Unix and Linux servers are typically packaged with syslogd or syslog-ng, both of which are RFC compliant.

RMTLOG also demonstrates useful assembler and z/OS programming techniques including:

- Access Register programming;
- Extended MCS consoles, including the MSCOPER and MCSOPMSG macros for managing consoles and console message units;
- TCPIP sockets programming;
- MVS start/stop interface (Handle START and STOP operator cmds.)

The two principal components required for z/OS remote logging are the RMTLOG z/OS started task running on z/OS, and the remote Syslog server compliant with RFC 3164 and/or 5424.


## Quick Install Instructions:

1. Modify the ASM file member (or JCL below) to reflect your local libraries, and run.  The job should complete with a condition code of 0.

```
//Z90008A JOB (0),JMILLER,CLASS=A,NOTIFY=&SYSUID,MSGCLASS=T,REGION=64M
/*JOBPARM L=999999
//*-------------------------------------------------------------------
//* COMPILE AND LINK RMTLOG
//*-------------------------------------------------------------------
//* 12/09/2010 John C. Miller
//*-------------------------------------------------------------------
//* RMTLOG Remote logging program copyright 2010-2017 John C. Miller.
//*-------------------------------------------------------------------
//* Fix the jobcard as appropriate.  Then set the following values to
//* something appropriate for your site, and run this job.
//*-------------------------------------------------------------------
// JCLLIB ORDER=USER03.RMTLOG.V01B      <-- Name of this install PDS.
// SET  INSTLIB=USER03.RMTLOG.V01B      <-- Name of this install PDS.
// SET  LINKLIB=SYS2.LINKLIB            <-- Name of target loadlib.
// SET  SEZACMAC=TCPIP.SEZACMAC         <-- Your TCPIP SEZACMAC.
// SET  SEZATCP=TCPIP.SEZATCP           <-- Your TCPIP SEZATCP.
//*-------------------------------------------------------------------
//IPADDR   EXEC ASM$,M=IPADDR
//JULIAN   EXEC ASM$,M=JULIAN
//RECON    EXEC ASM$,M=RECON
//RLCLOS   EXEC ASM$,M=RLCLOS
//RLINIT1  EXEC ASM$,M=RLINIT1
//RLINIT2  EXEC ASM$,M=RLINIT2
//RLWRITE  EXEC ASM$,M=RLWRITE
//RMTLOG   EXEC ASM$,M=RMTLOG
/*
```

2. Create and customize a `PARMS` member (see below for details).

```conf
# Sample RMTLOG parameter file:
#
SERVER syslog.jmit.com
PROTO TCP
PORT  514
TIMESRC MESSAGE
CNAME RLG01
PRI  035
RETRYINT 15
RETRIES 0
SIZE  512
QLIMIT 524288
```

3. Create a started PROC.  You may need to create a STARTED RACF profile for the new started task.  Use member `RMTLOG$` (or below) in this PDS as an example, changing the data set names as appropriate.

```jcl
//RMTLOG PROC
//*--------------------------------------------------------------------
//* RMTLOG - Remote Syslog Task.
//*--------------------------------------------------------------------
//* 05/29/2010 John C. Miller.
//*--------------------------------------------------------------------
//LOGGER   EXEC PGM=RMTLOG,TIME=1440
//STEPLIB  DD DISP=SHR,DSN=SYS2.LINKLIB2
//PARMS    DD DISP=SHR,DSN=SYS1.PARMLIB(RMTLOG00)
//SYSTCPD  DD DISP=SHR,DSN=SYS2.TCPIP(TCPDATA)
//SYSPRINT DD SYSOUT=*
```

4. Configure and start a syslog server as described below.
5. Start RMTLOG with the command: `S RMTLOG`

You should see z/OS messages in the log file of the SYSLOG server.

**Note:** It does not matter which is started first, the RMTLOG started task,
the remote syslog server, or TCPIP.  If RMTLOG is started first, it will buffer z/OS syslog messages while it continues retrying the TCPIP socket init, and while waiting for the remote syslog server to start responding.

If RMTLOG is started before TCPIP and the remote syslog server, be sure that the `RETRYINT` and `RETRIES` settings are set appropriately in the `PARMS` member.


## How it Works

### INITIALIZATION:

RMTLOG is an APF authorized started task.  Upon initialization, RMTLOG establishes itself as a software only extended MCS console, with options specified to receive the z/OS HARDCOPY message set.  The RMTLOG started
task then sees all hardcopy messages.

The RMTLOG task runs with settings specified in a parameter file that is allocated to the PARMS DDNAME.  See Appendix B for details on the parameter file statements.

RMTLOG runs continuously until it is stopped.  It stays in a disabled wait until it is given something to do:  Either process a new Message Data Block (syslog message), process a stop or modify command, or send a message to the remote syslog server.

See Appendix A for the started task JCL and Appendix B for the Parameter file parameters and format.

## THE SYSLOG SERVER:

The syslog server is a server that accepts UDP datagrams or TCP connections, and writes the payload of these packets to a file as specified in RFC 3164 and/or RFC 5424.  For more information on the Syslog protocol (RFC 5424), see http://tools.ietf.or

## OPERATION:

The RMTLOG server runs as a started task, and should be started as early as possible during the IPL process, prior to TCPIP initialization.  In such a case, RMTLOG will capture and buffer SYSLOG messages while it
waits for TCPIP to initialize.  RMTLOG will deliver those buffered messages to the remote SYSLOG server once it is reachable via TCPIP.

**Start RMTLOG:**

To start the RMTLOG task, enter the z/OS console command: `S RMTLOG` or specify a similar START command in the `PARMLIB(COMMNDxx)` member, or in system automation software if used.

Notes when starting RMTLOG before TCPIP is up:

- RMTLOG can be started prior to TCP/IP initialization.  In such a case, the RETRYINT and RETRIES parameters in the PARMS member should be set to values that allow for an adequate retry period.
- The SIZE and QLIMIT parameters must be large enough to allow RMTLOG to queue all the hardcopy messages that are issued before TCPIP is initialized, and the remote SYSLOG server can be reached.

**Stop RMTLOG:**

To stop the RMTLOG task, issue the z/OS STOP command: `P RMTLOG`

**Communications or syslog server outages:**

If communications disruptions to the syslog server occur, or if the syslog server itself has an outage, RMTLOG will continue to queue z/OS messages up to the limits that are defined by the QLIMIT and SIZE parameters of the PARMS file. Once communications are restored, the queued messages are sent to the syslog server.

**Priority:**

The RMTLOG started task should be given a high enough dispatching priority to ensure that it continues passing Syslog records, and does not have console messages backed up.  RMTLOG should not need to be marked as non-swappable.

## Other Uses

Log processing tools such as Splunk can be used to view and process log data from a variety of hosts.  By using RMTLOG to get z/OS syslog messages onto a Linux or Unix system, tools such as Splunk can easily be used to examine z/OS messages.

## Future Enhancements

Some possible future enhancements to RMTLOG include:

1. Allow IP address for Syslog server;
2. Enhance MCS and TCP/IP error handling;
3. Specify multiple Syslog servers for failover.
4. Allow directives in the parameter file to send certain types of records to different servers if desired.
5. Allow directives in the parameter file to exclude or include certain
   records based on some kind of pattern matching;
6. Allow dynamic changing of parameters in the PARMS member either by allowing changes to be submitted via a MODIFY command, or by allowing the PARMS parameter member to be changed and then refreshed.
7. Establish ESTAE environment to make RMTLOG more bullet-proof.
8. Optimize the code to reduce the instruction path for each syslog record processed.  Currently the data gets copied twice; the program would be more efficient if it only moved the data once.

## Files in this repo:

- ASM      - Assemble all modules and load module for rmtlog.
- ASM$     - Assemble proc.
- IPADDR   - Support module for RMTLOG.
- JULIAN   - Support module for RMTLOG.
- PARMS    - Sample PARMS member for RMTLOG.
- RACFUSR  - Sample CLIST to set up RACF user, STARTED profile, etc.
- RECON    - Support module for RMTLOG.
- RLCLOS   - Support module for RMTLOG.
- RLCOMM   - Support module for RMTLOG.
- RLINCL   - Support module for RMTLOG.
- RLINIT1  - Support module for RMTLOG.
- RLINIT2  - Support module for RMTLOG.
- RLWRITE  - Support module for RMTLOG.
- RLWRITEL - Support module for RMTLOG.
- RMTLOG   - Main RMTLOG module.
- RMTLOG$  - Sample PROC for RMTLOG started task.
- RMTLOG00
- STR      - String manipulation macro for RMTLOG.

# Appendix

## Appendix A   Started PROCEDURE

```jcl
//RMTLOG PROC
//*--------------------------------------------------------------------
//* RMTLOG - Remote Syslog Task.
//*--------------------------------------------------------------------
//* 05/29/2010 John C. Miller.
//*--------------------------------------------------------------------
//LOGGER   EXEC PGM=RMTLOG,TIME=1440             <=== (1)
//STEPLIB  DD DISP=SHR,DSN=SYS2.LINKLIB          <=== (2)
//PARMS    DD DISP=SHR,DSN=SYS1.PARMLIB(RMTLOG)  <=== (3)
//SYSTCPD  DD DISP=SHR,DSN=SYS1.TCPIP(TCPDATA)   <=== (4)
//SYSPRINT DD SYSOUT=*
```

1. Execute statement. `TIME=1440` makes this task not time out.
2. APF authorized link library that contains the RMTLOG module. STEPLIB can be omitted if the RMTLOG module is in a linklist LPA List library.
3. `LRECL(80)` file containing the parameters for RMTLOG.
4. `TCPDATA` file, if required for your installation.  May be needed for dns name lookup of the remote syslog server, depending on the configuration of TCP/IP on z/OS.

## Appendix B   Parameter File

The parameter file is an LRECL(80) PDS member that contains settings for the RMTLOG started task.  The parameters are all space delim meaning that each line contains a parameter name, followed by one or more
spaces, followed by the value for that parameter.  Lines th begin with either  #  or  *  in column one are ignored i.e. are treated as comments.  A sample parameter member is supplied with the RMTLOG package
in member PARMS.

| Parameter |  Example  |          Description |
|-----------|-----------|----------------------|
| SERVER    |  rmtlog.jmit.com |    Host name of SYSLOG server.  This should be a partially or fully qualified domain name for the server that will receive the syslog records.  Be sure that the DNS servers specified in the z/OS TCPIP stack are able to resolve this name. |
| SERVERIP  |  70.102.3.45 | IP address of SYSLOG server. **Note**: :warning: SERVERIP or SERVER can be coded, but not both.  An error message will be issued if both SERVER and SERVERIP are coded. |
| HOSTAME    | myhost             | Overrides the local host name specified in the active TCPDATA file.  Some syslog servers use this value to affect how logs are handled. |
| PROTO      | TCP       |          UDP or TCP Specifies the protocol to use in communicating with the remote SYSLOG server.  If using the older BSD Syslog facility (described by RFC 3164) then only UDP can be specified here, as the BSD Syslog daemon only responds to UDP requests.  See Appendix C Syslog Server for more information on configuring the Syslog server. |
| PORT       | 514       |  Usually set to 514. UDP or TCP port to which RMTLOG will send Syslog messages. The Syslog Server must be listening on this port for Syslog requests.
| CNAME      | RLG01     |          Up to 8 characters.  The name of the Extended MCS console.  Mainly or documentary purposes.  This name can be used with the D EMCS command   e.g.: `D EMCS,FULL,CN=RLG01` |
| TIMESRC    | MESSAGE   |          **MESSAGE**: Takes the time stamp from the z/OS Message Data Block (MDB).  This will result in the remotely logged message having the same time stamp as the original local SYSLOG message **SYSTEM**:  Causes the current z/OS system time to be placed in the remotely logged message. |
| PRI        | 035       |          3 digit number. Default PRI value.  See Appendix C on Syslog Server, and RFC 3164 and RFC 5424 for an extensive treatment of the PRI value.  It can be left as its default value of 035 to start with. |
| RETRYINT   | 15        |          Retry seconds.  If the syslog server cannot be reached, the parameter determines how long to wait (in seconds) before retrying the connection. |
| RETRIES    | 0         |         How many times to retry before giving up, and issuing a fatal message.  A value of 0 means to never give up retrying, and is the recommended value. |
| SIZE       | 512       |          Number of megabytes allowed for the messages data space.  Minimum is 1, and maximum is 2048 (2gb) |
| QLIMIT     | 524288 |             Maximum number of MCS messages that can be queued.  Maximum value is 2147483647, (2 billion) which equates to X 7FFFFFFF. |

## Appendix C   Syslog Server

HARDWARE / OS PLATFORM The remote Syslog server can be any platform that is capable of running an RFC 3164 or RFC 5424 compliant syslog server. Most mainst Linux distributions come with a syslog server installed that
is capable of satisfying the requirements for such a server.  Red Hat En Linux (RHEL) 5.3 was used as the development platform for RMTLOG, and the sample configurations in this document are what would be us RHEL or
CENTOS Linux.  An Intel based Linux or Solaris server is an inexpensive but capable platform for hosting the remote Syslog.

**SYSLOG SOFTWARE:**

Many Linux distributions come with the BSD Syslog facility already installed.  BSD Syslog only accepts UDP syslog requests, not TCP.  best results, it is recommended that BSD Syslog be either uninstalled or
disabled, and Syslog-ng (Next Generation) be installed inste Syslog-ng can be configured to accept either UDP or TCP syslog requests, and TCP connections offer the benefits of greater reliabilit and proper
sequencing of records even under congested network situations. Syslog-ng can be downloaded from the Syslog-ng home page a http://www.balabit.com/network-security/syslog-ng/

An Open Source (free) version of Syslog-ng is available at this site, as is a  Premium  (i.e. not free) version.  The free version wo fine for the purposes of RMTLOG.  This site has information on configuring
Syslog-ng, but a sample configuration (such as was used in development of RMTLOG) is shown below.

**SAMPLE SYSLOG-NG  CONFIGURATION FOR USE WITH RMTLOG**:

```conf
@version: 3.0
#
# Syslog-ng configuration for use with RMTLOG
#
options {
  use_fqdn(no);
  keep_hostname(yes);
  use_dns(no);
  dns_cache(no);
  long_hostnames(off);
  };

#---
# Default source for internal messages.
#---
source s_local {
internal();
unix-stream("/dev/log");
# messages from the kernel
file("/proc/kmsg" program_override("kernel: "));
};

#---
# Default destination for internal messages.
#---
destination d_messages { file("/var/log/messages"); };



#---
# RMTLOG source and destination.  The destination as coded here will res
# in a separate log file for each host that is logged in the directory:
# /var/log/HOSTS/<hostname>.  This can be changed as desired.
#---
source TCP_UDP {
  tcp( port(514));
  udp( port(514));
  };

destination logip {
  file("/var/log/HOSTS/$HOST" perm(0600) create_dirs(no) dir_perm(0700)
  };

#---
# Source and destination connections.
#---
log {
  source(s_local);
  destination(d_messages);
  source(TCP_UDP);
  destination(logip);
  };

```

### RED HAT ENTERPRISE LINUX (RHEL) 5.X CONFIGURATION

**Listening:**

If a BSD Syslog server is chosen, then the following changes need to be made to the RHEL configuration:  In the file /etc/sysconfig/syslog, add the  -r  flag to SYSLOGD_OPTIONS.  When finished, it should look something like this: `SYSLOGD_OPTIONS="-m 0 -r"`

If the Syslog-ng syslog server is installed, the above change is not needed.  Syslog-ng already listens on both UDP and TCP port 514 TCP directive in the Source entry in the configuration file.

**Firewall:**

The firewall must be instructed to allow inbound datagrams / packets on port 514.  For RHEL5, select GUI as follows:  System / Admini Level and Firewall.

Click the  Other Ports  arrow, and then click the  Add  button, specify the port number (e.g. 514) and type (UDP or TCP).

**OTHER CONFIGURATION ISSUES:**

Log file size and location:  As Syslog data can grow to be quite large, it is strongly recommended that the Syslog files be mounted on a separate, dedicated file prevent the Syslog server system from crashing
if the log files become  full.  It is also recommended that the file system that is to as large as possible to prevent running out of storage.

**Log Rotation:**

Use of the logrotate package on Linux platforms is highly recommended. Logrotate will automatically break up the log file into a siz and compress these files for maximum space efficiency.  For more
information, Google on `logrotate`.

**Network connectivity:**

Hosts that are sending Syslog records to the Syslog server should have robust, high-speed network connectivity to the Syslog server

**Dedication:**

It is strongly recommended that the Syslog server be dedicated solely to recording syslog records from remote hosts.  Other workloads timely processing of Syslog recording requests.

## Appendix D - RMTLOG Messages

* `RLG000I` RMTLOG Starting shutdown.
* `RLG002I` Console not deactivated, was not active.
* `RLG003I` MCS console activated.
* `RLG004I` MCS Console failed to activate.
* `RLG005I` MCS Console deactivated.
* `RLG006I` MCS Console deactivate failed.
* `RLG009I` Waiting for connection to syslog server.
* `RLG010I` Error processing MDB.
* `RLG011I` RMTLOG is active.
* `RLG012I` Issuing MCSOPMSG RESUME.
* `RLG016I` Message queueing stopped, memory limit.
* `RLG017I` Message queueing stopped, queue depth reached.
* `RLG018I` Message queueing stopped, internal error.
* `RLG019I` Message queueing stopped, alert percentage reached.
* `RLG020I` RMTLOG Starting initialization phase 2.
* `RLG021I` RMTLOG Starting initialization phase 1.
* `RLG023I` Default value assigned for parameter:
* `RLG024I` Neither SERVER nor SERVERIP coded, one is required - exiting.
* `RLG025I` SERVER and SERVERIP both coded, but only one allowed - exiting.
* `RLG032I` TCP protocol selected.
* `RLG033I` UDP protocol selected.
* `RLG034I` Unable to determine local hostname.
* `RLG035I` Error creating socket.
* `RLG036I` Error connecting socket.
* `RLG037I` Waiting for TCPIP initialization.
* `RLG053I` Error opening parameter file.


# Release Notes:

* 09/21/2016 - Tested and operational on z/OS 2.0 without reassembly.
* 11/10/2010 - v1.1 - Added SERVERIP parameter to allow specifying syslog server by ipv4 address instead of by domain name.  I am not calling this code beta any longer, as it has been in production on at least one z/OS system for over 5 years.  However it would be helpful to have any feedback or bug reports.
* 06/20/2010 - v1.0 (beta) Original version, developed on z/OS 1.10. I'm calling this a beta version mainly because it's only been run at a couple of sites.