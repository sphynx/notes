---
description: How to get all your email from Google Mail in a nice searchable format
---

# GMail backup

## Attempt \#1: Google Takeout - didn't fully work

Google has a special service called [Google Takeout](https://takeout.google.com/settings/takeout) designed to download all your user data from Google accounts, including GMail and Google Photos. It's easy to use: you just choose what labels you want to export and in which format, they will take some time to prepare a zip archive with a single .mbox file. Mbox is just a giant text file containing all your messages concatenated \(including chat messages from Google Talk\). In my case the archive was around 2 Gb and .mbox file was around 7 Gb. Then you can import that file in an email client \(say Mozilla Thunderbird\) to read, search and index your email.

So far so good, but it didn't quite work for me, since most of my e-mail is in Russian. It has turned out that this .mbox file is encoded in UTF-8 and before preparing it, Google has just converted every symbol which is not ASCII or UTF-8 into � \(which is a Unicode character for "replacement character"\), ignoring charset field in "Content-Type" MIME headers. It has also happened [to someone else](https://webapps.stackexchange.com/questions/71153/takeout-breaks-my-non-ascii). Hence, that contents of non-UTF-8 messages was basically lost and my Thunderbird was unable to read any messages in KOI8-R or Windows-1251 encodings used for Russian language before UTF-8 era.

However, if your email is mostly in English or in UTF-8 you'll be fine and I think this is the easiest way to go.

## Attempt \#2: GYB \(Got Your Back\) - worked fine!

["Got Your Back"](https://github.com/jay0lee/got-your-back) is an open source command line tool working with Google Mail through Google API over HTTPS. It's written in Python, works on Mac, Win and Linux and authorises itself through some Google services, obtaining OAuth 2.0 token and stuff. It sounds complicated, but the tool will guide you through the process. The user manual is [here](https://github.com/jay0lee/got-your-back/wiki).

The tool synced my email under one hour: it created a hierarchy of directories of this shape: `year/month/day/some-email-id.eml`. That .eml format is just plain text containing message headers and plain text email content in the original encoding \(no automatic conversion to �!\). Also it has created an sqlite database for correspondence between GMail labels and messages. It's possible to incrementally update the backup, so it doesn't take long to update the backup.

Then again, it's possible to search through those .eml files with just `ripgrep` or other text searching utilities. Or you can import it all in Mozzila Thunderbird \(using their Import/Export add-on available from `Options -> Extensions`\) and search from there. However, note that labels will be lost in this way \(since they are in the sqlite database\). I don't care much about labels to be fair.

## Other solutions

* I've also tried `offlineimap` which looked good, but it took forever to sync my email \(after a week it was still working on it!\) and it was often failing with intermittent errors, so I ditched it. Also it required setting up a non-trivial config for GMail before you can even start using it.
* A friend of mine has successfully used [mbsync](http://isync.sourceforge.net/) to sync his Google email into MailDir format in about an hour. I haven't tried this solution.

