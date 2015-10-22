# Pacemaker Debug Tips
This document describes a series of ways to debug problems with Pacemaker, looking into the logs, understanding cluster's dynamics and find a quick approach.

## Debugging stonith resources problems

Stonith resources are quite like standard resources, excepts that for declaring them you need a specific pcs syntax, since they are dedicate to fence operations and, for this reason, quite uniques.
Suppose you need to declare a stonith resource to be able to fence a controller which is reachable via ipmi. You will use something similar to this:

    pcs stonith create stonith-overcloud-controller-0 fence_ipmilan pcmk_host_list="overcloud-controller-0" ipaddr="<IMPI-IP>" action="reboot" login="<YOUR-LOGIN>" passwd="<YOUR-PASSWORD>" delay=20 op monitor interval=60s

This will define a stonith resource named "stonith-overcloud-controller-0" to be started wherever the cluster will decide.

From a pcs point of view, this is what the cluster configured:

    Resource: stonith-overcloud-controller-0 (class=stonith type=fence_ipmilan)
     Attributes: pcmk_host_list=overcloud-controller-0 ipaddr=10.1.8.100 action=reboot login=qe-scale passwd=d0ckingSt4tion delay=20
     Operations: monitor interval=60s (stonith-overcloud-controller-0-monitor-interval-60s)

Now, if the resource comes up without problems you might be done. Problems comes when it fails:

    * stonith-overcloud-controller-0_start_0 on overcloud-controller-2 'unknown error' (1): call=333, status=Error, exitreason='none',

So we need to investigate what is wrong with our resource, starting from what we must know: how fence operations works.

Each stonith operation is driven by the execution of a script, in this case the cluster will use **fence_ipmilan**, that can of course be launched by hand. The command works in this way:

    [heat-admin@overcloud-controller-0 ~]$ fence_ipmilan -P -a <IMPI-IP> -o status -l <YOUR-LOGIN> -p <YOUR-PASSWORD>
    Status: ON

Since the result is *Status: ON* for us it's a grat tip: the problem is not on the system side, but on the cluster's one, since the command **WORKS**.
If this command failed we need to focus our attention on the system, to understand, for example, if the destination machine was reachable, the user and the password were right, and so on. But in this case we need to focus on Pacemaker. So, understanding that the problem was first see on *overcloud-controller-2* we can move on this machine and have a look to the logs.

In the logs (in this case /var/log/pacemaker.log, but maybe you've configured your cluster differently), searching for the string "stonith-overcloud-controller-0" make us discover some informations.
The declaration of the resource:

    Oct 22 06:10:14 [28785] overcloud-controller-2.localdomain stonith-ng:   notice: stonith_device_register:       Added 'stonith-overcloud-controller-0' to the device list (2 active devices)

The fact that this resource's monitor is not started:

    Oct 22 06:10:28 [28789] overcloud-controller-2.localdomain       crmd:   notice: process_lrm_event:     Operation stonith-overcloud-controller-0_monitor_0: not running (node=overcloud-controller-2, call=304, rc=7, cib-update=157, confirmed=true)

And the effective start:

    Oct 22 06:10:28 [28786] overcloud-controller-2.localdomain       lrmd:     info: log_execute:   executing - rsc:stonith-overcloud-controller-0 action:start call_id:310

Then, after a couple of lines we find our problem:

    Oct 22 06:10:26 [28785] overcloud-controller-2.localdomain stonith-ng:   notice: log_operation: Operation 'monitor' [14272] for device 'stonith-overcloud-controller-2' returned: -201 (Generic Pacemaker error)
    Oct 22 06:10:26 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-2:14272 [ Failed: Unable to obtain correct plug status or plug is not available ]

The main problem about this message is that "Failed: Unable to obtain correct plug status or plug is not available" is not such an helpful message. We already verified that we have got the fence agent on the system, and it also works. So, what else can we do? We need more informations, and to obtain them we can enable the debug for the stonith resource.
How this can be done? By looking into the description of the fence_ipmilan there's a useful option that can be enabled:

    [heat-admin@overcloud-controller-0 ~]$ sudo pcs stonith describe fence_ipmilan
    Stonith options for: fence_ipmilan
    ...
    ...
      verbose: Verbose mode
    ...
    ...

Traducing it into a command:

    [heat-admin@overcloud-controller-0 ~]$ sudo pcs stonith update stonith-overcloud-controller-0 set verbose=true

From now on the resource will be more verbose inside the logs. So after cleaning its status:

    [heat-admin@overcloud-controller-0 ~]$ sudo pcs resource cleanup stonith-overcloud-controller-0
    Resource: stonith-overcloud-controller-0 successfully cleaned up

We can check again the logs, and there's finally what we are looking for:

    Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:     info: update_remaining_timeout:      Attempted to execute agent fence_ipmilan (monitor) the maximum number of times (2) allowed
    Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:   notice: log_operation: Operation 'monitor' [24100] for device 'stonith-overcloud-controller-0' returned: -201 (Generic Pacemaker error)
    Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [ Executing: /usr/bin/ipmitool -I lan -H <IMPI-IP> -U <YOUR-LOGIN> -P <YOUR-PASSWORD> -p 623 -L ADMINISTRATOR chassis power status ]
    Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [  ]
    Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [ 1  Get Session Challenge command failed ]
    Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [ Error: Unable to establish LAN session ]
    Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [ Unable to get Chassis Power Status ]
    Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [  ]
    Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:241

So, for some reason, the ipmitool (driven by fence_ipmilan) command fails. And we can verify it by hand, launching:

    [heat-admin@overcloud-controller-2 ~]$ /usr/bin/ipmitool -I lan -H 10.1.8.100 -U qe-scale -P d0ckingSt4tion -p 623 -L ADMINISTRATOR chassis power status
    Get Session Challenge command failed
    Error: Unable to establish LAN session
    Unable to get Chassis Power Status

So, what's the matter with this? Looking at the this command and comparing it to the one we launched on the top, we see that there is an evident difference. In our original command there was a -P which stays for "Use Lanplus to improve security of connection", but in this ipmitool command the -I option is "lan", which is used by default from Pacemaker.
Let's try first to use lanplus in the command:

    [heat-admin@overcloud-controller-2 ~]$ /usr/bin/ipmitool -I lanplus -H 10.1.8.100 -U qe-scale -P d0ckingSt4tion -p 623 -L ADMINISTRATOR chassis power status
    Chassis Power is on

It works. So, how to correct our configuration? First of all, let's see what options do we have with the ipmi stonith agent, by launching this command:

    [heat-admin@overcloud-controller-2 ~]$ sudo pcs stonith describe fence_ipmilan
    Stonith options for: fence_ipmilan
    ...
    ...
      lanplus: Use Lanplus to improve security of connection
    ...
    ...

That's our option! We can change our resource to support lanplus:

    [heat-admin@overcloud-controller-0 ~]$ sudo pcs stonith update stonith-overcloud-controller-0 set lanplus=true

And after a new cleanup:

    [heat-admin@overcloud-controller-0 ~]$ sudo pcs resource cleanup stonith-overcloud-controller-0
    Resource: stonith-overcloud-controller-0 successfully cleaned up

See how things are going now:

    [heat-admin@overcloud-controller-0 ~]$ sudo pcs stonith show
    ...
    ...
     stonith-overcloud-controller-0 (stonith:fence_ipmilan):        Started

The resource is finally started and working.
