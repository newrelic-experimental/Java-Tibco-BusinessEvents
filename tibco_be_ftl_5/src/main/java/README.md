#   Instrumentation Classes

Reference the [examples](../../examples/java/targetclass/packagename/) and the [classes being instrumented](../../examples-to-instrument/java/targetclass/packagename/) for help with specific instrumentation methods. By convention of the gradle build script these examples will not be compiled into your instrumentation jar.

This is where you should add your instrumentation classes.  Use the following steps as a guide.

*Keep in mind, this tool gives you the ability to alter runtime behavior.  Be careful not to break functionality or degrade performance of what you're instrumenting.*

1. Find a class or interface you want to instrument. See [below](#finding-good-instrumentation-points) for tips on what to instrument.
2. Create a folder structure to match the classpath.
3. Create a new empty (optionally abstract) class with name matching step 1.
4. Add the `@Weave` annotation to the class. If the target is an interface or super class, be sure to specify the weave type.
5. Add a method for each method in the orignal class to instrument. Not all methods must be defined. Methods can be declared abstract.
	Method name, arguments, and return type should match the original method exactly.
	(This is where it is handy to have the library included as a dependecy since it allows you to reference those classes.)
	Note: method visibility and thrown exceptions don't need to match as they're not considered part of a method signature.
6. Each non-abstract declared method must call `Weaver.callOriginal()` which will provide access to the original result.
	This call can be wrapped in a try/catch for specialized exception handling.
	The original result should be returned if the method is non-void (though something different can be returned if there's a good reason).
7. Add other method calls or `@Trace` annotations accordingly. Remember, if a transaction isn't already started by some other instrumentation, you will need to replace the first `@Trace` with @Trace(dispatcher=true). Please reference our [API documentation](https://docs.newrelic.com/docs/java/java-agent-api) for details on the API, with specific emphasis on how [converting a transaction to a web request](https://docs.newrelic.com/docs/java/java-instrumentation-by-annotation#web-request).
8. If the instrumentation doesn't behave as expected, see [below](#debuggingvalidating-instrumentation) for debugging tips.

##  Naming Metrics
When creating instrumentation, especially when creating metrics or naming transactions, it is important to understand the impact metric names can have. Please review our documentation to understand ["Metric Grouping Issues"](http://docs.newrelic.com/docs/features/metric-grouping-issues). Basically, metric names and transaction names should be mostly static. They should be common across app instances and never contain unique generated text. URLs are dangerous to name on since they often change regularly and (especially REST style URLs) contain user generated data or IDs.

##  Finding Good Instrumentation Points
When choosing an instrumentation point, it helps to have a clear understanding of how the library or framework works. Often stepping through the code in a debugger is a great way to understand its strucutre and behavior.  Running a thread profile in New Relic on a running application can also provide insight on key classes.

Instrumenting stable classes or interfaces is better than instrumenting something that changes often. Interfaces tend to be more stable than classes. If an instrumented class's strucutre changes, the instrumentation must be updated to support that class.

By choosing a parent class to instrument, you often also reduce the possibility of execution bypassing your instrumentation.  As child classes or implementations of an interface are added, your instrumentation will automatically match resulting in a more durable module.

Be very careful not to instrument common methods or interfaces such as `Object.toString()`, `Object.equals()`, `Callable.call()`, `Runnable.run()`. These methods are very common and instrumenting will cause performance degredation with little benefit.

In cases where there is a significant branch point, such as routing to controllers, it may be tempting to instrument all the controllers.  Often a better choice is to instrument the one caller, rather than many callees.

Some frameworks make significant use of async patterns. The java agent doesn't have a way of handling async in modules yet. Try to find instrumentation points that execute on a single thread.

##  Debugging/Validating Instrumentation
When writing instrumentation, it is best to start with a functional application with simulated load that can be restarted very quickly. As you instrument make small incremental changes, validating behavior at each step.

Increasing the agent's log level can help in debugging or validating. Log messages generated by a module using the `AgentBridge.logger` will have the module name (org.mycorp.instrumentation.sample-0.1) included in the log message. When using lower log levels, filter the log on your module's name. To verify metrics and other data sent to the collector, enable the agent's `audit_mode`. Audit output will be sent to the agent's log.

In addition to analyzing agent logs, it is important to verify the result of your instrumentation in the New Relic interface.  As you make changes, it is helpful to add [deployment markers](https://docs.newrelic.com/docs/java/recording-deployments-with-the-java-agent)

```shell
java -jar newrelic.jar deployment
```

to the UI to make it easier to compare changes.  Remember, it often takes at least 3 minutes before seeing new data in the UI. [Enabling](https://docs.newrelic.com/docs/java/java-agent-configuration#cfg-sync_startup) `sync_startup` can reduce the time to report data. 
