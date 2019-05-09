## Use Cases

### How should you manage multiple rules of engagement using Orchestra?
Q. For instance, how do you use Orchestra to represent multiple order routing destinations? An order routing hub is responsible for translating incoming messages into the FIX format accepted by each destination broker or market. 

A. To simplifiy the problem, each Orchestra file should only represent one service offering. In other words, one set of message structures and workflows exposed to clients of a system. Therefore, there should be one Orchestra file representing the hub to its direct clients, and one Orchestra file for each of the order destinations. The logic of the routing hub can use the set of Orchestra files to inform decision making about which routes have a particular capability, e.g. which ones accept a certain order type or message type.

## Orchestra Syntax

### How do you encode that a field is conditionally required depending on the value of another field?

Q. Say, for example, that a SecurityIDSource (tag 22) associated with SecurityID (48) can be SEDOL (code 2), ISIN (4), or RIC (5). In the rules of enagement, Currency (15) is usually an optional field, except when SecurityIDSource is ISIN, Currency is required. How can this rule be encoded in Orchestra?

A. Insert the following snippet into a message to encode the rule. The `<when>` element contains a Score DSL expression that tells when the rule is effective. It says that the Currency field (tag 15) is required *when* the value of the SecurityIDSource field in the same message is equal to the value of the code named ISIN. (See the Orchestra specification for the Score expression syntax.)
```xml
<fixr:fieldRef id="15">
	<fixr:rule name="currencyRule" presence="required">
		<fixr:when>SecurityIDSource==^ISIN</fixr:when>
	</fixr:rule>
</fixr:fieldRef>
```

**© Copyright 2019 FIX Protocol Ltd.**