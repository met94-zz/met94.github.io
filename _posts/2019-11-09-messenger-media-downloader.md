---
layout: post
section-type: post
title: Downloading media from Facebook Messenger conversations
category: utilities
tags: [ 'nodejs', 'typescript', 'facebook', 'messenger' ]
---

## Introduction

[Facebook Messenger](https://www.messenger.com/) is a messaging app and platform developed by Facebook. If you have a facebook account, you are probably familiar with Messenger.
Basically  it allows users to chat with each other, but also to send videos, photos, audio which I will refer to as media.
I don't know how about you guys but I prefer to share everything I do with my closest friends and I'm not much of an Instagram/Snapchat guy, so my messenger conversations are full of media.
After a few years, I had this thought that it could be a good idea to download all the stuff and have it in a cloud.
I tend to destroy my phone every year and a half just before backing up my data, so Messenger is the only place I can restore at least media from.<br/>
That's how I ended up coding [MessengerChatMediaDownloader](https://github.com/met94/MessengerChatMediaDownloader), a utility to download media from messenger conversations.

## Unofficial Facebook Chat API

The utility uses [Unofficial Facebook Chat API for Node.js](https://github.com/Schmavery/facebook-chat-api) without which I couldn't create the utility, so big thanks to guys standing behind it.<br/>
<br/>
Also, I would like to join to their disclaimer:<br/>
<code data-trim>
Disclaimer: We are not responsible if your account gets banned for spammy activities such as sending lots of messages to people you don't know, sending messages very quickly, sending spammy looking URLs, logging in and out very quickly... Be responsible Facebook citizens.
</code>
## Usage

### Command line options
<pre>
-r, --reset - Resets the saved session, allows to relog to Facebook.<br/>
-a, --all - Download photos/videos/audios from all conversations.<br/>
-l, --list - List all conversations and their threadIds.<br/>
-t, --thread &lt;threadId&gt; - Download photos/videos/audios from the conversation with given threadID.<br/>
-i, --infinite - Keep retrying until all operations succeed.<br/>
-h, --help - Print help.<br/>
-V, --version - Print version.<br/>
</pre>
### NOTE
There seem to be some kind of API calls limit so if you attempt to dump media from a large conversation or all conversations, you will most likely hit the limit.
That's why there's is -i, --infinite option, so the utility will keep retrying to dump everything until it succeeds.

### Downloading media from a single conversation
In order to download media from a single conversation, we need to have its threadID.<br/>
We can get threadIDs for all conversations by running the utility with -l, --list option.<br/>
<pre><code data-trim>
node app.js -l -i
</code></pre>

<a href="/img/posts/messenger-media-downloader/run_l_list.png" data-lightbox="img1">
	![alt text](/img/posts/messenger-media-downloader/run_l_list.png "Run -l, --list")
</a>
*The utility run with -l, --list option*

Meanwhile I got a timeout because of my shitty internet connection but finally, the details of my conversations got saved to threadIDs.txt file as stated by the last line.
It may take a while to go through all conversations so be patient.

<a href="/img/posts/messenger-media-downloader/run_l_list_result.png" data-lightbox="img1">
	![alt text](/img/posts/messenger-media-downloader/run_l_list_result.png "Run -l, --list results")
</a>
*Results*

The results contain conversation name, messages count and threadID. They are also sorted by messages count so you can check who do you chat the most with :)<br/>
Let's say I wanted to download all media from my conversation with Gingełboy. To do that I would run the utility with -t 100005555555343 option.
<pre><code data-trim>
node app.js -t 100005555555343 -i
</code></pre>

<a href="/img/posts/messenger-media-downloader/run_t_threadid.png" data-lightbox="img1">
	![alt text](/img/posts/messenger-media-downloader/run_t_threadid.png "Run -t, --thread &lt;threadId&gt;")
</a>
*The utility run with -t, --thread &lt;threadId&gt; option*

Downloaded files will be outputted to *./outputs/&lt;conversation_name &#124;&#124; threadID&gt;/*.

<a href="/img/posts/messenger-media-downloader/run_t_threadid_result.png" data-lightbox="img1">
	![alt text](/img/posts/messenger-media-downloader/run_t_threadid_result.png "Run -t, --thread &lt;threadId&gt; results")
</a>
*Results*

### Downloading media from all conversations
This is rather straightforward, all you need to do is run the utility with -a, --all option and it will go through all conversations and download media.
Have in mind that if you have a lot of conversations and since there is an API calls limit, this task will take a lot of time, probably hours.
<pre><code data-trim>
node app.js -a -i
</code></pre>
<br/>

## Conclusion
Here you go, the utility to download media from Messenger conversations. This is a simple typescript Node.js project, nothing really fancy.
Feel free to contribute at [github.com/met94/MessengerChatMediaDownloader](https://github.com/met94/MessengerChatMediaDownloader).<br/>
By the way, I have downloaded 31 074 files with a total size of 4,37GB from all conversations. Share your result if you dare to let it go through all your messages.