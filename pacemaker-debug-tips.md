# Pacemaker Debug Tips

A particular situation in which a service by hand is running correctly but it fails into Pacemaker.
The command works in this way:

[heat-admin@overcloud-controller-0 ~]$ fence_ipmilan -P -a 10.1.8.100 -o status -l qe-scale -p d0ckingSt4tion
Status: ON

And the associated definition goes like this:

pcs stonith create stonith-overcloud-controller-0 fence_ipmilan pcmk_host_list="overcloud-controller-0" ipaddr="10.1.8.100" action="reboot" login="qe-scale" passwd="d0ckingSt4tion" delay=20 op monitor interval=60s

From a pcs point of view, this is what is saw:

 Resource: stonith-overcloud-controller-0 (class=stonith type=fence_ipmilan)
  Attributes: pcmk_host_list=overcloud-controller-0 ipaddr=10.1.8.100 action=reboot login=qe-scale passwd=d0ckingSt4tion delay=20 
  Operations: monitor interval=60s (stonith-overcloud-controller-0-monitor-interval-60s)

The resource tries to run into overcloud-controller-2, but it fails:

* stonith-overcloud-controller-0_start_0 on overcloud-controller-2 'unknown error' (1): call=333, status=Error, exitreason='none',

In the logs (in this case /var/log/pacemaker.log), searching for the string "stonith-overcloud-controller-0" make us discover this:

Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:     info: update_remaining_timeout:      Attempted to execute agent fence_ipmilan (monitor) the maximum number of times (2) allowed
Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:   notice: log_operation: Operation 'monitor' [24100] for device 'stonith-overcloud-controller-0' returned: -201 (Generic Pacemaker error)
Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [ Executing: /usr/bin/ipmitool -I lan -H 10.1.8.100 -U qe-scale -P d0ckingSt4t
ion -p 623 -L ADMINISTRATOR chassis power status ]
Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [  ]
Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [ 1  Get Session Challenge command failed ]
Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [ Error: Unable to establish LAN session ]
Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [ Unable to get Chassis Power Status ]
Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:24100 [  ]
Oct 22 06:45:46 [28785] overcloud-controller-2.localdomain stonith-ng:  warning: log_operation: stonith-overcloud-controller-0:241

So, for some reason, the command fails. And we can verify it by hand, launching:

[heat-admin@overcloud-controller-2 ~]$ /usr/bin/ipmitool -I lan -H 10.1.8.100 -U qe-scale -P d0ckingSt4tion -p 623 -L ADMINISTRATOR chassis power status
Get Session Challenge command failed
Error: Unable to establish LAN session
Unable to get Chassis Power Status

So, what's the matter with this? Looking at the first command we launched, we see that there is an evident difference. In our original command there was a -P which stays for "Use Lanplus to improve security of connection", but in this ipmitool command the -I option is "lan", which is used by default from Pacemaker.
That is confirmed by this:

[heat-admin@overcloud-controller-2 ~]$ /usr/bin/ipmitool -I lanplus -H 10.1.8.100 -U qe-scale -P d0ckingSt4tion -p 623 -L ADMINISTRATOR chassis power status
Chassis Power is on

So, how to correct our configuration? First of all, let's what options do we have with the ipmi stonith agent, by launching this command:

[heat-admin@overcloud-controller-2 ~]$ sudo pcs stonith describe fence_ipmilan
Stonith options for: fence_ipmilan
...
...
  lanplus: Use Lanplus to improve security of connection
...
...

That's our option! After deleting the old stonith resource:

[heat-admin@overcloud-controller-0 ~]$ sudo pcs stonith delete stonith-overcloud-controller-0                                                                                                                      
Deleting Resource - stonith-overcloud-controller-0

We can create the new one, with the right option declared:

[heat-admin@overcloud-controller-0 ~]$ sudo pcs stonith create stonith-overcloud-controller-0 fence_ipmilan pcmk_host_list="overcloud-controller-0" verbose="true" pcmk_host_check="static-list" ipaddr="10.1.8.100" action="reboot" login="qe-scale" passwd="d0ckingSt4tion" lanplus="true" delay=20 op monitor interval=60s

[heat-admin@overcloud-controller-0 ~]$ sudo pcs stonith show
...
...
 stonith-overcloud-controller-0 (stonith:fence_ipmilan):        Started

And we are done!
