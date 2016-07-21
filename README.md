# lossy
Wrapper for linux tc and netem tools. Simulate lossy/shaped networks the easy way.

Lossy uses tc (traffic control) to build a chain of qdiscs (queues for packets) and simulate packet loss, delay 
or limited network bandwidth. It uses standard netem (network emulator) and tbf (token bucket filter) qdiscs for 
that purpose.

However using tc directly can be tricky because of several known issues, so this tool is made to simplify usage and 
provide built-in detection/workarounds for some problems.

Lossy sets rules that match incoming/outgoing traffic and applies policies to these packets: 
* To enabled packet loss/delay specify netem settings (it is better to check 'man netem' for detailed description.
* To limit network bandwidth specify tbf settings (see 'man tbf')

#Usage 

```
Usage: sudo lossy -n 'delay 100ms 10ms 25% loss 0.1%' \
                  -t 'rate 0.5mbit burst 10kb limit 10k' \
                  --from 127.0.0.1:* \
                  --to *:80
```

Options:

```
-s                                   print status.

-c                                   clear all rules.

-t|--tbf <tbf-settings>              set shaping settings that are passed to token bucket filter, 
                                     example 'rate 0.5mbit burst 10kb limit 10k'.
                                     See 'man tbf' for details. Default ''.
                          
-n|--netem <netem-settings>          set delay/loss settings that are passed to netem, 
                                     example 'delay 100ms 10ms 25% loss 0.1%'.
                                     See 'man netem' for details. Default ''.
                              
--from <host[/mask]:port|none>       match only packets from this address, 
                                     example '127.0.0.1:80', '*:80', '127.0.0.1/24:*', '*:*'.
                                     Special value 'none' - match none. 
                                     Default 'none'. Host with mask can be specified.
                          
--to <host[/mask]:port|none>         match only packets from to this address, 
                                     example '127.0.0.1:80', '*:80', '127.0.0.1/24:*', '*:*'.
                                     Special value 'none' - match none. 
                                     Default 'none'. Host with mask can be specified.
                        
-p|--protocol <protocol>             matches only this protocol packets, 
                                     example 'ip', 'tcp', 'udp'. Default 'ip' - matches ip, tcp, udp, 
                                     icmp and other ip packets.

-i|--interface <interface>           matches only on this interface, 
                                     example 'eth0', 'lo'. Default 'eth0'.

-r|--reorder-if-jitter <true|false>  if jitter is specified for netem then should it reorder 
                                     packets or send them in order. Default 'false' - send 
                                     in order.
```

# Important notes

* Must be run as root.

* Clears existing rules on specified interface before adding new rules, so if you launched it with '-i eth0' and then '-i lo'
  then rules on eth0 will remain. When invoking lossy with -c you can also add -i to specify interface to clear.

* Options --from and --to each is applied both to incoming and outgoing traffic. Example: -netem 'delay 100ms' --to '*:*',
  we ping this machine and get 200ms roundtrip time because incoming packets matches *:* and reply packet also matches rule.
  If you want to affect only upstream/downstream you will have to specify ip or mask so that packets to/from this machine
  are not matched. Example: this machine has ip 192.168.1.1 and we add rule --to 192.168.1.1 this will result in 100ms
  rountrip, incoming packets will have 100ms delay while outgoing will have 0ms delay.

* Lossy uses ethtool to verify environment correctness, it is not required to run, but may tell you about some problems
  that can cause incorrect behavior.

* All settings are applied both to outgoing traffic and incoming traffic, so if you specify delay 100ms then roundtrip
  time will be 200ms. Use --from and --to to match packets more granularly.

* Remember that your ssh session is affected by lossy also, so if you specify large delay, jitter, packet loss you may
  not be able to disable it via ssh. However all settings are reset after system reboot.

* If both tbf and netem enabled then netem goes after tbf.

* No need to specify 'limit' in tbf settings if netem is enabled, because netem also has 'limit' option and internal
  pfifo qdisc, so traffic will mainly be affected by this setting which we set to 20 (packets) by default.

* If 'large segment offload' option (also known as TSO/GSO) is enabled in kernel (check with 'ethtool -k eth0') then
  inside kernel packets can be larger than MTU and be broken into MTU-size chunks only by network interface. This leads
  to tbf limiting speed to very low value (10-20Kb/sec) despite of settings. If this is the case recommended approach
  is to disable TSO/GSO, this doesn't affect functionality, but may slightly increase CPU usage in case of large amounts
  of traffic.

* Netem has a reordering bug/feature - if jitter is specified like 'delay 100ms 50ms' then packets will be reordered. Docs
  suggest adding child pfifo qdisc, but it doesn't work in 3.2-4.2 kernels, but there is undocumented behavior that if
  'rate' netem option is specified then no reordering occurs. Option was added in attempt to bring tbf functionality into
  netem but has this side effect. So we simply have -r flag to prevent reordering, it results in adding large 'rate' value.
  Seems several rates can be specified and the last one takes effect so we prepend our value to allow overriding.

* When configuring tbf set burst and limit high enough. Min burst is rate/250 (HZ=250 in many linux kernels) so 40k is
  required to reach 10Mb/sec speed. Sometimes packets can even fail to queue at all if limit is low (-s shows drops
  increasing, but sent bytes stays the same) so always check your configuration with iperf.
