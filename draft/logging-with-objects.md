# Logging With Objects

*Sven Van Caekenberghe* June 9, 2014

Logging, generating some form of output during the execution of a computer program, for monitoring or for later study, is useful everywhere and is thus a prime candidate for a framework. It seems however that it is hard to reach a consensus in this area, each developer having their own reasons for not liking some existing framework and/or preferring their own solution. Maybe stripping logging down to its bare essentials is a solution.

Most existing logging frameworks are based on strings being the central thing being produced as log output by a running program. They differ in what extra attributes they add, like tags, severity levels, and so on. They also enforce a particular way the logging is actually done, thus creating an intrusive dependency.

This document describes the design of the 'Logging With Objects' micro framework.

## Logging Objects

What if we allowed and even encouraged **any object** to be a thing that can be logged ? This would allow each developer to chose the exact information they want to log. The framework could provide some useful super classes to inherit from, but should not require them.

Although any object can be used as log object, there are some semantics that should be respected:

- log objects should be **static or constant**, so that they or their sub objects do not change after they have been send off to the log
- log objects with their sub objects should consist of **a single, separate graph** that can serialised, the single graph definition depending on the serialisation framework being used
- log objects should have **a main string representation** suitable for classic textual logging
- there should be some **communality among all log object types** to make later processing or analysis possible, inheritance, traits or shared protocols can be used for this
- it might be useful to be able to sort log objects in the order in which they were generated
- it might be useful to be able to uniquely identify log objects, even globally
- it might be useful to support equality and hashing

Conceptually, it is best to think of log objects as representing *something that has happened or that is about to happen*. Hence, the term log event can be used as well. Here are some examples:

- UserLoggedIn
- UserLoggedOut
- WebPageRendered
- RestCallCompleted
- PrintJobScheduled
- PrintJobCompleted
- ServerErrorOccurred 
- AboutToSaveToFile
- DidSaveToFile

Obviously, some namespace prefixes have to be added. Maybe an Event, LogEvent or Signal suffix is useful as well.

Note how these names automatically suggest useful attributes to add: username, time to execute, web page or REST call URI, filename.

Here are some attributes that are quite common, though none of them should be required:

- timestamp
- ID, UUID, GUID
- process/thread ID
- username
- session ID
- package/application tag
- server ID
- severity level or classification

The value types of these attributes can be important as well, but a more loose definition gives more flexibility.

Note how in most cases, in a well structured object oriented program, *you will already have objects that can become useful parts of log objects*:

- request/response
- call/command
- action
- announcement
- event

Performance is an issue that can be solved by reusing objects that are already in the flow of computation. Remember to make sure that they remain constant. If critical, a local flag could prevent log object creation altogether.

The key idea is to *include clean, structured information* that can then be useful when looking at the log events. This structured information is precisely what is almost completely lost, or very hard to recover, when logging just strings.

The framework should provide in a couple of base classes for quickly building your own log event objects. But it would be even better to define a couple of traits or protocols describing useful attribute sets for log events.

## Logging

The single thing that a running program should do is **create log objects and send them off to some central destination** that collects them in the order they arrive.

The framework should not directly define how log objects are send off, nor how to find this central destination. The scope of being a central destination could be per application, image, server, or even across servers. Smaller scopes could be aggregated into large ones.

The only requirement is that log objects are created and added to a log, in order. The addition can be triggered by a #log, #emit or #signal message. It is up to the application developer to add convenience methods to make this easy. Automatic fill in of as much attributes as possible, based on context, will make the actual logging code lighter.

## Processing Logs

What happens with logs should be totally separate from the way the logs are generated. Obviously, the details of the processing are linked to the attributes of the log objects.

Since we insist on logging objects, the main goal is to keep them somehow in their natural state. An obvious approach is to keep the last N log objects in memory. Another approach is to serialise them as such to external persistent storage. FUEL, STON or JSON, but also any database, come to mind.

**Processing logs thus becomes the same as working with any collection of more or less uniform objects**. The standard collection API and tools can be used.

Any framework functionality offered here will add requirements to log objects, in terms of the attributes that they should provide. Traits are ideal to define these requirements.

## Announcements

The announcement framework (another micro framework) can be used to distribute log events from the producers to the actual consumers. Announcements are a mechanism, not a requirement as such.

## Notifications

Another possible mechanism for log event producing code to send off objects is to use notifications, a subclass of exception. To  collect them, a standard exception handler could then be used. Notifications are a mechanism, not a requirement as such.

## Conclusion

Adding logging to your application or framework according to the idea of 'Logging With Objects' should help in removing the lock in anxiety of such a wide cross cutting concern. You can define your own log objects (maybe conforming to a couple of minimal protocols) and select your own mechanism to send the off to the log. Just connect your log object stream to the framework's post processing tools and your are done. 

To make it easier and/or to serve as examples, you can reuse some log object base classes or some of the transfer mechanisms offered.