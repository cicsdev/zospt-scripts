# Scripting with z/OS Provisioning Toolkit: Removing Old Containers

A few months ago we delivered [z/OS Provisioning Toolkit][zospt], allowing you to bring
up CICS regions in minutes, and purge them just as quickly. Being a command that's run
from USS---from a shell---z/OS PT is easy to use as part of a more complicated script. In
this short example, we're going to write a script to be able to remove all containers in
the system older than _x_ days. This can then be scheduled to run every night, to ensure
developers don't leave their CICS regions (or any other container contents) sat around
unnecessarily. Treat your z/OS PT-created CICS regions like livestock, not pets,
remember. All source code for this example is on [CICSDev on GitHub][gh].

I've started by provisioning myself a CICS TS V5.3 region in a container. I can see this
container, and in fact any container I---_chpoole_---own (running or not), using the
`zospt ps -a` command:

```bash
> zospt ps -a
2017-06-27 15:14:21  IBM z/OS Provisioning Toolkit V1.0.2
2017-06-27 15:14:21  Connecting to z/OSMF on host winmvs29.hursley.ibm.com port 27820.
2017-06-27 15:14:22  z/OS level is V2R1.
2017-06-27 15:14:22  z/OS Management Facility level is V2R2.

NAMES        	IMAGE  	OWNERS 	CREATED            	STATE      	TEMPLATE	CONTAINER ID
CICS_CICPP002	cics_53	chpoole	2017-06-27T13:40:17	provisioned	cics_53 	c5a3f854-0cfc-48c8-8279-5582b982c550
```

I've also stored my password in my bash profile, `$HOME/.bashrc`, as `export
zospt_pw=hunter2`. When `zospt` sees this, it doesn't need to ask me for my password each
time, perfect for a script.

So what I need to do is take the output of the above command, and identify any containers
that are too old. The first thing to do is to remove the header from that output, so we
only have the list of containers themselves. I can do this easily with the `tail`
command, saying "only show me from line 7 onwards":

```bash
> zospt ps -a | tail -n +7
CICS_CICPP002	cics_53	chpoole	2017-06-27T13:40:17	provisioned	cics_53 	c5a3f854-0cfc-48c8-8279-5582b982c550
```

Looking good. Now, I need to identify which containers are too old. To do this, I've
written a quick perl script[^ported] to take this output, and give me back the names of the old
containers. I take the time of each container's creation, and convert
to [Unix time][ut]. I can then check what the
current Unix time is, and see if the difference is too
great. The [full script is on GitHub][gh], but the core logic is
as such:

```perl
my @time_date = split /[-T:]/, $fields[3];
  if (abs(time()-seconds_since_epoch(@time_date)) > ($days*$seconds_per_day)) {
    print $fields[0] . "\n";
  }
```

I've written the perl script to also accept the number of days (an integer or decimal) of
age, such that our line of shell script now looks like this:

```bash
> zospt ps -a | tail -n +7 | find-old-containers 0
CICS_CICPP002
```

I.e., "show me all the containers older than 0 days". If no value is given, 3 is the
default. Running this gives me the name of my one and only container, as I've asked it
for all containers older than _right now_. To finish the script, we can make use of
`zospt rm -f`, to (force) remove a container. This takes the container name as its first
argument, so we'll use `xargs` to pass it that name:

```bash
zospt ps -a | tail -n +7 | find-old-containers 0 | xargs -n1 zospt rm -f
```

This has the effect of running `zospt rm -f CICS_CICPP002`, and will repeat that command
for every old container found. I'm assuming that `find-old-containers`, the perl script,
is somewhere in your `$PATH` so it can be found. We can store this line that we've built
up in an executable file itself, saving it as `remove-old-containers`, again somewhere in
your `$PATH`.

```bash
#!/bin/bash -

