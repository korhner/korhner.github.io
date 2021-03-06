---
layout: post
title: "A memory leak caused by dynamic creation of log4j loggers"
date: 2015-03-17 21:54:46
categories:
- Java
- Debugging
published: true
---

At the company I work for, we had a situation where a highly loaded server that was handling several thousands requests per second consumed memory increasingly, and after about 30 days, it would become unusable and required a restart.
By looking at our monitoring tools, we concluded it was clearly a memory leak, and we figured it must be an easy one to detect as memory exhibited an almost perfect linear grow.
First thing we did was we took a heap dump and looked at most frequent instances and to our shock over 30 GB of memory was occupied by log4j loggers!

<!--more--> 

## Hunting the memory leak

We started to try to isolate the problem and found a possible red flag - for every client that connected, we generated a new logger containing class name and IP address of the client.
However, we didn't keep a reference to it and as soon as the request finishes nothing from our code referenced the logger and we thought it would be garbage collected.
Is it possible that log4j loggers are kept in memory forever and never removed?
It was very easy to set up an experiment to test the hypothesis. I created a minimal piece of code that created a lot of dynamically named loggers and attached a profiler to it.
For this simple case, Java VisualVM, which comes with Java is more that enough.

{% highlight Java %}
for( int i = 0; i < 100000; i++) {
    Logger.getLogger("Logger - " + i);
}
{% endhighlight %}

jvisualvm can be found in bin folder of your jdk. I run the test code from IDE and made it stop after creating the loggers.
After that, I opened jvisualvm, found my process at Application list and took a heap dump (can be done by right clicking on process and selecting Heap Dump).
After the dump is generated, I opened 'Classes' tab. Here is how it looks like:

![_config.yml]({{ site.baseurl }}/assets/img/log4j_leak_cls.png)

We can see that there is more than 100,000 instances of log4j classes, each holding strings that can really add up to size and eat heap memory.
As we also have the same number of java.utilHashtable$Entry instances, we can anticipate log4j is probably caching every logger we create. 
Lets dig further and right click on org.apache.log4j.Logger and click 'Show in Instances View'.
This shows us all the instances of this type. Select an instance that we know we created (you can check that name property is something like Logger - 454), right click on 'this' reference in References view and click on 'Show Nearest GC Root'.
GC roots are objects special for garbage collector. Garbage collector collects those objects that are not GC roots and are not accessible by references from GC roots. You should something like this:

![_config.yml]({{ site.baseurl }}/assets/img/log4j_leak_root.png)


Going from bottom to top, we can see that our logger is referenced by a Hashtable$Entry which is referenced by a Hashable which is referenced by a Hierarchy. Hm.
Since there is only one instance of Hierarchy this has to be the root of the problem. Let's dig into the log4j source to see what creating loggers does:

{% highlight Java %}
public Logger getLogger(String name, LoggerFactory factory) {
  synchronized(ht) {
    Object o = ht.get(key);
      if(o == null) {
        logger = factory.makeNewLoggerInstance(name);
        logger.setHierarchy(this);
        ht.put(key, logger);
        updateParents(logger);
        return logger;
      } else if(o instanceof Logger) {
        return (Logger) o;
      }
  }
{% endhighlight %}

I removed a lot of code to emphasize the critical part - if we don't have a logger with this name a new instance is created and put in ht Hashtable!  
The solution was very simple, we just replaced creating a new logger with a static one.
This doesn't mean you should be conservative and use one logger per application - it is still a good idea to create loggers named by logical parts of your application of even named by class names as soon as you don't create new loggers uncontrollably.
Share your favourite tools and methods for hunting memory leaks in comments and subscribe for more interesting debugging adventures!