---
layout: post
title: "RestKit - How to set custom header for HTTP request"
date: 2013-12-29 00:00:00 +0000
meta_description: "How to set custom header in RestKit. This blog post explains 2 ways to set custom HTTP header to requests sent via RestKit"
component: Network
component_class: "label-network"
---
There are 2 different approaches to solve this problem. We'll go thru each of them and describe when to use them.

You can set custom header:

* on `RKObjectManager` level
* on `RKObjectRequestOperation` level

# Set HTTP Header for RKObjectManager

If you use `RKObjectManager` a lot, or subclass it to keep all networking code related to one particular resource in one manager. This is the way to go.

Every `RKObjectManager` has HTTPClient property, which is an instance of `AFHTTPClient`. So you can set custom HTTP header by calling `-setDefaultHeader:value:` on `HTTPClient`.

Example:

#### AKObjectManager.m
{% highlight objective-c %}
...
[sharedManager.HTTPClient setDefaultHeader:@"Authorization" value: [NSString stringWithFormat:@"token %@", PERSONAL_ACCESS_TOKEN]];
...
{% endhighlight %}

Or if you're using just `RKObjectManager`, you can do something like this:

#### ViewController.m
{% highlight objective-c %}
    [[RKObjectManager sharedManager].HTTPClient setDefaultHeader:@"X-API-TOKEN" value:SERVICE_PROVIDER_API_TOKEN];

    // all of the requests made below this point will have "X-API-TOKEN" header
    [[RKObjectManager sharedManager] getObjectsAtPath:@"/randomPath" parameters:nil success:^(RKObjectRequestOperation *operation, RKMappingResult *mappingResult) {
        // processing goes here
    } failure:^(RKObjectRequestOperation *operation, NSError *error) {
        // error handling goes here
    }];
{% endhighlight %}



# Set HTTP header for RKHTTPRequestOperation
In some situations customizing all requests coming out of `RKObjectManager` is not an option (for a number of reasons!)

To set HTTP header for `RKHTTPRequestOperation`, you need to provide `NSMutableURLRequest` object with header set using method `setValue:forHTTPHeaderField:`.

Let's figure out exact plan of how we can create `RKHTTPRequestOperation`:

1. define URL path - it will be used to build `NSMutableURLRequest` and `RKResponseDescriptor`
2. define `RKResponseDescriptor` that will be used to process response from API according to specified mapping and URL path
3. build full URL
4. build `NSMutableURLRequest`
5. **set custom header** on a `NSMutableURLRequest` object created in step 4
6. create `RKObjectRequestOperation` specifying `NSMutableURLRequest` and array of `RKResponseDescriptor` objects
7. set completion and failure blocks
8. enqueue `RKObjectRequestOperation` into object manager's operation queue

Below is an example of how you can re-write `UserManager`'s method to pull some data from API. You can review original method here: [AKGithubClient - UserManager.m](https://github.com/restkit-tutorials/AKGithubClient/blob/master/AKGithubClient/RestKit/Managers/UserManager.m#L16)

#### UserManager.m
{% highlight objective-c %}
- (void) loadAuthenticatedUser:(void (^)(User *))success failure:(void (^)(RKObjectRequestOperation *, NSError *))failure {
    NSString *urlPath = @"/user";

    RKResponseDescriptor *authenticatedUserResponseDescriptors = [RKResponseDescriptor responseDescriptorWithMapping:[MappingProvider userMapping] method:RKRequestMethodGET pathPattern:urlPath keyPath:nil statusCodes:RKStatusCodeIndexSetForClass(RKStatusCodeClassSuccessful)];

    NSURL *url = [NSURL URLWithString:urlPath relativeToURL:self.baseURL];
    NSMutableURLRequest *urlRequest = [[NSMutableURLRequest alloc] initWithURL:url];
    [urlRequest setValue:[NSString stringWithFormat:@"token %@", PERSONAL_ACCESS_TOKEN] forHTTPHeaderField:@"Authorization"];

    RKObjectRequestOperation *objectRequestOperation = [[RKObjectRequestOperation alloc] initWithRequest:urlRequest responseDescriptors:@[authenticatedUserResponseDescriptors]];

    [objectRequestOperation setCompletionBlockWithSuccess:^(RKObjectRequestOperation *operation, RKMappingResult *mappingResult) {
        if (success) {
            User *currentUser = (User *)[mappingResult.array firstObject];
            success(currentUser);
        }
    } failure:^(RKObjectRequestOperation *operation, NSError *error) {
        if (failure) {
            failure(operation, error);
        }
    }];

    [self enqueueObjectRequestOperation:objectRequestOperation];
}
{% endhighlight %}


That's it! You select, which one is appropriate for your situation!