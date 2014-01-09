---
layout: post
title: "RKObjectManager high level overview"
date: 2014-01-10 00:00:00 +0000
meta_description: "RKObjectManager - understand how it works internally. Understand where to override default behavior"
component: Network
component_class: "label-network"
---
Overview with pretty pictures and advise on best points to override default behavior.

After reading this post, I want you to:

* understand how `RKObjectManager` works internally
* know about methods to implement in subclass to override default behavior

## High-level diagram

![RKObjectManager flow diagram](/images/RKObjectManager_diagram_flow.png)

### Legend

*Pumpkin* circle - step number in the flow
  
*Green* rectangle - place for customization (to override default behavior)

## Walkthrough

1. This is the starting point of the flow - you requesting or updating some objects. Usuauly you call one of these methods on `RKObjectManager` instance:
  * `– getObjectsAtPath:parameters:success:failure:`
  * `– getObjectsAtPathForRelationship:ofObject:parameters:success:failure:`
  * `– getObjectsAtPathForRouteNamed:object:parameters:success:failure:`
  * `– getObject:path:parameters:success:failure:`
  * `– postObject:path:parameters:success:failure:`
  * `– putObject:path:parameters:success:failure:`
  * `– patchObject:path:parameters:success:failure:`
  * `– deleteObject:path:parameters:success:failure:`
2. Next, `RKObjectManager` tries to determine if you're making request by path (e.g. to load new objects) or you're making request by object (e.g. to update existing object, to POST/PUT/PATCH an object). If you're making request to load new objects, go to **Step 4**.
3. `RKObjectManager` aware that we're making request for an object. Are we doing this for an instance of `NSManagedObject`?
4. If yes, manager will create instance of `RKManagedObjectRequestOperation`, if no it'll be an instance of `RKObjectRequestOperation`. Reason for that - managing persistence of CoreData objects is a lot of boilerplate code and RestKit handles that very well for us. So from an end-user perspective it's really as easy as operating with objects that are based on `NSObject` class (i.e. w/o any persistence).

  **NOTE**: `RKManagedObjectRequestOperation` inherits from `RKObjectRequestOperation`, so starting from now I'll be refering to both of these operations as `RKObjectRequestOperation`, though the actual class may be `RKManagedObjectRequestOperation` depending on context when this operation is executed.
5. `RKObjectManager` understands what class do we want to use based on type of request we're performing. In this step, RKObjectManager going to create an `RKHTTPRequestOperation` and assign it to instance of `RKObjectRequestOperation`. Later on, `RKObjectRequestOperation` will "watch" for execution of `RKHTTPRequestOperation` and once it's done, will map deserialized response. Current step finishes two-step initialization process of `RKObjectRequestOperation` (you can see area marked with green background).
6. Add newly created `RKObjectRequestOperation` to the `RKObjectManager` operation queue using method `- enqueueObjectRequestOperation:`

## Customization

Next thing to talk about - customziation of `RKObjectManager`. First of all take another look on a diagram, briefly you can spot there 2 green rectangles.
Let's go over each of them and talk thru methods that get called to override.

First area includes steps **4** and **5**. It's really all about initialization of `RKObjectRequestOperation`. Below are methods to override:

  * `- requestWithObject:method:path:parameters:` - method that constructs `NSMutableURLRequest` object that will be used later on to construct actual `RKObjectRequestOperation`. 
  This is great place to customize actual HTTP request that will be fired to API server. For example, you can set timeout, allow/disallow cellular access, change/update HTTP header in the implementation of this method.

  * if you need to alter behavior on a lower level, i.e. on HTTP request level, you need to provide subclass implementation of `RKHTTPRequestOperation` and register it on `RKObjectManager` by calling method `registerRequestOperationClass:`. `RKRKHTTPRequestOperation` inherits from `AFHTTPRequestOperation` and conforms to `NSURLConnectionDelegate` and `NSURLConnectionDataDelegate` protocols. Check them out.

Second area is really simple, cuz it's just one method:

* `enqueueObjectRequestOperation:` - this method enqueues already prepared object request operation. Personally, I find couple use cases for this situation. For example, when `RKObjectRequestOperation` is really a background request, and you don't do anything with UI (e.g. sending some stats or storing some low-priority user data), you would want to set properties `successCallbackQueue` and `failureCallbackQueue` to be a background queues. That means all operations triggered by `RKObjectManager` will run success/failure callbacks in background queue, which is kinda makes sense in some situtations. Another use case, would be to set "will map response" block using method `setWillMapDeserializedResponseBlock:` on `RKObjectRequestOperation` if you need to notify something about started mapping operation or change incoming JSON.

## Conclusions

Alright, I hope you find these tips useful for your RestKit use. As you can see RestKit is so flexible and provides great support for customization for any need.