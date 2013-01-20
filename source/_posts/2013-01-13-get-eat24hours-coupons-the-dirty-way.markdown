---
layout: post
title: "Eat24 coupons the dirty way"
date: 2013-01-13 15:13
comments: true
categories: Hacks
---

I've been using [Eat24](http://eat24hours.com) a lot recently to order food and they always seems to have coupons on their Facebook page.

Here's a one-liner to get the latest coupons. Simply fill in your access token:

```bash
curl https://graph.facebook.com/eat24/feed?access_token={your_token} \
| python -mjson.tool | grep "coupon code '" \
| sed -r "s/.*code '([a-z]+)'.*/\1/g" | sort | uniq
```
