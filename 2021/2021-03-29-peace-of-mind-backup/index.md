---
title: Peace of mind backup
date: 2021-03-29
summary: A backup solution that should last, be secure, and keep your mind at peace
tags: ['security']
githubFiles: true
---

## Table of Contents

## Introduction

If you are semi-paranoid, have a lot of data and want a backup-solution that last, this post might be for you.
I've been trying to get a backup solution that I'm happy with for a while, but after using it for a while, there is always the questions of `what if the house burns down`, `what if I forgot the backup key` that hunts me..

We don't just need to backup the data and encrypt it. We also need to backup the key used for the encryption, somehow..

This is the idea:

* Use https://restic.net on all pc's with different encryption keys per pc.
* Store those encryption-keys in an gpg (symmetric) encrypted base64 encoded text.
* The encrypted cipher-text can be stored on multiple locations, gmail, private github gist, usb sticks, printouts, whatever.
* The decryption key is stored in your password safe, but also physical. So if you can recover 2 of 3 physical keys (cylinders), you can get to the decryption key.
  * Don't haveto be cylinders.. Use your imagination. Can be printouts as well. You will understand what I mean soon..

## Actual content

### Passphrase for the encrypted text of secrets

The data will be encrypted using gpg with the `--symmetric` option and a very long passphrase (~135 characters).
The reason for the length is that we are going to split it in 3, then make 3 physical tokens so that we will need 2 of them to get the password.

Generate a passphrase of around 135 characters but use words, upper, lower, sprinkle with some random, space, characters and so on..
For simplicity, let say you ended up with `z1-z2-z3-z4-z5-z6`, split it up in 3, so you have

* a: `z1-z2`
* b: `z3-z4`
* c: `z5-z6`

Merged together, your passphrase is `z1-z2z3-z4z5-z6` (or you can keep the delimiters, upto you..)

Store each part so you will need 2 out of 3 to get the whole passphrase.

In container:

* 1: store `a` and `b`
* 2: store `b` and `c`
* 3: store `c` and `a`

For the storage, I would highly recommend https://cryptosteel.com/product/cryptosteel-capsule-trio/. They are a little expensive, but they are solid.

:::tip Cryptosteel
It's a little tedious to put every letter correct. Verify that you got it correct multiple times.
They are also a little expensive, but they will last! If you want them sealed, you can use example a thick layer of Marine epoxy, it will be a fairly distinct structure that you can take a picture of to make sure it havent been opened.
:::

:::tip Alternative
Use your imagination, you can use paper as well, laminate it. or maybe even scratch the code on something..
:::

You should also store 4-5 random characters in each container that is the same in all of them. This will be usefull when you want to find the encrypted text later, since you can also use those same 4-5 characters as a comment in your ciphertext. This make the text searchable/findable later. In our case, that is `neuz`

You will also need a delimeter between each part of the code.

The actual text you store will therefor be:

* In container `a`: `1/3: z1-z2|2/3: z3-z4||neuz`
* In container `b`: `2/3: z3-z4|3/3: z5-z6||neuz`
* In container `c`: `3/3: z5-z6|1/3: z1-z2||neuz`

So if you can get a hold on 2 out of 3, you got the key, but where do you hide them? Use your imagination here.. Trusted friends, safe, bury it down in the forest, or something.. Just make sure that noone can get to 2 of them without you knowing.

:::note Secret sharing the real way
There are multiple ways of splitting a secret like we do here. There is one very popular called https://en.wikipedia.org/wiki/Shamir's_Secret_Sharing, which can take any x of y to get the secret. We use a simple, manual way here because

* We don't want to try to figure out how we did it 10-20 years down the line..
* We want others to be able to decipher this as well
* Writing, and reading totally random characters from a cryptosteel is more error-prone than using words.
* Shamir really won't give us much in our case since it is so easy to do this manually..
* If someone finds 1 capsule, they still misses ~35 characters, which is not feasable to bruteforce/guess anyway
:::

