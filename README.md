# PageSpeed Tutorial
### How to get the fastest sites in the West

This is a tutorial on how to build sites to acheive the best [Pagespeed Insights](https://developers.google.com/speed/pagespeed/insights/) scores. A lot of this is specific to the [Volcanic](https://www.volcanic.co.uk) platform, but will apply to other sites aswell. I have attempted to explain *why* each section is important and how it helps improve the score.

The following points can be completed in any order, and each step will help the process.

## Key Points

These are the key points to consider that we will be optimising:
* Prevent stylesheet load blocking the page render
* Prevent javascript load from blocking page render
* Reduce file sizes to reduce time before render starts
* Optimise images to prevent wasted bandwidth
* Leverage client side caching to the best of our ability
* MOBILE - Achieve 100/100 UX scores


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


