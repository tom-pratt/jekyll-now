---
published: true
layout: post
title: Morphous - OrchardCMS as a REST webservice
---
Orchard is a .NET content management system, its modern and flexible framework makes it a good basis for all sorts of web applications, I find it an excellent choice for building REST web services. I tend to work with web services and mobile apps but you can just as easily use them from javascript in the browser or anywhere else. Morphous is a project to expose many of Orchard's main features in a RESTful manner.

The beauty of using Orchard as a basis is that it is (primarily) a website, with a wealth of MVC functionality and a rich content management system, so it's not just about WebAPI at all. Most mobile apps that rely on a behind-the-scenes web service tend to have a fully functional browser based equivalent. Think of a News app or Social Media app, you can always log into your account in a browser and experience more or less the same content and functionality. In this scenario, Orchard offers loads of benefits, because when you set up the data structures and everything that the web service needs, the website naturally begins to emerge.

If you use Xamarin for mobile app development then you're already used to thinking about maximising code reuse. Using Orchard for your web service and website is quite similar. Using a framework such as MvvmCross or Xamarin.Forms for your mobile app development can also help to reduce the mental overload and increase efficiency.

When I tackle a typical project, I'm often faced with something like this:
<ul>
	<li>Web Service</li>
	<li>Web site</li>
	<li>iOS app</li>
	<li>Android app</li>
	<li>Windows Phone app</li>
</ul>
But in my mind I think of it like this:
<ul>
	<li>Orchard project</li>
	<li>MvvmCross project</li>
</ul>
Which makes things a lot more manageable. Not only have we cut five projects down to two, but they're both made with C# and .NET. This is the sort of thing that can be achieved by a small team or even a single developer.

<strong>Morphous Source Code</strong>

While most of the Morphous API code is contained in a module, some changes to the Orchard core were required too so you will need both the Morphous.Api module and the Morphous.Orchard fork of OrchardCMS. Morphous.Orchard is kept up to date with the offical Orchard repo, and the changes to it are minimal anyway. If you add the official Orchard repo as a remote you can easily pull in any changes yourself.

Instructions for getting setup can be found here:
<a href="https://github.com/Morphous/Morphous.Api" target="_blank">https://github.com/Morphous/Morphous.Api</a>


&nbsp;
<h3>Example - Newslapse</h3>

Let's go for something obvious like a News website and app and see how Orchard can be used to tackle the web service and website. We'll create a web service that serves up news "Articles" and we'll also provide a way to sort them into categories. If you want to follow along then get the Morphous source code above.

The Orchard admin panel looks a lot like WordPress, so there's a pretty good chance that whoever will be managing the content can get to grips with it no problem.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/1_new_orchard.jpg"><img class="alignnone wp-image-37 size-large" src="http://www.sylapse.com/wp-content/uploads/2015/07/1_new_orchard-1024x253.jpg" alt="1_new_orchard" width="765" height="189" /></a>

Orchard lets you build up custom Content Definitions through the UI of the website, which is pretty impressive. Content Definitions are things like Page, Blog Post, Comment or any other pieces of "content" you want to be able to edit and display on the website. In this new Orchard website go to the Content Definition section using the menu on the left. Create a new Content Definition and call it "Article". You then literally just tick the "Parts" you want it to have, give it a Title Part and a Body Part then click save.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/2_parts_list.jpg"><img class="alignnone size-full wp-image-38" src="http://www.sylapse.com/wp-content/uploads/2015/07/2_parts_list.jpg" alt="2_parts_list" width="843" height="269" /></a>

Once you've saved the content definition you can also add fields. Add a Media Library Picker Field and name it "Image". The CommonPart always gets added automatically and contains various meta data. There are some differences between parts and fields, you can have multiple fields of the same type on one content definition for example. It's possible to code your own parts and fields from scratch but this example uses Orchard built in ones.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/3_new_parts.jpg"><img class="alignnone size-large wp-image-39" src="http://www.sylapse.com/wp-content/uploads/2015/07/3_new_parts-1024x428.jpg" alt="3_new_parts" width="765" height="320" /></a>

