---
title: "How to play Among Us on your Macbook :video_game:"
layout: post
date: 2020-10-02 18:00
headerImage: false
tag:
- osx
- game

category: blog
author: zainpatel
description: "Among Us has been recently re-kindled and is taking the world by storm, including my group of friends. This post is a quick guide to get Among Us working on your Mac."
---

![](https://steamcdn-a.akamaihd.net/steam/apps/945360/header.jpg)

If you're anything like me, then you've got yourself a Macbook (or 2 - definitely not using my work laptop to play games, no, not me! :eyes:) and are loathe to dual boot Windows via Bootcamp or spin up a heavy VM via Parallels just to be able to play Among Us. This posts is a quick guide to get Among Us running on your laptop/computer running OSx. I'm running on the latest Catalina version and haven't tested this with any previous versions of OSx.

## 1. Buy Among Us on Steam

The creators of the game deserve that sweet Steam revenue for the awesome game they've created. This is different to some of the other approaches out there on the web that get you to play Among Us on your Mac via Bluestacks (which frequently crashes) and uses the free Among Us mobile client via the Android Emulator.

Also... *skins*.

*Note*: You won't be able to install Among Us on Steam, since you're running on OSx and it's only been built for Windows. This is okay and to be expected.

*Note 2*: Buying it on Steam also means that we're likely to get the Mac client if and when the Among Us creators decide to update Among Us to have one (unlikely with all the work they have cut out for them though)

## 2. Download PlayOnMac

Install [PlayOnMac](https://www.playonmac.com/en/) - it's roughly 600mb to download and installs/unzips to about 2gb, which is a little larger than what I was expecting.

From what I can tell, PlayOnMac bundles together some kind of wine emulator/client that runs on OSx - get ready for lots of windows-style dialogue pop ups, bringing back some nostalgic and decidedly creepy feels seeing it on your Macbook screen.

When opening PlayOnMac for the first time, you may get a popup (if you're on Catalina) that will deny you access to open the application. You can get past this by going to Security in your System Preferences and clicking "Open anyway" on the "General" tab.

## 3. Install Windows-Steam on PlayOnMac

Launch the PlayOnMac application and press "Install" on the GUI that opens up. Open the "Games" tab on the top and search for *Steam* (not Among Us!). Hit install and run through the installer.

If you get anything about Wine asking you to install Gecko for "embedded HTML", feel free to cancel - it isn't relevant to us.

This should install Steam in `/Users/<user>/Library/PlayOnMac/wine_prefix/Steam` which is a virtual drive that mirrows Windows structure of `Program Files (x86)/Steam/steam.exe`.

## 4. Open up Steam

You'll need to login to Steam, likely pasting in a security code from your email and Steam should then open up. You might see a blank screen on the Steam GUI. If you do, close Steam and exit PlayOnMac and follow the next step.

## 5. Fix Steam

Open up PlayOnMac again, left-click on Steam and then click "Configure" in the sidebar that pops up on the left of the GUI. In the "General" tab of the configuration editor, paste the following in the "Arguments" field: `wine steam.exe -no-browser +open steam://open/minigameslist` which ensures that Steam only opens the mini-Library list that is compatible with Wine.

## 6. Re-open Steam

Re-open Steam (by hitting "Run") and go ahead and install Among Us, it's a 50mb download and 200mb unzipped and installed, so should be relatively quick. You can now run the game!

Don't forget to switch the controls from "Mouse" to "Mouse+Keyboard".

## 7. Red sus

![](https://vignette.wikia.nocookie.net/among-us-wiki/images/a/a6/1_red.png/)