# traffic

## Name

*traffic* - handout addresses according to assignments.

## Description

The *traffic* plugin is an advanced load balancer that allows traffic steering, weighted responses
and draining of endpoints. It receives (via gRPC) *assignments* that define the relative
importance of the endpoints. The plugin takes care of handing out responses that adhere to
this assignment. Assignments will need to be updated frequently to ensure load spreading, without
new updates *traffic* will hand out responses according to the last received assignment. When there
are no assignments for a name (yet), the responses will be let through as-is.

An assignment covers a "service name", which for all intent and purposes is just a domain name. With
each service name a number of backends are expected. A backend is defined as a IP:port pair and a
number telling how often this backend should be be handed out, relative to the other backends. A
negative number means the backend exists, but should not be handed out (drain it).

The *traffic* plugin will only balance for A, AAAA and SRV requests (see Assignments below). Also
note *traffic* works with any plugin that returns responses; i.e. *traffic* doesn't mandate a
backend storage; it can work with *file*, or *kubernetes*, etc. .

The *traffic* plugin has no notion of draining, drop overload and anything that advanced, *it just
acts upon assignments*.

## Syntax

~~~
traffic
~~~

This enables traffic loadbalancing for all subdomains of the server block.

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
information. This is an example assignment for the service "www.example.org":
~~~
assignments:
  assigment:
    - service: www.example.org
        - backend: 192.168.1.1:443
            use: 1
          backend: 192.168.1.2:443
            use: 3
          backend: 192.168.1.3:443
            use: -1
~~~
This particular one has 3 backends, one of which is to be drained (192.168.1.3), the remaining to
tell *traffic* that 192.168.1.2 should be handed out 3 times more often than 192.168.1.1.

When *traffic* sees a reply for a service it is authoritative for the one of the following will
occur depending on the query.

For an A query it will check all A records in the answer section with the backends for this
service. Any non-matching addresses *will be removed from the reply*. The remaining matching
addresses will be ordered according to the assignment.

For an AAAA query the same things happens as with an A query except we look for AAAA records.

For SRV queries things are different. And 2 things can happen, depending if the additional section
contains address records. If there are no address to be found the query it let through as-is. When
there are addresses to be found they are matched against any received assignments and only the SRV
records and address records that match an assignment are let through.

## Bugs

This plugin doesn't work with DNSSEC. Queries for, or replies with CNAMEs and DNAME need some more
thought.
