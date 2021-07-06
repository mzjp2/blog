---
title: "Fining myself for snoozing alarms :alarm_clock:"
layout: post
date: 2019-09-24 01:00
headerImage: false
tag:
- automation
- monzo
- IFTTT
- API

category: blog
author: zainpatel
description: "Using Apples new iOS 13 Shortcut Automations alongside IFTTT webhooks to set up automatic fines for snoozing alarms in the morning"
---

With Apples new [iOS 13](https://www.apple.com/ios/ios-13/https://www.apple.com/ios/ios-13/) release comes the much-hyped Shortcut automations, that let you run shortcuts based on a number of triggers.

This got me thinking about what I could use and after reading a few posts in the [Monzo Community Forum](https://community.monzo.comhttps://community.monzo.com) I decided to try my hand at creating a shortcut that would run automatically whenever I snoozed my alarm in the morning and transfer £1 to a designated pot in my Monzo accocunt. This post details how I set it up.

## Creating an IFTTT webhook
First steps first, you'll need to create a webhook on IFTTT, which is just a fancy term for having IFTTT give you a convenient URL that runs a specific action whenever you open said URL. This'll come in handy, because it so happens that the Shortcut app is capable of opening a URL as an action. So, how do you create a webhook?

1. Navigate to [ifttt.com](ifttt.com) and sign in.
2. Press explore in the top right and press the plus button next to "Make your own applets from scratch"
3. Press the "+This" text, search and open the "Webhooks" service in the "Choose a Service" page.
4. Select the "Receive a web request" option and give it an informative name, like `alarm_snoozed` and create it.

    ![](https://i.imgur.com/wKtePEb.png)

5. Press the "That" text and search/select Monzo. Choose "Move money into a pot".
6. Choose the pot that'll hold your fines in the dropdown along with the amount you want to fine yourself.

    ![](https://i.imgur.com/KjSGP8S.png)

7. Create your action!
8. On your home page, open the "Webhooks" service and press documentation, located on the top right.
9. Take note of your key, located on the top.

![](https://imgur.com/KSyLqV9.png)

## Creating a Shortcut - way 1

You'll need to ensure that you're on iOS 13.1 to be able to do this. Open your settings app, find the "Shortcuts" setting page and enable "Allow Untruusted Shortcuts".

![](https://imgur.com/YjNGSyy.png)

1. Open this link: [https://www.icloud.com/shortcuts/3fc2b776e5944b2f852fd79ca4b2db52](https://www.icloud.com/shortcuts/3fc2b776e5944b2f852fd79ca4b2db52) on your phone and press "Get Shortcut". 

    ![](https://imgur.com/7r5Hj4i.png)

2. Press "Add Untrusted Shortcut" and enter your `IFTTT trigger` and `IFTTT key` when asked.

    ![](https://imgur.com/PE5JNwW.png)

3. Open the `Automation` tab and press the plus button, pressing "Create Personal Automation".
4. Press "Alarm" under "Time of Day" and select "Is Snoozed" along with a specific alarm if you want.

    ![](https://imgur.com/mS46uXN.png)

5. Press next and then press "Add Action", then "Apps", then select the Shortcut app and press "Run Shortcut"
6. Select "Shortcut" and choose the "Monzo Snooze Fine" shortcut.

    ![](https://imgur.com/m8b5IUF.png)

7. Press Next, toggle "Ask Before Running" off and press Done.

You're all done! 

## Creating a Shortcut - way 2

If you don't want to enable untrusted shortcut settings and would rather makae it from scratch, this is how:

1. In shortcuts, press the plus button to create a new shortcut.
2. Press "Add action", select "Web" and then press "Get Contents of URL"
3. Enter "https://maker.ifttt.com/trigger/*YOUR-TRIGGER*/with/key/*YOUR-KEY*" replaceing the bits in bold with the relevant stuff.
4. Press "Done" and name your shortcut something descriptive, like "Monzo Snooze Fine" then follow steps 3-7 in `Way 1`.

You're all done!

## Future work

Make the shortcut send in specific values that you can use to set the fine. One quirky thing I want to try is fining an amount equivalent to the time of snooze, so if I snooze at 7:15 it'll fine me £7.15. This can be done by passing in some `JSON` when sending a request to the URL, which IFTTT can then use to pass along to Monzo for the pot transfer.
