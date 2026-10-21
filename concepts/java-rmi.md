# Java RMI

## What It Is
Java Remote Method Invocation lets a program call a method on an object living on a different JVM as if it were local. The registry (default port 1099) is a naming service: a client asks it where to find a bound remote object, then calls into it directly.

## How It Works
A normal method call runs in the same process you're in. RMI's trick is making a call to a different machine look the same to the programmer: `remoteObject.doSomething()` gets packaged up, sent over the network, run on the other end, and the result comes back.

Every RMI endpoint also exposes a Distributed Garbage Collector (DGC) object automatically, whether the application developer meant to expose it or not. Its job is mundane: `dirty` means "I'm still holding a reference to this, don't collect it," `clean` means "I'm done." Because it's bundled with RMI itself rather than something a developer chose to add, it shows up on basically every RMI service regardless of what the application actually does.

RMI method calls, DGC included, are not authenticated by default. Combined with a default configuration that will load classes from a remote URL to satisfy deserialization (see [Unsafe deserialization](unsafe-deserialization.md)), this becomes a trust boundary failure: a request that names a class the target doesn't have locally can cause the target to fetch and run code it was never meant to trust, without any credential ever being checked.

## In My Setup
I found this on my isolated Metasploitable practice target, port 1099, identified by nmap as `java-rmi`/`rmiregistry`. It corresponds to CVE-2011-3556, a known, long-patched misconfiguration in old Java RMI defaults. The core lesson for me was that this is a trust boundary failure, not a credential failure: no username or password is involved anywhere in the path.

## Related Notes
- [Unsafe deserialization](unsafe-deserialization.md)
- [Metasploitable](../services/metasploitable.md)
- [Kali](../services/kali.md)

## Tags
#concept #java #rmi
