# What is in an Orchestra File?

## Overview

Orchestra is *not* a communication protocol in the ususal sense, nor is it an intermediary for messages on the wire. Rather, it describes message structures and behaviors of interacting peers about their communications rules--technically, metadata. The benefits of receiving such metadata include the ability to accurately prepare for behaviors in advance rather than discovering them at runtime, potentially leading to unexpected errors or breakdowns. 

### At the Application Layer
 
Orchestra defines messages and behaviors as viewed by the application layer, not bits and bytes as viewed from lower technical protocol layers. Another way to say it is that Orchestra deals with *semantics*. 

Note that Orchestra was designed to work for practically any interactive message protocol, not just for FIX. Even within the family of FIX protcols, there are multiple message encodings--including the traditional tag value format, FIXML, and binary encodings such as SBE--but they can all have the same business semantics.

### A Standard File Format

The Orchestra standard has two file schemas, one for message structures and workflows of messages, and the other describes protocol stacks and session configurations. In this tutorial, we'll cover the first one, which is historically a successor to FIX Repository. Therefore, another name for it is Repository 2016 Edition.

To ensure that an Orchestra file is interoperable, it must conform to an XML schema that is published as part of the Orchestra standard. See the XSD files [here](https://github.com/FIXTradingCommunity/fix-orchestra/tree/master/repository2016/src/main/resources/xsd) in GitHub.

## Message Structures

A message is a complete unit of data sent on the wire. A message is composed of fields, components, and repeating groups. Let's dive into the concepts and how Orchestra represents them.

### Field

A field is the smallest unit of business semantics. A field has an identifier, usually called a tag. In FIX, tags are a number, and FIX experts know common tags by heart, e.g. tag 44 is the identifier of the Price field. Tags are persistent--they remain the same forever for a given field and can never change or be reused. Only new field tags can be added.

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
All `<field>` XML elements are contained by their parent element `<fields>`.

Key parts of a field definition:

* The `name` attribute gives a humanly understood name. Ususally, this name is not sent on the wire.
* The `id` attribute, commonly known as tag, is a numeric identifier of a field. In tag value encoding, the tag is sent on the wire.
* The `abbrName` is an abbreviated name, which is used by a field element in FIXML since some field names can be very long.
* The `type` attribute tells the name of the datatype of a field. 
* The `scenario` attribute gives a particular useage of a field. We'll explain scenarios when we get to message definitions.
* Every message element in an Orchestra file can have documentation as shown. Also they can be documented with pedigree, the version history of the element, not shown here for simplicity.

### Datatype 

FIX has about 20 predefined datatypes. A datatype is a value space from an application perspective., that is, the range of possible values that a field can convey. In the example above, the Symbol field is of datatype `String`, a sequence of characters. There are other datatypes for numeric values, dates, times, and so forth. 

For each datatype, mappings can be defined for multiple message encodings. Technically, the rules for encoding a datatype in an ecoding is called its lexical space. Usually, you won't need to deal with that unless you are describing a new, non-FIX protocol, so we won't go into that in this tutorial. You can read about it in the Orchestra specification if you need to.

### Code set

There is a special kind of datatype that you probably do need to know, however. A code set describes a finite set of valid values of a field, also known as an enumeration, rather than a continuous range of values. A code set has an underlying datatype. In FIX, codes may be of String, int, or char datatypes. Lately, new code sets have all been numeric.

Here's an example of a code set of `char` type (simplified):

```xml
<fixr:codeSet type="char" id="40" name="OrdTypeCodeSet" scenario="base" supported="supported">
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

```xml
<fixr:field type="OrdTypeCodeSet" id="40" name="OrdType" abbrName="OrdTyp"/>
```

### Component Block

A component block is an assembly of fields that can be reused in multiple messages. The FIX standard defines many components. Instrument block is one of the most common ones; it is used in over half of FIX message types.

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

* A `<fieldRef>` is a reference to a defined `<field>`. The `id` attribute is the key for matching the reference to the field. So, `<fixr:fieldRef id="55"` matches the Symbol field as defined above. The point is, you only have to define a field once with its name, datatype, documentation, and so forth, but you can reuse that definition as many times as needed.
* Components may be nested. A `<componentRef>` is a reference to a contained component. Like a fieldRef, the `id` attribute is the key for matching. In this example, the component with `id="1060"` is called `SecurityXML`. All of the fields defined for that component are nested within the Instrument block.
* Suffice it say that a `<groupRef>` is a reference to a `<group>` repeating group that we'll talk about next. Like components, repeating groups can be nested within components or other repeating groups.

(We'll explain the `presence` attribute when we get to message.)

### Repeating Group

A repeating group is defined with the same contained elements as a component. The difference is that a repeating group can have multiple instances when sent on the wire. For example, the `Parties` group can carry data for multiple parties to a transaction. (Most repeating group names end in "Grp" but not that one.)

When sent on the wire in tag value encoding, the instances of a repeating group are preceeded by a counter to aid a message decoder. The counter is of a special datatype called `NumInGroup`, in essence an ordinal number.

Here's a simplified definition of the Parties group with its NumInGroup tag.

```xml
<fixr:group id="1012" name="Parties" abbrName="Pty" added="FIX.4.3" scenario="base">
	<fixr:numInGroup id="453"/>
	<fixr:fieldRef id="448" scenario="base" presence="optional"/>
	<fixr:fieldRef id="447" scenario="base" presence="optional"/>
	<fixr:fieldRef id="452" scenario="base" presence="optional"/>
</fixr:group>
```

### Message

Now that we have defined the building blocks of a message, we can define a message. It largely works the same way as component, but message has some extra parts.