Once we've created our Article content definition we see that we now have the ability to create new Articles.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/4_create_article.jpg"><img class="alignnone size-full wp-image-40" src="http://www.sylapse.com/wp-content/uploads/2015/07/4_create_article.jpg" alt="4_create_article" width="237" height="247" /></a>

Before we create any Articles we'll add a way to assign Articles into news categories. This is also really easy using Orchard's Taxonomies feature. Go to the Taxonomies section using the menu on the left. Create a new Taxonomy and call it "Articles". Then add some "Terms" to the taxonomy. Terms are basically just categories and a Taxonomy is just a group of categories. So in this Taxonomy, I've added the following terms for categorizing news articles.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/5_taxonomy.jpg"><img class="alignnone size-large wp-image-41" src="http://www.sylapse.com/wp-content/uploads/2015/07/5_taxonomy-1024x349.jpg" alt="5_taxonomy" width="765" height="261" /></a>

Now we just require one little tweak to our Article content definition before we go ahead and actually create some Articles. We're going to add a Taxonomy Field and name it "Category", it allows us to link articles to relevant categories. Go back to the Content Definition section and edit the Article definition so that it looks like this:

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/6_article_with_taxonomy.jpg"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/6_article_with_taxonomy-1024x537.jpg" alt="6_article_with_taxonomy" width="765" height="401" class="alignnone size-large wp-image-230" /></a>

Now we're ready to create some Articles and assign them to the various categories as we go. Create a new Article by clicking in the box at the top left of the Orchard admin screen. I'll copy and paste some in from a real news website for this.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/main_article-1.jpg" rel="attachment wp-att-497"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/main_article-1-1024x752.jpg" alt="main_article" width="765" height="562" class="alignnone size-large wp-image-497" /></a>

I'll add in a few more just like that. Once we've got some articles, we're ready to start using our web service. Now we'll use Postman for Chrome or a similar REST client to hit our API.

&nbsp;
<h3>Web Service</h3>

We're about to use the Morphous API. While Orchard naturally renders content items as HTML for your website, this custom feature just serializes the same content items as JSON instead. The JSON is then served up using WebAPI.

This is how you get a content item by its ID. Whenever you edit a piece of content through the Orchard admin panel, you can see its ID up in the browser URL bar. So like that, I found the ID for the Taxonomy we created which contains a list of Terms (categories). Don't forget the headers. Alternates means the same thing that it does usually in Orchard, we can use it to tweak the format of the JSON but we'll look at it properly in a later post. So lets get that Taxonomy, the ID in this case was 12.

<em><strong>Taxonomy</strong></em>

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/8_get_taxonomy.jpg" rel="attachment wp-att-495"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/8_get_taxonomy.jpg" alt="8_get_taxonomy" width="808" height="852" class="alignnone size-full wp-image-495" /></a>

As you can see it's returned our Taxonomy along with its list of Terms. We can use those terms and their IDs to drill down further. So lets get another content item, the "Top Stories" term, in the same way that we got the whole Taxonomy. They're all just content items, even the ones that contain a list of child content items. We can see that the Top Stories ID is 13.

<em><strong>Term</strong></em>

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/9_get_term.jpg" rel="attachment wp-att-494"><img src="http://www.sylapse.com/wp-content/uploads/2015/07/9_get_term.jpg" alt="9_get_term" width="818" height="811" class="alignnone size-full wp-image-494" /></a>

Orchard has an option called DisplayType where content items can be displayed as either Detail or Summary, you can see it in the JSON for each item and child item. Summary mode just omits certain parts or fields compared with Detail. Child content items are shown as Summary and top level items are shown as Detail. That's why requesting a Term directly now shows a list of Articles, and the Articles themselves are being shown in Summary mode. Morphous API matches pretty much all the features and conventions of the normal front end display process in Orchard. Such as DisplayType, Placement.info and use of Alternates. If you want to request a content item in summary mode directly just add <code>?displayType=Summary</code> to the url. We'll stick to the basics in this post and cover that other stuff later but basically you can control what parts/fields show up in Detail/Summary mode and things like that.

Lets drill down further into an individual Article and see its Detail equivalent. Taking the first one which has an ID of 25.

<em><strong>Article</strong></em>

