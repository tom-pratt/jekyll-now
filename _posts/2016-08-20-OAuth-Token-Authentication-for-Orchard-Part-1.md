---
layout: post
title: OAuth Token Authentication for Orchard - Part 1
published: true
---
Lots of apps and websites require that users create an account, maybe to store user preferences, user data or to allow for special permissions. Orchard comes with user accounts built in and all we want to do is extend it a little to make this functionality available to our mobile client apps.

To do so we'll create an Orchard Module which can be dropped into any Orchard project. The module will contain an API AccountController with a /Register endpoint for creating new accounts. There will also be an OAuth /Token endpoint for logging in with a username and password and receiving a valid access token in return.

We'll be using the [http://tools.ietf.org/html/rfc6749#section-1.3.3](Resource Owner Password Credentials) OAuth flow. Which means the user types their Orchard username and password directly into your app which are sent off to your Orchard website and exchanged for an access token. We won't implement refresh tokens because mobile apps can't easily keep a client secret which would be required for refresh tokens. You just have to set an expiry time on the access tokens that you're comfortable with.

**Source Code**

The Morphous.TokenAuth module can be used in any Orchard project, you don't need the Morphous.Orchard fork for it to work:
[https://github.com/Morphous/Morphous.TokenAuth](https://github.com/Morphous/Morphous.TokenAuth)

And here is a blog post from bitoftech.net about Owin, OAuth and tokens which helped me a lot when I first started on this topic. My post will be similar in lots of ways but built up around Orchard instead of ASP.NET Identity.
[http://bitoftech.net/2014/06/01/token-based-authentication-asp-net-web-api-2-owin-asp-net-identity/](http://bitoftech.net/2014/06/01/token-based-authentication-asp-net-web-api-2-owin-asp-net-identity/)

[Part 2 - The follow up blog post](http://www.sylapse.com/orchard/oauth-token-authentication-for-orchard-part-2/)

*Note that there are some small name changes in the repository code vs the code samples in this post.


<h3>Requirements</h3>
There's a convenient way to get started and that's by looking at one of the Visual Studio template projects which comes with this type of authentication built in. Once we've worked out what the important bits are, we can try to recreate it in Orchard, using the Orchard user management system.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/1_webapi_template.jpg"><img class="alignnone size-full wp-image-90" src="http://www.sylapse.com/wp-content/uploads/2015/07/1_webapi_template.jpg" alt="1_webapi_template" width="784" height="588" /></a>

This template gives us ASP.NET Identity (the standard .NET user management system) for user accounts and OAuth for issuing tokens. When we combine the two, we can put the [Authorize] attribute on our controllers to restrict access and get information about the current user. The OAuth stuff comes implemented as an OWIN middleware. If you don't know much about OWIN then for now it's enough to know that OWIN middlewares contain code that is executed when a request comes into our website before it hits the relevant controller, which is a good time to do authentication.

Let's take a look at some of the files and code in the template project and use it to figure out which bits we need to recreate and modify when we move over to Orchard. The Global.asax file in the root of the project might be a good place to start. We can see a few things that we might expect in this file, WebApi and Routing are being configured along with some other stuff. But there's no mention of authentication/OAuth/tokens yet.

```
namespace WebApiTest {
    public class WebApiApplication : System.Web.HttpApplication {
        protected void Application_Start() {
            AreaRegistration.RegisterAllAreas();
            GlobalConfiguration.Configure(WebApiConfig.Register);
            FilterConfig.RegisterGlobalFilters(GlobalFilters.Filters);
            RouteConfig.RegisterRoutes(RouteTable.Routes);
            BundleConfig.RegisterBundles(BundleTable.Bundles);
        }
    }
}
```

Move on to the Startup.cs file. It looks promising because not only is there a method named ConfigureAuth but we can see the OwinStartup attribute too, this attribute just causes the Startup class to be detected and executed when the website first starts up. This is where you can register your Owin middlewares.

```
[assembly: OwinStartup(typeof(WebApiTest.Startup))]
namespace WebApiTest {
    public partial class Startup {
        public void Configuration(IAppBuilder app) {
            ConfigureAuth(app);
        }
    }
}
```

Right click and use Go To Definition on the ConfigureAuth method. Once you're there, you can see it registers some useful services against the Owin context (ApplicationDbContext and ApplicationUserManager), they can then be retrieved from the OwinContext later in a controller. These are specific to ASP.NET Identity, Orchard actually has it's own user management services which we will get in our ApiControllers via dependency injection. Orchard also has cookie authentication set up already so really the <em>OAuthOptions</em> and the bit on line 20 are the bits we're interested in.

```
public void ConfigureAuth(IAppBuilder app) {
     // Configure the db context and user manager to use a single instance per request
    app.CreatePerOwinContext(ApplicationDbContext.Create);
    app.CreatePerOwinContext&lt;ApplicationUserManager&gt;(ApplicationUserManager.Create);

    // Enable the application to use a cookie to store information for the signed in user
    app.UseCookieAuthentication(new CookieAuthenticationOptions());
    app.UseExternalSignInCookie(DefaultAuthenticationTypes.ExternalCookie);

    // Configure the application for OAuth based flow
    OAuthOptions = new OAuthAuthorizationServerOptions {
        TokenEndpointPath = new PathString(&quot;/Token&quot;),
        Provider = new ApplicationOAuthProvider(PublicClientId),
        AuthorizeEndpointPath = new PathString(&quot;/api/Account/ExternalLogin&quot;),
        AccessTokenExpireTimeSpan = TimeSpan.FromDays(14),
        AllowInsecureHttp = true
    };

    // Enable the application to use bearer tokens to authenticate users
    app.UseOAuthBearerTokens(OAuthOptions);
}
```

This sets up some options and then finally, registers the OAuth middleware. Configuring and registering the OAuth middleware will be our Step 1 when we move over to Orchard. The TokenEndpointPath is the URL that clients post their username and password to in exchange for a token. The Provider is important too so head over to the ApplicationOAuthProvider class and check out its GrantResourceOwnerCredentials method. It's what decides if the username and password are valid or not and either issues a token or returns an error message. It also generates some Cliams which will be encrypted into the token. Our Claims will be a bit simpler so don't worry about these ones too much, they're specific to Identity.

```
public override async Task GrantResourceOwnerCredentials(OAuthGrantResourceOwnerCredentialsContext context) {
    var userManager = context.OwinContext.GetUserManager&lt;ApplicationUserManager&gt;();
    ApplicationUser user = await userManager.FindAsync(context.UserName, context.Password);

    if (user == null) {
        context.SetError(&quot;invalid_grant&quot;, &quot;The user name or password is incorrect.&quot;);
        return;
    }

    ClaimsIdentity oAuthIdentity = await user.GenerateUserIdentityAsync(userManager, OAuthDefaults.AuthenticationType);
    ClaimsIdentity cookiesIdentity = await user.GenerateUserIdentityAsync(userManager, CookieAuthenticationDefaults.AuthenticationType);

    AuthenticationProperties properties = CreateProperties(user.UserName);
    AuthenticationTicket ticket = new AuthenticationTicket(oAuthIdentity, properties);
    context.Validated(ticket);
    context.Request.Context.Authentication.SignIn(cookiesIdentity);
}
```

Creating a provider like this will be our Step 2. Ours will use Orchard's IMembershipService to validate a client's username as password.

The last thing to look at in this Visual Studio template is the AccountController class. It's an ApiController for all other account related actions besides actually issuing a token. Open it up and look at some of the methods, the essential ones are things like "Register" and "ChangePassword". Have a glance through it, it contains lots of Identity specific terminology, all of which we will implement using Orchard's own user accounts system. Creating an AccountController is our Step 3.

<h3>Over to Orchard</h3>

The next thing to do is set up a new Orchard website. It's easy enough, all you need to do is download the source code from the Orchard website, unzip it and open the solution file in Visual Studio. If you run this in the browser then you can enter an admin username and password and choose your database type, either setup a new database in SQL Server or just choose SQL Server Compact for simplicity. Now we're in and ready to start writing our own module. If you don't know how to create an empty module in Orchard then <a href="http://docs.orchardproject.net/Documentation/Building-a-hello-world-module" target="_blank">read this</a>. Orchard contains a Code Generation feature so it can do the hard work for you leaving you with a new project in Visual Studio in the Modules folder which is ready for you to start coding in. Make sure you're using Orchard 1.9 or newer because 1.9 is the first one that gives you the ability to easily add Owin middlewares. I called the module HttpAuth.

You should have a module which looks like this in Visual Studio

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/2_module_structure.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/2_module_structure.jpg" alt="2_module_structure" width="426" height="425" class="alignnone size-full wp-image-91" /></a>

Remember to enable your module in Orchard like you did for Code Generation when you created the module.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/3_enable_module.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/3_enable_module.jpg" alt="3_enable_module" width="723" height="312" class="alignnone size-full wp-image-92" /></a>

We're ready to go now so lets remember the steps we picked out earlier when we looked at the Visual Studio template solution.
<ol>
<li>Register the OAuth middleware.</li>
<li>Create a custom Provider which validates user login details.</li>
<li>Create an AccountController so new users can create an account.</li>
</ol>

<strong><em>Register the Middleware</em></strong>

In Orchard, we don't have to worry about the Owin Startup class, Orchard takes care of that and gives us an extensibility point for adding middlewares in the form of the IOwinMiddlewareProvider interface. Create a folder in your Orchard module called "Middleware" and create a class in there called "AuthMiddleware". Paste in the following:

```
using HttpAuth.Providers;
using Microsoft.Owin;
using Microsoft.Owin.Security.OAuth;
using Orchard.Environment;
using Orchard.Owin;
using Orchard.Security;
using Orchard;
using Owin;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Web;

namespace HttpAuth.Middleware {
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

                        app.UseOAuthAuthorizationServer(oAuthOptions);
                        app.UseOAuthBearerAuthentication(new OAuthBearerAuthenticationOptions());                        
                    }
                }
            };
        }
    }
}
```

<em>UPDATE - Orchard now uses Nuget instead of a lib folder for most things so this next bit is easier than when the post was first written and the next couple of paragraphs are less relevant now.</em>

We haven't created the AuthProvider yet but ignoring that for now, we see that we're missing some references. We're gonna get them from Nuget but we have to be careful because Orchard stores all the DLLs it needs in a "lib" folder. If we start Nugeting in different versions of DLLs that Orchard already has in its lib folder then we'll end up with runtime errors. We either need to nuget in the same versions or we need to reference them straight from the "lib" folder. We'll just be sure to nuget in the same versions this time.

The Nuget package we need is Microsoft.Owin.Security.OAuth, if you search for it in the Package Manager GUI then the current version is 3.0.1 (at the time of writing). You'll see it has a dependency on Microsoft.Owin 3.0.1 or higher. Orchard already has a Microsoft.Owin DLL in the lib folder and it's only version 3.0.0. Looking on the <a href="ttps://www.nuget.org/packages/Microsoft.Owin.Security.OAuth" target="_blank">Nuget website</a> we can go back through previous versions until we find one that doesn't have conflicting dependencies, in our case this is version 3.0.0. So Open the package manager console and make sure you set the Default Project to be your module and not Orchard.Web. Then type in 

<blockquote>Install-Package Microsoft.Owin.Security.OAuth -Version 3.0.0</blockquote>

 and hit enter.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/4_nuget_owin.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/4_nuget_owin.jpg" alt="4_nuget_owin" width="753" height="309" class="alignnone size-full wp-image-93" /></a>

That should sort out the missing references (except for the missing AuthProvider which we'll make after this). The AuthMiddleware class implements IOwinMiddlewareProvider. Orchard will detect and execute it for us and pass in an IAppBuilder (app), which is an important Owin object which we can use to configure middlewares. The AuthMiddleware class supports dependency injection and we inject Orchard's IWorkContextAccessor, it doesn't get used here but we pass it on to our AuthProvider. What we really need is the IMembershipService but we can't inject it directly here so we'll use the IWorkContextAccessor to resolve it later. I'm not entirely sure why injecting the IMembershipService doesn't work, it did used to work in an older version of Orchard. It's worth noting the last two lines in our middleware registration:

```
app.UseOAuthAuthorizationServer(oAuthOptions);
app.UseOAuthBearerAuthentication(new OAuthBearerAuthenticationOptions());
```

This is where we're registering the OAuth middleware, it's slightly different to what we saw in the Visual Studio WebApi template solution. The template uses an extension method from Identity, whereas we are using more general extension methods from the OAuth assembly. I'm sure the Identity one would be calling these at some point.

<strong><em>Create the Provider</em></strong>

Create a folder called Providers and in it, a class called AuthProvider. Paste in this:

```
using Microsoft.Owin.Security.OAuth;
using Orchard;
using Orchard.Environment;
using Orchard.Security;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Security.Claims;
using System.Threading.Tasks;
using System.Web;

namespace HttpAuth.Providers {
    public class AuthProvider : OAuthAuthorizationServerProvider {
        private readonly IWorkContextAccessor _workContextAccessor;

        public AuthProvider(IWorkContextAccessor workContextAccessor) {
            _workContextAccessor = workContextAccessor;
        }

        public override Task ValidateClientAuthentication(OAuthValidateClientAuthenticationContext context) {
            context.Validated();
            return Task.FromResult&lt;object&gt;(null);
        }

        public override Task GrantResourceOwnerCredentials(OAuthGrantResourceOwnerCredentialsContext context) {
            context.OwinContext.Response.Headers.Add(&quot;Access-Control-Allow-Origin&quot;, new[] { &quot;*&quot; });
            var membershipService = _workContextAccessor.GetContext().Resolve&lt;IMembershipService&gt;();

            var user = membershipService.ValidateUser(context.UserName, context.Password);
            if (user == null) {
                context.SetError(&quot;invalid_grant&quot;, &quot;The user name or password is incorrect.&quot;);
            } else {
                var identity = new ClaimsIdentity(context.Options.AuthenticationType);
                identity.AddClaim(new Claim(ClaimTypes.Name, context.UserName));
                context.Validated(identity);
            }
            
            return Task.FromResult&lt;object&gt;(null);
        }
    }
}
```

This is pretty similar to the Provider we saw in the Visual Studio template solution. We just use the IMembershipService to check if the username and password are valid for a user registered with this Orchard website. If not we set an error, if they are valid then we validate them. We generate a claim for them which is just a key value pair which will be encrypted into the token that they receive. We add the username as a claim using the <em>ClaimTypes.Name</em> string for the key. Using this particular key for the claim will help to make the username conveniently accessible when the user sends an authorized request to one of our ApiControllers, as we will see later on.

We're ready to build and run our Auth module for the first time now because we have everything we need to issue tokens to registered users. The only registered user we have so far is the admin user which you created when you ran your Orchard website for the very first time. Open up a REST client of your choice, I use the Postman Chrome app.

Set up your request like this.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/5_token_request.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/5_token_request.jpg" alt="5_token_request" width="865" height="555" class="alignnone size-full wp-image-94" /></a>

First of all enter the address, if you run the website straight from Visual Studio then it will look like the one in the picture, you might have set it up in IIS in which case it will be different. Make the request a POST, make sure you add the content as form-urlencoded, then enter the above key value pairs but change the username and password to the ones you set when you created your Orchard website.

Click Send and you should get your access token back (woop!). Remember it's got the username Claim encrypted into it, plus they tell you how long it's valid for incase it's of use to your client app. You can put a break point in the GrantResourceOwnerCredentials method in the AuthProvider class if you want to see Orchard's IMembershipService in action validating the username and password.

This is great because now clients can essentially log in to their account and receive a token which they can use for authorization in all future communication with our API. If like me, you're creating APIs with mobile app clients in mind, then it's pretty obvious that the next thing we'll need is a way to allow new users to create an account through the API. But there is another way to create an account already, because Orchard lets you create an account through the web browser. Run your website and log in to the admin area, at the bottom of the menu on the left, expand the Settings menu item and click Users.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/6_orchard_settings.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/6_orchard_settings.jpg" alt="6_orchard_settings" width="273" height="257" class="alignnone size-full wp-image-95" /></a>

Then tick "Users can create new accounts on this site" and click save.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/7_orchard_usersettings.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/7_orchard_usersettings.jpg" alt="7_orchard_usersettings" width="717" height="395" class="alignnone size-full wp-image-96" /></a>

Now navigate to http://localhost:30321/OrchardLocal/Users/Account/Register and you can create new accounts. Create one and use Postman to request another token using the new user details that you just created. Notice that we're starting to mix using our Orchard website through the browser and using our Orchard based API. This is nice because it's common to want to build a system where users access their account through your website or through your app seamlessly. 

This HttpAuth module feels like it could be useful in lots of different projects that require some sort of mobile client access to your Orchard system, and Orchard modules are very easily reusable. So lets get this AccountController out the way already!

<strong><em>Implement the AccountController</em></strong>

Before we create the AccountController we'll just set up some routing. Orchard does do this automatically for ApiControllers with the following format:

<blockquote>www.mywebsite.com/api/{modulename}/{controller}/{id}</blockquote>

Which is verb based and fine for CRUD style controllers, but for our AccountController we really want the action name in there too. So in the root of your module create a class called Routes and paste in the following (if you're missing a reference to System.Web.Http then don't worry, we'll fix it in a moment):

```
using Orchard.Mvc.Routes;
using Orchard.WebApi.Routes;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Web;
using System.Web.Http;

namespace HttpAuth {
    public class HttpRoutes : IHttpRouteProvider {

        public void GetRoutes(ICollection&lt;RouteDescriptor&gt; routes) {
            foreach (var routeDescriptor in GetRoutes()) {
                routes.Add(routeDescriptor);
            }
        }

        public IEnumerable&lt;RouteDescriptor&gt; GetRoutes() {
            return new[] {
                new HttpRouteDescriptor {
                    Priority = 5,
                    RouteTemplate = &quot;api/account/{action}/{id}&quot;,
                    Defaults = new {
                        area = &quot;HttpAuth&quot;,
                        controller = &quot;Account&quot;,
                        id = RouteParameter.Optional
                    }
                }
            };
        }
    }
}
```

This is similar to how we implemented our OWIN middleware provider and if you're familiar with Orchard then you probably regularly set up the routing for MVC controllers, which is very similar.

When people create an account they need to send up some details such as email address, password and confirmation password. In your scenario you might want to capture other information too but this is the minimum. We need a model to hold this, I copied the one called RegisterBindingModel from the Visual Studio template solution. Create a folder called TransferModels and a file named AccountBindingModels. Paste in this:

```
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;
using System.Linq;
using System.Web;

namespace HttpAuth.TransferModels {
    public class RegisterBindingModel {
        [Required]
        [EmailAddress]
        [Display(Name = &quot;Email&quot;)]
        public string Email { get; set; }

        [Required]
        [StringLength(100, ErrorMessage = &quot;The {0} must be at least {2} characters long.&quot;, MinimumLength = 6)]
        [DataType(DataType.Password)]
        [Display(Name = &quot;Password&quot;)]
        public string Password { get; set; }

        [DataType(DataType.Password)]
        [Display(Name = &quot;Confirm password&quot;)]
        [Compare(&quot;Password&quot;, ErrorMessage = &quot;The password and confirmation password do not match.&quot;)]
        public string ConfirmPassword { get; set; }
    }
}
```

It contains the information required to create an account and it has some validation attributes on the properties for checking ModelState. Next, create a class named AccountController in the Controllers folder.

This is one of those times where we have to be a bit careful with references. If we tell Visual Studio to add an empty WebApi 2 Controller for us then it's going to import references automatically, but it won't get them from Orchard's "lib" folder and we'll most likely get runtime errors. So just add an empty class and then add the following code:

```
using HttpAuth.TransferModels;
using Orchard.Localization;
using System;
using System.Web.Http;
using Orchard.Users.Services;
using Orchard.Users.Models;
using Orchard.Security;

namespace HttpAuth.Controllers {
    [Authorize]
    public class AccountController : ApiController {
        private readonly IMembershipService _membershipService;
        private readonly IUserService _userService;

        public AccountController(
            IMembershipService membershipService,
            IUserService userService) {
            _membershipService = membershipService;
            _userService = userService;
            T = NullLocalizer.Instance;
        }

        public Localizer T { get; set; }

        [AllowAnonymous]
        [HttpPost]
        public IHttpActionResult Register(RegisterBindingModel model) {
            if (model == null) {
                return BadRequest();
            }

            if (!ModelState.IsValid) {
                return BadRequest(ModelState);
            }

            if (!string.IsNullOrEmpty(model.Email)) {
                if (!_userService.VerifyUserUnicity(model.Email, model.Email)) {
                    AddModelError(&quot;NotUniqueUserName&quot;, T(&quot;User with that email already exists.&quot;));
                    return BadRequest(ModelState);
                }
            }

            var user = _membershipService.CreateUser(new CreateUserParams(model.Email, model.Password, model.Email, null, null, true));
            if (user != null) {
                return Ok();
            }

            return InternalServerError();
        }

        public void AddModelError(string key, LocalizedString errorMessage) {
            ModelState.AddModelError(key, errorMessage.ToString());
        }
    }
}
```

First of all, we'll fix the missing references, but this time we won't get them from nuget. It's all stuff that Orchard already has. So right click on "References" in your module in Visual Studio and click "Add Reference". First we'll add System.Web.Http, click "Browse" on the left of the Reference Manager, then click the "Browse" button at the bottom. Navigate to the "lib" folder in the root of your Orchard project folder then go into the "aspnetwebapi" folder, then add the "System.Web.Http" DLL. Now click "Solution" on the left of the Reference Manager and tick the "Orchard.Users" project.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/8_references.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/8_references-1024x420.jpg" alt="8_references" width="765" height="314" class="alignnone size-large wp-image-97" /></a>

Now we can have a look at the Register method. It's pretty easy to follow what's going on. I found the code for this in the AccountController in the Orchard.Users module. That's what we were hitting when we created a new user account through the browser a few steps ago. It's worth going and having a look at it because getting familiar with the Orchard source code is an essential part of getting comfortable with Orchard in general. I was able to strip out a fair bit because we leave the validation to the DataAnnotations on our RegisterBindingModel. Whereas Orchard's version checks these in code and adds the ModelState errors manually. I left out the bit about confirmation emails just for simplicity, but if you want confirmation emails for your system then everything you need is in that AccountController in the Orchard.Users module so you could easily add it in to your WebApi AccountController. Also I haven't bothered to log the user in as part of the Register request. If you wanted to generate a token as part of the registration and return it all in one go then you could definitely modify it to do that. We just use the email address for the username.

The IMembershipService and IUserService that we got via dependency injection are some handy Orchard services that make creating a new user in code nice and easy.

This should be working now so run the website and fire up Postman. Set up your request like this.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/9_register_request.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/9_register_request.jpg" alt="9_register_request" width="1009" height="405" class="alignnone size-full wp-image-98" /></a>

If everything is working then you should get a 200 Ok response. It looks good but lets convince ourselves by going into the Orchard admin panel and clicking on the Users section in the menu on the left (not to be confused with the Users subsection of the Settings menu item).

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/10_user_created.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/10_user_created.jpg" alt="10_user_created" width="993" height="361" class="alignnone size-full wp-image-99" /></a>

Our newly registered user should be there. Try hitting the /Token url from Postman with these new account details.

At this point, our module allows clients to create accounts and to sign in and receive their Token. It might be the bare minimum, but still, we are at a point where we can start to make an actual API which relies on our authentication module.

<h3>Create an API</h3>

We'll create some API functionality in a brand new Orchard module. This just highlights that the HttpAuth module that we created doesn't need to have any knowledge of the rest of your website so you could use it in lots of different projects. I like to add these reusable modules as Git submodules to a main project repository. So if you add features or make improvements to it while working on one project, then all your other projects can easily pull in the changes and get the benefits.

Use Orchard's codegen feature again to create an empty module, I just called it MyApi. So you should have something like this:

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/11_myapi_module.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/11_myapi_module.jpg" alt="11_myapi_module" width="413" height="353" class="alignnone size-full wp-image-100" /></a>

Next we'll create a new class in the Controllers folder which will inherit from ApiController. Like when we created the AccountController earlier, we have to be careful not to accidentally add a reference to a newer version of System.Web.Http than the one in the Orchard "lib" folder.

We'll keep it simple, just to test the Authorization is working so paste in this:

```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Web;
using System.Web.Http;

namespace MyApi.Controllers {
    public class ValuesController : ApiController {
        [Authorize]
        public string Get() {
            return &quot;Hello, you are authorized.&quot;;
        }
    }
}
```

Now we'll test it, so run the website and make sure you enable your new module in the Orchard admin area. Next it's really important that you sign out of your Orchard website altogether, or the cookie authentication that comes with being signed in in the browser is going to affect our ability to test the API. So once you've signed out on the browser fire up Postman, first lets send a non authorized request. Set it up like this:

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/12_non_authorized.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/12_non_authorized.jpg" alt="12_non_authorized" width="915" height="347" class="alignnone size-full wp-image-101" /></a>

We haven't set up any routing so it's using the default Orchard format for WebApi controllers that was mentioned earlier and we are hitting our new ValuesController with a Get request. Unsurprisingly, it tells us that we are unauthorized, because we haven't included a valid token with our request.

Obtain a token by hitting the /Token url, refer to the Postman screenshot earlier in the post if you need to remind yourself of the exact format. Once you have your token, include it in a new request like this:

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/13_authorized.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/13_authorized.jpg" alt="13_authorized" width="909" height="353" class="alignnone size-full wp-image-102" /></a>

Success! By including the Authorization header in our request with a valid token, the OAuth middleware was able to do its job and authorize the request. So the request was able to get all the way to the controller action marked with [Authorize] and we got some data back.

This is working nicely because neither our HttpAuth module or the MyApi module have any knowledge or dependency on eachother whatsoever. So it's easy to see how the HttpAuth module could grow into something really useful and reusable if you're into building APIs in Orchard.

I'll finish this post here because it's getting long. You've seen how to use OAuth in your Orchard project to generate authorization tokens, all using Orchard's built in user management system. I'll do a Part 2 to follow up this post where I'll show how to get information about the currently authorized user in your ApiControllers. Remember earlier in this post when we embedded that special Claim into the token with the user's username? Part 2 will show how we actually put that to good use.
