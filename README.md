# PageSpeed Tutorial
### How to get the fastest sites in the West

This is a tutorial on how to build sites to acheive the best [Pagespeed Insights](https://developers.google.com/speed/pagespeed/insights/) scores. A lot of this is specific to the [Volcanic](https://www.volcanic.co.uk) platform, but will apply to other sites aswell. I have attempted to explain *why* each section is important and how it helps improve the score.

The following points can be completed in any order, and each step will help the process.

## Key Points

These are the key points to consider that we will be optimising:
* Prevent stylesheet load blocking the page render
* Prevent javascript load from blocking page render
* Replace `{% content_for_header %}`
* Reduce file sizes to reduce time before render starts
* Optimise images to prevent wasted bandwidth
* Leverage client side caching to the best of our ability
* MOBILE - Achieve 100/100 UX scores
* 3rd party javascripts


## Prevent Render Blocking - CSS

When using an external stylesheet, the browser has to do another lookup to fetch it before it can parse the CSS and start applying the rules to the DOM. This second lookup takes time, and on mobile data connections the initial HTTP handshake can take as much time as the file download (incidentally why the mobile score is often lower than desktop, it is tested on a throttled connection).

The Google recommended solution is to use `<style>` tags in the `<head>` to inline enough CSS for the above the fold content to be rendered, while you asynchronously load the rest of the CSS after the page has rendered. The idea is that the user is looking at the top of the page at the time the style "pops" in for the rest of the page, and they don't notice.

Currently, the platform does not have any support for a seperate "above the fold" stylesheet, but large gains can be achieved by inlining the entire stylesheet in the head.

Replace

```liquid
{{ 'stylesheets/application.css' | u_asset_url | u_stylesheet_tag }}
{{ 'stylesheets/application.css' | asset_url | stylesheet_tag }}
```
with
```liquid
{{ 'stylesheets/application.css' | inline_style_from_universal }}
{{ 'stylesheets/application.css' | inline_style_from_manifest }}
```
**Please note:** Currently, the inlining of CSS is a bit temperamental on DEV, so I would only swap to inlined style just before an export to staging/production. Using the seperate stylesheets in dev also allows you to take advantage of the development assets caching we have implemented to speed up dev page load times.

## Prevent Render Blocking - Javascript

Similar to CSS, any JS in the `<head>` blocks rendering. Any external files in the head have to have finished downloading before the browser will continue parsing and rendering the body. When you consider that the vast majority of javascript is attaching listeners to elements *after* the DOM is ready, holding up the ready process harms the entire process. The faster the DOM is ready the faster we can attach our listeners and do cool stuff.

There is an argument that a JS file is only truly not render blocking if it is explictly loaded asynchronously, or is loaded in a deferred manner by other inline JS. However, experiments have shown that loading the JS outside `<head>` and just before the closing `</body>` is sufficient to pass the rule in Pagespeed Insights.

```html
    .
    .
    .
    {{ 'javascripts/application.js' | u_asset_url | u_javascript_tag }}
    {{ 'javascripts/application.js' | asset_url | javascript_tag }}
  </body>
</html>
```

**WARNING:** This now means that jQuery is loaded at the END of the page. Read on to find out how this affects your code.

Although browsers wait for every external script to be loaded before any JS is executed, it is important to remember that all scripts **are executed in the order they appear in the DOM**. This means that you inline `<script>` up by the main nav will execute before theme's JS file is loaded. Libraries such as jQuery are now loaded at the very bottom of the DOM, so everyone's favourite function `$(document).ready()` is NOT going to work.

This does not mean that you cannot use jQuery at all, however. jQuery functions are perfectly safe to use as long as when they are called, jQuery is loaded and ready to go. Above I said that the majority of jQuery is used for attaching listeners and plugins to different DOM elements, and that this has to run when the DOM is ready to have listeners attached. You will know that most of the code you write sits inside `$(document).ready()`, a function that only runs the code inside when the DOM is ready. All we need to do is replace `$(document).ready()` (which won't work because no jQuery) with a native JS version.

```javascript
// This will work
document.addEventListener("DOMContentLoaded", function(event) { 
  $(".chosen").chosen();
});

// This will complain about missing jQuery
$(document).ready(function(){
  $(".chosen").chosen();
})
```

See how you can use jQuery inside the event listener? On the first pass of the javascript execution, all that runs is the vanilla JS that sets up functions to run when `DOMContentLoaded` fires. For that event to fire, the external JS which includes jQuery must have been downloaded and initialised, meaning that now `$` is defined and our code works as expected.

This removes the jQuery dependency from everything that happens before page load, making it safe to load at the bottom of the DOM. The load order of any plugins in the `application.js` file is unchanged, so all dependencies there behave the same as they always have. One slight problem is that the `DOMContentLoaded` event is not available in IE8, so if you need it there then you will have to fake it yourself. Good Luck.

## Replace `{% content_for_header %}`

`{% content_for_header %}` includes a lot of code that is the same for each site, or can be automated, such as Google Analytics. This includes `<meta>` elements as well as `<scripts>`. Now, we want all the `<meta>` elements to still be in the `<head>`, but we want all the javascript to be in the footer.

This means that the `{% content_for_header %}` has been split into two new tags to be used when the javascript is in the footer.

`{% header_meta_content %}` contains only header specific content, i.e. page title, metadescription, OpenGraph data etc. This can be put at the top of the `<head>`.

`{% platform_js %}` contains only `<script>` elements, and can be put after the other javascript, just before the `</body>` tag.

##Reduce File Sizes

Although Pagespeed Insights makes little mention of file size unless the html is huge, it is known that the page rank algoritm used in the search index does care about total page speed, and download time on mobile devices. Care needs to be made to ensure that unused javascript libraries are not just left in themes, or some copy and pasted css that doesn't even style any element on the page.

During experiments, stripping out some unused css and js gained 5 points on the mobile score, making the jump into the 85+ green band.

##Optimise Images

As an addition to general file size reduction, a specific mention needs to be made regarding images. Images are often the largest external resource loaded by a site; a single image can be larger than all the HTML, CSS and JS combined.

Any images that are in the theme's images folder are now compressed when the theme is deployed. The deploy server attempts lossless compression, but if at all possible it is better for the image to be resized and compressed using a tool like Photoshop before it is even added to the theme.

It is also crucial that images are loaded in the resolution needed. Some themes load an image with 1200x700 resolution, but once the css rules are applied it is displayed at 90x50 pixels. The browser will still have to download the huge image files before it can shrink them with css. For some sites a large image is needed for the responsive design, but on others the images keep a constant size on all devices and as such can be requested in the right resolution.

If you hover over an image in the inspector in Chrome, it gives you the current dimensions, as well as the 'Natural' resolution of the image file. These should ideally be identical.

##Leverage Client Side Caching

The majority of these changes involve adding cache headers to the requests server side, but it is worth knowing that some 3rd party resources, like tracking or chat widgets, often have missing or less than perfect cache settings. Adding a plugin at the request of a client can knock 10 points off the score. There is nothing you can do about this, but you can at least inform the client that the plugin they have insisted on is what is ruining their score, and not anything to do with our platform.


## Achieve 100/100 UX

Pagespeed Insights provides quite a good explanation on how to fix this, and most of our sites pass by default due to good design choices. The only common failure point is `<a>` tags getting too close to gether on mobile screen sizes which fails the tap target size rule. This can easily be fixed with some extra line height or padding when on mobile.

##3rd Party Resources

These were touched on in the 




