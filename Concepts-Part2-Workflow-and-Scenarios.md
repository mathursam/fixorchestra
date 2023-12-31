*Orchestra Concepts Part 2:*

# Workflow and Scenarios

**In addition to message structures, an Orchestra file can also convey the workflow, or expected message exchange behaviors of a service offering. Scenarios are the key to Orchestra's power.**

## Overview

A session in FIX or other financial industry protocol involves an exchange of application messages of various types (aside from the messages of the underlying session and transport protocols). The expected message exchange behaviors are important part of the rules of engagement between counterparties. To use a service offering, a counterparty must send the right kind of message for the circumstances.

Some messages are part of a synchronous request/response pattern while others are asynchronous or unsolicited. Overall, we call these application behavioral patterns *workflow*. The Orchestra XML schema provides the means to express what kinds of responses are possible to a message, and when each possibility applies.

In FIX, message types are overloaded for meaning. An ExecutionReport (35=8) is sent when when an order is booked, when it executes immediately— either partially or fully, when it is rejected, and so forth. The message layouts of ExecutionReport events are slightly different for each use case. For instance, an execution conveys a trade quantity and price, while a rejection conveys a reject reason instead. Each use case of a message type is known in Orchestra as a *scenario*. Unlike legacy data dictionaries, Orchestra provides a way to specify each scenario individually rather than as a conglomeration of all its use cases.

### What's it good for?

Receiving machine-readable definitions of message exchange behaviors can be used for ...

* Generating stubs of application code
* Writing messaging test cases and validations
* Generating documentation for humans

## Actors and Flows

An actor plays a role with regard to a system described by an Orchestra file. For example, in an order routing system, one actor would be "trader" and another would be "market". An `<actor>` element is defined for each role in an Orchestra file. 

A `<flow>` element represents a stream of messages from one actor (source) to another (destination). Often, a pair of flows constitute a conversation between actors. For example, orders flow from trader to market while executions flow back from market to trader.

Here's an example with three flows between actors. As you can see, flows can have various attributes, but we won't dive into those details right now since we want to concentrate on message exchange. (You can refer to the Orchestra specification or XML schema for details.) We'll also show in a later tutorial how actors can have state information that influence messaging behaviors.

```xml
<fixr:actors>
	<fixr:actor name="BuySide"/>
	<fixr:actor name="Market"/>
	<fixr:flow name="OrderEntry" source="BuySide" destination="Market"
messageCast="unicast" reliability="idempotent"/>
	<fixr:flow name="Executions" source="Market" destination="BuySide"
messageCast="unicast" reliability="recoverable"/>
	<fixr:flow name="MarketData" source="Market" destination="BuySide"
messageCast="multicast" reliability="bestEffort"/>
</fixr:actors> 
```  

## Message Workflows

Each message defined in an Orchestra file belongs to a flow. From its `flow` attribute, you can infer the source and destination of the message from the `<flow>` object of the same name.

Here's an example of an order message on an "OrderEntry" flow, which is sent from BuySide to Market.

```xml
<fixr:message msgType="D" flow="OrderEntry" id="14" name="NewOrderSingle" abbrName="Order">
```

In the previous tutorial, we explored message structures. The NewOrderSingle message definition would contain its members within a `<structure>` element as we showed before, but we'll skip that for the moment because now we want to explore behaviors.

A sibling to the `<structure>` element is the `<responses>` element where workflow is defined. The `<responses>` parent element may contain any number of `<response>` elements.

For now, we'll concentrate only on sending message responses, but be aware that Orchestra also can specify triggering of other kinds of events as well, including timers and actor state changes. (See the XML schema for all forms.) Each response that results in sending a message contains a `<messageRef>` element. 

Here's a simplified view of the possible message responses to a NewOrderSingle message:

```xml
<fixr:responses>
	<fixr:response name="orderAck">
		<fixr:messageRef name="ExecutionReport" msgType="8" implMinOccurs="0" implMaxOccurs="1" id="9" scenario="base" implMinOccurs="1">
		</fixr:messageRef>
		<fixr:when>$Market.SecMassStatGrp[SecurityID==in.SecurityID].SecurityTradingStatus != ^TradingHalt and $Market.Phase == "Open"</fixr:when>
	</fixr:response>
	<fixr:response name="invalidStateRejection">
		<fixr:messageRef name="ExecutionReport" msgType="8" implMinOccurs="0" implMaxOccurs="1" id="9" scenario="rejected" implMinOccurs="1"/>
		<fixr:when>$Market.Phase == "Closed"</fixr:when>
	</fixr:response>
	<fixr:response name="badInstrumentRejection">
		<fixr:messageRef name="BusinessMessageReject" msgType="j" implMinOccurs="0"  implMaxOccurs="1" id="43" scenario="base" implMinOccurs="1">
		</fixr:messageRef>
		<fixr:when>!exists $Market.SecMassStatGrp[SecurityID==in.SecurityID]"</fixr:when>
	</fixr:response>
	<fixr:response name="orderFilled" sync="asynchronous">
		<fixr:messageRef name="ExecutionReport" msgType="8" implMinOccurs="0" id="9" scenario="traded" implMinOccurs="1">
		</fixr:messageRef>
		<fixr:when>$Market.SecMassStatGrp[SecurityID==in.SecurityID].SecurityTradingStatus != ^TradingHalt and $Market.Phase == "Open"</fixr:when>
		<fixr:annotation >
			<fixr:documentation >One or more fills may occur; the first one may be synchronous.</fixr:documentation>
		</fixr:annotation>
	</fixr:response>
</fixr:responses>
```

