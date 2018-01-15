Inadyn planning of IPv6 support
===============================

## What Works Already ##

* If ```checkip-server``` or ```checkip-command``` return an IPv6
  address, it can be used to update IPv6-only hostnames.
* The ```iface``` option may find an interface's IPv6 address, if
  that's the only address family (rare).


## Current IPv6 Support Problems ##

* Cannot determine two different addresses to use for one update
  request.
* DNS lookup for cache preseeding is currently IPv4-only
* The ```iface``` option might return a "temporary" address used for
  IPv6 Privacy Extensions, which is not useful for dynamic DNS.


## Basic Assumptions ##

Trying to describe the program's function as abstract as possible:

1. Domain names should be mapped to changing IP addresses.
2. Each service provides the user with individual authentication
   credentials.
3. The domains may have "alias" records below them which need to be
   addressed differently for updates.
4. Several names can be updated to the same address in one request by
   a special "group" or "location" update request.
5. For each DNS name, there are multiple possible record types which
   can coexist e.g. for different address families.
6. An update of several record types should take only one HTTP update
   request if the provider supports this.


## Implementation Ideas ##

* Which record type needs to be looked up for each provider section
  when preseeding cache?
  * store at least one IPv4 and one IPv6 address
  * possibly also MX?
* Store multiple address types in cache file, one per line
  * If checkip / iface result matches any of them, skip update

* Which parameter to use in the update protocol is provider specific.
  The ```myip``` parameter usually accepts any string valid for the
  named record type.  Only if multiple address types should be
  updated, additional parameters must be supplied (e.g. ```myipv6```)

* This could be extended for future support of MX or other record
  types, BUT:
  * MX and other records are not supported by getaddrinfo(), so cache
    preseeding is a totally different story.
  * This is a very special and very provider-specific use-case,
    already covered by ```ddns-path``` with placeholders as a
    workaround.
  * Will focus on the common IPv4 and IPv6 record types.


## Configuration Syntax ##

1. How to specify what records to update?  (especially when using
   provider's default checkip settings)
  * Supersede the global ```allow-ipv6``` setting?
  * Give each provider section a list option ```{ "ipv4", "ipv6" }```.
    Looks rather complicated.
  * Individual ```use-ipv4``` and ```use-ipv6``` boolean settings, at
    least one must be true.  Mirrored in provider info structure with
    defaults.  Defaults are backward-compatible for IPv4 only case.
  * Only one option ```addrtype = ipv4 | ipv6 | both```.  Not
    extensible to other record types.

2. ```checkip-*``` implicitly defines address type from the server's
   response.

3. ```iface``` option should be available for each provider section
   instead of globally, as an equal alternative to ```checkip-*```

4. IPv6 and IPv4 may need different ```checkip-*``` or ```iface```
   settings.


### Config Options as Lists ###

Extend the ```checkip-*``` and ```iface``` options to use lists for
multiple record types.

* Make ```checkip-server``` a list and see if we can collect both IPv6
  and IPv4 addresses.
* Need matching ```checkip-path``` setting per server.
* Cannot easily mix ```checkip-*``` and ```iface``` options.
* Not backwards-compatible because lists always need the curly braces.
* Lists can provide unlimited entries, we only need a predefined set
  (of currently two).

Example:

	checkip-server = { ipv6.example.net, ipv4.example.net }
	checkip-path = { /, / }
	checkip-command = { ... , ... } # which one to use now?
	addrtype = { ipv6, ipv4 } # defaults to only a single ipv4 entry, backwards-compatible


### Nested Record Sections ###

Add any number of "record" sections within each provider section.
Each one describes where to get the corresponding address from.

* Each section contains ```checkip-*``` or ```iface``` options.  Error
  if multiple sources are defined in one "record" section.
* Optionally force the expected IP address family with an ```addrtype```
  option.  Bail out if the determined address does not fit.
* Or use the section title to specify the address family?  If it can
  be optional, leave it out for auto detection, otherwise specify
  ```auto```.

* Fall-back for current option names, they are treated as a "record"
  section with auto-detected address family.
  * Could also handle the ```addrtype``` option this way to force the
    type if only one should be used.
  * If a proper "record" section is present, the fall-backs are no
    longer allowed.

* Reusable to implement MX record support later as well?

Examples:

	record { # explicit IPv6 from command
		addrtype = ipv6
		checkip-command = ...
	}
	record v4 { # explicit IPv4 (section title) from HTTP request
		checkip-server = ...
		checkip-path = ...
		checkip-ssl = no
	}
	record { # implicit address type from HTTP response
		checkip-server = ipv6.example.net
	}
	record mx { iface = ... } # possible future extension
