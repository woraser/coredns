# traffic

## Name

*traffic* - handout addresses according to assignments.

## Description

The *traffic* plugin is an advanced load balancer that allows traffic steering, weighted responses
and draining of endpoints. Endpoints are either domain names or IP:port pairs.

*Traffic* receives (via gRPC) *assignments* that define the weight of the endpoints. The plugin
takes care of handing out responses that adhere to this assignment. Assignments will need to be
updated frequently to ensure load spreading, without new updates *traffic* will hand out responses
according to the last received assignment. When there are no assignments for a service name (yet),
the responses will be let through as-is.

An assignment covers a "service name", which for all intent and purposes is just a domain name. With
each service name a number of backends are expected. A backend is defined as either a IP:port pair
or a domain name. Each backend comes with a fractional number indicating and how often this backend
should be be handed out. A zero means the backend exists, but should not be handed out (drain it).
The backends per assignment must be of the same type; either IP:port pairs or domain names. This
requirement may be relaxed in a future version of the *traffic* plugin.

Depending on the type of the backend, *traffic* can either load balance replies containing address
records or replies with domain names in them. In more detail it will do the following, depending on
the type:

IP:port pairs:

:   This will load balance queries for A or AAAA records.

Domain names:

:   This will load balance queries for SRV and MX records.

The actual load-balancing algorithm is straight forward and mostly involves matching up resource
records, but see Assignments below for more details. The *traffic* plugin has no notion of draining,
drop overload and anything that advanced, *it just acts upon assignments*.

Note the this plugin works with any plugin that returns responses; i.e. *traffic* doesn't mandate a
backend storage; it can work with *file*, or *kubernetes*, etc. . You can just overlay this on top
of those and have way to steer traffic around.

## Syntax

~~~
traffic
~~~

This enables traffic loadbalancing for all (sub-)domains named in the server block.

## Examples

~~~ corefile
example.org {
    traffic
    forward . 10.12.13.14
}
~~~

This will add load balancing for domains under example.org; the upstream information comes from
10.12.13.14; depending on received assignments replies will be let through as-is or balanced.

## Assignments

Assignments are given in protobuf format, but here is an example in YAML conveying the same
information. This is an example assignment for the service "www.example.org", only containing
IP:port pairs as backends.

~~~
assignments:
  assigment:
    - service: www.example.org
        - backend: 192.168.1.1:443
            assign: 0.33
          backend: 192.168.1.2:443
            assign: 1.00
          backend: 192.168.1.3:443
            assign: 0.00
~~~

This particular one has 3 backends, one of which is to be drained (192.168.1.3), the remaining two
tell *traffic* that 192.168.1.2 should be handed out in *all* responses, and 192.168.1.1 only in
(roughly) one third of the responses.

When *traffic* sees a reply for a service it is authoritative for, the following will occur. On
seeing a reply the for a known service name the answer section will be matched to the backends.
For A and AAAA queries this means matching on the owner name, for SRV and MX this will match on
the target name (and port). Any unknown backends will be removed from the set as will be backends
that are assigned zero load. Next, one record will be selected as the one to be handed out for this
query; this will be the *only remaining* record in the reply; all other records are stripped from
it.

If all assignments are removed from a reply, and we're left with an empty response, *traffic* will
return a NODATA response. It will *never* sent an NXDOMAIN. The SOA record will be synthesised, and
has a low TTL (and negative TTL) of 5 seconds.

Any RRSIG are *stripped* from messages where *traffic* is authoritative for.

## Bugs

This plugin does not work well with DNSSEC.

## TODO

Queries for, or replies with CNAMEs and DNAME need some more thought. Add source address information
(geographical load balancing) to the assignment? This can be handled be having each backend
specificy an optional source range there this record should be used. For IPv4 this must a /24 for
IPv6 a /64.
