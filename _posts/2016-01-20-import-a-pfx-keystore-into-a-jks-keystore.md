---
layout: post
title:  "Import a pfx keystore into a jks keystore"
date:   2016-01-20 09:13:12
categories: code
---

A project I was working on requires a JKS keystore, which holds a variety of different keys for a variety of different purpoes.  I wanted to use the same keystore to serve SSL certificates via Tomcat, below is how I achieved this.

```
$ keytool -importkeystore -srckeystore my.domain.com.pfx -srckeystoretype pkcs12 -srcstorepass myPass -destkeystore tomcat.keystore -deststoretype jks -deststorepass myPass
```

I then needed to change the alias:

```
$ keytool -changealias -keystore tomcat.keystore -alias src-alias-numbers-long -destalias tomcat -storepass myPass
```

I thing to be careful of is that I needed my JKS keystore and PFX keystore to have the same password.
