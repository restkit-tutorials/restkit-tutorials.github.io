---
layout: post
title: "Building iOS app with RestKit. Part 1"
date: 2013-12-28 00:00:00 +0000
meta_description: "How to build iOS app with RestKit. RestKit guide. RestKit Mapping."
component: Guide
component_class: "label-guide"
---
How-to build iOS app with RestKit. 
We're building Github Client. Part 1.


In the bottom you'll find link to github repo with code.
# API
As an API for our application we'll use so popular [GitHub API](http://developer.github.com/v3/).

Why?

* their API is really good
* you're familiar with concepts of Github, so I don't need to explain

## Prerequisites
* Github account
* Github's Personal Access Token (to get one go to **Account settings** -> **Applications** -> **Create new Personal Access Token**)
* iOS 7 project with RestKit installed ([How to install RestKit](https://github.com/restkit/restkit#installation))

(this project will be also available on Github)

To fully cover extensibility and maintainability we'll need to pick up few different resources that we want to incorporate into our application:

* Users
* Stars

Seems fairly reasonable to start developing against these 5 sets of features.

# Code Organization
This topic is well covered in this tutorial [Code Organization in RestKit-based app](/code-organization-in-restkit-based-app/) so make sure you're comfortable with approach taken there.

# API Endpoints
* [Get a single user](http://developer.github.com/v3/users/#get-a-single-user)
* [List starred repositories](http://developer.github.com/v3/activity/starring/#list-repositories-being-starred)
* [Check if repo is starred](http://developer.github.com/v3/activity/starring/#check-if-you-are-starring-a-repository)
* [Star repository](http://developer.github.com/v3/activity/starring/#star-a-repository)

Next step will be going to those links and reviewing API responses. 

# Step 1. Defining Models
Now we need to setup some models:

* User
* Repository

Let's define some attributes that we want to pull from Github API.

#### User.h
{% highlight objc %}
#import <Foundation/Foundation.h>

@interface User : NSObject

@property (nonatomic, strong) NSString *login;
@property (nonatomic, strong) NSNumber *userId;
@property (nonatomic, strong) NSString *avatarUrl;
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSNumber *publicReposCount;
@property (nonatomic, strong) NSNumber *publicGistsCount;

@end
{% endhighlight %}

Let's setup `Repository` model!

#### Repository.h
{% highlight objc %}
#import <Foundation/Foundation.h>

@class User;

@interface Repository : NSObject

@property (nonatomic, strong) NSNumber *repositoryId;
@property (nonatomic, strong) NSString *name;
@property (nonatomic, strong) NSString *description;
@property (nonatomic, strong) NSString *apiUrl;
@property (nonatomic, strong) NSString *starsCount;
@property (nonatomic, strong) NSNumber *privateRepository;
@property (nonatomic, strong) NSNumber *forkRepository;
@property (nonatomic, strong) User *owner;

@end
{% endhighlight %}

There're no implementations yet in any of these models. So I'm posting here just *.h files.
Let's move to next step!

# Step 2. Setup ObjectMapping
Time to tie up our NSObject models with Github's API.
I assume you followed this guide [Code Organization in RestKit-based app](/code-organization-in-restkit-based-app/) and you have created `MappingProvider` class.

Ok, let's talk about this mapping thing.
You have a JSON response and 2 classes User and Repository.
You need to map JSON response and create instances of User and Repository. RestKit will easily allow you to easily do that.

And here's what you need to do:

* create instance of `RKObjectMapping` - `mapping`
* define "rules" for transforming JSON in a format of `NSDictionary`, where **key** is the source key from JSON response and **value** is a destination key in User/Repository model
* assign "rules" for transforming JSON into instance of User/Repository using method `[mapping addAttributeMappingsFromDictionary:mappingDictionary];`


Header file is not so interesting, so make sure to check out implementation file of mapping provider.

#### MappingProvider.h
{% highlight objc %}
#import <Foundation/Foundation.h>

@class RKObjectMapping;

@interface MappingProvider : NSObject

+ (RKObjectMapping *) userMapping;
+ (RKObjectMapping *) repositoryMapping;

@end
{% endhighlight %}

#### MappingProvider.m
{% highlight objc %}
#import "MappingProvider.h"
#import <RestKit/RestKit.h>
#import "User.h"
#import "Repository.h"

@implementation MappingProvider

+ (RKObjectMapping *)userMapping {
    RKObjectMapping *mapping = [RKObjectMapping mappingForClass:[User class]];
    NSDictionary *mappingDictionary = @{@"login": @"login",
                                        @"id": @"userId",
                                        @"avatar_url": @"avatarUrl",
                                        @"name": @"name",
                                        @"public_repos": @"publicReposCount",
                                        @"public_gists": @"publicGistsCount"
                                        };

    [mapping addAttributeMappingsFromDictionary:mappingDictionary];

    return mapping;
}

+ (RKObjectMapping *)repositoryMapping {
    RKObjectMapping *mapping = [RKObjectMapping mappingForClass:[Repository class]];
    NSDictionary *mappingDictionary = @{@"id": @"repositoryId",
                                        @"name": @"",
                                        @"full_name": @"fullName",
                                        @"description": @"description",
                                        @"url": @"apiUrl",
                                        @"stargazers_count": @"stargazersCount",
                                        @"watchers_count": @"watchersCount",
                                        @"private": @"isPrivateRepository",
                                        @"fork": @"isForkedRepository"};

    [mapping addAttributeMappingsFromDictionary:mappingDictionary];
    [mapping addRelationshipMappingWithSourceKeyPath:@"owner" mapping:[self userMapping]];

    return mapping;
}

@end
{% endhighlight %}

# Step 3. Setup Object Manager
Before we continue, remember you needed to prepare couple of prerequisites and one of them was **Personal Access Token for Github**. This is the time to get it and start using.
Most of the requests in Github API require user authorization. So that API able to understand who's making requests. For the sake of simplicity of this tutorial, we're not building login screen, instead we'll be just supplying these **Personal Access Token** in every request, so that we can perform requests that require authorization. The way we do that is by defining literals (shame on me!) in _AKGithubClient-Prefix.pch_ (again, for sake of simplicity and keeping focus on topic)

**Note**: Token shown here is invalid (for obvious reason), so make sure to create yours!

**Note #2**: Make sure to not have slash at the end of `BASE_URL`. So that you will specify path when loading objects with slash at the beginning, e.g. `/user` which makes it a bit more explicit and readable.

#### AKGithubClient-Prefix.pch
{% highlight objc %}
#define BASE_URL @"https://api.github.com"
#define PERSONAL_ACCESS_TOKEN @"6fba03e4e425010d3bf108717280529f79b5c70a"
{% endhighlight %}

Here's the plan so far:

1. Create a subclass of `AKObjectManager` with name `UserManager`
2. Define public method `loadAuthenticatedUser`
3. Define private methods `setupRequestDescriptors` and `setupResponseDescriptors`

#### UserManager.h
{% highlight objc %}
#import "AKObjectManager.h"

@class User;

@interface UserManager : AKObjectManager

- (void) loadAuthenticatedUser:(void (^)(User *user))success failure:(void (^)(RKObjectRequestOperation *operation, NSError *error))failure;

@end
{% endhighlight %}

#### UserManager.m
{% highlight objc %}
#import "UserManager.h"
#import <RestKit/RestKit.h>
#import "MappingProvider.h"
#import "User.h"

@implementation UserManager

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

### Implementation Details
Now that you've looked at the code, let's go over and explain every bit of RestKit's code.
As you know, inheritance structure of `UserManager` can be represented as `UserManager` -> `AKObjectManager` -> `RKObjectManager`. That means every method from `RKObjectManager` is available in our `UserManager`.

Let's review how this manager works:

0. RestKit uses AFNetworking library for doing all the network stuff. So that RestKit follows AFNetworking's pattern on communicating responses back to the objects that invoked it - **blocks**. I'd advise to follow same pattern, as it greatly reduces complexity if you follow same pattern as these 2 libraries.
1. All of the RestKit's requests in `RKObjectManager` are asynchronous. They get posted into operation queue and processed (behind the scenes).
2. On the first line of that method we invoke `getObjectsAtPath:parameters:success:failure:`
This method will fire GET request to the specified path.
3. When server responds with some JSON, `UserManager` knows how to process that JSON, because you've specified that via `RKResponseDescriptor` mechanism. 
Take a look at method `setupResponseDescriptors`. We've created an instance of `RKResponseDescriptor` specifying what mapping we should use, when processing requests for a given **path** and given **HTTP method**. Very flexible and re-usable architecture.
4. Now, all you need to do is implement how your method `loadAuthenticatedUser:failure:` will communicate back `User` object.

If you were performing all of the steps described here by yourself, and you run the app, all requests will fail, **guess why?**

Because you never used that Github's `PERSONAL_ACCESS_TOKEN` that we've been talking about since the beginning of the post! So Github will always return **401 Unathorized** to all your requests.

Let's do that now!

### Customizing `AKObjectManager`
OK, now `AKObjectManager` really comes into play. When the task comes to add custom header to all of the HTTP requests coming out of the app, this is the right place to do it.

At the time of `sharedManager` initialization, set header like this:

#### AKObjectManager.m
{%highlight objc %}
...
        [sharedManager.HTTPClient setDefaultHeader:@"Authorization" value: [NSString stringWithFormat:@"token %@", PERSONAL_ACCESS_TOKEN]];
...
{% endhighlight %}
This will set default header value for the "Authorization" header.
Our request should work and you should get 200 Succcess response now and see the result in app!

You can get access to repo here [AKGithubClient](https://github.com/restkit-tutorials/AKGithubClient)

# Part 2
Below is a list of things that will be covered in part #2:

- browsing stars
- starring repo
- more RestKit ticks
