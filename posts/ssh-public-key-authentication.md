---
title: Public key authentication with Java over SSH
description: This article shows how to securely connect (i.e. establish ssh connection) to the remote host from java application.
date: 2015-11-23
tags:
  - java
  - ssh
layout: layouts/post.njk
permalink: "ssh-public-key-authentication/index.html"
---

## Introduction

This article demonstrates how to securely connect to a remote host from a Java application by establishing an SSH connection. It also covers configuration details for enabling public key authentication and securing SSH keys.

Public key authentication allows users to establish an SSH connection without typing in a password. The key advantage is that the password is never transmitted over the network, reducing the risk of compromise.

For security, the private key should be stored in the SSH keychain and protected with an encryption passphrase.

## Generate Key Pair

The first step is to generate a private/public key pair on the server where your Java application will run.

You can generate the key pair by executing the following command:

```text
ssh-keygen -t rsa
```

Here is the output from my local development box:
```text
vladimir.stankovic@PCSVLADA ~
$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vladimir.stankovic/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/vladimir.stankovic/.ssh/id_rsa.
Your public key has been saved in /home/vladimir.stankovic/.ssh/id_rsa.pub.
```

Private key is identified as ```id_rsa``` and public key as a ```id_rsa.pub```. 

## Copy public key to remote host

The `ssh-copy-id` command copies the public key of your default identity to a remote host. If you want to use a different identity, specify it with the `-i <identity_file>` option.


```text
vladimir.stankovic@PCSVLADA ~
$ ssh-copy-id -i /home/vladimir.stankovic/.ssh/id_rsa root@www.svlada.com
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@www.svlada.com's password:
Number of key(s) added: 1
Now try logging into the machine, with:   "ssh 'root@www.svlada.com'"
and check to make sure that only the key(s) you wanted were added.
```

## Connect to remote host from Java

I used the [JSch library](http://www.jcraft.com/jsch/) to establish the SSH connection.

The most important step is configuring the `com.jcraft.jsch.Session` object and adding publickey to the list of preferred authentication options.

Here is a sample configuration for public key authentication: 
```java
    JSch jsch = new JSch();
    Session session = null;
    String privateKeyPath = "/home/vladimir.stankovic/.ssh/id_rsa";
    try {
        jsch.addIdentity(privateKeyPath);	    
        session = jsch.getSession(username, host, port);
        session.setConfig("PreferredAuthentications", "publickey,keyboard-interactive,password");
        java.util.Properties config = new java.util.Properties(); 
        config.put("StrictHostKeyChecking", "no");
        session.setConfig(config);
    } catch (JSchException e) {
        throw new RuntimeException("Failed to create Jsch Session object.", e);
    }
```

The next step is to connect to the remote host and execute an arbitrary command over SSH:

```java
    String command = "echo \"Sit down, relax, mix yourself a drink and enjoy the show...\" >> /tmp/test.out";
    try {
        session.connect();
        Channel channel = session.openChannel("exec");
        ((ChannelExec) channel).setCommand(command);
        ((ChannelExec) channel).setPty(false);
        channel.connect();
        channel.disconnect();
        session.disconnect();
    } catch (JSchException e) {
        throw new RuntimeException("Error durring SSH command execution. Command: " + command);
    }
```

## Source code

For a complete example, see the code in the linked [Git repository](https://github.com/svlada/ssh-public-key-authentication).
