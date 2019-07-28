While working on the implementation of secure and HIPAA compliant chat, I've encountered problem using Virgil crypto library with Alpine linux v3.8 and Open JDK 1.8.

Initial clue was the following exception, meaning there was some problem loading libraries compiled by Virgil:

`org.springframework.web.util.NestedServletException: Handler dispatch failed; nested exception is java.lang.NoClassDefFoundError: Could not initialize class com.virgilsecurity.crypto.virgilcryptojavaJNI`

Further analysis showed that upon initialization of the Virgil context, precompiled .SO library was created in the tmp directory: `/tmp/virgil_crypto_java3134786209685506331.so`.

Next step is to list `virgl_crypto_java` shared libraries by executing the following command:

`ldd /tmp/virgil_crypto_java3134786209685506331.so`

Output of the ldd command is as follows:
> ldd (0x7f47776a8000)
> libstdc++.so.6 => /usr/lib/libstdc++.so.6 (0x7f4776fbc000)

> libm.so.6 => ldd (0x7f47776a8000)

> libgcc_s.so.1 => /usr/lib/libgcc_s.so.1 (0x7f4776daa000)

> libc.so.6 => ldd (0x7f47776a8000)
> Error loading shared library ld-linux-x86-64.so.2: No such file or directory (needed by /tmp/virgil_crypto_java3134786209685506331.so)

Let's look at the following error message:`Error loading shared library ld-linux-x86-64.so.2`. Now, we need to figure out where `ld-linux-x86-64.so.2` file resides and in order to see that we will use [the following website](https://pkgs.alpinelinux.org/contents?file=ld-linux-x86-64.so.2&path=&name=&branch=&repo=&arch=). We can see that libc6-compat needs to be installed and we will do that by executing the following command: `apk add libc6-compat`. 

So everything should work now? Right? Well, yes, it worked until Alpine Linux [realesed patch](https://git.alpinelinux.org/aports/commit/?id=7b32fee49798e36cb5a7dfde30183f9717472cf6) that broke our code. I've decided to check the Alpine linux git repository and stumbled upon the following commit - [click here](https://git.alpinelinux.org/aports/commit/?id=7b32fee49798e36cb5a7dfde30183f9717472cf6). By default `libc6-compat` is not installed because Alpine Linux comes with musl compiler by default.



cp /lib64/ld-linux-x86-64.so.2 /lib/

https://pkgs.alpinelinux.org/contents?file=ld-linux-x86-64.so.2&path=&name=&branch=&repo=&arch=

apk list 
