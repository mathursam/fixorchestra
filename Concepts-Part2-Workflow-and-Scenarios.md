*Orchestra Concepts Part 2:*

# Workflow and Scenarios

**In addition to message structures, an Orchestra file can also convey the workflow, or expected behaviors of a service. Scenarios are the key to Orchestra's power.**

## Overview

A session in FIX or other financial industry protocol involves an exchange of application messages of various types (aside from the messages of the underlying session and transport protocols.) Some messages are part of a synchronous request/response pattern while others are asynchronous or unsolicited. In general, we call these application behavioral patterns *workflow*. Orchestra schema provides the means to express what kinds of responses are possible to a request message, and when each possibility applies.

In FIX, message types are overloaded for meaning. An ExecutionReport is sent when when an order is booked, when it executes immediately--either partially or fully, when it is rejected, and so forth. The message layouts of ExecutionReport events are slightly different. For instance, an execution conveys a trade quantity and price, while a rejection conveys a reject reason instead. Each use case of a message type is known in Orchestra as a *scenario*. Unlike legacy message definitions, Orchestra provides a way to specify each scenario individually rather than as a conglomeration of all its use cases.

### What's it good for?

Receiving machine-readable definitions of message exchange behaviors can be used for ...

* Generating stubs of application code
* Writing messaging test cases and validations
* Generating documentation for humans

## Actors and Flows

An actor plays a role with regard to a system described by an Orchestra file. For example, in an order routing system, one actor would be "trader" and another would be "market". An `<actor>` element is defined for each role in an Orchestra file. A `<flow>` element represents a stream of messages from one actor (source) to another (destination). Often, a pair of flows constitute a conversation between actors. For example, orders flow from trader to market while executions flow back from market to trader.

Here's an example with three flows between actors. As you can see, flows can have various attributes, but we won't dive into those details right now since we want to concentrate on message exchange. We'll also show in a later tutorial how actors can have state information that influence messaging behaviors.

```xml
<fixr:actors>
	<fixr:actor name="BuySide"/>
	<fixr:actor name="SellSide"/>
	<fixr:flow name="OrderEntry" source="BuySide" destination="SellSide"
messageCast="unicast" reliability="idempotent"/>
	<fixr:flow name="Executions" source="SellSide" destination="BuySide"
messageCast="unicast" reliability="recoverable"/>
	<fixr:flow name="MarketData" source="SellSide" destination="BuySide"
messageCast="multicast" reliability="bestEffort"/>
</fixr:actors> 
```  

## Messages Flows

Each message defined in an Orchestra file belongs to a flow. From its `flow` attribute, you can infer the source and destination of the message from the `<flow>` object of the same name.

Here's an example of an order message on an "OrderEntry" flow:
```xml
<fixr:message msgType="D" flow="OrderEntry" id="14" name="NewOrderSingle" abbrName="Order">
```

In the last tutorial, we explored message structures. The NewOrderSingle message definition would continue with its field members within a `<structure>` element as we showed before, but we'll skip that for the moment because now we want to explore behaviors.

A sibling to the `<structure>` element is the `<responses>` element where workflow is defined. The `<responses>` parent element may contain any number of `<response>` elements.

For now, we'll concentrate only on sending message responses, but be aware that Orchestra also can specify triggering of other kinds of events as well.  Each response that results in sending a message contains a `<messageRef>` element. 

Here's a simplified view of the possible message responses to a NewOrderSingle message:

```xml
<fixr:responses>
	<fixr:response name="orderAck">
		<fixr:messageRef name="ExecutionReport" msgType="8" implMaxOccurs="1" id="9" scenario="base" implMinOccurs="1">
		</fixr:messageRef>
		<fixr:when>$Market.SecMassStatGrp[SecurityID==in.SecurityID].SecurityTradingStatus != ^TradingHalt and $Market.Phase == "Open"</fixr:when>
	</fixr:response>
	<fixr:response name="invalidStateRejection">
		<fixr:messageRef name="ExecutionReport" msgType="8" implMaxOccurs="1" id="9" scenario="rejected" implMinOccurs="1"/>
		<fixr:when>$Market.Phase == "Closed"</fixr:when>
	</fixr:response>
	<fixr:response name="badInstrumentRejection">
		<fixr:messageRef name="BusinessMessageReject" msgType="j" implMaxOccurs="1" id="43" scenario="base" implMinOccurs="1">
		</fixr:messageRef>
		<fixr:when>!exists $Market.SecMassStatGrp[SecurityID==in.SecurityID]"</fixr:when>
	</fixr:response>
	<fixr:response name="orderFilled" sync="asynchronous">
		<fixr:messageRef name="ExecutionReport" msgType="8" implMaxOccurs="unbounded" id="9" scenario="traded" implMinOccurs="1">
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
* Every message and messageRef has a `scenario` attribute. If not provided, its value is `base` by default. To restate the point above more precisely, a messageRef matches a message definition by combination of its `id` and `scenario` attributes. That combination defines a specific use case of a message type. The structure of ExecutionReport message of scenario `rejected` is different than scenario `traded`, and so forth.
* Each of the responses shown here also contains a `<when>` element. That element contains a Score DSL expression that must evaluate to true or false. If it evaluates to true, then its response is triggered. (If there is no `<when>` element in a response, then it is triggered unconditionally.)
* Responses can have other attributes such as a minimum or maximum number of times it can occur. For example, acknowledgement can only occur once for a given order. Also, responses can be specified as synchronous or asynchronous.

Not shown here, a response can also tell how the key identifiers of a response message relate to the original message. For example, an ExecutionReport generally carries the ClOrdId of the original order message.

## Next

How to encode conditional expressions using the Score DSL