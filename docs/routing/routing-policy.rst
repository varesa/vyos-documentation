
Routing-policy
--------------

Routing Policies could be used to tell the router (self or neighbors) what routes and their attributes needs to be put into the routing table.

There could be a wide range of routing policies. Some examples are below:

  * Set some metric to routes learned from a particular neighbor
  * Set some attributes (like AS PATH or Community value) to advertised routes to neighbors
  * Prefer a specific routing protocol routes over another routing protocol running on the same router

Routing Policy Example
~~~~~~~~~~~~~~~~~~~~~~

**Policy definition:**

.. code-block:: sh

  #Create policy
  set policy route-map setmet rule 2 action 'permit'
  set policy route-map setmet rule 2 set as-path-prepend '2 2 2'  
  
  #Apply policy to BGP
  set protocols bgp 1 neighbor 1.1.1.2 route-map import 'setmet'
  set protocols bgp 1 neighbor 1.1.1.2 soft-reconfiguration 'inbound' <<<< *** 
  
  *** get policy update without bouncing the neighbor

**Routes learned before routing policy applied:**

.. code-block:: sh

  vyos@vos1:~$ show ip bgp
  BGP table version is 0, local router ID is 192.168.56.101
  Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
                r RIB-failure, S Stale, R Removed
  Origin codes: i - IGP, e - EGP, ? - incomplete
  
     Network          Next Hop            Metric LocPrf Weight Path
  *> 22.22.22.22/32   1.1.1.2                  1             0 2 i  < Path 
  
  Total number of prefixes 1

**Routes learned after routing policy applied:**

.. code-block:: sh

  vyos@vos1:~$ sho ip b
  BGP table version is 0, local router ID is 192.168.56.101
  Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
                r RIB-failure, S Stale, R Removed
  Origin codes: i - IGP, e - EGP, ? - incomplete
  
     Network          Next Hop            Metric LocPrf Weight Path
  *> 22.22.22.22/32   1.1.1.2                  1             0 2 2 2 2 i < longer AS_path length
  
  Total number of prefixes 1
  vyos@vos1:~$ 

Prefix filtering example
~~~~~~~~~~~~~~~~~~~~~~~~

**Unfiltered routes advertised by r1:**

.. code-block:: sh

  vyos@r1:~$ show ip bgp neighbors 1.1.1.2 advertised-routes
  BGP table version is 8, local router ID is 1.1.1.1, vrf id 0
  Default local pref 100, local AS 65001
  Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
                 i internal, r RIB-failure, S Stale, R Removed
  Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
  Origin codes:  i - IGP, e - EGP, ? - incomplete

     Network          Next Hop            Metric LocPrf Weight Path
  *> 0.0.0.0/0        192.168.15.1             0         32768 ?
  *> 10.0.0.0/8       0.0.0.0                  0         32768 ?
  *> 10.0.0.0/24      0.0.0.0                  0         32768 ?
  *> 10.0.1.0/24      0.0.0.0                  0         32768 ?
  *> 172.16.1.0/24    0.0.0.0                  0         32768 ?
  *> 172.16.2.0/24    0.0.0.0                  0         32768 ?


**Filtering imported routes on r2:**

.. code-block:: sh

  # Import 10.0.0.0/8 and longer prefixes within (length 8 to 32)
  set policy prefix-list IMPORT rule 1 action 'permit'
  set policy prefix-list IMPORT rule 1 le '32'
  set policy prefix-list IMPORT rule 1 prefix '10.0.0.0/8'
  # Implicit deny at the end

  set policy route-map IMPORT rule 1 action 'permit'
  set policy route-map IMPORT rule 1 match ip address prefix-list 'IMPORT'
  # Implicit deny at the end

  set protocols bgp 65002 neighbor 1.1.1.1 address-family ipv4-unicast route-map import 'IMPORT'

Installed routes after filtering:

.. code-block:: sh

  vyos@r2:~$ show ip bgp
  BGP table version is 11, local router ID is 1.1.1.2, vrf id 0
  Default local pref 100, local AS 65002
  Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
                 i internal, r RIB-failure, S Stale, R Removed
  Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
  Origin codes:  i - IGP, e - EGP, ? - incomplete

     Network          Next Hop            Metric LocPrf Weight Path
  *> 10.0.0.0/8       1.1.1.1                  0             0 65001 ?
  *> 10.0.0.0/24      1.1.1.1                  0             0 65001 ?
  *> 10.0.1.0/24      1.1.1.1                  0             0 65001 ?

**Filtering exported routes on r1:**

.. code-block:: sh

  # Match exact prefixes by default
  set policy prefix-list EXPORT rule 1 action 'permit'
  set policy prefix-list EXPORT rule 1 prefix '10.0.0.0/8'
  set policy prefix-list EXPORT rule 2 action 'permit'
  set policy prefix-list EXPORT rule 2 prefix '172.16.1.0/24'

  set policy route-map EXPORT rule 1 action 'permit'
  set policy route-map EXPORT rule 1 match ip address prefix-list 'EXPORT'

  set protocols bgp 65001 neighbor 1.1.1.2 address-family ipv4-unicast route-map export 'EXPORT'

Advertised routes after filtering:

.. code-block:: sh

  vyos@r1:~$ show ip bgp neighbors 1.1.1.2 advertised-routes
  BGP table version is 8, local router ID is 1.1.1.1, vrf id 0
  Default local pref 100, local AS 65001
  Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
                 i internal, r RIB-failure, S Stale, R Removed
  Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
  Origin codes:  i - IGP, e - EGP, ? - incomplete

     Network          Next Hop            Metric LocPrf Weight Path
  *> 10.0.0.0/8       0.0.0.0                  0         32768 ?
  *> 172.16.1.0/24    0.0.0.0                  0         32768 ?

