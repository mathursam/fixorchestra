*Orchestra Concepts Part 3:*

# Conditional Expressions

**Conditional expressions tell when something should happen or when a rule applies. Orchestra provides a language called Score to form expressions that machines can read and evaluate.**

## Overview

Rules of engagement between counterparties often depend on the values in a message or external state information. One use case is *conditionally required* fields—whether a field is required is contingent on the value of another field. In the FIX specifications, this has traditionally been expressed in text comments, often following the phrase "required if..." Another use case for condional expressions is to select when one workflow reponse is triggered among several alternatives. Score expressions can also be used to set field values.

The problem with text narratives is that are designed for human readability. Unfortunately, they are not easily parsed by computer programs, and even to humans, the narrative is often vague or ambiguous. At least in the case of a conditionally required field, the condition is usually next to the field definition. In the case of workflow, the problem is usually worse—either the flow is not explained at all, or the documentation of the flow is disconnected from the message definitions, requiring humans to put it together in their heads.

The goal of Orchestra is to make rules of engagement machine readable, precise, and unambiguous. The rest of this tutorial gives a tutorial for how to form machine readable condtional expressions.

### What's it good for?

Receiving machine-readable expressions can be used for ...

* Writing messaging test cases and validations
* Evaluating conditions at run-time
* Populating messages from field assignment expressions

## Score DSL

Score is the part of Orchestra to form expressions. It is what is called a Domain Specific Language (DSL). It was designed to be easy to learn for software developers since it is highly similar to the expression syntax of common programming languages including C, C++, C#, and Java. It should also look familiar to anyone who has written spreadsheet formulas. It was expected that there would be these two groups of users, so the syntax allows some alternatives. For example, an advanced business user can use the word "or" to combine logical expressions while a software engineer uses the symbol "||" to mean the same thing but using a form common in technical expression syntaxes. The two symbols are interchangeable in Score. 

The complete syntax of Score is defined in the Orchestra specification. However, this tutorial won't cover every feature of Score, but it should give enough of the gist to get you started.

### Conditionally required field

Let's say that a message definition contains this field reference:

```xml
<fixr:fieldRef id="99" scenario="base" presence="optional">
	<fixr:rule name="StopOrderRequiresStopPx" presence="required">
		<fixr:when>OrdType == ^Stop || OrdType == ^StopLimit</fixr:when>
	</fixr:rule>
	<fixr:annotation>
        <fixr:documentation>
         Required for OrdType = "Stop" or OrdType = "Stop limit".
        </fixr:documentation>
	</fixr:annotation>
</fixr:fieldRef>
```

It is a reference to the stop price field definition, matching on its `id` attribute:

```xml
<fixr:field type="Price" id="99" name="StopPx" abbrName="StopPx"/>
```

Before we break down the expression in the `<when>` element, see if you can guess its meaning without looking at `<documentation>` that pretty much gives it away.

I hope that if you understand FIX, that meaning was apparent, but let's analyze it carefully.

The field is by default optional in this message. That is, the field may be present in the message but is not required. However, the contained `<rule>` named "StopOrderRequiresStopPx" overrides the default presence by setting `presence="required"`. 

When is the rule activated? It depends on the `<when>` element. The content of that element is a Score DSL expression. When the expression evalutes to true, then the rule applies, and consquently, stop price is required in the message.

Let's examine the expression. Remember, that the symbol "||" is a synonym for the word "or". So there are two expressions connected by "or". If either of the parts is true, the whole expression evaluates to true.

Score also recognizes a logical "and" that connects two logical expressions; both parts must be true for the whole expression to be true. A synonym for "and" is the symbol "&&".

There is another non-word symbol in the expression "==", meaning equality. If the two entities on either side of an equality symbol evaluate to the same value, then the expression is true. 

(If you are not a programmer, I have to explain why it is a double equal sign. The reason is that equal sign actually has two meanings. One is a test of equality as used here. The other meaning is assignment, as when you assign a value to a variable in algebra. Since Score supports both tests of equality and assignment, it has to distinguish the two meanings. Therefore, like many programming languages, it uses a single equal sign for assignment and a double equal for equality.)

Like the logical operators, equality has both a mathmatical style symbol "==" and word style "eq". You can use the style that is more comfortable for you. In addition to equality, there are symbols for inequality.

| Token     | Name       |
| --------- | ---------- |
| \== or eq | equals     |
| \!= or ne | not equals |

The equality symbol is a comparions operator. It is just one kind of comparison. The other so-called relational operators are:

| Token     | Name                  |
| --------- | --------------------- |
| \< or lt  | less than             |
| \<= or le | less than or equal    |
| \> or gt  | greater than          |
| \>= or ge | greater than or equal |

