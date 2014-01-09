---
layout: post
title: "Logging in RestKit"
date: 2014-01-02 00:00:00 +0000
meta_description: "Logging in RestKit. How to enable logging in RestKit. Guide on logging in RestKit."
component: Support
component_class: "label-support"
---
In this post we'll talk about flexible and extensive logging available in RestKit.

Logging implementation in RestKit based on [LibComponentLogging](http://0xc0.de/LibComponentLogging) and offers variety of options that could be useful during app development and debugging.

## Configuring Logging in RestKit

Use method `RKLogConfigureByName` to set logging level to particular components.

`RKLogConfigureByName(name, level)` - expects 2 parameters:

- `name`, C string, log component name you want to set log level to. List of available components avaialble below
- `level`, log level. List of available aliases for log levels available below

Example:
{% highlight objc %}
RKLogConfigureByName("RestKit/Network", RKLogLevelTrace);
RKLogConfigureByName("RestKit/ObjectMapping", RKLogLevelDebug);
{% endhighlight %}

You can clone [AKGithubClient app](https://github.com/restkit-tutorials/AKGithubClient) and test out some basic changes, to see the results, you'll be impressed with the amount of useful debug information available for your eyes, right at the time when you need it.

### Example output

{% highlight ruby %}
T restkit.network:RKObjectRequestOperation.m:148 GET 'https://api.github.com/user':
{% endhighlight %}
This is how debug line look like, as you can see it starts from single capital letter T, that means log level for component is set to "Trace". Next you can see RestKit component that printed this line, it is `restkit.network` in this case. Then class name and line number for log statement.

## Configure logging from environment variables
Feature that I like the most in the RestKit's logging approach - configuration using environment variables.
Go to *Product -> Scheme -> Edit Scheme...*. (or press **CMD + <**). Find section "Environment Variables" and start customizing!

Name for environment variable should be in following format:
`RKLogLevel.<Component separated by .>`, for example:

* `RKLogLevel.RestKit.Network`
* `RKLogLevel.RestKit.ObjectMapping`
* `RKLogLevel.RestKit.Network.CoreData`

Available values for environment variables are:

* Default
* Critical
* Error
* Warning
* Info
* Debug
* Trace

(you can also set these values as numeric, starting from 0)

Ok, so now that you've confingured your scheme, you need to make RestKit to configure logging from environment variables. To do this, you simply need to call method `RKLogConfigureFromEnvironment();` somehwere in the beginning of initialization. After that, RestKit will process environment variables and adjust logging for specified components accordingly. 


## RestKit logging components

1. `"App"` - app wide component that will allow the end-user of RestKit to leverage the power of RKLog
2. `"RestKit"` - "super" component. All of the components below are part of this parent one
3. `"RestKit/Network"` - for all network related stuff. Classes invovled are `RKHTTPRequestOperation`, `RKObjectRequestOperation` and `RKObjectManager` (because of 2 previous)
4. `"RestKit/Network/CoreData"` - just one class - `RKManagedObjectRequestOperation` - for pulling objects and storing them in CoreData
5. `"RestKit/CoreData"` - for everything about CoreData in RestKit
6. `"RestKit/CoreData/Cache"` - very narrow component used when debugging CoreData's entity attribute cache.
7. `"RestKit/ObjectMapping"` - component responsible for all mapping operations. When you have issues trying to figure out **why the hell mapping doesn't work** - configure log level for this component to be `RKLogLevelTrace`
8. `"RestKit/Support"` - component responsible for MIME serialization and Date Formatting.


## Log Level Aliases
RestKit supports multiple levels of logging. Their names are pretty descriptive and I am sure you are smart to understand them.
I'm posting aliases here, so that you can always refer to them when you need to.

1. `RKLogLevelOff`
2. `RKLogLevelCritical`
3. `RKLogLevelError`
4. `RKLogLevelWarning`
5. `RKLogLevelInfo`
6. `RKLogLevelDebug`
7. `RKLogLevelTrace`


