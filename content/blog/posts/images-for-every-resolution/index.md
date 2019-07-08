---
title: Using images for every resolution
date: "2019-07-08T02:44:59.046Z"
description: Learn how browsers can choose the right image for different screen resolutions
---

A couple of days ago I needed to add an icon next to a text into a website I've been working on.

```html
<img src="really-big-icon.png" alt="" />
```

It was really easy and everything looks nice.

But then, a coworker mentioned that I was delivering a really big file for this small icon. Indeed, by the styles the icon size should be **25px height**, and the original icon was **150px**. So I went ahead, resize the icon an updated the URL

```html
<img src="icon-25.png" alt="" />
```

but this time, the icon looked plain bad on my screen. _—What's wrong?_ I wonder myself.

## High-density screens

After a bit of search, I realized I was using a high-density screen. In particular, I've been using a somewhat recently MacBook Pro that has a Retina display, which provides a really crispy image all the time —but not now.

The solution I found implies using the `srcset` attribute with a list of higher resolution images separated by commas, and let the browser choose what is best for the current screen. In my case, it looked like this

```html
<img src="icon-25.png" srcset="icon-25.png 1x, icon-50.png 2x" alt="" />
```

But I still didn't understand why should I provide an image with the double of the pixels. Pixels are pixels. Right?

## "CSS px"

So I kept reading and I found an interesting article [^1] where it states, when we talk about pixels in CSS, we refer to a special unit.

When we talk about `CSS px`, we are not talking about _screen pixels_.

`CSS px` are defined to be small but visible, and in contrast to inches or cm, **its size depends from where the user should see the screen**. So a CSS `1px` in a phone will be smaller than a CSS `1px` in a TV because the phone is a few centimeters from our face, but the TV can be a couple of meters away. Anyway, we should be able to see them in both places.

CSS provides this unit, `CSS px`, that allow us to work for all these different devices without having to worry on _how many screen pixels per inch my screen has_. **All that complexity is hidden behind CSS px**. And, of course, in the other CSS units that are `px` based like `em` and `rem`.

In my MacBook Pro screen, the relationship between _screen pixels_ and `CSS px` is `2x`. In order for images to look crisp, I need to provide a resource with the double of the size.

But how do I knew the relationship between _screen pixels_ and `CSS px`? Thankfully the browser exposes a property called `devicePixelRatio`. I opened DevTools and I typed

```javascript
window.devicePixelRatio // my result was "2"
```

This result depends on the screen the browser is being used. If you happen to have another connected screen, this result may vary. Regular desktop screens should return `1px`. The newest phones are returning `3px`.

## How many images should I provide?

This depends on the users of your website. If you need to support mobile device users with high-end phones, you should provide 3x, 2x and 1x versions of your assets.

## Is "srcset" supported by browsers?

This is important. At the time of writing this article, support for `srcset` attribute is pretty good [^2] —a good thing to notice is if we keep the `src` attribute **we will deliver an image to older browsers too**.

For this case, it makes sense to deliver the 1x URL in `src`. 

In my example, the code for my icon looks like this

```html
<img
    src="icon-25.png"
    srcset="icon-25.png 1x, icon-50px 2x, icon-75.png 3x"
    alt=""
    />
```

## Conclusion

Summarizing,

- Know about your users and which devices they are using
- Define which resolutions you should provide: 1x, 2x and/or 3x
- Provides the assets they need by using `srcset` attribute
- Deliver the experience for unsupported browsers by using `src` attribute
- If it makes sense, remember to add an alternative text using `alt` attribute

That's all for now. Have a good day!

[^1]: Bert Boss wrote an article for the W3C explaining the different units available in CSS, a bit of history, and which should we use depending on each situation https://www.w3.org/Style/Examples/007/units.en.html
[^2]: Worldwide support is over 90%. Anyway, you should understand who your users are, what devices are using, and provide what makes sense for your case https://caniuse.com/#search=srcset