In the end, you should now have a passphrase that you can make sure you will never loose.

:::tip Password storage
It's an good idea to store this password in your normal password-safe as well. It's not an `emergency only` password, rather a password that you can't loose that easiely.
:::

### The encrypted text of secrets

Now it's time to actually store some data in the encrypted text file, for this, we will use gpg with many flags to make things as secure as possible.

:::tip Alternative
There are a lot of alternatives to gpg --symmetric for this. I use gpg --symmetric here because

* It is well established tool
* It got a builtin way to store the ciphertext as something I can send on mail
* I can have an unencrypted string (comment) in it.
:::

This part is easiest told using a simple-ish shell-script..

The gist of it is:

* Makes an file with name of `bcktxt.*.enc` where `*` is your unique id
* Encryption is done with gpg and --symmetric with all the max of security
  * `--symmetric`: Encrypt using a single key, not a private/public key
  * `--s2k-mode 3`: Mangles passphrases more times
  * `--s2k-count 65011712`: Together with `s2k-mode 3` this ensure that that we need to hash this emount of bytes (around 65 MiB), for each time we want to try a password. This makes bruteforce much much harder since you can't bruteforce a million tries a second. Maybe a couple of hundred..
  * `--s2k-digest-algo SHA512`
  * `--cipher-algo AES256`: Symmetric cipher algo
  * `--no-symkey-cache`: Tell gpg not to cache our key (you must enter it every time)
  * `--armor`: Creates an ascii-armored file (which can be sent/stored as plaintext)
  * `--comment "..."`: The comment, being in the form `$uniqueID yyyy-mm-dd`, will be as plain-text in the output so we can search for the uniqueID whereever we decide to put it..
* If `bcktxt.*.enc` exists, we will
  * decrypt it and put the decrypted file on ram (`/dev/shm/`). We will warn if it is not a ramdisk
    * On OSX, we will create a new temporary ramdisk which we will cleanup after we are done.
  * Using an umask of `0177` so only we can read 
  * Open it using $EDITOR or nano. Warns if `vim` to remind you to turn off vim's temp-files/swap-files
* If script exists or crashes, we will do a cleanup
  * Tries to use shred on the decrypted file
  * Removes it
  * Cleanup temporary ramdisk if on osx

#### Helper script

Available <g-link :to="$page.post.githubFiles + '/encedit'">here</g-link>.

Some of the best options for gpg is available above, but if you want a finished "solution", you can just take what you want from a script I made for this.

Here is an example-output

```bash{outputLines: 2-29,31,33-40}{promptUser: "user"}{promptHost: "macbook"}{codeTitle: "On your mac"}
./encedit
found no encrypted file, what should the unique id be?
neuz
Running on OSX
Making new ramdisk device
Making filesystem on it (/dev/disk4)
Started erase on disk4
Unmounting disk
Erasing
Initialized /dev/rdisk4 as a 2 MB case-insensitive HFS Plus volume
Mounting disk
Finished erase on disk4 bcktxttmp
Will use /Volumes/bcktxttmp/enc-30384.md as tempfile for the decrypted file
Found no encrypted file (bcktxt.neuz.enc, a new one will be created
# editing files here...
Running cleanup...
No shred-util found, using normal rm
Unmounting, secure-erasing and ejecting temp ramdisk at /Volumes/bcktxttmp (dev: /dev/disk4)
Volume bcktxttmp on disk4 unmounted
Started erase on disk4 bcktxttmp
Pass: 1
Pass: 2
Pass: 3
Pass: 4
Pass: 5
Pass: 6
Pass: 7
Finished erase on disk4 bcktxttmp
Disk /dev/disk4 ejected
ls
bcktxt.neuz.enc encedit
cat bcktxt.neuz.enc 
-----BEGIN PGP MESSAGE-----
Comment: neuz 2021-03-29

jA0ECQMKHO9lmxlEQ57/0lgBz9Xxui2w6Xq0Jsya9TCpspagEGD+0IF8J26ipf+i
fGpyg+DEnLto/Wjl2fkwptsMSzvTJwPDkuQj0GTab8/bGb9fqN9Hrtvcg2s8csuY
cS1upUw2UVI0
=QHk1
-----END PGP MESSAGE-----
```

