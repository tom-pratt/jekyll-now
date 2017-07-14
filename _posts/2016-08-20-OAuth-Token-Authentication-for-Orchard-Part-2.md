---
layout: post
title: OAuth Token Authentication for Orchard - Part 2
published: true
---
This is Part 2 of adding OAuth Token Authentication to Orchard. In Part 1 we implemented everything we need to allow users to register with our Orchard website over WebAPI and for them to login and receive a Token which they could use to make authorized requests.

<strong>Source Code</strong>

The Morphous.TokenAuth module can be used in any Orchard project, you don't need the Morphous.Orchard fork for it to work:
<a href="https://github.com/Morphous/Morphous.TokenAuth" target="_blank">https://github.com/Morphous/Morphous.TokenAuth</a>

Lets add something else that will be useful in a lot of ApiControllers and that brings us back to the Claim we added to the token in the AuthProvider class in Part 1. We often want to get the current user in our controllers, usually we want to get their data out the database and display it for them or something like that. We'll cheat a little to begin with and then tidy it up after. Following on with the code we wrote in part 1... overwrite the code in the ValuesController with this:

[code language="csharp"]
using Orchard.Security;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Web;
using System.Web.Http;

namespace MyApi.Controllers {
    [Authorize]
    public class ValuesController : ApiController {
        private readonly IMembershipService _membershipService;

        public ValuesController(
            IMembershipService membershipService) {
            _membershipService = membershipService;
        }
        
        private IUser _orchardUser;
        public IUser OrchardUser {
            get {
                if (_orchardUser == null)
                    _orchardUser = _membershipService.GetUser(User.Identity.Name);
                return _orchardUser;
            }
        }

        [Authorize]
        public string Get() {
            return &quot;Hello, you are authorized. Id: &quot; + OrchardUser.Id;
        }
    }
}
[/code]

We inject the IMembershipService and we've added this OrchardUser property of type IUser, which is an Orchard interface for a user. Having an IUser in the controller is convenient because you often want to load data related to the current user, here we just print out the Id property to prove that we've got the user. The IMembershipService's GetUser method expects an Orchard username. Luckily for us this happens to be available in the "User.Identity.Name" property, note that this isn't an Orchard property, it's just a part of ApiController. So how does the ApiController know the Orchard username of the client that's making the request? It's thanks to the Cliam that we added to the Token in the AuthProvider class when the Token was generated. More specifically, it's thanks to the special key string we used to add the claim, it looked like this:

[code language="csharp"]
identity.AddClaim(new Claim(ClaimTypes.Name, context.UserName));
[/code]

So this works and we now have the current user which we could start using. But Orchard developers know that this isn't how we usually access the current user when we're in a Controller, usually we use the CurrentUser property on the WorkContext. At the moment this only works for requests that come in authorised with a cookie, Orchard does that out of the box. We just need to inform Orchard who the current user is when a request comes in with an authorisation token.

We need to set the current user for an incoming request after the OAuth middleware has authenticated the user but before it hits the controller. We can do this by adding a second Owin middleware to the pipeline. Open the AuthMiddleware class and add the second middleware so that the whole class looks like this.

[code language="csharp"]
using HttpAuth.Providers;
using Microsoft.Owin;
using Microsoft.Owin.Security.OAuth;
using Orchard.Owin;
using Orchard.Security;
using Orchard;
using Owin;
using System;
using System.Collections.Generic;

namespace HttpAuth.Owin {
    public class AuthMiddleware : IOwinMiddlewareProvider {
        private readonly IWorkContextAccessor _workContextAccessor;

        public AuthMiddleware(IWorkContextAccessor workContextAccessor) {
            _workContextAccessor = workContextAccessor;
        }

