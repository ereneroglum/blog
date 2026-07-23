+++
date = '2026-07-23T16:16:31+03:00'
title = 'Email Sieving'
tags = [ "email", "stalwart", "dovecot", "postfix", "opendkim", "sieve", "mail", "smtp", "jmap", "imap", "thunderbird" ]
+++

I have recently switched to a self managed email solution.
I use [Stalwart Mail](https://stalw.art/) as my email server and it handles
almost all of the email features I can think of perfectly.

I used to have a postfix + dovecot + opendkim as my email server. To be honest
the setup worked perfectly and I had no reason to switch, but I wanted to
experiment with this new and shiny email server called Stalwart.

One of my complaints compared to my older setup is that in the Stalwart's
Web UI, instead of hiding the paid features they chose to put a lock icon
besides them. I understand having a paid and a free model. They ofcourse need to
be able to fund their work, and it is hard to fund open source, but having that
lock icon instead of having nothing sometimes becomes really annoying.

One of the features of this email server is that the users can upload their
sieve scripts to run on the server side when they recieve an email. This feature
also exists in dovecot but I never thought about setting it up mostly because
I saw it as a useless feature. I use Thunderbird as my email client, though
I am thinking about switching to Neomutt or a JMAP client, and Thunderbird
includes client side filtering for mails. From Thunderbird I can move specific
mails to their respective folders with message filters. It works but only when
I open my Thunderbird on my computer. Sometimes I want to check my emails from
my phone but I don't have the filters on the phone. So they all end up in the
main mailbox sitting until I open my PC and run the filters on them.

Now that I switched to Stalwart I can utilize the sieve function to sort these
mails in the server side. I am using a very simple script and I will explain it
line by line.

The whole script is as following:

```sieve
require ["fileinto"];

if anyof (
    header :contains "X-Spam-Flag" "YES",
    header :contains "X-Spam-Status" "Yes"
) {
    stop;
}

if header :contains "to" "digikey@<my domain>" {
    fileinto "Digikey";
    stop;
}

if header :contains "to" "farnell@<my domain>" {
    fileinto "Farnell";
    stop;
}
```

It is a very simple script. ```require ["fileinto"];``` is what allows me to use
`fileinto` function to move mail between folders. I want spam mails to remain
in spam folder so I use

```
if anyof (
    header :contains "X-Spam-Flag" "YES",
    header :contains "X-Spam-Status" "Yes"
) {
    stop;
}
```

this checks if the mails is spam and immediatly terminates the sieve script
if it is marked as such. Then the script follows with a number of blocks:

```
if header :contains "to" "<service>@<my domain>" {
    fileinto "<Folder>";
    stop;
}
```

When I sign up to services I use `<service>@<mydomain>` as my email so when they
send messages these blocks in sieve scripts capture them and put them into their
respective folders. I wish I knew this trick when I first started using email
but better late then newer.