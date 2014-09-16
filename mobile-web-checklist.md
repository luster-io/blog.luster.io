  This is a list of I've been compiling over the last couple years while
building high-performance mobile frontends.  Some of these are broadly
applicable to any mobile website, some are specifically for people building
apps.

###Don't put touch interactions too close to edges of the screen.

  Unfortunately, mobile web apps are considered second class citizens.  There
are browser default touch interactions that you can't disable.  Swipe from left
back button in safari is the worst offender.  On android, swipe interactions
that start too close to the bottom will cause google now to activate.  When
you're designing your app, ensure   that there is enough margin that even the
fattest of fingers won't accidentally   trigger safari's awful swipe back
gesture.

##Remove the click delay

  When you tap something on the mobile web, there is a 300ms delay between when you tap and when the event fires.  This has been fixed on android 4.4+, but is still an issue on iOS.  The solution is to use a library like fastClick.

  FastClick does some magic to overwrite the default click events with ones that fire immediately when the user removes their finger from the screen.

###Better fixed header when input is focused.

    How much do you hate it when your fixed header unfixes itself any time
  someone brings up their onscreen keyboard? A lot? Me too.

    The solution is to not actually fix your header over top of the content.  You put the content in a `absolute` container with overflow scroll.

  And then put the header in a sibling container that is also `absolute`.  This
  way, scrolling the content doesn't actually affect the header at all since
  neither are fixed over top of each other.  This technique also works well with the next technique for Preventing Overscroll.

  WARNING: This will cause the double tap at the top of mobile safari to no
  longer scroll to the top of the page, since it's no longer the body scrolling.

  ###Prevent overscroll on the body.

  If you want your web app to feel app like, you have to get rid of the
  overscroll.  Overscroll is when you pull a scroll container past it's boundary.  When this happens on the body element, an ugly grey background appears and your entire viewport shifts.

  You can fix this really easily in cordova/phonegap with this simple xml config.

  ```
  <preference name="DisallowOverscroll" value="true" />
  ```

  It's also possible to fix this issue in plain javascript.

  ```
  module.exports = function(el) {
    el.addEventListener('touchstart', function() {
      var top = el.scrollTop
        , totalScroll = el.scrollHeight
        , currentScroll = top + el.offsetHeight

      //If we're at the top or the bottom of the containers
      //scroll, push up or down one pixel.
      //
      //this prevents the scroll from "passing through" to
      //the body.
      if(top === 0) {
        el.scrollTop = 1
      } else if(currentScroll === totalScroll) {
        el.scrollTop = top - 1
      }
    })

    el.addEventListener('touchmove', function(evt) {
      //if the content is actually scrollable, i.e. the content is long enough
      //that scrolling can occur
      if(el.offsetHeight < el.scrollHeight)
        evt._isScroller = true
    })
  }

  document.body.addEventListener('touchmove', function(evt) {
    //In this case, the default behavior is scrolling the body, which
    //would result in an overflow.  Since we don't want that, we preventDefault.
    if(!evt._isScroller) {
      evt.preventDefault()
    }
  })
  ```

  ##Make things that shouldn't be selectable, unselectable

    It's really annoying when an trying to interact with something causes
  something on the page to be selected.  Adding `user-select: none` to
  everything, except for the the things that a user would want to copy paste can
  cut way down on these interactions being accidentally triggered.

    Adding `-webkit-touch-callout: none;` to a thing prevents a tap and hold
  from open a context menu on the link or image.

  ```css
  user-select: none;
  -{prefix}-user-select: none;
  -webkit-touch-callout: none;
  ```