<a href="http://www.sylapse.com/wp-content/uploads/2016/08/10_get_article.jpg" rel="attachment wp-att-491"><img src="http://www.sylapse.com/wp-content/uploads/2016/08/10_get_article.jpg" alt="10_get_article" width="821" height="733" class="alignnone size-full wp-image-491" /></a>

The JSON for this Article contains an Image property which is coming from the Media Library Picker Field, which was hidden in Summary mode. It has the image URL which we could use to load the image in our planned news app, plus some other information about the image.

This is all starting to look very promising. Infact there's enough there already to make a pretty functional news app. Not only with the capability of downloading new Articles but even the ability to download the Categories themselves. So there's no need to hard code the categories into the app because you can get them from the web service. You can build a nice dynamic UI and then you can add or remove categories and articles just by playing around with the content in the Orchard admin area.

&nbsp;
<h3>Website</h3>
Let's do one last thing before concluding. So far we've been totally focused on building up a useful web service to be consumed by a mobile app. Now we can see how far we've come in terms of building our website as well, because I claimed that by setting up our data structures, we would naturally make some inroads into building the website.

All I'm going to do is add the Taxonomy that we created to the website's navigation bar. To do this you go to the Navigation section in the Orchard admin area and then add a Taxonomy Link which you can see at the bottom right. Then just choose the correct Taxonomy, we only created one, and save.

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/11_navigation.jpg"><img class="alignnone size-large wp-image-47" src="http://www.sylapse.com/wp-content/uploads/2015/07/11_navigation-1024x465.jpg" alt="11_navigation" width="765" height="347" /></a>

Once that's done we can navigate to the home page of the website. We can see that the Terms (categories) in the Taxonomy are all now links in the navigation bar. This is a little bit like the first API call we made when we hit the whole Taxonomy, only we've swapped JSON for HTML.

<em><strong>Taxonomy</strong></em>

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/12_render_taxonomy.jpg"><img class="alignnone size-large wp-image-48" src="http://www.sylapse.com/wp-content/uploads/2015/07/12_render_taxonomy-1024x192.jpg" alt="12_render_taxonomy" width="765" height="143" /></a>

Clicking on Top Stories is similar to our second API call to a specific Term. Notice that the images aren't present for this list of summaries, it's the exact same data we saw in JSON form earlier. Again worth noting that we can easily customize things like this in Orchard using Placement.info and template overrides and if we want we can target the API or the HTML output independently of eachother.

<em><strong>Term</strong></em>

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/13_render_term.jpg"><img class="alignnone wp-image-49 size-full" src="http://www.sylapse.com/wp-content/uploads/2015/07/13_render_term.jpg" alt="13_render_term" width="987" height="551" /></a>

Clicking one of the "more" links takes us through to the whole article. Unsurprisingly, this is the equivalent to our third API call to a specific Article. It shows us the whole Article with the image and everything else. Orchard shows quite a lot of meta data by default when rendering, it's another case of using the Placement.info file or template overrides to hide stuff like that and to add all of your own styling by modifying the HTML/CSS.

<em><strong>Article</strong></em>

<a href="http://www.sylapse.com/wp-content/uploads/2015/07/14_render_article.jpg"><img class="alignnone size-full wp-image-50" src="http://www.sylapse.com/wp-content/uploads/2015/07/14_render_article.jpg" alt="14_render_article" width="989" height="565" /></a>

So all of the data we've been building up was always available through the front end of the website. That is what a CMS is for after all. So at this point we basically have a working web service and website, both serving up the same data which can all be easily managed through the Orchard admin panel. Not bad considering we haven't written a single line of code yet and the whole thing took a matter of minutes!

We're only a CSS file and a Placement.info file away from making the website look respectable. Then we could come up with a simple mobile app (or any other form of client) to consume the API and display the data.

<h3>Conclusion</h3>
I hope this has been a compelling example and has shown the Morphous project and OrchardCMS to be a worthy choice for building web services. The next few blog posts will cover topics such as:
<ul>
	<li>Placement, Alternates, DisplayType etc</li>
	<li>Dealing with non success results</li>
	<li>Client app examples using Xamarin (for iOS and Android)</li>
	<li>User accounts and token based authentication</li>
	<li>POSTing content from clients</li>
</ul>## A New Post

Enter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
