---
title: 'Virgil crypto on Alpine linux v3.8'
description: Fun times with gcc, musl and Alpine linux v3.8.
date: 2019-01-06
tags:
  - alpine-linux
  - gcc
  - musl
  - crypto
layout: layouts/post.njk
permalink: "fun-times-with-gcc-musl-alpine-linux/index.html"
---

This article is a quick-start guide for debugging issues with the [Virgil crypto library](https://github.com/VirgilSecurity/virgil-crypto), Alpine Linux v3.8, and OpenJDK 1.8.

While testing different crypto libraries for a pet project, one candidate was Virgil. After a few hours of setup, I had a working prototype. Everything ran smoothly on macOS and Windows development machines, but things started breaking after deployment to Amazon.

The first clue came from the following exception, which suggested a problem loading libraries compiled by Virgil:

```java
org.springframework.web.util.NestedServletException: Handler dispatch failed; 
nested exception is java.lang.NoClassDefFoundError: 
Could not initialize class com.virgilsecurity.crypto.virgilcryptojavaJNI
```

Further analysis showed that Virgil copies precompiled `.SO` to tmp directory: `/tmp/virgil_crypto_java3134786209685506331.so`. 

Logical step was to list shared libraries `virgl_crypto_java` by executing the following command: `ldd /tmp/virgil_crypto_java3134786209685506331.so`

Output of the ldd command:

```bash
ldd (0x7f47776a8000)
libstdc++.so.6 => /usr/lib/libstdc++.so.6 (0x7f4776fbc000)
libm.so.6 => ldd (0x7f47776a8000)
libgcc_s.so.1 => /usr/lib/libgcc_s.so.1 (0x7f4776daa000)
libc.so.6 => ldd (0x7f47776a8000)
Error loading shared library ld-linux-x86-64.so.2: No such file or directory (needed by /tmp/virgil_crypto_java3134786209685506331.so)
```

Let's look at the following error message: `Error loading shared library ld-linux-x86-64.so.2`. 

We need to figure out where `ld-linux-x86-64.so.2` file resides. That can be checked [the following website](https://pkgs.alpinelinux.org/contents?file=ld-linux-x86-64.so.2&path=&name=&branch=&repo=&arch=). We can see that `libc6-compat` needs to be installed. This can be done by running the command: `apk add libc6-compat` or by adding the following command to the docker file: `RUN apk add --update --no-cache libc6-compat`. By default, `libc6-compat` is not included, since Alpine Linux ships with the musl C library instead of glibc.

So everything should work now? Right? Well, yes, it worked until Alpine Linux [released patch](https://git.alpinelinux.org/aports/commit/?id=7b32fee49798e36cb5a7dfde30183f9717472cf6) that broke the code. I've decided to check the Alpine linux git repository and stumbled upon the following commit - [click here](https://git.alpinelinux.org/aports/commit/?id=7b32fee49798e36cb5a7dfde30183f9717472cf6).

The problem with this commit is that previously `/lib64` was just a symlink to the `/lib` directory. With this change, that is no longer the case.

As a result, `ld-linux-x86-64.so.2` is copied to `/lib64` instead of `/lib`, which breaks the code. A quick (and dirty) workaround is to manually copy the file to `/lib` (e.g. `cp /lib64/ld-linux-x86-64.so.2 /lib/`).
