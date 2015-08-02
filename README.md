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
    logadmins:alice,bob
    
    [containers]
    www.example.com:user2
    syslog.local.domain:@logadmins=loguser
    *:@wheel

    [hosts]
    localhost:@wheel,user3
    
    [manager]
    move:@managers
    startstop:@managers

user1 has access to any container, as the group @wheel, while user2 only have access to www.example.com.

Alice and Bob are members of the logadmins-group and are able to access the container syslog.local.domain, but only as loguser, not root.

@wheel and user3 are allowed to get a shell on the local OpenVZ host. Only the group "managers" are allowed to start and stop containers and move them to other hosts. user1 is a member of  managers but has to use a special key with "admin" as suffix.

In ~root/.ssh/authorized_keys you need to add the public keys generated in the step above.

If permissions are correct everything should be up and running now. (Probably not. Since this is still in early development.)

## Usage

```
VZ shell

Usage: vzsh [options] [action]

Options:
        -k suffix
                Select key to use.
        -u
                Do not update container states.
        -q
                Be quiet.

Actions:
        [-r] container [command]
                Run command/shell in container.
        -x container [command]
                Run command/shell in container with X11 forwarding.
        -l
                Show list of all containers on all hosts.
        -m move container host
                Move container to host.
        -m moveall host1 host2 [host3]
                Move all containers from host1 to host2 [and host3]
        -m start container
                Start container
        -m stop container
                Stop container
        -g [suffix]
                Generate ssh-keyspairs. After keys been generated
                distribute your /home/yes/.vz/vzsh-yes-user.pub to all vz-hosts
                you need to access. Admin also has to update
                vzshd.ini to give you persmissions.
        -t
                Create folder and empty hostfile-template, implies -g.
        -h container [command]
                Run command/shell on host of container.
        -?
                Show this.
```

First thing you could do is to run 'vzsh -t' to get a .vz-directory and an empty hosts file and also generate an ssh-key to run vzshd with.

Example key:
```
command="~/bin/vzshd user=yownas" ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCvnBflmZrFValum2bW/hSvzt2WihNjXLMoIjkXaGbc994gxxssQg0/7FdtRboWkvOO4ytJSn2JBXzMpjq3jxc1FOTiZ3GgMO5Odv76+plLVUFtX0wG8Kycye9/wcRHK8Jy6TG4/AdVjpFTudWIK5GOpx28gueKDy1KhBFhEFf2DITOSjwKiUTgmPddADB3itbkBmAi1gJXed6XUjFrWeJCZTc0Swsn4OrMfK34uCtQ9j3fy16de42Uh0/dHt43HAuyCV+A7HT+/WtVagGjuorFD2f1wUt7V0FVXvhyyiotTWpIpnfouPTDfPOzwXF+fHS4lfpFepBzUNBEl4YsaUCp vzsh-yownas-user
```

When added to ~root/.ssh/authorized_keys on the host it will allow you to login as root but not give you a shell, instead it will run ~/bin/vzshd regardless of which command you told it to run. Is this super-secure and hacker resistant? No, and yes. No, the vzshd is just a shell script and I would be surprised if there wasn't some way to trick it to execute any command. On the other hand, only people with ssh-keys added are able to execute the script. As long as only administrators have keys and keep them safe you are ok....well, ok-ish. If you need super-security, let the non-admins play with their own cluster of servers.

When you've done that, add hosts to your ~/.vz/hosts file and you are done.

    -l

Simply list all containers on all hosts. (That you have access to.)

    [-r] container [command]

Run a command in the container or get a root-shell. (-r is not needed) What the script does is to log into the host and then runs 'vzctl enter' (or 'vzctl exec' if you send a command).

    -x container [command]

Get a shell or run a command with X11 forwarding. This is a bit more complicated. It doesn't run ssh directly. Instead it logs in as above, starts a sshd with -i so it sends data via stdin/stdout rather than over the network and then connects to it, allowing for X11 forwarding.

Basically you are tunneling X11 over the text-only console 'vzctl exec' give you. The advantage of doing it like this instead of using ssh directly to the container is that you do not need to have access to the network the container runs in (only the host). Sshd doesn't need to run and you don't need to know the root-password or have any keys installed.

    -h container [command]

Use this to get a shell or run a command on the host that contains the container. Basically a ssh to the host, but using the generated key.

    -g [suffix]
    -k suffix -g

Generate a new key or show a key that is already generated. Use ths suffix if you want to generate keys apart from the first 'user' key you generate. See the example ~/.vz/vzshd.ini above.

    -k suffix (other commands)
    
You can also use -k suffix or set VZSH_KEY to select different privilege levels. Default it to use your "user" key, but maybe you want to use an "admin" key to access the containers as root or separate "manager" key to start/stop or move containers.

    -m command

This will run different management command.

    -m move container newhost

Move a container to a new host.

    -m moveall oldhost newhost1 [newhost2 [newhost3]

Move all containers on oldhost to newhost1 (and newhost2 (and newhost3)) in a round-robin fashion. This could be used to empty a host if you need to do maintainance.

    -m start container
    -m stop container

Start or stop the container. I leave it as a challenge to the reader to figure out which one does what.

    -u
    (Or set VZSH_UPDATE to 0 or false)

For every command the script execute it will run ssh to all hosts in your ~/.vz/hosts file and look for containers. Sometimes this can be a bit time-consuming, a host may be down or moving a dns-server-container causes trouble. Using -u will skip this part and assume that the state-file created last time is accurate.

    -q

Does some things a bit less verbose.

## Examples

    vzsh testct.example.com uptime

Get uptime from testct.example.com.

    vzsh -k admin -h testct.example.com vzctl destroy testct.example.com

Using your admin-key, login on the host running testct.example.com and remove it. This is a good example why you would like to have separate admin-keys, to avoid doing something dangerous by mistake.

    echo "This is a nameserver." | vzsh ns1.mydomain 'cat > /etc/motd'
    echo "This is a nameserver." | vzsh -u ns2.mydomain 'cat > /etc/motd'

Put "This is a nameserver." in /etc/motd on ns1 and ns2. The second time we assume that ns2 hasn't moved and use -u to speed things up.

    rsync -va --rsh=vzsh mycontainer.mydomain:/root ~/backup

Use rsync to make a backup of root home-folder in mycontainer.

   export VZSH_UPDATE=0
   export VZSH_KEY=administrator
   vzsh webhost.test ls
   vzsh -k webmaster webhost.test ls

Without going through all hosts to update the state of the containters, list files in the container, first logging in with your administrator key and then with the webmaster key. (-k overrides the environment variable.)

## Copyright

License: [The MIT License (MIT)](LICENSE)
