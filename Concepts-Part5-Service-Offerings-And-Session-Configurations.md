*Orchestra Concepts Part 5:*

# Service Offerings and Session Configurations

**The Orchestra Interfaces schema provides a standardized way to specify service offerings and configure sessions for connectivity.**

## Overview

Every system that allows customers or partners to connect uses some stack of communications protocotols. Those protocols must be specified to the peers in order to establish connectivity. We call an application-layer protocol a service offering. A firm may have several service offerings for different business phases, such as order routing, market data, reference data, and post-trade clearing and settlement. The Orchestra Interfaces schema provides a standardized way to specify those APIs.

Commonly, order entry customers are directed to a dedicated gateway by a listening address and port, and sessions are assigned specific identifiers. In FIX, sessions are traditionally identified by a combination of SenderCompID and TargetCompID, but other protocols have different identifiers. It is now common to use public/private key infrastructure to securely identify parties. The Orchestra Interfaces schema provides features for  transport connectivity and user identification, either by declared IDs or with security keys.

## Service Offerings

Each service offering in an Orchestra Interfaces file is specified by an `<interface>` element under the root `<interfaces>`. The application layer protocol for a service offering is named in a `<service>` element. A service is configured with an *orchestration*, another Orchestra file in the repository schema. It contains the message structures and workflow associated with that service in formats described by parts 1 and 2 of this tutorial. 

An orchestration file may be distributed to peers in a bundle with an interfaces file or may be accessible through a web inteface.

### Protocol Stack

Below application layer, a service runs over a protocol stack that includes a session protocol, message encoding, transport, and possibly other protocols. At each layer, a protocol version and configuration may be given.

Here's a sample interface definition with two orchestrations. It all works over TagValue encoding, FIXT session layer, and TCP transport.

```xml
<fixi:interface name="Private">
	<!-- one or more service offerings, with local orchestration file or internet address -->
	<fixi:service name="orderEntry" orchestration="https://mydomain.com/orchestra/orderEntry.xml"/>
	<!-- the protcol stack -->
	<fixi:userInterface name="ATDL" orchestration="https://mydomain.com/orchestra/algo.xml"/>
	<fixi:encoding name="TagValue"/>
	<fixi:sessionProtocol name="FIXT.1.1" reliability="recoverable" orchestration="https://mydomain.com/orchestra/session.xml"/>
	<fixi:transport name="TCP"/>
</fixi:interfacer> 
 ```

 ## Session Configuration

 For each service offering, a party may be assigned one or more sessions. A session configuration includes session identifiers, transport settings, and possibly security keys. Each `<sessions>/<session>` element is contained by an `<interface>` that describes its service offering.

 Here's and example of a session configuration:

```xml
<fixi:session name="XYZ-ABC">
	<!-- inherits services and protocols from interface -->
	<!-- alternate addresses are supported -->
	<fixi:transport address="10.96.1.2:567" use="primary"/>
	<fixi:transport address="10.96.2.2:567" use="secondary"/>
	<!-- there can be any number of identifiers -->
	<fixi:identifier name="SenderCompID">
		<fixi:value>XYZ</fixi:value>
	</fixi:identifier>
	<fixi:identifier name="TargetCompID">
		<fixi:value>ABC</fixi:value>
	</fixi:identifier>
	<!-- tells when session becomes effective so it can be configured in advance -->
	<fixi:activationTime>2019-08-07T09:30:00Z</fixi:activationTime>
</fixi:session>
```

A `<session>` element can contain configuration for any protocols in the stack that is specific to the session. In this case, the transport is always TCP for the service offering, but each session contains transport settings that are unique to that session. In this example, a primary and secondary port are offered for fail-over.

Optionally, security keys may be provided for a session in a `<securityKeys>` element. It must follow the textual encoding format described in IETF RFC 7468. Optional parts include certificates and private keys.


### Back
[Orchestra Concepts Part 4: Actors and External States](https://github.com/FIXTradingCommunity/fix-orchestra/wiki/Concepts-Part4-Actors-And-External-States)

**© Copyright 2019 FIX Protocol Ltd.**