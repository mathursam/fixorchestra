# Concepts Part 1: Message Structures

**An Orchestra file contains message definitions and their building blocks.**

## Overview

Orchestra is *not* a communication protocol in the ususal sense, nor is it an intermediary for messages on the wire. Rather, it describes message structures and behaviors of interacting peers about their communications rules--technically, metadata rather than data. The benefits of receiving such metadata include the ability to accurately prepare for behaviors in advance rather than discovering them at runtime, potentially leading to unexpected errors or breakdowns. 

### At the Application Layer
 
Orchestra defines messages and behaviors as viewed by the application layer, not bits and bytes as viewed from lower technical protocol layers. Another way to say it is that Orchestra deals with *semantics*. 

Orchestra was designed to work for practically any interactive message protocol, not just for FIX. Even within the family of FIX protcols, there are multiple message encodings--including the traditional tag value format, FIXML, and binary encodings such as SBE--but they can all have the same business semantics.

### A Standard File Format

The Orchestra standard has two file schemas, one for message structures and workflows of messages, and the other describes protocol stacks and session configurations. In this tutorial, we'll cover the first one, which is historically a successor to FIX Repository. Therefore, another name for it is Repository 2016 Edition.

To ensure that an Orchestra file is interoperable, it must conform to an XML schema that is published as part of the Orchestra standard. See the repository2016 XSD files [here](https://github.com/FIXTradingCommunity/fix-orchestra/tree/master/repository2016/src/main/resources/xsd) in GitHub.

## Message Structures

Orchestra is not a communications protocol, so it does not deal with instances of messages on the wire, but rather with templates for formation of messages. A message is a complete unit of data sent on the wire. It is composed of fields, components, and repeating groups. Let's dive into these concepts and how Orchestra represents them.

### Field

A field is the smallest unit of business semantics. A field always has the same meaning wherever it used. For example, the Price field is used in many message types, but it always means "price per unit of quantity".

A field has an identifier called a tag. In FIX, tags are numeric, and FIX experts know common tags by heart, e.g. tag 44 is the identifier of the Price field. Tags are persistent--they remain the same forever for a given field and can never change or be reused. Only new field tags can be added.

How the Symbol (55) field is represented in Orchestra (simplified):

```xml
<fixr:field type="String" id="55" name="Symbol" abbrName="Sym" scenario="base">
	<fixr:annotation>
	<fixr:documentation purpose="SYNOPSIS">
         Ticker symbol. Common, "human understood" representation of the security.
    </fixr:documentation>
	</fixr:annotation>
</fixr:field>
```

Key parts of a field definition:

* The `name` attribute gives a humanly understood name. Typically, this name is not sent on the wire by message processors; it's metadata to aid humans.
* The `id` attribute, commonly known as tag, is a numeric identifier of a field. In tag value encoding, the tag is sent on the wire.
* The `abbrName` is an abbreviated name, which is used as a shorter tag in FIXML elements since field names can be very long.
* The `type` attribute tells the name of the datatype of a field (explained below). 
* The `scenario` attribute gives a particular useage of a field. We'll explain scenarios when we get to message definitions.
* Every message element in an Orchestra file can have humanly readable documentation as shown. 

Also, message elements can be documented with *pedigree*, the version history of the element, not shown here for simplicity.

All `<field>` XML elements are contained by their parent element `<fields>` in an Orchestra file.

### Datatype 

FIX has about 20 predefined datatypes. A datatype is a *value space*, that is, the range of possible values that a field can convey. In the example above, the Symbol field is of datatype `String`, a sequence of characters. There are other datatypes for numeric values, dates, times, and so forth. 

For each datatype, mappings can be defined for multiple message encodings, such as XML, SBE, and so forth. Technically, the rules for encoding a datatype is called its *lexical space*. Usually, you won't need to deal with that unless you are describing a new, non-FIX protocol, so we won't go into that in this tutorial. You can read about it in the Orchestra specification if you need to.

### Code set

There is a special kind of datatype that you probably do need to know about, however. A code set describes a finite set of valid values of a field called codes, as opposed to a continuous range of values. A code set is also known as an *enumeration*.

A code set has an underlying datatype for all its code values. In FIX, codes may be of `String`, `int`, or `char` datatypes. Lately, new code sets have all been numeric.

Here's an example of a code set of `char` type that enumerates order types (simplified):

```xml
<fixr:codeSet type="char" id="40" name="OrdTypeCodeSet" scenario="base">
    <fixr:code value="1" id="40001" name="Market"/>
    <fixr:code value="2" id="40002" name="Limit"/> 
    <fixr:code value="3" id="40003" name="Stop"/>
    <fixr:code value="4" id="40004" name="StopLimit"/>
    <fixr:code value="A" id="40010" name="OnClose"/>			
</fixr:codeSet>
```

Like a field, a code set and the codes that it contains have a humanly understood `name` attribute, and a numeric `id`.

Each code has a `value`, the value sent on the wire. In the example, many of the code values look like numbers, but they are actually characters, as shown by code value "A".

A code set may be used as a datatype by one or more fields. In FIX Latest, more than 20 fields share `SecurityIDSourceCodeSet`, but its codes only need to be defined once in an Orchestra file.

All `<codeSet>` XML elements are contained by their parent element `<codeSets>` in an Orchestra file.

Here's how the code set above is assigned as the datatype of a field:

```xml
<fixr:field type="OrdTypeCodeSet" id="40" name="OrdType" abbrName="OrdTyp"/>
```

In a message, when OrdType field (tag 40) is sent on the wire, the character value of the field signifies one order type from the code set.

### Component Block

A component block is a collection of fields that can be reused in multiple message types. The FIX standard defines many components. Instrument block is one of the most common ones; it is used in over half of FIX message types.

A simplified definition of Instrument block:

```xml
<fixr:component id="1003" name="Instrument" abbrName="Instrmt">
    <fixr:fieldRef id="55" scenario="base" presence="optional"/>
    <fixr:fieldRef id="48" scenario="base" presence="optional"/>
    <fixr:fieldRef id="22" scenario="base" presence="optional"/>
    <fixr:groupRef id="2071" scenario="base" presence="optional"/>
    <fixr:componentRef id="1060" scenario="base" presence="optional"/>
    <fixr:annotation >
	<fixr:documentation purpose="SYNOPSIS" >
         The Instrument component block contains all the fields commonly used to describe a security or instrument. Typically the data elements in this component block are considered the static data of a security, data that may be commonly found in a security master database. The Instrument component block can be used to describe any asset type supported by FIX.
    </fixr:documentation>
    </fixr:annotation>
</fixr:component>
```

The elements of a component may be any combination of the following:

* A `<fieldRef>` is a reference to a defined `<field>` within the `<fields>` container. The `id` attribute is the key for matching the reference to the field. So, `<fixr:fieldRef id="55"` matches the Symbol field as defined above. The point is, you only have to define a field once with its name, datatype, documentation, and so forth, but you can reuse that definition as many times as needed.
* Components may be nested. A `<componentRef>` is a reference to a contained component. Like a fieldRef, the `id` attribute is the key for matching. In this example, the component with `id="1060"` is called `SecurityXML`. All of the fields defined for that component are nested within the Instrument block.
* Suffice it say that a `<groupRef>` is a reference to a `<group>` repeating group that we'll talk about next. Like components, repeating groups can be nested within components or other repeating groups.

We'll explain the `presence` attribute in depth when we get to message, but for now just know that "optional" means that field need not be sent on every message.

All `<component>` XML elements are contained by their parent element `<components>` in an Orchestra file.

### Repeating Group

A repeating group is like a component, but a repeating group can have multiple instances when sent on the wire. For example, the `Parties` group can carry data for multiple parties to a transaction. (Most repeating group names end in "Grp" but this one is an exception.)

When sent on the wire in tag value encoding, the instances of a repeating group are preceeded by a counter to inform a message decoder how many instances to parse. The counter is a field of a special datatype called `NumInGroup`, in essence a cardinal number. The only difference between a repeating group definition and a component is the addition of `<numInGroup>` element to specify the tag of the group counter. Otherwise, it contains the same field, component and nested group reference formats. 

Here's a simplified definition of the Parties group with its NumInGroup tag.

```xml
<fixr:group id="1012" name="Parties" abbrName="Pty" scenario="base">
	<fixr:numInGroup id="453"/>
	<fixr:fieldRef id="448" scenario="base" presence="optional"/>
	<fixr:fieldRef id="447" scenario="base" presence="optional"/>
	<fixr:fieldRef id="452" scenario="base" presence="optional"/>
</fixr:group>
```

All `<group>` XML elements are contained by their parent element `<groups>` in an Orchestra file.

### Message

Now that we have defined the building blocks of a message, we can define a message. Like a component definition, its structure also contains field, component and group references. However, a message has some extra parts.

First, `<message>` has an extra identifier, namely attribute `msgType`, the traditional FIX protocol message identifier. 

Another difference between the format of a message and component is that the structure of a message resides under a `<structure>` element. The reason for it is that A message can also have workflow aside from structure. That's what the `flow` attribute is about, too. We'll cover workflow in a later tutorial.

Here's a simplified definition of an ExecutionReport message that is sent when a trade occurs: 

```xml
<fixr:message msgType="8" flow="Executions" id="9" name="ExecutionReport" abbrName="ExecRpt" scenario="traded">
	<fixr:structure>
		<fixr:componentRef presence="required" id="1024"/>
		<fixr:fieldRef id="37" presence="required" scenario="base"/>
		<fixr:fieldRef added="FIX.5.0SP2" id="2422" scenario="base" presence="optional"/>
		<fixr:fieldRef id="11" implMaxLength="20" scenario="base" presence="optional">
			<fixr:assign>in.ClOrdID</fixr:assign>
		</fixr:fieldRef>
		<fixr:groupRef id="1012" scenario="base" presence="optional">
			<fixr:annotation>
			<fixr:documentation>
         Specifies party information related to the submitter.
            </fixr:documentation>
			</fixr:annotation>
		</fixr:groupRef>
        <fixr:componentRef presence="required" id="1025"/>
	</fixr:structure>
</fixr:message>
```

Message structure definitions have many optional attributes. One in the example, `implMaxLength` informs users that ClOrdID (tag 11) is limited to 20 characters.

All `<message>` XML elements are contained by their parent element `<messages>` in an Orchestra file.

### Presence

Orchestra provides these possible values of `presence`:

* **required** -- the field or component MUST be present.
* **optional** -- the field or component MAY be present; it may be conditionally required based on a rule.
* **forbidden** -- The field or component MUST NOT be present.
* **ignored** -- the field or component MAY be present but is not validated.
* **constant** -- the field has a constant value. (In some encodings, contants need not be sent on the wire.)

In the above example, fieldRef with `id="37"`, the OrderID, field has attribute `presence="required"`. This means that the rules of engagement state that in this message type, OrderID is a required field--it must be included in every instance of the message to be valid.

Frequently in FIX, fields are conditionally required, that is, it is required when some condition about the message is true. For example, StopPx field is required *when* the value of the OrdType field is Stop or StopLimit. Fortunately, Orchestra has syntax to express such rules -- Orchestra's **Score** Domain Specific Lanaguage (DSL). Score can also used to express assignment of field values. In the example, the ClOrdId field (tag 11) in the ExecutionReport is assigned the value from the incoming order message. We'll leave the syntax of that language for an advanced tutorial, but you should get the gist of the example.

### Scenarios

In FIX, message types are often overloaded for many different meanings. We call each specialization or use case of a message a *scenario*. For example, there may be scenarios of ExecutionReport message type for when an order is booked, when it trades, when it is rejected, or when an order is replaced.

Another reason to use scenarios is that different security types need different sets of fields to describe them. So an option order requires slightly different fields, like MaturityMonthYear (tag 200), that is not required for an equity order. The interesting thing here is that MaturityMonthYear is not directly included in the definition of NewOrderSingle (msgtype="D"), but rather is included in the Instrument component.

To deal with different views of the Instrument block, we could define scenarios for it, one for equities, one for options, and so forth. Each is distinguished by its `scenario` name. An equity only requires a Symbol or SecurityID, but an option also need MaturityMonthYear, StrikePrice, and put or call classification.

```xml
<fixr:component id="1003" name="Instrument" abbrName="Instrmt" scenario="equity">
    <fixr:fieldRef id="55" scenario="base" presence="requried"/>
    <fixr:fieldRef id="48" scenario="base" presence="optional"/>
    <fixr:fieldRef id="22" scenario="base" presence="optional"/>
</fixr:component>

<fixr:component id="1003" name="Instrument" abbrName="Instrmt" scenario="option">
    <fixr:fieldRef id="55" scenario="base" presence="required"/>
    <fixr:fieldRef id="48" scenario="base" presence="optional"/>
    <fixr:fieldRef id="22" scenario="base" presence="optional"/>
    <fixr:fieldRef id="200" scenario="base" presence="required"/>
    <fixr:fieldRef id="201" scenario="base" presence="required"/>
    <fixr:fieldRef id="202" scenario="base" presence="required"/>
</fixr:component>
```

Another common reason to use scenarios is to have different views of code sets. Inbound orders may use one scenario of PartyRole while outbound execution reports use a more enriched version with more codes in the code set.

## Next

In subsequent tutorials, we explain concepts of workflow, conditional expressions, and other details of Orchestra.



