---
title: "OpenPNEでメール無差別送付"
date: 2013-10-22 23:34:19
draft: false
slug: "openpneでメール無差別送付"
---

opCommunityTopicToolKit.class.php

内の

[php]

> if ($r-&gt;getIsReceiveMailPc() &amp;&amp; $memberPcAddress)

> if ($r-&gt;getIsReceiveMailMobile() &amp;&amp; $memberMobileAddress)

[/php]
これを
[php]

> if ($memberPcAddress)

if ($memberMobileAddress)
[/php]

こうすればいいね!
