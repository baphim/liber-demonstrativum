# Unsafe Deserialization

## What It Is
A vulnerability class where reconstructing an object from a byte stream (deserializing it) can trigger code execution as a side effect, not just reconstruct data. Not specific to Java RMI, it shows up anywhere a program accepts serialized data from somewhere it doesn't fully trust.

## How It Works
Serialization turns a live object into bytes so it can cross a network or get written to disk. Deserialization reverses that. In Java specifically, rebuilding an object from bytes isn't passive data parsing, constructing an object can run code (static initializers, constructors) as part of just bringing it into existence.

The dangerous combination is two things at once: the receiver will deserialize bytes without being picky about what class they claim to represent, and it will go fetch and load that class from wherever the sender points it, if it doesn't already have the class locally. Neither piece alone is the bug. Together, "send data" becomes "name a class, have the target import it, and let importing it run your code."

The attacker isn't injecting code into existing logic, they're getting the target to willingly import and run code it was told to trust, because it trusted the data about where to find it.

## In My Setup
This is the mechanism underneath the Java RMI issue I found on my Metasploitable practice target (see [Java RMI](java-rmi.md)). It's worth watching for this same shape on other services later: anything that deserializes attacker-reachable input is worth asking "where does it get the class or type definition from, and does it check."

## Related Notes
- [Java RMI](java-rmi.md)
- [Metasploitable](../services/metasploitable.md)

## Tags
#concept #deserialization #vulnerability-class
