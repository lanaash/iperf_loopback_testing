# IPerf Loopback Tetsing

## Intro

How you could potentially use an IPerf server and client within a Service Provider infrastructure to test the effective throughput (“goodput”) of a customer Leased Line/Ethernet service.
This could only really be of value when testing symmetric Ethernet circuits – same upload and download bandwidth - as the customer router (CPE) will be hair pinning the traffic!
Some potential benefits for the Service Provider are being in control of the testing equipment and test results and no remote engineer or customer assistance required.

## Basic design 
IPerf Client and Server are in the same LAN but have static ARP entries to each other via the next hop router (e.g. see “ip neigh add” command).
The next hop Juniper router will filter forward any IPerf traffic received from either IPerf host towards the CPE.
Return traffic from the CPE is sent unicast forwarding.

## Example config for JunOS

    #
    # Filter to match IPerf traffic and then lookup the iperfTraffic routing-instance
    #
    set firewall family inet filter iperfFilter term 1 from destination-port 8080
    set firewall family inet filter iperfFilter term 1 then routing-instance iperfTraffic
    set firewall family inet filter iperfFilter term 2 from source-port 8080
    set firewall family inet filter iperfFilter term 2 then routing-instance iperfTraffic
    set firewall family inet filter iperfFilter term 3 then accept
    #
    # Example IPerf infra interface with the firewall filter
    #
    set interfaces ge-0/0/0 description “IPerf LAN”
    set interfaces ge-0/0/0 unit 0 family inet filter input iperfFilter
    set interfaces ge-0/0/0 unit 0 family inet address 192.168.20.6/29
    #
    # Example CPE “WAN” interface
    #
    set interfaces ge-0/0/5 description “CPE WAN”
    set interfaces ge-0/0/5 unit 0 family inet address 10.0.0.1/30
    #
    # Just a static default via the CPE in the iperfTraffic routing-instance
    #
    set routing-instances iperfTraffic instance-type forwarding
    set routing-instances iperfTraffic routing-options static route 0.0.0.0/0 next-hop 10.0.0.2
    #
    # Filter just the CPE WAN IP addressing for import to RIB group – don’t want IPerf infra!
    #
    set policy-options policy-statement p2pOnly term 10-slash-16 from protocol direct
    set policy-options policy-statement p2pOnly term 10-slash-16 from route-filter 10.0.0.0/16 orlonger
    set policy-options policy-statement p2pOnly term 10-slash-16 to rib iperfTraffic.inet.0
    set policy-options policy-statement p2pOnly term 10-slash-16 then accept
    set policy-options policy-statement p2pOnly then reject
    #
    # RIB group
    #
    set routing-options interface-routes rib-group inet IPerf-rib
    set routing-options rib-groups IPerf-rib import-rib iperfTraffic.inet.0
    set routing-options rib-groups IPerf-rib import-rib inet.0
    set routing-options rib-groups IPerf-rib import-policy p2pOnly