. $HOME/.bashrc
zospt ps -a | tail -n +7 | find-old-containers 0 | xargs -n1 zospt rm -f
```

I've loaded my z/OS PT password as a first step, but this could be written in the script
itself as `export zospt_pw=hunter2`. Doing it my way allows me to only store my password
in my bash profile, and to lock down that file such that no-one else can read it, while
allowing this script file to be read by anyone.

If we run this command from the shell, it'll remove all my containers. But let's run it
from JCL, so we can easily add it to the scheduler. To do this, I'm going to use
`BPXBATCH`, passing it the name of the script to execute, and giving it a filename to
write its output to. The JCL looks like this:

```jcl
//REMCON   JOB CLASS=M,MSGCLASS=H,NOTIFY=&SYSUID,
//  MSGLEVEL=(1,1)
//REMOVE EXEC PGM=BPXBATCH,REGION=8M
//STDIN    DD PATH='/u/chpoole/bin/remove-old-containers',
//            PATHOPTS=(ORDONLY)
//STDOUT   DD PATH='/u/chpoole/tmp/bpxout',PATHOPTS=(OWRONLY,OCREAT),
//            PATHMODE=SIRWXU
```

Now, if we submit this, we can see that it ran successfully:

```jcl
---- TUESDAY,   27 JUN 2017 ----
 IRR010I  USERID CHPOOLE  IS ASSIGNED TO THIS JOB.
 ICH70001I CHPOOLE  LAST ACCESS AT 15:58:16 ON TUESDAY, JUNE 27, 2017
 $HASP373 REMCON   STARTED - INIT 15   - CLASS M        - SYS MV2D
 IEF403I REMCON - STARTED
 -                                         --TIMINGS (MINS.)--
 -JOBNAME  REMOVE  PROCSTEP    RC   EXCP    CPU    SRB  CLOCK   SERV
 -REMCON           STEPNAME    00     10    .00    .00    .00    310
 -REMCON           *OMVSEX     00     16    .00    .00    .00    194
 -REMCON           *OMVSEX     00   1470    .00    .00    .90   1719
```

So let's use `tail` to take a look at the `STDOUT` file (updated dynamically), and see
what it did:

```bash
> tail -f /u/chpoole/tmp/bpxout
2017-06-27 16:00:06  IBM z/OS Provisioning Toolkit V1.0.2
2017-06-27 16:00:06  Connecting to z/OSMF on host winmvs29.hursley.ibm.com port 27820.
2017-06-27 16:00:09  Performing deprovision on container CICS_CICPP002.
2017-06-27 16:00:11  Checking if the CICS region is active - Complete.
2017-06-27 16:00:16  Performing normal CICS shutdown and wait - Complete.
2017-06-27 16:00:30  Deleting standard CICS data sets - Complete.
2017-06-27 16:00:32  Deleting CICS log stream model - Complete.
2017-06-27 16:00:43  Deleting zFS - Complete.
2017-06-27 16:00:53  Deleting CICS security configuration - Complete.
2017-06-27 16:00:53  Returning CMCI port - Complete.
2017-06-27 16:00:53  Returning http port - Complete.
2017-06-27 16:00:56  Returning https port - Complete.
2017-06-27 16:00:56  Returning applid - Complete.
2017-06-27 16:00:57  Deletion of environment completed successfully.
2017-06-27 16:00:58  Deleted container CICS_CICPP002.
```

Success! We've now automated the deletion of any container that I own, older than _x_
days. To be able to have this script run and remove _any_ old container, regardless of
the user who created it, we need to run this piece of JCL (and `zospt` itself) as a z/OS
MF _domain administrator_. When that user runs `zospt ps -a`, they'll see all containers
in the system (for their domain). Add the `USER` parameter to the job card (or just run
as that user), and set the z/OS PT password for that user.

What are you going to script z/OS Provisioning Toolkit to do?




[zospt]: https://developer.ibm.com/cics/2017/01/10/provisioning-a-cics-liberty-development-environment-in-minutes-with-the-zos-provisioning-toolkit/
[gh]: https://github.com/cicsdev/zospt-scripts
[ut]: https://en.wikipedia.org/wiki/Unix_time
[tools]: https://www-03.ibm.com/systems/z/os/zos/features/unix/ported/
[rocket]: http://www.rocketsoftware.com/zos-open-source?cm_mc_uid=68352275219715004763813&cm_mc_sid_50200000=1500630482&cm_mc_sid_52640000=1500630482
[^ported]: Perl for z/OS is available from [IBM's Ported Tools site][tools],
    or [Rocket Software][rocket].