Here are some important elements and attributes to notice:

* Each of the `<messageRef>` elements contains the identifiers of a message that must be defined elsewhere in the Orchestra file. That is, the `id` of the messageRef must match the `id` of a corresponding `<message>` element. That's where you would find the `<structure>` of the response message.
* Every message and messageRef has a `scenario` attribute. If not provided, its value is "base" by default. To restate the point above more precisely, a messageRef matches a message definition by combination of its `id` and `scenario` attributes. That combination defines a specific use case of a message type. The structure of ExecutionReport (35=8) message of scenario `rejected` is different than scenario `traded`, and so forth. Not all the responses need be the same message type, as shown by the response of BusinessMessageReject.
* Each of the responses shown here also contains a `<when>` element. That element contains a Score DSL expression that must evaluate to true or false (called a predicate in formal logic). If it evaluates to true, then its response is triggered. If there is no `<when>` element in a response, then it is triggered unconditionally.
* Responses can have other attributes such as a minimum or maximum number of times it can occur. For example, `implMinOccurs="0" implMaxOccurs="1"` on the acknowledgement response says that it can occur at most once for a given order but may never occur. By default, there is no limit to the number of times a response can trigger.
* Also, responses can be specified as synchronous or asynchronous. The fill response is specified as `sync="asynchronous"` since the timing of a trade is not determined by the time that the order was entered into the market. It could occur seconds, minutes or hours later (if ever).

Here's how the workflow looks as a UML sequence diagram:

![Sequence Diagram](./media/NewOrderSingle-base.png)

Not shown in the example, a response can also tell how the key identifiers of a response message relate to the original message. For example, an ExecutionReport (35=8) generally carries the ClOrdId (11) field of the original order message.

## Message Scenarios

Let's compare to of the responses in the example above but with some details stripped out.

```xml
<fixr:response name="orderAck">
	<fixr:messageRef name="ExecutionReport" msgType="8" id="9" scenario="base"/>
</fixr:response>
<fixr:response name="orderFilled">
	<fixr:messageRef name="ExecutionReport" msgType="8" id="9" scenario="traded"/>
</fixr:response>
```

The two responses both refer to a response message `<messageRef>` and both of them are ExecutionReport (35=8) messages. The events are not the same though, and although both responses have `msgType="8"`, the messasge layouts are not the same. Both ExecutionReport messages have the usual identifiers, only a trade produces a price and quantity while an order acknowledgement does not. Likewise, and ExecutionReport for order expiration or rejection would contain different fields. Unlike legacy FIX data dictionaries, Orchestra can provide a precise message layout for each event.

Here's a simplified view of the "traded" scenario message definition:

```xml
<fixr:message msgType="8" category="SingleGeneralOrderHandling" flow="Executions" id="9" name="ExecutionReport" abbrName="ExecRpt" scenario="traded">
	<fixr:structure>
		<fixr:componentRef presence="required" added="FIX.2.7" id="1024"/>
		<fixr:fieldRef id="37" presence="required"/>
		<fixr:fieldRef id="11" implMaxLength="20" scenario="base">
			<fixr:assign>in.ClOrdID</fixr:assign>
		</fixr:fieldRef>
		<fixr:fieldRef id="17" presence="required" scenario="base"/>
		<fixr:fieldRef id="39" presence="required" scenario="base"/>
		<fixr:componentRef presence="required" id="1003" scenario="base"/>
		<fixr:fieldRef id="54" presence="required" scenario="base">
			<fixr:assign>in.Side</fixr:assign>
		</fixr:fieldRef>
		<fixr:fieldRef id="40" scenario="base" presence="optional">
			<fixr:assign>in.OrdType</fixr:assign>
		</fixr:fieldRef>
		<fixr:fieldRef id="59" scenario="base" presence="optional">
			<fixr:assign>in.TimeInForce</fixr:assign>
		</fixr:fieldRef>
		<fixr:fieldRef id="151" presence="required" scenario="base"/>
		<fixr:fieldRef id="14" presence="required" scenario="base"/>
		<fixr:fieldRef id="60" scenario="base" presence="optional"/>
		<fixr:componentRef presence="required" id="1025" scenario="base"/>
	</fixr:structure>
</fixr:message>
```

One of the implications of scenario design is that the multitude of optional fields tends to fall away. Rather, each scenario contains just the required fields for its use case. Message layouts are thus much more deterministic than without scenarios.

## Next

How to encode conditional expressions using the Score DSL [Orchestra Concepts Part 3: Condtional Expressions](https://github.com/FIXTradingCommunity/fix-orchestra/wiki/Concepts-Part3-Conditional-Expressions)

### Back
[Orchestra Concepts Part 1: Message Structures](https://github.com/FIXTradingCommunity/fix-orchestra/wiki/Concepts-Part1-MessageStructures)

**© Copyright 2019 FIX Protocol Ltd.**