---
title: "Setting Up My Mobile Application Security Testing: Mass Downloading APK files"
date: 2024-06-12 21:35:27 +0300
categories: [Security Testing, Mobile]
tags: [tagmobile, bugbounty, hacking]
image: /assets/img/mobile1.png
alt: "mass downloading apks"
---

## The Problem
Hunting on a prorgram with many APK files is good. Security testing too. Its always a problem when they provide a bunch of App IDs that makes it hard to download them individually.

There are a couple of ways to do this. I however use APKEEP to mass download the apk files from the app store.

## The Tool
I use [apkeep](https://github.com/EFForg/apkeep "Get apkeep here").

### Installation
```
cargo install apkeep
cargo install --git https://github.com/EFForg/apkeep.git (for latest commit)
```
### Usage
For a single downlaod, use:
```
apkeep -a <app-id> -d <sources> <output dir>
```
The supported download sources are:
```
apk-pure, google-play, f-droid, huawei-app-gallery
```
To download the `instagram` apk from `apk-pure` do:
```
apkeep -a com.instagram.android -d apk-pure .
```
For mass download, there is an option to load IDs into a CSV file and read from a specific column with
```
apkeep -c <csv file> -f <field containing IDs>
```
If the above is a hustle, a bash for loop can be written:
```bash
for i in $(cat ids.txt) #ids.txt is the file containing the apk IDs.
do 
apkeep -a $i -d apk-pure . #The dot inicated the output destination folder
done
```
![downloading!](/assets/img/ids.png "Some example apk files")

If the ids are present, they will download. If not, a `Could not get download URL` message will be displayed.

The good thing with apkeep is it can be used to download historical versions of the app if available from the sources.

![downloading!](/assets/img/ids-versions.png "apk files with different versions")

Downloading from Google Play Store will however require authentication. This can be done by following the steps [here](https://github.com/EFForg/apkeep/blob/master/USAGE-google-play.md)

## Feeling a little Paranoid?
In the likely case you do not trust the store, feel free to scan your files on [virustotal](https://www.virustotal.com "scan with Virustotal"), [Kasperskey](https://opentip.kaspersky.com), [OPSWAT Metadefender Cloud](https://metadefender.opswat.com "scan with metadefender") or a more personalized apk scanner like [Koodous](https://koodous.com "scan with koodous") which gives more details.