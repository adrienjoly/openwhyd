---
title: Let's turn an Openwhyd playlist into a static Jekyll site
published: true
description: "In this tutorial, I explain how to get rid of a back-end (i.e. web server + database): by downloading the data in JSON format and writing a Jekyll template that will be used to render web pages from this data."
tags: tutorial, jekyll, music, youtube
---

Today I'm giving a coding challenge to myself:
Make [my openwhyd.org profile](https://openwhyd.org/adrien) playable from a pure client-side HTML page (i.e. without API/back-end server), using Jekyll.

# Why?

Openwhyd is a platform on which music lovers collect songs that they've found online, on Youtube, Soundcloud, Bandcamp and several other streaming platforms. It allows them to make cross-source playlists, to play them from their laptop or iPhone, and to share their discoveries with other music lovers who subscribed to them.

![screenshot of my openwhyd profile](https://thepracticaldev.s3.amazonaws.com/i/l8kz1srasn617in3brht.png)

Behind the scenes, Openwhyd is a web application powered by Node.js and MongoDB. This means that we have to pay our hosting provider a monthly bill to keep it running on a server. (FYI, this costs $15/month)

So far, we have been lucky enough to be sponsored by DigitalOcean: they gave us 1 year of free hosting! But this gift will not last forever, we will soon have to ask users again for donations, in order to pay the hosting bills. Which is not a fun thing to do, for us, nor for our users. (see [our OpenCollective page]())

This situation got me thinking: **would it be possible to keep using Openwhyd without having to pay for hosting?**

As I'm a big fan of GitHub Pages, the first solution that came to mind was to get rid of the back-end part of Openwhyd. Turning it into a pure client-side web app (a.k.a. static web app), in which the database would be replaced by data files rendered with Jekyll.

Of course, it would be very complicated to cover all the features of Openwhyd without using a back-end. (e.g. adding tracks to playlists from the web UI, and fetching the stream of latest tracks from users you subscribed to)

But I figured that just being able to play your own playlists would be a good compromise between utility for users and technical complexity to make it work.

If I get this to work, it would allow some Openwhyd users to still be able to play the tracks that they spent years collecting, even if our back-end were to go offline some day.

# Design of the technical solution

Here's what currently happens when a user wants to play tracks from their profile, or any of their playlists:
1. They open a browser tab to the corresponding URL; (e.g. https://openwhyd.org/adrien)
2. The back-end fetches the 20 latest tracks of that playlist and renders a HTML page containing that list of tracks;
3. The user's browser displays the page and loads a bunch of JavaScript files, notably `WhydPlayer.js` that is responsible to making any listed track playable;
4. When the user clicks on one of the tracks, `WhydPlayer.js` extracts the source of that track (stored in the DOM node corresponding to that track) and streams it from there;
5. When reaching the end of the track being currently played back, `WhydPlayer.js` plays the following track; (still by extracting data from the DOM)
6. When reaching the last track of the page, `WhydPlayer.js` asks the back-end for the next page of tracks and appends them to the list.

Based on this (simplified) flow, we can notice that the back-end is only solicited on steps 2 and 6: to fetch and render tracks from the playlists. The rest is done by `WhydPlayer.js` and other client-side logic.

Step 2 could easily be done on a static web hosting provider like GitHub Pages: we would just have to store the user's playlists as HTML pages. The `WhydPlayer.js` would still work from it, as long as the DOM structure contains all the data that it needs to stream the music of each track.

Step 6 could be worked around: if HTML pages contain the full list of tracks, there would be no need to fetch pages of next tracks!

Now, let's propose an action plan:
1. Clone Openwhyd's repository, and get rid of its back-end files;
2. Create a HTML page that holds one track, and make it playable with `WhydPlayer.js`;
3. Download the track listing of an Openwhyd profile (or playlist);
4. Turn the HTML page into a Jekyll template that will render the track list;
5. Patch the UI, so that the user can navigate between their profile and playlist pages.

Constraint: I'm going to do my best to apply as few alterations as I can to Openwhyd's codebase (including its file structure), in order to these steps reproducible on its future versions too.

> Note: I'm writing this article while following these steps.


# Step 1. Clone (and clean) Openwhyd's repository

Let's head to Openwhyd's official open source repository: [github.com/openwhyd/openwhyd](https://github.com/openwhyd/openwhyd).

From there, git noobs will be happy to download it as a `zip` file, thanks to the "Clone and download" button. The pros will use the command line:

```bash
$ git clone https://github.com/openwhyd/openwhyd.git
$ cd openwhyd
```

You'll see several folders and files in the resulting `openwhyd` directory. Let's get rid of everything that is not relevant for our static version of Openwhyd:

```bash
$ rm Dockerfile
$ rm docker-compose.yml
$ rm -r scripts
$ rm -r whydMisc
$ rm -r whydJS/test
$ rm -r whydJS/config
$ rm -r whydJS/screenshots
$ rm -r whydJS/app/controllers
$ rm -r whydJS/app/data
$ rm -r whydJS/app/emails
$ rm -r whydJS/app/lib
$ rm -r whydJS/app/models
$ rm -r whydJS/app/workers
$ rm -r whydJS/app/views
$ rm whydJS/app/*.*

# delete all files from whydJS/, except package.json
$ find whydJS/* -type f -maxdepth 0 | egrep -v "package\.json" | xargs rm

# delete all folders from public/, except a few ones
$ find whydJS/public/* -maxdepth 0 | egrep -v "css|fonts|html|images|js|swf" | xargs rm -r

# delete all folders from images/
$ find whydJS/public/images/* -type d -maxdepth 0 | xargs rm -r

# delete all files from public/css, except a few ones
$ find whydJS/public/css/* -maxdepth 0 | egrep -v "[^-]common|userPlaylistV2|userProfileV2" | xargs rm

# delete public/html/*, except a few ones
$ find whydJS/public/html/* -maxdepth 0 | egrep -v "channel|\bYoutubePlayerIframe" | xargs rm -r

# delete public/js/*, except a few ones
$ find whydJS/public/js/* -maxdepth 0 | egrep -v "autoresize|\bjquery|mustache|playem|soundmanager|swfobject|ui|whyd\.js|Player\.js" | xargs rm -r

# delete whydJS/app/templates/*, except a few ones
$ find whydJS/app/templates/* -maxdepth 0 | egrep -v "feed\.html|mainTemplate|posts\.html|userPlaylistV2\.html|userProfileV2\.html|whydPlayer" | xargs rm -r

# let's see the tree of remaining folders
$ ls -R | grep ":$" | sed -e 's/:$//' -e 's/[^-][^\/]*\//--/g' -e 's/^/   /' -e 's/-/|/'
   |-docs
   |---img
   |-whydJS
   |---app
   |-----templates
   |---public
   |-----css
   |-----fonts
   |-----html
   |-----images
   |-----js
   |-----swf
```

Ok, now we have much fewer files to distract us from our mission!


## Step 2. Create a HTML page with a playable track

Let's try a naive move: save a playlist page from openwhyd.org to a HTML file and see if we can play a track from it.

For that, I'm gonna open [my profile page](https://openwhyd.org/adrien) on Google Chrome, and press `Cmd-S` to save it as a "Simple HTML file":

![saving my openwhyd profile page into a HTML file](https://thepracticaldev.s3.amazonaws.com/i/m5sliwng4hwy08mpcdm6.png)

Of course, if I try to open that download file in my browser, it looks completely broken and it's impossible to play tracks from it. That is because all the references to JavaScript, CSS and image files are broken.

Extract from the resulting HTML file:

```html
    <title>Adrien Joly's tracks - Openwhyd</title>
    <link href="/css/common.css?1.1.1" rel="stylesheet" type="text/css" />
    <link href="/css/browse.css?1.1.1" rel="stylesheet" type="text/css" />
    <link href="/css/tipsy.css?1.1.1" rel="stylesheet" type="text/css" />
    <link href="/css/userProfileV2.css?1.1.1" rel="stylesheet" type="text/css" />
    <link href="/css/userPlaylistV2.css?1.1.1" rel="stylesheet" type="text/css" />
    <link href="/css/dlgEditProfileCover.css?1.1.1" rel="stylesheet" type="text/css" />
    <script src="/js/jquery-1.10.2.min.js"></script>
    <script src="/js/jquery-migrate-1.2.1.js"></script>
    <script type="text/javascript" src="/js/jquery.history.js"></script>
    <script src="/js/soundmanager2-nodebug-jsmin.js"></script>
```

As you can see, that page uses absolute paths to all its assets. So we're gonna have to setup a web server locally that will give access to these resources from these paths.

In order to respect that file structure, it would make sense to move the HTML file in the `whydJS/public/` folder. That way, we can consider that is folder is gonna be the root of our web server, making both our HTML file and the asset subfolders accessible through it.

So I'm running the following commands:

```bash
# (assuming that you are still in the openwhyd folder we created earlier)
$ cd ./whydJS/public
$ move ~/Downloads/my-openwhyd-profile-playlist.html 
$ python -m SimpleHTTPServer # to start python's http server
Serving HTTP on 0.0.0.0 port 8000 ...
```

Now, let's open `http://0.0.0.0:8000/my-openwhyd-profile-playlist.html` in Google Chrome:

![screenshot of the HTML page of my openwhyd profile after opening it from my local server](https://thepracticaldev.s3.amazonaws.com/i/5m42tfhar8yuq293yq9k.png)

As you can see, some elements of the UI still look broken, but the layout is working and you may even be able to play some of the tracks by clicking on them! 

...Except Youtube tracks, because Openwhyd's Youtube player was patched to play tracks through another domain name. In our case, we don't want this behaviour, so let's make the patch script think that we're running locally (even if we're not):

```
# let's force `var isLocal = true;`, in whydJS/public/js/playem-youtube-iframe-patch.js
$ sed -i -e "s/var isLocal = /var isLocal = true; /" ./whydJS/public/js/playem-youtube-iframe-patch.js
```

This should make it work better, after refreshing the page!


# 3. Store the track listing into a data file

So far, we have a static HTML page in which the tracks are hard-coded. As we will want to turn that page into a template that will be able to render any playlist, let's download a playlist in raw format, without the HTML rendering.

Fortunately, Openwhyd makes it easy to download a raw track list in JSON format, from any page. We just have to add the `?format=json` suffix to the URL of that page.

In my case, I want the track list of my profile, so I am going to download the JSON content of `https://openwhyd.org/adrien?format=json` and save it to `openwhyd/whydJS/public/_data/tracks.json`.

The contents of the file should look like this:

```json
[{"_id":"5b4b6b81314fcb64d074e462","uId":"4d94501d1f78ac091dbc9b4d","uNm":"Adrien Joly","text":"","pl":{"name":"classical & epic","id":1},"name":"Max Richter - November","eId":"/yt/2Bb0k9HgQxc","ctx":"bk","img":"https://i.ytimg.com/vi/2Bb0k9HgQxc/default.jpg","src":{"id":"https://totoromoon.wordpress.com/2018/07/13/max-richter-november-sarajevo/","name":"MAX RICHTER November &amp; Sarajevo | totoromoon"},"nbP":8,"nbR":1,"lov":[]},{"_id":"5b34c9731db44a36a795f96a","uId":"4d94501d1f78ac091dbc9b4d","uNm":"Adrien Joly","text":"","pl":{"name":"epic coding session soundtrack","id":53},"name":"alexanderbrandon - sunrise on vanaar","eId":"/bc/alexanderbrandon/sunrise-on-vanaar#https://t4.bcbits.com/stream/6b5b84ad5900cbb22b532a778f63701d/mp3-128/3200645745?p=0&ts=1530272235&t=01ec41132994dcb46b2a5f5c213a53c39cddb5dc&token=1530272235_39982de236fdfd2e66cf14379f8ba69b44f02f51","ctx":"bk","img":"//s0.bcbits.com/img/bclogo.png","src":{"id":"https://alexanderbrandon.bandcamp.com/album/aven-colony-original-game-soundtrack","name":"▶︎ Aven Colony (Original Game Soundtrack) | Alexander Brandon"},"lov":[]},{"_id":"5b326e261db44a36a795f844","uId":"4d94501d1f78ac091dbc9b4d","uNm":"Adrien Joly","text":"smiling band from cuba, fantastic female drummer !","pl":{"name":"Jazz meets World","id":61},"name":"Yissy García & Bandancha: NPR Music Tiny Desk Concert","eId":"/yt/Y1Kv_sFZHqI","ctx":"bk","img":"https://i.ytimg.com/vi/Y1Kv_sFZHqI/default.jpg","src":{"id":"https://www.youtube.com/watch?v=Y1Kv_sFZHqI&feature=youtu.beq","name":"Yissy García &amp; Bandancha: NPR Music Tiny Desk Concert - YouTube"},"nbP":9,"lov":[]},{"_id":"5b2e34f51db44a36a795f4ff","uId":"4d94501d1f78ac091dbc9b4d","uNm":"Adrien Joly","text":"","pl":{"id":2,"name":"rock","collabId":null},"name":"KADAVAR - Come Back Life (OFFICIAL MUSIC VIDEO)","eId":"/yt/4xgi91s7zf8","img":"https://i.ytimg.com/vi/4xgi91s7zf8/0.jpg","repost":{"pId":"520de33ae4d15ab75a0030f5","uId":"51e92a777e91c862b2af478a","uNm":"Thomas Radioyes"},"lov":[],"nbR":0,"nbP":6},{"_id":"5b2b72c11db44a36a795f361","uId":"4d94501d1f78ac091dbc9b4d","uNm":"Adrien Joly","text":"chiptune","pl":{"name":"epic coding session soundtrack","id":53},"name":"Ujico*/Snail's House - Hello [Ordinary Songs
[...]
```

Feel free to reformat this JSON file using [JSON Pretty Print](https://jsonformatter.org/json-pretty-print), or to convert it to YAML using [json2yaml.com](https://www.json2yaml.com/) if you want, but this is not required for the Jekyll template to work.

> Note: if you don't remember your Openwhyd username, you can also use the URL `https://openwhyd.org/me?format=json`.


# 4. Turn the HTML page into a Jekyll template

In step 2, we had made Openwhyd's player work on the list of tracks that were dumped in a static HTML file. Now, let's turn this HTML file into a template, so that the list of tracks can be loaded from our JSON file. (like we had done in my previous tutorial: [How to maintain a collection of music albums online, using Jekyll and Github Pages](https://dev.to/adrienjoly/how-to-maintain-a-collection-of-music-albums-online-using-jekyll-and-github-pages-3hd6))

Basically, what we want to do is to delete the elements that represent tracks from our HTML file, and replace them by a Jekyll template.

In Openwhyd, tracks are displayed using `<div>` elements with the `post` classname:

```bash
# (assuming that you are in the openwhyd/whydJS/public, where your HTML file is)
$ grep '<div class="post' *.html

                <div class="posts">
<div class="post " data-pid="5b546297314fcb64d074e8d3"
<div class="post " data-pid="5b4b6b81314fcb64d074e462"
<div class="post " data-pid="5b34c9731db44a36a795f96a"
[...]
```

First, using our favorite code editor, let's cut and paste all these `<div>` elements into a separate HTML file (e.g. `posts.html`), so that we can use this HTML code as an example when we'll write our Jekyll template.

![moving post divs from the playlist's html file to an external file](https://thepracticaldev.s3.amazonaws.com/i/7nb38i0vsm1o4fnzkpls.png)

After doing this operation, your playlist's HTML file should contain a `<div class="posts">` element that just contains the `<!-- TRACKS TAB CONTENT -->` comment. That's where we are now going to add our Jekyll template.

In order to make it easier to build that template, let's copy the code of one of the posts that we moved out of our HTML file, except its child nodes that have the classes `ago`, `stats`, `btns` and `ext`, back under the `<div class="posts">` element.

After re-formating the HTML, it should look like this:

```html
<div class="posts">

  <!-- TRACKS TAB CONTENT -->
  
  <div class="post "
        data-pid="5b546297314fcb64d074e8d3"
        data-initialpid="5b540c4d314fcb64d074e891" 
        data-time="1532256919000" >

    <div class="playBar"></div>

    <a class="thumb"
        href="//youtube.com/watch?v=u8rVPIV-Y4Y"
        target="_blank"
        data-eid="/yt/u8rVPIV-Y4Y"
        onclick="return playTrack(this);"
        style="background-image:url('https://i.ytimg.com/vi/u8rVPIV-Y4Y/default.jpg');">
      <img src="https://i.ytimg.com/vi/u8rVPIV-Y4Y/default.jpg">
      <div class="play"></div>
    </a>

    <h2>
      <a href="/c/5b546297314fcb64d074e8d3"
          class="no-ajaxy"
          onclick="toggleComments('5b546297314fcb64d074e8d3');return false;">
          Paraphon Tree - Firefly (Official Music Video)</a>
    </h2>

    <p class="author">
      <span style="background-image:url('/img/u/4d94501d1f78ac091dbc9b4d');"></span>
      <a href="/u/4d94501d1f78ac091dbc9b4d">Adrien Joly</a> added this track
      to <a href="/u/4d94501d1f78ac091dbc9b4d/playlist/12">post-rock / progressive</a>
      via <a href="/u/544c39c3e04b7b4fca803438">Stefanos Mavrogenis</a>
    </p>
  </div>

</div>
```

Now, let's turn this sample post into a Jekyll template that can be used to render any track.

In order to do that, we replace all the hard-coded values by *mustache* placeholders that contain the path of that value in any track from our `tracks.json` file.

For example, I replace the following HTML code:

```html
<a href="/u/4d94501d1f78ac091dbc9b4d">Adrien Joly</a> added this track
to <a href="/u/4d94501d1f78ac091dbc9b4d/playlist/12">post-rock / progressive</a>
```

... by this template code:

```html
<a href="/profile">{{ track.uNm }}</a> added this track
to <a href="/playlist-{{ track.pl.id }}">{{ track.pl.name }}</a>
```

It should end up like this:

```html
<div class="posts">

  <!-- TRACKS TAB CONTENT -->

  {% for track in site.data.tracks %}
  <div class="post"
        data-pid="{{ track._id }}"
        data-initialpid="{{ track.repost.pId }}">

    <div class="playBar"></div>

    <a class="thumb"
        href="#"
        data-eid="{{ track.eId }}"
        onclick="return playTrack(this);"
        style="background-image:url('{{ track.img }}');">
      <img src="{{ track.img }}">
      <div class="play"></div>
    </a>

    <h2>
      <a href="#">{{ track.name }}</a>
    </h2>

    <p class="author">
      <span style="background-image:url('{{ user.img }}');"></span>
      <a href="/profile">{{ track.uNm }}</a> added this track
      to <a href="/playlist-{{ track.pl.id }}">{{ track.pl.name }}</a>
      via <a href="https://openwhyd.org/u/{{ track.repost.uId }}">{{ track.repost.uNm  }}</a>
    </p>
  </div>
  {% endfor %}

</div>
```

In order to test the rendering of this template, the `python -m SimpleHTTPServer` that we ran above will not help. So let's stop it (press `Ctrl-C`) and use Jekyll instead.

Here is how I set Jekyll up from my `openwhyd/whydJS/public/` folder:

```bash
# let's make sure we have ruby installed
$ ruby -v
ruby 2.3.7p456 (2018-03-28 revision 63024) [x86_64-darwin15]

# let's install jekyll in openwhyd/whydJS/public/
$ bundle init
$ bundle install --path vendor/bundle
$ bundle add jekyll
$ bundle install

# this command installs jekyll in ./vendor/bundle => let's git-ignore it (optional)
$ echo /whydJS/public/vendor >>../../.gitignore
$ echo /whydJS/public/_site >>../../.gitignore
```

When it's fully set up, we can run the Jekyll server by typing this command:

```bash
$ bundle exec jekyll serve
    Server address: http://127.0.0.1:4000
  Server running... press ctrl-c to stop.
```

Let's open `http://127.0.0.1:4000/my-openwhyd-profile-playlist.html`.

![screenshot of the Jekyll template failing to render](https://thepracticaldev.s3.amazonaws.com/i/dff1qzrwhpfr374d26j8.png)

The good news is that our page is properly served by Jekyll. The bad news is that our template failed to render. It looks like Jekyll did not interpret the template from the HTML file.

I looked up online to see what's Jekyll process (and criteria) to render templates, and found this:

> Jekyll looks for files with front matter tags (the two sets of dashed lines --- like those in index.md) and processes the files (populating site variables, rendering any Liquid, and converting Markdown to HTML).

(source: [Convert an HTML site to Jekyll | Jekyll](https://jekyllrb.com/tutorials/convert-site-to-jekyll/))

So let's add two sets of dashed lines at the very top of our HTML file:

```html
---
---
<!DOCTYPE html>
<html lang="en">
[...]
```

Normally, the `jekyll serve` command should have detected this modification, and the track list should be rendered correctly from the JSON file, when opening/refreshing `http://127.0.0.1:4000/my-openwhyd-profile-playlist.html` in your web browser!

![jekyll is rendering the track list from our JSON file](https://thepracticaldev.s3.amazonaws.com/i/rkfj96tx0uikhlugsytu.png)

Great job!


# Next steps

I'm now done with my objective of the day: we turned an Openwhyd profile into a static Jekyll site. The list of tracks was downloaded in JSON format and used by Jekyll to render it into HTML, following the template that we wrote.

The next step will be to fix the navigation, so that visitors of the static version of my Openwhyd profile can also browse my playlists, following the same principle.

In order to do that, I intend to use [Jekyll collections](https://jekyllrb.com/docs/collections).

**Until I get back to writing the sequel to this article, let me know in the comments if you have alternative ideas on how to make this work!**