```javascript
if(navigator.userAgent.match(/Android/i))
  window.addEventListener('contextmenu', function (e) { e.preventDefault() })
```

  ##Always use momentum scrolling.

    Now that our scrolling is in a seperate container, it lost it's momentum!
  To get it back, we have to add `-webkit-overflow-scrolling: touch;` to the
  container with `overflow: scroll` on it.

    This gives the element momentum scrolling, instead of ending the scroll the
  moment the user takes their finger off the page.

  ##Get rid of ugly grey tap highlight.

    By default mobile web browsers have a tap highlight, so that users
  get feedback when they tap something.  Unfortunately it looks aweful,
  and is a dead giveaway that your app isn't native.

  The really easy solution is to add this to your css.  Since you NEVER
  want the default highlight.

  ```css
  * {
    -webkit-tap-highlight-color: rgba(0,0,0,0);
  }
  ```

  ##Use `:active` states

    So we got rid of the tap highlight color, so we need to give user's
  feedback when they tap something that's tappable.

    The easy way to do this is to add `:active` states to tappable things.

  ```
  button:active {
    /* some example */
  }
  ```

  ##Only animate transforms, opacity, and filter.

  This is a big one.  Animating properties like width, height, box-shadow, or
  anything that isn't `transform` `opacity` or `filter`.  Animating any other property
  will cause repaints and/or reflows.  Your desktop maybe be powerful enough
  to handle repainting your page as it changes 60 times per second, but your phone definitely isnt.

  ##translateZ or will-change to animated elements.

  To improve the performance of your animations, add `transform: translateZ` and/or `will-change: transform` or `will-change: opacity`  This lets the browser know that you're going to be animating this property and it should upload a layer to be composited on the GPU.

  ##Don't use jquery animate or fade.

  Jquery Animate uses a setInterval instead of requestAnimationFrame, and doesn't
  have good support for animating css transforms, which are bascically the only
  thing you can safely animate on mobile (except opacity). Just use CSS
  animations, transitions, Velocityjs and/or Impulse.

  ##Don't resize images clientside.

    When on a mobile device, or a high pixel density desktop display, or any
  display really, avoid resizing images on the client.

    The way it works is this.  The browser decodes the image from whatever format it's in (jpeg, png, whatever), into a bitmap, which it then resizes and caches.

    However, browsers have a limited cache size for resized images.  Once that cache fills up, the older ones get evacuated from the cache.  This means that as you're scrolling up and down the page,  you'll constantly run into images that are not in the cache.  These will have to be decoded and resized again on the fly.

    This on the fly resize will cause one of two things, either it will cause your scrolling to jank, or it'll cause the scrollable area on your mobile site to be white, while the mobile browser draws it in the background.  This isn't ideal, because your users won't actually see what their scrolling through, because it can be loaded fast enough. The solution is to serve and download images at the resolution they'll be displayed.

  ##Paint before you animate.

    Like we said previously, one of the keys to fast animations is to ensure that your animations aren't competing with repaints.  Repaints not caused directly by your animation are just as bad.

    Let's say a user clicks a button to go to another page, if we render (handlebars, react, angular, etc) the page and then immediately animate it in, the painting and the animating will happen concurently, and it'll jank. The solution is to defer the animation until after you've painted.  Luckily this is really easy to do.  First, append your new view to the DOM, but make sure it's off the screen or transparent.  Then, in a `requestAnimationFrame` call, do your animation.  That's it!  The requestAnimationFrame callback won't be called until the paint is done, and we're ready to animate.

    CAVEAT, if you're rendering a large page, your animation won't jank, but it may a long wait before the animation runs.  This is just because painting a lot of content takes a long time.  Generally when this is the case, I try to just paint everything "above the fold" before I animate the view in, and then start painting everything else in asyncronously once it's loaded.

  ###Provide your own navigation

    If a user adds your app to their homescreen, they won't have access to the back or forward buttons, so they need to be able to navigate your entire app without the use of the forward or back buttons.  Using the common native app metaphor of transitioning between views works really well.  You can use CSS animations to setup the transitions, and pushState to setup the back button functionality for each page.

  ###Test your back and forward buttons and linkability

    Even though your app provides it's own navigation, you can't just ignore the browser's default navigation. Not to mention, if you launch your app on HN, the top comments will revolve around the fact that your app breaks back button functionality.  Learn to use pushState, or find a framework that handles it for you.

  ###Hide with opacity or translate off screen if you plan to animate an element

    If you have an element that needs to be able to appear onscreen in a moments
  notice, don't just `display: none;` it.  If you do, it will have to be painted
  before it can appear.  So for things like hidden menus, it's best to hide them
  by setting `opacity` to `0` or by translating the item off screen (`translate3d(-9999px, 0, 0)`.  That way it's painted, loaded on the GPU, and ready to go, and won't have an initial jank, when you pull it onscreen.

  ###Don't do work in scroll or touch event handlers.

    You may be tempted to set style properties in event handlers.  These events
  happen much more often than you draw to the screen.  This means you're causing
  the browser to do extra work recalculating styles, when the result of those
  calculations will never be drawn.  It's best to put the result of the event
  somewhere, and then use that stored result in a requestAnimationFrame loop.
  That way you never do more style calculations than absolutely necessary.

    This is especially true of scroll events, since on mobile devices, scrolling
  is handled in a seperate thread from your javascript.  However, if you have
  scroll events, the scroll thread has to "wait" on the result of the scroll
  handler, since it may change the scrollTop, or call event.preventDefault(), in
  which case the browser would have to immediately stop scrolling.

  ###Don't use a custom scroll implementation unless you REALLY need to.

    But you shouldn't need to.

    Unless you're trying to do something crazy fancy like paralax.  Or you want
  things to whizz, spin, or spring in as the user scrolls down.  However, I've
  never seen entrance effects or paralax done well on mobile, so be prepared for
  it not to work out.  The danger is that not only do your neat effects jank,
  but your scrolling janks out along with them.

  There are a few reasons why it wouldn't work.  Even though you can build
  scrolling that feels very similar to native with something like Impulse, there
  are still problems with implementing your own scrolling.  One is that you're
  limiting the length of your content.  Mobile browsers do some crazy things to
  make scrolling smooth.  They only load some of the painted content, and
  asyncronously paint and load the content in.  If you do scrolling yourself, you
  don't get that.  Your scrollable area has to fit entirely into gpu memory.  If
  it doesn't you're going to have to paint portions of it as you scroll, and
  there is no way that's going to be smooth.


  ###Prevent user scaling.

    If you're building an app, you probably don't want to allow the user to
  arbitrarily zoom in and out.

  ```
  <meta name="viewport" content="width=device-width,height=device-height,user-scalable=no,initial-scale=1.0,maximum-scale=1.0,minimum-scale=1.0">
  ```

    If you use this meta tag, it will prevent the user from scaling, as well as
  scaling due to input focus and scaling due to device orientation.

  ##Hide chrome when your app is added to the homescreen.

    If someone likes your app enough to add it to their homescreen, why
  not make the experience even better for them.  You can remove the address bar,
  forward back button, and other miscellaneous browser controls with a few meta tags.

  ```html
  <!-- android -->
  <meta name="mobile-web-app-capable" content="yes">
  <!-- iOS -->
  <meta name="apple-mobile-web-app-capable" content="yes">
  <meta name="apple-mobile-web-app-status-bar-style" content="black">
  <meta name="apple-mobile-web-app-title" content="My App">
  ```

  ##Setup touch icons for the user's homescreen.


  ##Add to homescreen splash screen

  ```
  <link rel="apple-touch-startup-image" href="img/l/splash.png">
  ```

  ##Offline Cacheing

    If user's are going to add your app to homescreen, it better work when they don't have internet.  That's where app-cache comes in.

  ##Offline Data
    Saving some data in localStorage or indexDB can make your app much more
    useful when offline.  If your app has data that a user may want to access
    when they aren't online, store it.

  ## IE cleartype.

  <meta http-equiv="cleartype" content="on">
