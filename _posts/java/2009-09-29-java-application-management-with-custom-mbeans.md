---
title: 'Java Application Management With Custom MBeans'
summary: 'I show how to develop your own functional MBeans for managing and monitoring your application.'
tags: java management mbeans
category: article
legacydate: 9/29/2009 22:19
legacyId: 307
layout: post
---

{% assign align='right' %}
{% assign src='misc/beans.png' %}
{% include image.html %}

Java 5 added the ability to work with MXBeans as part of the core Java platform, via the java.lang.management package, and specifically the MBeanServer API. In this article, I'm going to walkthrough of a quick example on how to surface components of your application into the platform MBeanServer so that you can maintain your application with any standard management utility, like [VisualVM](http://java.sun.com/javase/6/docs/technotes/guides/visualvm/).

Out of the box, the standard platform from Java 5 and on has supported a number of management facilities, allowing system administrators and developers to monitor the running VM, and currently active code. This suite of tools include:

* Detailed Memory Monitoring Facilities
* Garbage Collector Tools
* Thread and Lock/Deadlock Monitoring
* JIT Compilation Monitoring
* Runtime Information

Additionally, several Java libraries/frameworks contribute their own MBeans when on Java 5: [Hibernate](http://docs.jboss.org/jbossas/jboss4guide/r2/html/ch13.html), [Apache Tomcat](http://tomcat.apache.org/tomcat-6.0-doc/monitoring.html) (along with most other application servers, actually), [JRuby](http://jruby.org/), and others. MXBeans are a good option for surfacing administrative metrics and features as they are a Java standard, and are designed to allow for remote connections that can not only gather information, but can also make remote-procedure calls. From an implementation standpoint, MXBeans are simple to implement as long as you follow the (simple) rules.

The `java.lang.management` package makes it easy for you to contribute your own MBeans specific to your application. Most applications typically have some component or metric that would be valuable to monitor. Some possibilities include:


* Average requests for a given time frame to a particular portion of the app
* In-memory data-structure performance/metrics (pools, factories, etc)
* Volatile component switches (turning on and off experimental/dynamic features)
* Performance of remote connections (message queues, web services, etc)
* Configuration properties (viewing, as well as potentially manipulating)
* Tuning Parameters (throttles, pool sizes, etc)
* Active users

The options are only limited by the concepts and scope of your application. Of course, some apps may already have administration panels or something similar (assuming they are internet-based), but MXBeans are accessible by a variety of tools, allowing for more management options. In many cases, using MXBeans can make your application much more accessible to admins, particularly because of the remote connection support. Additionally, for client apps it can provide another avenue to interact with a client deploy that may be misbehaving.

For the purposes of this article, let's say we have a MessageQueue in our application that acts as grand-central-station for messages between different components. In a highly concurrent application, this component is going to need to be well tuned, which means it should probably be monitor-able and configurable in the running system. One possible idea for a message queue could look something like this:

{% highlight java %}
public class MessageQueue {
  public void postMessage(Message m) {...}
  public void listenForMessages(MessageListener ml) {...}
}
{% endhighlight %}

The general idea is an object where component A can post messages for which component B can listen. Assuming the application is designed to handle it, this message queue can handle the message dispatching in an asynchronous manner, allowing the application to utilize multiple threads more naturally. This form of dispatching is becoming more common in modern applications that are multi-threaded.

So we've decided we want this message queue class to be MXBean accessible. Some attributes and operations we've decided might be valuable for this message queue include:

* number of worker threads (viewable and modifiable)
* number of listeners
* number of queued messages
* average delay for message dispatch
* Status of the queue (suspended), and the ability to suspend/resume the queue

A well written component will probably have the ability to gather all (or most) of this information for the client already. The first step to surfacing this component in MBean form is to figure out the API of what we want to share. For MXBeans that means starting with a Java interface. The basic rules for this interface are as follows:

* Data types passed around are restricted to a few different things: primitives and the corresponding wrapper types, Strings, Enums, "composite" objects, and Lists/Maps containing any of the former.
* Composite objects are classes with only "getter" methods that return one of the aforementioned data-types, and they have to have a special static factory method.
* "Attributes" for a platform MXBean may be defined using standard getter/setter syntax (`int getThreshold()` and `void setThreadhold(int)` for example).
* Operations are methods with a void return type that take any number of the aforementioned data-types as arguments.

Extensive rules for the interface are defined in the [Platform MBean Server documentation](http://java.sun.com/j2se/1.5.0/docs/api/java/lang/management/ManagementFactory.html).

To be able to share our MessageQueue information on the MXBean server, we simply need an interface using these rules that defines the attributes and operations. To do this, I'm creating an interface called MessageQueueMBean. The interface name is important: it must be the class-name of the object provided to the mbean server (in our case, MessageQueue) followed by "MBean". Here is the interface with all of the above information surfaced:

{% highlight java %}
public interface MessageQueueMBean {
  int getNumberOfWorkerThreads();
  void setNumberOfWorkerThreads(int count);
  int getListenerCount();
  int getQueuedMessageCount();
  long getAverageDispatchDelay();
  void suspend();
  void resume();
  boolean isSuspended();
}
{% endhighlight %}

The interface becomes the 'contract' for the MXBean server, so it's important to declare everything here that you want surfaced - any method not on the interface, even if it's on the implementation, will not be available. So now, we need an object we can provide to the Platform MBeanServer that implements our interface. Many people simply add the MBean interface to the existing component they are monitoring, and in this case, this is a perfectly acceptable approach for what we want to do here. You could always code an adapter or specialized inner-class for the same goal.

Rather than implement a fully-working message queue for this example, I've created a "mock" queue that simply increments counters and flips flags to represent an actual working queue. This example provides our base example API and also implements the MBean interface.

{% highlight java %}
public class MessageQueue implements MessageQueueMBean {

  private int workerThreadCount = 5;
  private boolean suspended;
  private int messageCount;
  private int listenerCount;

  public void listenForMessage(MessageListener l) {
    listenerCount++;
  }

  public void postMessage(Message m) {
    if(suspended) {
      System.out.println("Message not added, system suspended.");
    }
    else {
      messageCount++;
    }
  }
  @Override
  public boolean isSuspended() {
    return suspended;
  }
  @Override
  public void resume() {
    suspended = false;
  }
  @Override
  public void suspend() {
    suspended = true;
  }

  @Override
  public long getAverageDispatchDelay() {
    // generate a pretend time between 0 and 1000 ms.
    long time = Math.round(1000 * Math.random());
    return time;
  }
  @Override
  public int getListenerCount() {
    return listenerCount;
  }
  @Override
  public int getNumberOfWorkerThreads() {
    return workerThreadCount;
  }
  @Override
  public int getQueuedMessageCount() {
    return messageCount;
  }

  @Override
  public void setNumberOfWorkerThreads(int count) {
    System.out.println("Worker thread count changed to: " + count);
    workerThreadCount = count;
  }
}
{% endhighlight %}

*(This example is simply meant to show how the MBean concept works, and is certainly not intended to show how to actually implement this message queue concept)*

So we now have an object that implements the contract expected by the `MBean` model. Now we simply need to register this with the Java MBeanServer. To do so, we need to come up with a categorization for our object.  Generally speaking, this is achieved by providing a domain (such as com.realjenius), followed by a number of key-value pairs. The domain provides a top-level grouping, and the key-value pairs provide additional nested groupings. For our example, we'll only use the 'name'  key as an additional category. Therefore, our object name will simply be 'com.realjenius:name=MessageQueue'. The full documentation on these naming rules [is available here](http://java.sun.com/j2se/1.5.0/docs/api/javax/management/ObjectName.html). From this point, registering the bean is straightforward:

{% highlight java %}
MessageQueue queue = // ... get from somewhere.
MBeanServer server = ManagementFactory.getPlatformMBeanServer();
server.registerMBean(queue, new ObjectName("com.realjenius:name=MessageQueue"));
{% endhighlight %}

There are a variety of checked exceptions that I am omitting here (already registered, not compliant mbean, etc); you will want to handle those as appropriate for your circumstances. Unregistering (something that should be done if the component is decommissioned in the running system) can be done via the inverse method:

{% highlight java %}
MBeanServer server = ManagementFactory.getPlatformMBeanServer();
server.unregisterMBean(new ObjectName("com.realjenius:name=MessageQueue"));
{% endhighlight %}

Once we have the object registered, we can begin using it through one our favorite tools like JConsole or VisualVM. Here is a quick example program that shows how this all can work together:

{% highlight java %}
public class ExampleProgram {

    public static void main(String[] args) throws Exception {
        MessageQueue queue = new MessageQueue();
        MBeanServer server = ManagementFactory.getPlatformMBeanServer();
        server.registerMBean(queue, new ObjectName("com.realjenius:name=MessageQueue"));
        queue.listenForMessage(new MessageListener("do_something"));
        while(true) {
            Thread.sleep(5000);
            queue.postMessage(new Message("do_something"));
        }
    }
}
{% endhighlight %}

This program simply creates a message queue object, registers it with the platform MBean server, and then exercises it in an ever-repeating loop. Real usage would obviously differ from this, but this should give you the idea of how MBeans work.

If I connect to my running VM instance via VisualVM (which ships with JDK 6, and has plug-ins which can be installed for MBeans), here is what I see on the MBeans tab:

{% assign src='/custom-mbeans/mbeans1.png' %}
{% include image.html %}

If we let the program run for a while, and revisit this we will see the numbers shifting as the queue is "filling up":

{% assign src='/custom-mbeans/mbeans2.png' %}
{% include image.html %}

Additionally, the number of worker threads is marked blue to let us know it is editable (it is a text field). This number can be changed, and as we change it, our program logs the change (due to the sysout in the mock queue we created).

{% assign src='/custom-mbeans/mbeans3.png' %}
{% include image.html %}

If we go to the operation tab we can see that we have our suspend and resume as options:

{% assign src='/custom-mbeans/mbeans4.png' %}
{% include image.html %}

Suspending the queue will adjust the 'isSuspended' result, and will also begin logging to the console that our queue is, in fact, suspended.

{% assign src='/custom-mbeans/mbeans5.png' %}
{% include image.html %}

### On DynamicMBean

I should also mention the class [DynamicMBean](http://java.sun.com/j2se/1.5.0/docs/api/javax/management/DynamicMBean.html), which allows you to take an existing object and register it at runtime without having to create an MBean interface. This is valuable if you want to take an existing object, at runtime, and register it as an MBean, or if you have a legacy component (or component for which you don't own the source code) that you want to surface. I wouldn't advocate it over the interface approach, otherwise, however.