        public IEnumerable&lt;OwinMiddlewareRegistration&gt; GetOwinMiddlewares() {
            return new[] {
                new OwinMiddlewareRegistration {
                    Priority = &quot;1&quot;,
                    Configure = app =&gt; {
                        var oAuthOptions = new OAuthAuthorizationServerOptions
                        {
                            TokenEndpointPath = new PathString(&quot;/Token&quot;),
                            Provider = new AuthProvider(_workContextAccessor),
                            AccessTokenExpireTimeSpan = TimeSpan.FromDays(14),
                            AllowInsecureHttp = true
                        };

                        // Enable the application to use bearer tokens to authenticate users
                        app.UseOAuthAuthorizationServer(oAuthOptions);
                        app.UseOAuthBearerAuthentication(new OAuthBearerAuthenticationOptions());
                    }
                },
                new OwinMiddlewareRegistration {
                    Priority = &quot;2&quot;,
                    Configure = app =&gt; {
                        app.Use(async (context, next) =&gt; {
                            var workContext = _workContextAccessor.GetContext();
                            var authenticationService = workContext.Resolve&lt;IAuthenticationService&gt;();
                            var membershipService = workContext.Resolve&lt;IMembershipService&gt;();
                            
                            var orchardUser = membershipService.GetUser(context.Request.User.Identity.Name);
                            authenticationService.SetAuthenticatedUserForRequest(orchardUser);

                            await next();
                        });
                    }
                }
            };
        }
    }
}
[/code]

Notice the priority property which determines the order the middlewares are invoked. We need this new middleware to run second so that the user has already authenticated by then. Just like in the ValuesController at the start of this post we can get the current user's username from <em>User.Identity.Name</em> on the current request. Then we just use Orchard's IMembershipService and IAuthenticationService to set the current user for the request.

Lets change our ValuesController to look a bit more Orchardy. Overwrite the whole file with this.

[code language="csharp"]
using Orchard;
using System.Web.Http;
using Orchard.Core.Contents;

namespace HttpAuth.Controllers {
    public class ValuesController : ApiController {
        private readonly IOrchardServices _orchardServices;

        public ValuesController(IOrchardServices orchardServices) {
            _orchardServices = orchardServices;
        }
        
        public IHttpActionResult Get() {
            if (!_orchardServices.Authorizer.Authorize(Permissions.DeleteContent)
                return Unauthorized();

            return Ok(&quot;Your ID is &quot; + _orchardServices.WorkContext.CurrentUser.Id + &quot; and you are authorized to delete content&quot;);
        }
    }
}
[/code]

You'll need to install this Nuget package into your MyApi module to be able to use the <em>Unauthorized()</em> method.


<blockquote>
Install-Package System.Net.Http -Version 4.0.0
</blockquote>

First thing to notice is that we no longer need that OrchardUser property in the controller to manually go and get the current user. We can get the current user from the WorkContext, which was the main thing we were trying to achieve. Another benefit of having informed Orchard who the current user is, is that we can now use the Authorizer. Orchard uses Authorizer instead of the [Authorize] attribute and it gives very granular control over Orchard related permissions and Orchard developers will probably recognise it.

Lets make a couple of requests to the new version of the ValuesController. First lets make a request as the site admin, who we would expect to be authorized for <em>Permissions.DeleteContent</em>. First you'll need to get an access token, so if you need to, go back to <a href="http://www.sylapse.com/uncategorized/oauth-token-authentication-for-orchard-part-1/" target="_blank">Part 1</a> and look at the request for doing that. When you've got one, make a new request to the ValuesController like this:

<a href="http://www.sylapse.com/wp-content/uploads/2015/11/1_authorized_request.png"><img src="http://www.sylapse.com/wp-content/uploads/2015/11/1_authorized_request-1024x306.png" alt="1_authorized_request" width="765" height="229" class="alignnone size-large wp-image-389" /></a>

So, just like we thought, the admin user is authorized to delete content on this Orchard site. Lets see what happens if we make a request from a non admin user. I'm going to use tom@sylapse.com, which I created in Part 1, so after getting a token for that non admin email address, this is what happens when we make the same request.

<a href="http://www.sylapse.com/wp-content/uploads/2015/11/1_unauthorized_request.png"><img src="http://www.sylapse.com/wp-content/uploads/2015/11/1_unauthorized_request-1024x277.png" alt="1_unauthorized_request" width="765" height="207" class="alignnone size-large wp-image-390" /></a>

This user is unauthorised to delete content (they're a registered user, they just don't have the delete permission). Notice that this unauthorised response looks a little different to what we saw in Part 1. That's because we're not using the <em>[Authorize]</em> attribute any more which was picking up the error message set in the AuthProvider class. Here we just use the <em>Unauthorized()</em> method and we haven't set any error messages. We could set an error message here if we wanted, we could tell users they don't have permission to delete content, and if they weren't signed in at all we could tell them they're not authorized because they're not signed in. The Authorizer gives you that fine grained control.

That's all for now on this topic, I hope it has helped you. Feel free to get in touch with any questions.
