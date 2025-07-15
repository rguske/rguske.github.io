# Youtube Hugo-Shortcode workaround - when {{< youtube >}} won´t work


Inspired by a recent post from my friend, <a href="https://gohugo.io/" target="_blank">(Go)Hugo</a> and Git mentor <a href="https://twitter.com/embano1" target="_blank">Michael Gasch</a> :wink:, where he describes how he fixed the Hugo <a href="https://gohugo.io/content-management/shortcodes/" target="_blank">Shortcode</a> for Twitter to suppress the default thread view, it came to my mind immediately that I faced a similar issue when I started blogging with the Shortcode which embeds a Youtube-Video into your post.

<!--more-->

See how Michael fixed the normal behavior for the Twitter Shortcode `{{</* tweet */>}}` here...

<center> {{< tweet user="embano1" id="1071143487528206336" >}} </center>

...and add it to your Shortcode folder to make advantages from.

## *When {{< youtube xxxxxxxxxx >}} won´t work for your theme*

{{< admonition warning "Attention" true >}}
*This applies only to the `singleViewStyle = "casper"` configuration in your config.toml.*

*With `singleViewStyle = "caspertwo"` the shortcode {{< youtube xxxxxxxxxx >}} works.*
{{< /admonition >}}

Besides embedding a tweet into your post e.g. to give a statement more expression/ more emphasis, or to comment a topic related tweet, or simply because it´s pretty cool, a Youtube-Video is also a good alternative.

I´m running the <a href="https://themes.gohugo.io/hugo-casper-two/" target="_blank">*Casper-Two*</a> Theme for my Blog and back when I wrote <a href="https://rguske.github.io/post/sometimes-you-gotta-run-before-you-can-walk/" target="_blank">"*Sometimes you gotta run before you can walk*"</a>, I absolutely wanted to have the <a href="https://www.youtube.com/watch?time_continue=2&v=1VrHeInDwwg" target="_blank">*IRON MAN I First Armor test*</a> Video included and it didn´t work straight from the start.

### Let me show you what I mean

I´ll add the video <a href="https://www.youtube.com/watch?v=2xkNJL4gJ9E&list=PLLAZ4kZ9dFpOnyRlyS-liKL5ReHDcj4G3&index=9" target="_blank">"Shortcodes | Hugo - Static Site Generator | Tutorial 9"</a> here at this point of the post by using `{{</* youtube 2xkNJL4gJ9E */>}}` in my post `*.md` file. :point_down:

Normally you would see the mentioned Youtube video here :point_up: but it seems that the Shortcode won´t work for my chosen theme.

## You are not alone

The best thing to do first, from my point of view, is not to google the (mis-)behavior, but rather go to the Github-Repo from your theme and have a look at the *Issues*. If there exists one, comment...if not...raise one.

So I had a look at the issues for <a href="https://github.com/eueung/hugo-casper-two" target="_blank">Casper-Two</a> and indeed, there´s already <a href="https://github.com/eueung/hugo-casper-two/issues/5" target="_blank">one</a> open which covers this issue. I commented this as well and a really short period of time later, <a href="https://twitter.com/SalarRahmanian" target="_blank">Salar Rahmanian</a> had the code snippet for it.

This is simply great and one of thousands of things I´m really impressed with regarding Github and the Community.

## The Code

As Michael already described it in his post, we have to create a new empty file and this time we´ll call it `yt.html`.

```shell
cd <HUGO_BLOG_ROOT>
touch layouts/shortcodes/yt.html
```

Put the following lines of code into the `yt.html` file and save it.

```html
<iframe src="https://www.youtube.com/embed/{{ index .Params 0 }}?start={{ index .Params 1 }}"
        style="position: absolute; top: 0; left: 0; width: 560; height: 315;"
        allowfullscreen frameborder="0"
        title="YouTube Video">
</iframe>
```

This time I´ll make use of our new Shortcode `{{</* yt */>}}` instead of {{< youtube xxxxxxxxxx >}} *et voilà*...

{{< youtube 2xkNJL4gJ9E >}}

...here we go :thumbsup:. Thanks again Salar!

<center>{{< tweet user="vmw_rguske" id="1024709843712724993" >}}</center>

Getting more interest in building your own Shortcodes?

**Go here:** <a href="https://gohugo.io/templates/shortcode-templates/" target="_blank">Create Your Own Shortcodes</a>

> *"You can extend Hugo’s built-in shortcodes by creating your own using the same templating syntax as that for single and list pages."*

**<center>Thanks for reading.</center>**
