# How to Access an Orchestra File

**An Orchestra file contains message definitions and workflows. When you receive an Orchestra file from a trading counterparty, how do you access it programmatically to discover its elements?**

An Orchestra file is encoded in XML and must conform to the schema published in GitHub project [FIXTradingCommunity/fix-orchestra](https://github.com/FIXTradingCommunity/fix-orchestra). There are actually two XML schemas in Orchestra, in modules `repository2016` and `interfaces2016`. The first is for message definitions and workflows, and the latter is for defining service offerings, protocol stacks, and sessions. For this tutorial, we'll work with the repository schema, but interfaces access works the same way technologically.

## Prequisites
For this tutorial, we'll work in the Java programming language, using Maven as the build tool. So you'll need Java 8 or later and Maven 3 for the steps below. (Of course, XML can be read and manpulated in other programming languages using appropriate tools.)

## Step 1: Create a new project
Create a new simple Maven project. IDEs such as Eclipse or Intellij IDEA have menus and dialogs for this.

Code to access or update an Orchestra file is available as open source. To make it easy on yourself, let Maven retrieve a pre-built excutable for you. If you'd rather dive into detail, you can build the dependency yourself from source--see below.

### Use pre-built XML bindings
XML bindings relieve you of having to deal with XML details such as elements and attributes. Rather, you just use plain old classes and objects (POJOs).

Orchestra XML bindings generated with Java Architecture for XML Binding (JAXB) are already baked into an open-source project for you to use. Add this snippet to the `<dependencies>` element of your Maven project file. (Check if there is a later version available in [Maven Central repository](https://mvnrepository.com/artifact/io.fixprotocol.orchestra/repository2016).) When you build your project, Maven will retrieve each dependency in your project automatically.

```xml
<dependency>
  <groupId>io.fixprotocol.orchestra</groupId>
  <artifactId>repository2016</artifactId>
  <version>1.3.0-RC4</version>
</dependency>
```

*Note:* if you're using Java 9 or later, you'll have to add dependency for module `javax.xml.bind` since JAXB has been moved out of the base module.

### Alternative: Generate XML bindings from source
If you wish to build the XML bindings from scratch, retrieve the source for the module described above from GitHub module [repository2016](https://github.com/FIXTradingCommunity/fix-orchestra/tree/master/repository2016). Tell your IDE to build the project, or issue this command line from the root directory of the dependency project:
```
mvn install
```
In the `generate-sources` phase of the build, JAXB source code is generated to access or update elements of an XML file that conforms to the Orchestra schema. Then the source code is compiled, packed into a .jar file, and stored in you local Maven repository, making it available to your new project.

## Step 2: Create a class to report Orchestra messages
Let's create a class to report the messages contained in an Orchestra file. For this to work, the class has to parse the XML file and convert XML constructs into Java objects. The class will have a method to accept an input stream for the file, and an output stream to send text to. Start with this outline:

```java
package com.example.orchestra.sample;

import java.io.InputStream;
import java.io.PrintStream;
import javax.xml.bind.JAXBContext;
import javax.xml.bind.JAXBException;
import javax.xml.bind.Unmarshaller;
import io.fixprotocol._2016.fixrepository.Repository;

public class RepositoryReporter {
  
  
  public void report(InputStream is, PrintStream os) {
    try {
      // parse the XML
      Repository repository = unmarshal(is);
      // report messages
      // TODO
    } catch (JAXBException e) {
      // print exception
      os.format("ERROR: %s%n", e.getMessage());
    }
  }
  
  private Repository unmarshal(InputStream is) throws JAXBException {
    final JAXBContext jaxbContext = JAXBContext.newInstance(Repository.class);
    final Unmarshaller jaxbUnmarshaller = jaxbContext.createUnmarshaller();
    return (Repository) jaxbUnmarshaller.unmarshal(is);
  }

}
```
To report all the messages in the Orchestra file, insert this snippet into the `report()` method. It accesses a list of all messages in the repository and invokes a new method to report each message.

```java
      List<MessageType> messageList = repository.getMessages().getMessage();
      reportMessages(messageList, os);
```
The new method is:

```java
  void reportMessages(List<MessageType> messageList, PrintStream os) {
    // iterate all messages in the file
    for (MessageType message: messageList) {
      os.format("Message name: %s scenario: %s MsgType: %s%n", 
          message.getName(), message.getScenario(), message.getMsgType());
      // report message members
      // TODO
    }  
  }
```

To list the members of a message structure, insert this snippet into the `reportMessages()` method. The structure of a message is mixed collection of field, component, and repeating group references.

```java
      List<Object> members = message.getStructure().getComponentRefOrGroupRefOrFieldRef();
      reportMembers(members, os);
```
Add a new method to report each of the members. It has to check the type of each object in the mixed collection. To keep it simple, we'll only report the identifiers of each member, but you can expand the code to report more attributes.

```java
  void reportMembers(List<Object> members, PrintStream os) {
    for (Object member : members) {
      if (member instanceof FieldRefType) {
        FieldRefType fieldRef = (FieldRefType)member;
        os.format("\tFieldRef id: %d scenario: %s%n", fieldRef.getId(), fieldRef.getScenario());
      } else if (member instanceof ComponentRefType) {
        ComponentRefType componentRef = (ComponentRefType)member;
        os.format("\tComponentRef id: %d scenario: %s%n", componentRef.getId(), componentRef.getScenario());
      } else if (member instanceof GroupRefType) {
        GroupRefType groupRef = (GroupRefType)member;
        os.format("\tGroupRef id: %d scenario: %s%n", groupRef.getId(), groupRef.getScenario());
      }
    }
  }
```

### Your homework
Accessing fields, components, and repeating groups follows the same format, so you can write their accessors by following the style for messages. 

This simple tutorial reports on each element as it is accessed. However, a real application would likely need to collect information and cross-reference elements before processing them. For example, a FieldRef member of a message is only identified by its numeric ID (field tag). To get the field name, you have to cross-refernce the Field element with the same ID.

*Caution:* not all XML attributes are required, so some accessors may return `null`. Be careful to check for nullness before attempting to use a returned object reference.

## Step 3: Create a main and build the application
Create `main()` entry point to receive command-line arguments like so:

```java
public class RepositoryApp {
  public static void main(String[] args) {
    if (args.length < 1) {
      usage();
    } else {
      RepositoryReporter reporter = new RepositoryReporter();
      InputStream is;
      try {
        // first argument is an Orchestra file name
        is = new FileInputStream(args[0]);
        reporter.report(is, System.out);
      } catch (FileNotFoundException e) {
        System.err.println(e.getMessage());
      }
    }
  }

  public static void usage() {
    System.err.println("Usage: RepositoryApp <xml-file>");
  }
}
```
Tell your IDE to build the project, or issue this command line from the root directory of the project:
```
mvn install
```
 
Now your first Orchestra application is ready to run! Output should look something like this:
```
Message name: NewOrderSingle scenario: SecurityType-Future MsgType: D
	ComponentRef id: 1024 scenario: base
	FieldRef id: 11 scenario: base
	FieldRef id: 1 scenario: base
	FieldRef id: 21 scenario: SecurityType-Future
	ComponentRef id: 1003 scenario: SecurityType-Future
	FieldRef id: 54 scenario: SecurityType-Future
	FieldRef id: 60 scenario: base
	ComponentRef id: 1011 scenario: SecurityType-Future
	FieldRef id: 40 scenario: SecurityType-Future
	FieldRef id: 44 scenario: base
	FieldRef id: 59 scenario: SecurityType-Future
	ComponentRef id: 1025 scenario: base
  ```

#### Copyright 2019 FIX Protocol Ltd.