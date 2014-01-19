---
layout: post
title: "Code Organization in RestKit-based app"
date: 2013-12-21 00:00:00 +0000
meta_description: "Code setup in RestKit. Code Organization in RestKit. Best RestKit setup. Blog post about creating flexible and maintainable ios app with RestKit"
component: Support
component_class: "label-support"
---
Code organization and app structure are such an opinionated topics. Today we'll talk about following some design principles suggested by RestKit in our application and creating extendable application that will heavily utilize this library.

### TL;DR
* keep all RestKit mappings in separate class `MappingProvider`
* create `AKObjectManager` inherited from `RKObjectManager` to have a single point of customization for all HTTP requests
* create `<Resource>Manager` inherited from `AKObjectManager` to keep all logic and network processing (along with mapping) for that specific resource in one class

# Components
Here's a list of key components involved in any app development with RestKit:

- Object Mapping (parsing data and creating objects)
- Networking (performing actual HTTP requests)
- Object Management (retrieving, posting, deleting objects)

# Everything starts with models
For simplicity we'll assume that our app is online-only, so it does not use CoreData. That means all our models are based on `NSObject` class.
This one is really simple, because it's what you naturally do by default. You keep class for every model in separate files.

#### User.h
{% highlight objective-c %}
#import <Foundation/Foundation.h>

@interface User : NSObject

@property (nonatomic, strong) NSNumber *userId;
@property (nonatomic, strong) NSString *emailAddress;
@property (nonatomic, strong) NSString *login;

@end
{% endhighlight %}

# Object Mapping
Now, that we have our model ready, we work on mapping JSON response to our model. And for that task, you want to keep your mappings somewhat away from `RKObjectManager`.
Main advantage of this approach is to keep your mappings shareable across different `ObjectManagers`, so that you do not repeat yourself and can re-use one mapping in multiple object managers.

1. Create a class `MappingProvider` based on `NSObject`
2. In header file of `MappingProvider` class define public class methods that will return mappings for all responses you need.
I will skip implementation details at this point, as this is really out of scope of current topic.

#### MappingProvider.h
{% highlight objective-c %}
#import <Foundation/Foundation.h>

@class RKObjectMapping;

@interface MappingProvider : NSObject

+ (RKObjectMapping *) userMapping;
+ (RKObjectMapping *) repositoryMapping;

@end
{% endhighlight %}

Ok, now that you have model and mapping provider, let's continue to the last step - managers!

# Object Management
RestKit has base class for all sorts of things around object management - `RKObjectManager`.

### Step 1. 
Create a subclass of `RKObjectManager` and name it something like `AKObjectManager`. Of course, you can use any class prefix you want. In the future, we'll inherit all object managers from `AKObjectManager`. Reasons why you want another base class:

- allows you to keep setup of required HTTP headers in one place (do that once in `AKObjectManager`)
- easier to extend. For example if you want custom default behavior for all your HTTP requests, you override methods in `AKObjectManager`.
- more flexible structure


### Step 2. 
Define `sharedManager` public method in `AKObjectManager`. Also define 2 methods that potentially all subclasses of `AKObjectManager` will use. These 2 methods are for setting up `RKRequestDescriptor`s and `RKResponseDescriptor`s.

#### AKObjectManager.h
{% highlight objective-c %}
#import "RKObjectManager.h"

@interface AKObjectManager : RKObjectManager

+ (instancetype) sharedManager;

- (void) setupRequestDescriptors;
- (void) setupResponseDescriptors;

@end
{% endhighlight %}

#### AKObjectManager.m
{% highlight objective-c %}
#import "AKObjectManager.h"
#import <RestKit/RestKit.h>

@implementation AKObjectManager

+ (instancetype)sharedManager {
    NSURL *url = [NSURL URLWithString:BASE_URL];

    AKObjectManager *sharedManager  = [self managerWithBaseURL:url];
    sharedManager.requestSerializationMIMEType = RKMIMETypeJSON;
    /*
     THIS CLASS IS MAIN POINT FOR CUSTOMIZATION:
     - setup HTTP headers that should exist on all HTTP Requests
     - override methods in this class to change default behavior for all HTTP Requests
     - define methods that should be available across all object managers
     */

    [sharedManager setupRequestDescriptors];
    [sharedManager setupResponseDescriptors];

    [sharedManager.HTTPClient setDefaultHeader:@"Authorization" value: [NSString stringWithFormat:@"token %@", PERSONAL_ACCESS_TOKEN]];

    return sharedManager;
}

- (void) setupRequestDescriptors {
}

- (void) setupResponseDescriptors {
}

@end
{% endhighlight %}

### Step 3.
Every time you have a new resource that you need to load to an application you should create new class inherited from `AKObjectManager`, e.g. `UserManager`. Define method `sharedManager` using Apple's approach to singleton patter. Also define methods `setupRequestDescriptor` and `setupResponseDescriptor` (they can be private methods). These methods will be automatically called when you invoke `sharedManager`. And then, make sure to put all your code that adds request/response descriptors into these 2 methods respectively.
 
#### UserManager.h
{% highlight objective-c %}
#import "AKObjectManager.h"

@class User;

@interface UserManager : AKObjectManager

- (void) loadAuthenticatedUser:(void (^)(User *user))success failure:(void (^)(RKObjectRequestOperation *operation, NSError *error))failure;

@end
{% endhighlight %}

#### UserManager.m
{% highlight objective-c %}
#import "UserManager.h"
#import <RestKit/RestKit.h>
#import "MappingProvider.h"
#import "User.h"

static UserManager *sharedManager = nil;

@implementation UserManager

+ (instancetype)sharedManager {
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedManager = [super sharedManager];
    });

    return sharedManager;
}

- (void) loadAuthenticatedUser:(void (^)(User *))success failure:(void (^)(RKObjectRequestOperation *, NSError *))failure {
    [self getObjectsAtPath:@"/user" parameters:nil success:^(RKObjectRequestOperation *operation, RKMappingResult *mappingResult) {
        if (success) {
            User *currentUser = (User *)[mappingResult.array firstObject];
            success(currentUser);
        }
    } failure:^(RKObjectRequestOperation *operation, NSError *error) {
        if (failure) {
            failure(operation, error);
        }
    }];
}

#pragma mark - Setup Helpers

- (void) setupResponseDescriptors {
    [super setupResponseDescriptors];

    RKResponseDescriptor *authenticatedUserResponseDescriptors = [RKResponseDescriptor responseDescriptorWithMapping:[MappingProvider userMapping] method:RKRequestMethodGET pathPattern:@"/user" keyPath:nil statusCodes:RKStatusCodeIndexSetForClass(RKStatusCodeClassSuccessful)];
    [self addResponseDescriptor:authenticatedUserResponseDescriptors];
}

@end
{% endhighlight %}

### Step 4. 
Now you can load 'authenticated user' like this:
#### ViewController.m
{% highlight objective-c %}
#import "AKViewController.h"
#import "UserManager.h"
#import "User.h"

@interface AKViewController ()

@end

@implementation AKViewController

- (void)viewDidLoad
{
    [super viewDidLoad];
    
    [[UserManager sharedManager] loadAuthenticatedUser:^(User *user) {
        NSLog(@"user is: %@", user.login);
    } failure:^(RKObjectRequestOperation *operation, NSError *error) {
        NSLog(@"error occured: %@", error);
    }];
}

@end
{% endhighlight %}

Have an opinion? Post a comment, send a message or tweet.