---
layout: post
title:  "Android N 3D touch?"
description: "Android N 3D touch"
date:   2016-05-23
categories: android-N
tags: android-N
---

Android N preview 2 增加了一个新的接口：

Launcher shortcuts:

Now, apps can define shortcuts which users can expose in the launcher to help them perform actions quicker. These shortcuts contain an Intent into specific points within your app (like sending a message to your best friend, navigating home in a mapping app, or playing the next episode of a TV show in a media app). An application can publish shortcuts with ShortcutManager.setDynamicShortcuts(List) and ShortcutManager.addDynamicShortcut(ShortcutInfo), and launchers can be expected to show 3-5 shortcuts for a given app.

实现效果参考这个

[http://www.androidpolice.com/2016/04/21/android-n-feature-spotlight-launcher-shortcuts-give-apps-many-new-ways-to-provide-information-and-quick-actions/](http://www.androidpolice.com/2016/04/21/android-n-feature-spotlight-launcher-shortcuts-give-apps-many-new-ways-to-provide-information-and-quick-actions/)


update:
在 6-15日 发布的最终版本api中删除了此接口。
