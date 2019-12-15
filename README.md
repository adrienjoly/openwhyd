# Openwhyd-Jekyll (fork of [Openwhyd.org](https://github.com/openwhyd/openwhyd))

> Discover, collect and play music from Youtube, Soundcloud, Bandcamp, Deezer and other streaming platforms.

Openwhyd-jekyll is a fork of [Openwhyd.org](https://github.com/openwhyd/openwhyd) which objective is to provide some of its features without requiring a back-end.

Read more: [Let's turn an Openwhyd playlist into a static Jekyll site](https://dev.to/adrienjoly/lets-turn-an-openwhyd-playlist-into-a-static-jekyll-site-341a)

## Features

- Renders a profile page from a JSON dump
- Allows to play tracks in sequence

## Development

### Status of the project

This is a side-project started for learning and experimentation purposes. It's work in progress.

### Current tech stack

- Jekyll
- jQuery
- HTML + CSS
- [Playemjs](https://github.com/adrienjoly/playemjs) for streaming tracks continuously

### Setup and usage

(Requires the Ruby interpreter)

```bash
$ git clone https://github.com/adrienjoly/openwhyd-jekyll
$ cd openwhyd-jekyll/public
$ bundle install
$ bundle exec jekyll serve
$ open http://localhost:4000/my-openwhyd-profile-playlist.html
```

### Please use your own YouTube API Key 🙏

1. Go to https://console.developers.google.com/projectcreate => type a name for your project => click on the "done" notification when it's shown
2. After that, go to https://console.developers.google.com/apis/library => find "YouTube Data API v3", then click on it => click on the "activate" button
3. Click on the "create identifier" (or "create credentials") button => pick "YouTube Data API v3" again in the dropdown list of APIs,  "Web Browser (JavaScript)", "Public data", then submit => it will provide an API key
4. In the source code, paste that API Key as value for every definition of the `YOUTUBE_API_KEY` constant

## Support Openwhyd

Openwhyd is a collaborative and open-source project.

There are several way you can help Openwhyd! 💓

- If you're a **developer**, you can help us reply to our users' problems and questions from the [Music Lovers Club](https://www.facebook.com/groups/openwhyd/), maintain [issues](https://github.com/openwhyd/openwhyd/issues), or even better: [contribute to the codebase](docs/FAQ.md#id-love-to-contribute-to-openwhyd-how-can-i-help). (beginners are welcome too!)

- You can help Openwhyd sustain by [becoming a backer](https://opencollective.com/openwhyd/order/313) (*starting at $1/month, can be interrupted anytime*), or giving a [one-off donation](https://opencollective.com/openwhyd/donate). We publish our expenses transparently on [Open Collective](https://opencollective.com/openwhyd).

- If you have **other skills** you'd like to contribute to Openwhyd, please [tell us](https://github.com/openwhyd/openwhyd/issues/new?title=Hi,+I+want+to+help!).

- If you're out of time and money, you can still **spread the word** about openwhyd.org, e.g. by showing it to your friends or sharing your playlists on social networks.

Thank you in advance for your kindness! 🤗
