---
published: true
layout: post
title: Morphous Xamarin SDK for dynamic content
---
The Morphous SDK for Xamarin apps lets you add content to apps really easily, all using native platform UI. Create and manage content through a ready made CMS and the SDK can display it automatically. Then you just need to create a Theme to style the content in your app.

Watch the video for a overview of the Morphous SDK. It's a little bit long but the first 35 minutes give a full picture of how everything works and what it can do for you, the rest goes into the detail of creating a Theme to style your content in an Android app.

<center><iframe width="560" height="315" src="https://www.youtube.com/embed/ikcZk-GZvXM" frameborder="0" allowfullscreen></iframe></center>

&nbsp;
&nbsp;

There are three main steps to using the Morphous SDK

### 1. Create some content

Create and manage your content in Orchard CMS. Then add the Morphous modules which turn your content into JSON.

<center><a href="https://raw.githubusercontent.com/tom-pratt/tom-pratt.github.io/master/images/posts/morphousnativepost/web_screenshots_m.png" target="_blank"><img src="https://raw.githubusercontent.com/tom-pratt/tom-pratt.github.io/master/images/posts/morphousnativepost/web_screenshots_s.png" /></a></center>

&nbsp;

### 2. Add the Morphous SDK to your Xamarin app

To start showing content you just add the SDK to a Xamarin iOS or Xamarin Android project and point it at the URL of the CMS.

<center><a href="https://raw.githubusercontent.com/tom-pratt/tom-pratt.github.io/master/images/posts/morphousnativepost/app_unstyled_screenshots_m.png" target="_blank"><img src="https://raw.githubusercontent.com/tom-pratt/tom-pratt.github.io/master/images/posts/morphousnativepost/app_unstyled_screenshots_s.png" /></a></center>

&nbsp;

### 3. Create a Theme to style your content

Override .axml and .xib files and write any custom code needed to get your content styled and navigating the way you want it.

<center><a href="https://raw.githubusercontent.com/tom-pratt/tom-pratt.github.io/master/images/posts/morphousnativepost/app_styled_screenshots_m.png" target="_blank"><img src="https://raw.githubusercontent.com/tom-pratt/tom-pratt.github.io/master/images/posts/morphousnativepost/app_styled_screenshots_s.png" /></a></center>

&nbsp;

The example above uses Morphous to generate most of the UI. However, using the Morphous SDK allows you to show CMS content in your app exactly where you want it. Outside of the content views you're completely free to build your app however you want and would normally do so. Even within the content views you will see that you can easily override particular views and code to make visual changes or interact with things outside of content views.