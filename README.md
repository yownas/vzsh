# vzsh

## Introduction

This is a small script to help you log into your OpenVZ containers.

I realize this isn't for everyone but it solves a couple of problems you would run into if:
* Your containers aren't accessable via ssh. (ssh disabled, no account in containers or network behind horrible firewall)
* You have a lot of OpenVZ hosts and can't remember which one has your container.
* You want to allow some admins to only be able to access certain containers. 

For example, you have your hosts in your secure admin-network. In the user-network there are a bunch of mad scientists that will try to hack into any dns/mail/syslog-server you might set up. Also, they will intercept ssh and crack any password you might leave in /etc/shadow in the containters they need to log into for their research.

## How it works

vzsh is what admins will use to access the containers. It does this by first going through a list of hosts to find which one contains your container. Then ssh to the host and use "vzctl enter" or "vzctl exec" to get a shell. By doing this you don't need to use ssh to the container, sshd can even be disabled and root have a * in /etc/shadow.

vzshd is used on the host as a "server". vzsh doesn't get root-shell access to the host, but talkes to vzshd instead. vzshd will help vzsh get a list of containers and has a small access list to check if remote users are allowed to get a shell in the container.

## Installation

Copy vzsh somewhere where admin-users will have access to it. Users then need to create a folder ~/.vz/ where they put a file ~/.vz/hosts that contains a list of OpenVZ hosts. One per line, comments with # are allowed and anything written after the first word is treated as a comment.

~/.vz/hosts

    # This is a comment
    this.is.a.host  Also a comment.
    another.host

Then create a ssh-key-pair with vzsh -g and distribute it to all hosts you are supposed to have access to.

In the OpenVZ hosts you need to place vzshd in ~root/bin/vzshd and create an ini-file, ~root/.vz/vzshd.ini that controlls who can log in.

Example ~root/.vz/vzshd.ini:

    [groups]
    wheel:user1
    managers:user1_admin
    
    [containers]
    www.example.com:user2
    *:@wheel

    [hosts]
    localhost:@wheel, user3
    
    [manager]
    move:@managers
    startstop:@managers

user1 has access to any container, as the group @wheel, while user2 only have access to www.example.com. @wheel and user3 are allowed to get a shell on the local host. Only the group "managers" are allowed to start and stop containers and move them to other hosts. user1 is a member of  managers but has to use a special key with "admin" as suffix.

In ~root/.ssh/authorized_keys you need to add the public keys generated in the step above.

If permissions are correct everything should be up and running now. (Probably not. Since this is still in early development.)

## Copyright

License: [The MIT License (MIT)](LICENSE)