For example, you can test `in.OrdQty > 0`, meaning that the expression is true if OrdQty in the incoming message is greater than zero.

Whenever you compare values, the two values being compared must be of compatible datatypes. For example, you can compare two prices but it doesn't make sense to compare a price to a security identifier.

What is "OrdType" in the expression? It is the value of the OrdType field in the same message. The field has tag 40, but you don't have to remember that to write the expression because the Score implementation can cross-reference the tag to its field definition. 

```xml
<fixr:field id="40" name="OrdType" abbrName="OrdTyp" type="OrdTypeCodeSet"  scenario="base"/>
```

Score lets you reference other fields in the same message, and even reference an incoming message in an expression pertaining to its response. Since "OrdType" is followed by an equality symbol, there is a test to evaluate whether the value of the OrdType field to some other value.

There is one more non-word symbol in the expression: "^". To explain it, let's show another piece of the Orchestra file. Notice that the type of the OrdType (40) field above is "OrdTypeCodeSet". That is not the name of a built-in FIX datatype, but rather is the name of a code set, shown below (simplified).

```xml
<fixr:codeSet type="char" id="40" name="OrdTypeCodeSet" scenario="base" >
	<fixr:code value="1"id="40001" name="Market"/>
	<fixr:code value="2"id="40002" name="Limit"/>
	<fixr:code value="3" id="40003" name="Stop"/>
	<fixr:code value="4" id="40004" name="StopLimit"/>
</fixr:codeSet>
```

As you can see, "Stop" and "StopLimit" are names of codes in the OrdTypeCodeSet. Again, you don't need to know the code values on the wire to write the Score expression, only the names of the codes. So, Score expressions are humanly understandable as well as machine readable.

Let's now put together everything we've learned about a Score expression for a conditionally required field and see if your guess was right. The Score expression can be translated to natural language as: "if the value of the OrdType field in this message is equal to the code Stop or is equal to the code StopLimit", then the rule states that the stop price field is required. Not only is the Score expression shorter than the English equivalent, it is unambiguous and can be parsed and evaluated by computer code. The cost to an author is learning a few special symbols, but you already know most of the important ones.

### Conditional message responses

In the previous tutorial, we discussed workflow. A message can have one or more `<response>` elements. Like a field `<rule>`, a `<response>` can have a `<when>` element containing a Score expression to tell when the response is triggered. The expression syntax is exactly the same as for a conditionally required field. If the expression evaluates to true, the response is fired. Typically, message responses involve external states as well as values in an incoming message, so we'll defer a fuller explanation to the next tutorial.

### Field assignment

A Score expression can be used to assign a field value in a response message, as shown in the example below.

```xml
<fixr:fieldRef id="11" implMaxLength="20" scenario="base" presence="optional">
	<fixr:assign>in.ClOrdID</fixr:assign>
</fixr:fieldRef>
```

Tag 11 is the identifier of the ClOrdId field. The `<assign>` element contains a Score expression to assign a value to that field in the response. Unlike the `<when>` elements above, this Score expression is *not* a true/false predicate, but rather evaluates to a value in the datatype of ClOrdId. What is that?

```xml
<fixr:field type="String" id="11" name="ClOrdID" abbrName="ClOrdID" scenario="base"/>
```

Answer: datatype String. It is contrained in the definition of the response message by the attribute `implMaxLength="20"`. In short, the outgoing ClOrdID (11) field must be a String of no more than 20 characters.

The prefix "in." of the assignment expression refers to the incoming message that triggered the response. To put it all together, the Score expression assigns the incoming ClOrdID (11) value to the outgoing ClOrdID field. In other words, it echoes to the input to the output, up to 20 characters.

You can assign numeric values using arithmetic operations. For example, amount can be assigned quantity times price.

Supported arithmetic operators are:

| Token    | Name           |
| -------- | -------------- |
| \*       | multiplication |
| /        | division       |
| % or mod | modulo         |
| \+       | addition       |
| \-       | subtraction    |

Formulas work as you'd expect in a spreadsheet. Parenthesis can be used for grouping.

You can test for existence of a field using the `exists` keyword. For example `exists in.StopPx` evaluates to true or false depending on whether the StopPx field is found in an incoming message.

## Next

[Orchestra Concepts Part 4: Actors and External States](../Concepts-Part4-Actors-And-External-States)

### Back
[Orchestra Concepts Part 2: Workflow and Scenarios](https://github.com/FIXTradingCommunity/fix-orchestra/wiki/Concepts-Part2-Workflow-and-Scenarios)

**© Copyright 2019 FIX Protocol Ltd.**