* Works on Linux and OSX
* Takes care of storing the file temporary on a ramdisk with the correct permissions
* Removes the decrypted file securely
* Even if the operation is aborted
* Lets you edit the file easiely if it already exists
:::

### What to store

Since distributing your encrypted text might be a one-time thing in many cases, you should really try to put as many things in it as you will need a couple of years down the line. Store multiple passwords of many kinds so you can easiely just refeer to eg. `b:3` for backup-key 3 when you get a new server and you want it to have an unique backup-key.

Example.. Generate as many as you need in each category.. 3 backup-keys? 30?

```markdown
## Backup keys (pwgen -cnys 63 2 | awk '{print "* b:" NR-1 " " $0}')
* b:0 BJ6yH+tJB_2F}@HZl@^f5bXazZmDisC:ed!C|Qn<>nxse~REzG[tEf3~l#t!+lF
* b:1 k+=y+PARLrZ#;<<,2:$AAr,qsL~AYSu3HS31^nq}*\<gyW|0E*h@ibDvlte'VhY

## 32 char passwords (pwgen -cns 32 2 | awk '{print "* p1:" NR-1 " " $0}')
* p1:0 q8TbtmjIa4fqWyCPS6b6MZjDe1P0m9Of
* p1:1 uQCHxT8Gi1aHMLcN9dj2GLeNq1kfC7ty

## 10 char passwords (pwgen -cns 10 2 | awk '{print "* p2:" NR-1 " " $0}')
* p2:0 SI6jwig15J
* p2:1 uTjw7es55c

## ssh keys (ln -sf /dev/stderr /tmp/stderr; ln -sf /dev/stderr /tmp/stderr.pub; for i in $(seq 3); do echo "* ssh:${i}"; yes | ssh-keygen -N '' -t ed25519 -f /tmp/stderr > /dev/null; echo; done)

* ssh:1
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
QyNTUxOQAAACDNG+ZoUn5G6Oxy2BqsUjq6FDqsf1LCy+8rKvh7rqhOOQAAAJgvusinL7rI
pwAAAAtzc2gtZWQyNTUxOQAAACDNG+ZoUn5G6Oxy2BqsUjq6FDqsf1LCy+8rKvh7rqhOOQ
AAAECUjIlzZsjrolLBedxxZYKLqReFZrZiB3gjhRuje8L5T80b5mhSfkbo7HLYGqxSOroU
Oqx/UsLL7ysq+HuuqE45AAAAD3hlb3JAYmVuZGVyLmxhbgECAwQFBg==
-----END OPENSSH PRIVATE KEY-----
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIM0b5mhSfkbo7HLYGqxSOroUOqx/UsLL7ysq+HuuqE45 xeor@bender.lan

* ssh:2
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
...
-----END OPENSSH PRIVATE KEY-----
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOPMAeRCBAf57BvNone5YoQg+ajwW/XbrXEgqEvCk4fF xeor@bender.lan

* ssh:3
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAAAMwAAAAtzc2gtZW
...
-----END OPENSSH PRIVATE KEY-----
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIDrSmgne7VkPUBdgjAKtP1SXsbM9gAE/zUcUgxT4GL6k xeor@bender.lan

### What now?

Here are some suggestions for what to do now...

* Distribute your encrypted ciphertext.. To some friends on gmail, github secret gist, usb sticks, pc's, printouts?
* Setup restic to take backup of your data but use one of the keys you are storing in your encrypted distributed blob
