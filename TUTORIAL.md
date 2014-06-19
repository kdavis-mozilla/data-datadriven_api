# The Data-driven API Tutorial
This document provides a brief introduction on how to develop a distributed
application with the Data-Driven API (DDAPI) for AllJoyn. No prior knowledge of
the standard AllJoyn API is required. A basic proficiency with the C++ language
is assumed.

## Concepts

This section introduces you to the main concepts behind the DDAPI for AllJoyn.
If you are eager to look at a practical example, you can skip this part
(although we would advise you to revisit it later) and go directly to the
[Example section](#example-a-simple-home-security-system).

A distributed application consists of a set of individual applications
(henceforth called *Participants*) that run in a networked environment, and that
communicate over a common medium. This common medium is, in our case, the
AllJoyn *Bus*. In this communication, a Participant can assume the role of
*Provider*, which means it provides information, or offers a service, on the Bus,
or the Participant can play the role of *Consumer*, which means it consumes
information, or makes use of an offered service. In many cases, a single
Participant will play both roles.

### Strongly typed communication

All Participant interactions in AllJoyn are strongly typed: every message that
is exchanged between Participants has a well-defined type, structure and
meaning. An AllJoyn *Interface* is the formal specification of a set of
interactions that belong to the same subject domain. A Provider will expose one
or more *Objects* on the Bus that implement one or more such Interfaces.
Consumers discover Objects based on the Interfaces they implement, and interact
with the Objects (and hence with the Object's Provider) through the methods
defined in the Interface. The set of Interfaces that is provided and/or
consumed by your distributed application is called the application's *Data
Model*.

An AllJoyn Interface consists of the following parts:

* a name. Interface names follow a Java-like namespacing pattern, and should
  start with a reversed domain name. For example,
  `org.allseenalliance.sample.Foo` refers to an interface `Foo` in namespace
  `org.allseenalliance.sample`.
* a set of *Properties*. Properties are like member fields in an
  object-oriented programming language: they represent the observable state of
  the object implementing the interface. A Property has a name and a type.
  Types can be primitive (signed and unsigned integers, strings, boolean
  values, ...) or complex (arrays, structures, dictionaries).  
  **Note**: unlike standard AllJoyn, Properties in the DDAPI are read-only. If
  an individual Property must be directly changeable by a Consumer, we
  recommend you create a dedicated method call (see below) for that purpose.
* a set of *Signals*. Signals are emitted by Objects to notify interested
  Consumers of transient events that pertain to that Object. Take, for example,
  an Interface that represents a door. Whether the door is open or closed, is a
  part of the externally observable state of the door, and will hence be
  represented as a Property. On the other hand, the fact that someone passes
  through an open door is a transient event. This is best represented by a
  Signal. Signals have a name, and zero or more arguments. Each signal argument
  has a name and a type.
* a set of *Methods*. Methods are the only way in which a Consumer can directly
  interact with a given Object. Methods have a name, and a set of input and
  output arguments, which each have a name and a type.  
  **Note**: Methods may seem like a really convenient and obvious way to model
  all the Consumer-Provider interactions in your distributed application.
  However, the closely coupled interaction model that results from such a
  design does not hold up well in a truly distributed environment. A detailed
  argumentation for this point is beyond the scope of this document, but we
  would like to stress that the DDAPI is designed in such a way that it is easy
  for you to adopt a data-centric publish-subscribe interaction model that is
  much more robust and reusable for Internet of Things-like distributed
  use cases.

AllJoyn Interfaces are defined in a programming language independent XML
format. The AllJoyn Code Generator tool is used to process these XML
descriptions and create a convenient representation of the interface for the
programming language and platform of your choice. (For the first release of the
DDAPI, that means C++ on Linux. We're working hard on extending the list of
supported languages and platforms...)

### Objects and Interfaces

AllJoyn Interfaces serve as the formal specification of the kind of information
or service offered by a Provider. In order to offer said Interface to
Consumers, a Provider must create one or more Objects that each implement one
or more Interfaces. There are no limitations on the combination of Interfaces
that can be implemented by an Object.

For example, a single Provider may
simultaneously offer an Object that implements only Interface A, two Objects
that implement Interfaces A and B, and an Object that implements interface B
and C on the Bus.

Objects need a unique identifier to distinguish them from one another. That
unique identifier (called *ObjectId*) actually consists of two parts:

* a unique identifier for the Participant (Provider) that hosts the Object. In
  standard AllJoyn terms, this is the Provider's *unique bus name*.
* a unique identifier for the Object within the Provider. In standard AllJoyn
  terms, this is called the *object path*.

At the Provider side, you are free to choose the object path for your Objects,
but unless you have a very good reason for doing so, we advise you to let the
DDAPI generate a unique object path for you.

At the Consumer side, you can retrieve the two components from the ObjectId,
but we advise you to treat an ObjectId as a single, opaque value.

### Publish-Subscribe communication

The DDAPI is based on the publish-subscribe paradigm. This paradigm enables a
complete decoupling of information providers and consumers. This decoupling is
realized by the introduction of the *topic* concept. A topic is a grouping of
related information. Information providers publish their information on the
topic whenever they have it available, and information consumers subscribe to
the topics they are interested in. The publish-subscribe communication
framework then takes care of the delivery of relevant information to
subscribers. Consequently, providers and consumers are decoupled

* in space: there is no direct communication between providers and consumers.
  The topic acts as an intermediary. Hence, consumers must not care about the
  actual physical location of an information provider: it can be on the same
  system (even in the same process), or on a different system somewhere else in
  the network.
* in time: information providers publish information whenever they have it
  available, consumers consume the information when they have time. The
  communication framework makes sure that providers are not blocked until
  consumers are ready to receive the information, or that consumers are blocked
  waiting for providers to answer to a request for data.

In the DDAPI, we have applied the publish-subscribe principles in the following way:

* AllJoyn Interfaces serve as topics
* Providers publish information in one of two ways:
  - by updating observable Properties.
  - by emitting Signals.
* Consumers subscribe to an Interface by creating an *Observer* for that
  Interface. Via the Observer, they receive notifications whenever new Objects
  are discovered or removed from the Bus, Object Properties are updated, or
  Signals are emitted. At any point in time, the Consumer can retrieve the
  latest state (Properties) of all discovered Objects from the Observer, and
  interact with the Objects by means of Method calls.

The DDAPI still allows you to define service-oriented, RPC-heavy Interfaces and
interaction patterns. In fact, it is easier to implement such a system with the
DDAPI than with standard AllJoyn. But we urge you to at least consider a more
data-oriented, publish-subscribe based approach. You'll soon learn to
appreciate the elegance and benefits offered by this approach.

### The Provider role

As a Provider, it makes sense to reason in terms of Objects. Your role is
basically to manage a set of physical (say, an HVAC system) or virtual
(e.g. a media stream) resources, and provide a representation of these
resources on the AllJoyn Bus. It is a natural approach to represent each
resource as a single Object, and represent the different aspects of the
resource by different interfaces. For example, the HVAC system's temperature
sensor might be modeled as a TemperatureSensor interface, whereas the heating
and cooling elements would have a Heating and Cooling interface respectively.
That way, you can reuse the same interfaces (and a lot of the Provider logic)
if you manage a system that can only heat, or only cool, instead of doing both.

In short, these are the steps you have to take as a Provider:

1. Create a `BusConnection`, the DDAPI object that encapsulates your connection
   to the AllJoyn Bus.

2. Define a custom class for each kind of Object (each combination of
   Interfaces) you want to provide. You'll need to inherit from `ProvidedObject`
   (defined in the DDAPI library) and the various `FooInterface` classes (the
   Code Generator creates an `xyzInterface` class for each Interface `xyz`). In this
   class, you provide implementations for the Methods defined in the Interfaces,
   and any other business logic you may need.

3. Create an object of your class, and expose it on the bus by calling
   `object.UpdateAll()`.

4. Whenever the observable Properties of your object change, call
   `object::FooInterface.Update()`.

5. To emit a signal (say, signal `Bar` with 2 arguments), call `object.Bar(arg1,
   arg2)`.

6. When the object is no longer relevant, remove it from the Bus by calling
   `object.RemoveFromBus()`.

### The Consumer role

As a Consumer, you want to reason in terms of what you are interested in, and
not necessarily in terms of how the various Providers package that information
and services together in Objects. Therefore, the Consumer-side DDAPI is built
around Interfaces and not Objects. Consumers create Observers that discover all
Objects that implement a particular Interface, and interact with those Objects
through proxies that only expose that one Interface.

Obviously, sometimes you will want to correlate different aspects of the same
Object on the Consumer side (you'll want to know which Heating and Cooling
interfaces belong to the same HVAC unit). The DDAPI provides the means to
establish this link via the ObjectId. Proxies for the same object, but
discovered through different Observers (one for Heating and one for Cooling),
are guaranteed to have the same ObjectId.

In short, these are the steps you have to take as a Consumer:

1. Create a `BusConnection`.

2. For each Interface Foo you're interested in, create an `Observer<FooProxy>`.
   The Code Generator will create the `FooProxy` class for you.

3. If you're interested in receiving notifications for the discovered Object's
   lifecycle events (Objects appear, get property updates, disappear), you have
   to create a concrete subclass of `Observer<FooProxy>::Listener`, and pass an
   object of that subclass as an argument of the Observer constructor.

4. Attach signal listeners to the Observer. A `SignalListener<FooProxy,
   FooProxy::Bar>` will notify the Consumer whenever an Object that implements
   Interface Foo emits signal Bar.

5. At any time, you can iterate over (proxies for) the discovered Objects, or
   retrieve a proxy for a particular Object via its ObjectId.

6. To call a Method `Baz` on a remote Object, simply call `proxy->Baz(arg1,
   arg2)`. This returns an object of type
   `MethodInvocation<FooProxy::BazReply>`. Via this object, you can track the
   progress of the RPC call, and retrieve its final result or error code.

## Example: a simple home security system

As a tutorial, we'll build a simplistic home security system that monitors all
doors in your house, tells you whether they're open or closed, and who passes
through the door.

### Data model definition

First off, we need to think how we are going to represent a door in our system.
This is a critical step in the design of your application.  Design your
interfaces to encapsulate a single trait and nothing more.  This will allow for
future reuse of your interface. If the 'Door' interface contains nothing but
functionality and information related to opening and closing a door, it can be
reused later, say for a garage door. Therefore, additional aspects (e.g.,
spyhole functionality) are better modeled in a separate interface.

First, we need to think about the externally observable state of our door. In
this case, we constrain this to two simple properties:

* the location of the door (we want to be able to distinguish between front and
  back door).
* is the door open or closed?

Then, we must consider the transient events that relate to a door. In our
example, we have only one: the fact that a person passes through the door. We
model this as a signal.

Lastly, we want to enable remote opening and closing of doors. We'll use
methods to offer this functionality.

The resulting data model looks like this:

~~~ {.xml}
<node>
  <interface name="org.allseenalliance.sample.Door">
    <property name="open" type="b" access="read" />
    <property name="location" type="s" access="read" />

    <method name="Open">
      <arg name="success" type="b" direction="out"/>
    </method>
    <method name="Close">
      <arg name="success" type="b" direction="out"/>
    </method>
    <signal name="PersonPassedThrough">
      <arg name="who" type="s"/>
    </signal>
  </interface>
</node>
~~~

Argument and property types are specified as per the [Dbus
specification](http://dbus.freedesktop.org/doc/dbus-specification.html).

### Generate Helper Code

If we run our data model definition through the Code Generator, we get six
source code files:

~~~
$ ajcodegen.py -t ddcpp door.xml
$ ls -go
 -rw-rw-r-- 1   3725 May 21 10:17 DoorInterface.cc
 -rw-rw-r-- 1   3785 May 21 10:17 DoorInterface.h
 -rw-rw-r-- 1   4405 May 21 10:17 DoorProxy.cc
 -rw-rw-r-- 1   4644 May 21 10:17 DoorProxy.h
 -rw-rw-r-- 1   1891 May 21 10:17 DoorTypeDescription.cc
 -rw-rw-r-- 1   2014 May 21 10:17 DoorTypeDescription.h
~~~

The first two files are to be used by the provider, the next two files are
to be used by consumer. The last two files are used by both.

### Implement the Provider

#### Define business logic for the door

The first thing we need to do, is  define a class for the Objects that will
represent our physical doors on the AllJoyn Bus. This class must derive from
`datadriven::ProvidedObject` and from any generated interface class we want the
Object to implement. In this case,that means only
`gen::org_allseenalliance_sample::Door`.

~~~ {.cpp}
#include <datadriven/datadriven.h>
#include "DoorInterface.h"

using namespace std;
using namespace::gen::org_allseenalliance_sample;
class Door : public datadriven::ProvidedObject, public DoorInterface {
  public:
    Door(datadriven::BusConnection& busconn, qcc::String _location, bool _open = false) :
        datadriven::ProvidedObject(busconn),
        DoorInterface(this), open(_open), location(_location)
    {
        /* note that the "open" and "location" properties from the Door interface
         * are now (protected) fields of the DoorInterface abstract class. It's up
         * to the application logic defined in this class to properly initialize 
         * them and keep them up to date.
         */
    }
    ~Door() {}
};
~~~

**Note**: the order of inheritance and initialization is important! You should
always derive *first* from `ProvidedObject` and *then* from the generated
interface classes.

In our `Door` class, we must provide concrete implementations of the methods
defined in the Door interface.

~~~ {.cpp}
void Door::Open(OpenReply& reply)
{
    if (this->open) {
        /* door already open, so return success = false */
        reply.Send(false); /* reply.Send takes the method outargs as arguments */
    } else {
        /* OK, we can open it. */
        this->open = true; /* update the observable properties */
        this->DoorInterface::Update(); /* let the world know we are in a new consistent state */
        reply.Send(true); /* send method outargs back to caller */
    }
}

void Door::Close(CloseReply& reply)
{
    /* exercise left to the reader */
}
~~~

Note that every time the observable state of the door changes,
DoorInterface::Update() is called. This will inform the DDAPI that the state of
the door has changed and this will lead to an
`Observer<DoorProxy>::Listener::OnUpdate()` notification for every subscribed
Consumer.

#### Create an actual Door and publish it on the Bus

Now we create an actual Door object, and make it available on the AllJoyn Bus.
First, we need to set up a connection to the AllJoyn Bus via the
`BusConnection` object. You only have to do this once, even if you provide
multiple objects or if you provide and consume at the same time.

~~~ {.cpp}
int main()
{
    datadriven::BusConnection busConnection;
    if (ER_OK != busConnection.GetStatus()) {
        cerr << "Bus Connection not correctly initialized !!!" << endl;
        return EXIT_FAILURE;
    }

    publish_doors(busConnection);

    return EXIT_SUCCESS;
}
~~~

As the DDAPI does not use exceptions, you have to check the status after
construction manually.

Next, we create some Door objects and advertise them on the AllJoyn Bus.

~~~ {.cpp}
void publish_doors(datadriven::BusConnection& busConnection)
{
    //The front door
    Door frontDoor(busConnection, "front", false);
    assert(ER_OK == frontDoor.GetStatus());
    assert(ER_OK == frontDoor.UpdateAll()); //Advertise the door

    //The back door
    Door backDoor(busConnection, "back", false);
    assert(ER_OK == backDoor.GetStatus());
    assert(ER_OK == backDoor.UpdateAll()); //Advertise the door

    /* ... implement application logic here.
     * Note that we created the doors in this example as local variables.
     * At the end of this function, the Door objects go out of scope, at
     * which point they will remove themselves from the bus cleanly.
     */
}
~~~

This concludes an elementary provider.

### Implement the Consumer

#### Connect to the Bus and discover all Objects that implement the Door interface

As a Consumer, the first thing to do is to create a `BusConnection` and a `Listener` to
get Object life cycle notifications. The second thing is subscribing to all the
Interfaces you're interested in. The latter is done by creating an `Observer` of the
appropriate type.

~~~ {.cpp}
#include <datadriven/datadriven.h>
#include "DoorProxy.h"

using namespace::gen::org_allseenalliance_sample;

int main()
{
    datadriven::BusConnection busConnection;
    assert(ER_OK == busConnection.GetStatus());

    MyDoorListener dl = MyDoorListener();
    datadriven::Observer<DoorProxy> observer(busConnection, &dl);
    assert(ER_OK == observer.GetStatus());

    /* ... */

    return EXIT_SUCCESS;
}
~~~

#### Interacting with an Object through its proxy

So, what is this `DoorProxy`? At the consumer side, you cannot directly
work with Door objects (remember, they are at the provider side). That's why there
is a DoorProxy that will take care of the interactions with the actual Door object
on your behalf.

The DDAPI never passes around naked proxy objects, to avoid issues with
dangling references when the corresponding Object gets removed from the Bus.
Instead, all proxy object instances are wrapped in a `shared_ptr`.
The `shared_ptr` construction allows the DDAPI to ensure that there is only one
copy of each proxy object, and that it remains firmly in control of the life
cycle of that proxy object.

##### Access Properties

When you have a `std::shared_ptr<DoorProxy>`, you can always access the last known Properties for that door Object by calling `GetProperties()`.

~~~ {.cpp}
    const DoorProxy::Properties prop = door->GetProperties();
    qcc::String location = prop.location;
    bool open = prop.open;
~~~

##### Invoke Methods

To invoke a Method on a remote Object, simply invoke that method on its local
proxy. All method invocations in the DDAPI are asynchronous: they don't block
to wait for a reply, but rather return a `shared_ptr<MethodInvocation>` object that you can
use to track progress of the method call, and to retrieve the final result or
error message for the method call. If you *do* want to block until you receive a
reply, call the `GetReply()` method on the `MethodInvocation` object. 

~~~ {.cpp}
void open_door(shared_ptr<DoorProxy> door)
{
    std::shared_ptr<datadriven::MethodInvocation<DoorProxy::OpenReply> > invocation = door->Open();

    /* at any point in time, we can check whether the method invocation is finished */
    if (READY == invocation->GetState()) {
        cout << "We won't have to block..." << endl;
    } else {
        cout << "The following call will block..." << endl;
    }

    /* let's get the reply, even if we have to block to get it */
    DoorProxy::OpenReply reply = invocation->GetReply();

    /* we still need to check whether the method itself concluded successfully,
     * because we may get an error state due to a transport error or timeout
     * along the way.
     */
    if (ER_OK == reply.GetStatus()) {
        cout << "Opening of door " << (reply.success ? "succeeded" : "failed") << endl;
    } else {
        cout << "Invocation error." << endl;
    }
}
~~~

##### Check Object liveliness

One obvious reason why a method invocation may fail is that the Provider has
removed the Object from the Bus in the mean time. Therefore, you may want to
manually check the liveliness of an Object for which you hold a proxy. This is
possible with the `IsAlive()` method.

~~~ {.cpp}
    if (door->IsAlive) {
        // yes, it's alive as far as our Observer knows
    } 
~~~

Note that the status returned by `IsAlive` is only a hint, not a certainty.
We're dealing with a distributed system here, so the Observer can only report
the *last known status*. There is always a small chance that the Provider
removes the Object from the Bus in the exact same instant that you do the
liveliness check. 

#### Iterate over all discovered Objects

You can iterate over an `Observer` to get proxies for all discovered Objects.

~~~ {.cpp}
void list_doors(datadriven::Observer<DoorProxy> &observer)
{
    datadriven::Observer<DoorProxy>::iterator it = observer.begin();

    for (; it != observer.end(); ++it) {
        // *it is of type shared_ptr<DoorProxy> 
        // it-> dereferences twice, so the following line calls GetProperties on DoorProxy, not on the iterator.
        DoorProxy::Properties prop = it->GetProperties();
        cout << "Door location: " << prop.location.c_str() << " open: " << prop.open << endl;
    }
}
~~~

#### Get a proxy for a specific Object

If you know an Object's ObjectId, you can get a reference to the Object's proxy
by calling `Observer::Get`.

~~~ {.cpp}
void show_door(ObjectId& object_id)
{
    shared_ptr<DoorProxy> door = observer.Get(object_id);
    if (door) { // check for pointer validity
        DoorProxy::Properties prop = door->GetProperties();
        cout << "Door location: " << prop.location.c_str() << " open: " << prop.open << endl;
    } else {
        cerr << "ID " << object_id << " does not identify a live object on the bus." << endl;
    }
}
~~~

#### Get Object life cycle notifications

In many cases, you'll want to know when doors appear, disappear, open or close.
The `Observer` class accepts a listener via which you can get active
notifications whenever a door's properties change, when doors are discovered
(which is treated the same as a property change), and when doors are removed
from the Bus.

To create a listener, derive from `Observer<DoorProxy>::Listener` and implement
the appropriate callback methods:

* `OnUpdate()` is called when a new Door is advertised on the Bus or when a Door
  Object's Properties have been updated (e.g., the Door was open and has just
  been closed). When you add a listener to an Observer, the OnUpdate method
  will be called immediately for each Object that Observer knows about.

* `OnRemove()` is called when a Door is removed from the Bus. This does not
  necessarily imply the Door object itself was destroyed at the Provider side:
  the Provider may have temporarily removed the Object from the Bus, with the
  intention of bringing it back later. In this callback you can still see the last
  state of the Properties of the object but you cannot make any method calls
  anymore.

~~~ {.cpp}
class MyDoorListener : public datadriven::Observer<DoorProxy>::Listener {
  public:
    void OnUpdate(const std::shared_ptr<DoorProxy>& door)
    {
        /* Every DoorProxy has an ObjectId which can be regarded as the
         * unique identifier of the Door on the Bus. If you know the ObjectId,
         * you can always retrieve a proxy for an Object with the Observer::Get
         * method.
         */
        const datadriven::ObjectId& id = door->GetObjectId();
        const DoorProxy::Properties prop = door->GetProperties();
        cout << "Update for door " << id << ": location = "
             << prop.location.c_str() << " open = " << prop.open << "." << endl;
    }

    void OnRemove(const std::shared_ptr<DoorProxy>& door)
    {
        const datadriven::ObjectId& id = door->GetObjectId();
        const DoorProxy::Properties prop = door->GetProperties();
        cout << "Door " << id << "at location " << prop.location.c_str()
             << " has disappeared." << endl;
    }
};

int main()
{
    /* ... */

    MyDoorListener dl = MyDoorListener();
    datadriven::Observer<DoorProxy> observer(busConnection, &dl);
    assert(ER_OK == observer.GetStatus());

    /* ... application logic */
    
}
~~~

#### Get Signal notifications

Finally, we also want to be notified when someone passes through a door. You
can add listeners to an `Observer` to get active notifications when one of the
discovered Objects emits a specific Signal. In the notification callback, you
can get the signal arguments, and a proxy for the signal emitter.

To create a listener for the PersonPassedThrough signal, derive from
`SignalListener<DoorProxy, DoorProxy::PersonPassedThrough>`, and implement the
`OnSignal` callback method.

~~~ {.cpp}
class PPTListener : public datadriven::SignalListener<DoorProxy, DoorProxy::PersonPassedThrough> {
  public:
    void onSignal(const DoorProxy::PersonPassedThrough& signal)
    {
        const qcc::String who = signal.who; // signal argument
        const std::shared_ptr<DoorProxy> door = signal.GetEmitter();
        const DoorProxy::Properties prop = door->GetProperties();

        cout << who.c_str() << " passed through a door at location " << prop.location.c_str() << endl;
    }
};

int main()
{
    /* ... set up Observer */

    PPTListener l = PPTListener();
    assert(ER_OK == observer.AddSignalListener<DoorProxy::PersonPassedThrough>(l));

    /* ... application logic */

    assert(ER_OK == observer.RemoveSignalListener<DoorProxy::PersonPassedThrough>(l));
}
~~~

### Threading model

The DDAPI operates on a callback-driven model. For any non-trivial application,
it is important to know in which threads, and with which levels of concurrency,
you can expect the various callbacks to be invoked.

The DDAPI strives to make things simple for the developer. For this very
reason, we have a straightforward threading model with strong guarantees. The
DDAPI library reserves a single thread for all Consumer-related callbacks
(listener callbacks), and another thread for all Provider-related callbacks
(method invocation callbacks). Hence, there will never be two Consumer-related
callbacks or two Provider-related callbacks in flight at the same time. It is
possible to have one Provider-related and one Consumer-related callback in
flight concurrently.

The implication of this threading model is that you need to be somewhat careful
when performing blocking calls (e.g. `MethodInvocation::GetReply`) in a
callback. If there is a possibility of cyclic Method invocation (e.g.
Participant A calls a method on Participant B, which in turn calls a method on
A, which in turn calls a method on B), blocking the single Provider callback
thread in either A or B could result in a distributed deadlock.

### Build the application

The README.md file that is part of the source code distribution of the DDAPI
contains instructions on how to build applications against the DDAPI.
