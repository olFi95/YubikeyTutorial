# How to set up and use an Yubikey with OpenPGP
In this tutorial i will discuss how to set up an Yubikey with generated keys, how to manage the keys and how the Yubikey then can be used for encryption of emails and as login device for SSH.
I want to thank Thomas Goirand for his great Presentation at DebConf. I took almost everething i write below from his presentation that can be viewed over on Youtube: https://www.youtube.com/watch?v=xGsixSh6sC4

## Generate an Master key
In this section it is really important to use an clean and secure as possible machine. Preferably an Offline PC with an Life-CD.

First of all we have to generate the Master Key. To do so we type `gpg --full-gen-key`.
GPG will ask what type of Key we want to generate. We Select (1) aka "RSA and RSA". 
After that the Key-Length is configured, i chose 4096 because why less? 
I dont want the Master-Key to expire at any time, so i give the validity "0".
Then GPG wants our Name and E-Mail. It shall get them.

If we type `gpg --list-keys` we should see something like:
```
pub   rsa4096 2020-04-20 [SC]
      C8AAC3DB7F2C8F8E9B5A9790353A6519F974167F
uid        [ ultimativ ] Florian Dobisch (Example for the writeup) <Test2@Dobisch.online>
sub   rsa4096 2020-04-20 [E]
```
And with `gpg --fingerprint --fingerprint` something like:
```
pub   rsa4096 2020-04-20 [SC]
      C8AA C3DB 7F2C 8F8E 9B5A  9790 353A 6519 F974 167F
uid        [ ultimativ ] Florian Dobisch (Example for the writeup) <Test2@Dobisch.online>
sub   rsa4096 2020-04-20 [E]
      421A 4CAF 3948 886B C159  CFB6 EDAA 3FBE 9049 B3B1
```

For convenience later i export my Master-Key-Id to an variable like: `export keyid=C8AAC3DB7F2C8F8E9B5A9790353A6519F974167F`


## Delete the Default Encryption Key
Because we do not want an Key of any type but the master key to have an infinite Lifetime, we should delete it. To do so, we use `gpg --edit-key $(echo $keyid)`. Then we select the Key with `key 1`. The Key in question should now have an asterics next to it like this:
```
sec  rsa4096/353A6519F974167F
     erzeugt: 2020-04-20  verfällt: niemals     Nutzung: SC  
     Vertrauen: ultimativ     Gültigkeit: ultimativ
ssb* rsa4096/EDAA3FBE9049B3B1
     erzeugt: 2020-04-20  verfällt: niemals     Nutzung: E   
```
If the correct key (the one with the `Nutzung: E`) is selected we can delete it with `delkey` followed by `y`.

## Generate subkeys
After we generated the Master-Key we need to generate several subkeys from it.

To Derive an subkey from the master key do `gpg --edit-key --expert $(echo $keyid)` then `addkey`. From there on we configure the new Key to be an RSA key with option 8. Then We toggle sign and encrypt to off and authenticate on. I want my subkeys to expire after one Year, so I set the validity to `365d`.

Repeat this procedure to create an sign and an encrypt key. The only difference there are the capability-toggle-switches.

After all the Keys are Generated we should have something like 
```
sec  rsa4096/353A6519F974167F
     erzeugt: 2020-04-20  verfällt: niemals     Nutzung: SC  
     Vertrauen: ultimativ     Gültigkeit: ultimativ
ssb  rsa4096/EDAA3FBE9049B3B1
     erzeugt: 2020-04-20  verfällt: niemals     Nutzung: E   
ssb  rsa4096/112D178E261D0362
     erzeugt: 2020-04-20  verfällt: 2021-04-20  Nutzung: A   
ssb  rsa4096/4C4FE54289DD470B
     erzeugt: 2020-04-20  verfällt: 2021-04-20  Nutzung: S   
ssb  rsa4096/2C45D2DEBC6F037A
     erzeugt: 2020-04-20  verfällt: 2021-04-20  Nutzung: E   
[ ultimativ ] (1). Florian Dobisch (added the second mail) <Test3@Dobisch.online>
[ ultimativ ] (2)  Florian Dobisch (Example for the writeup) <Test2@Dobisch.online>
```

If yours looks similar enough save the keys with `save`.



## Export the keys

First of all we inspect our Keys again with `gpg --fingerprint --fingerprint`:

```
pub   rsa4096 2020-04-20 [SC]
      C8AA C3DB 7F2C 8F8E 9B5A  9790 353A 6519 F974 167F
uid        [ ultimativ ] Florian Dobisch (added the second mail) <Test3@Dobisch.online>
uid        [ ultimativ ] Florian Dobisch (Example for the writeup) <Test2@Dobisch.online>
sub   rsa4096 2020-04-20 [E]
      421A 4CAF 3948 886B C159  CFB6 EDAA 3FBE 9049 B3B1
sub   rsa4096 2020-04-20 [A] [verfällt: 2021-04-20]
      490D D6B9 6617 4A22 8627  9E2B 112D 178E 261D 0362
sub   rsa4096 2020-04-20 [S] [verfällt: 2021-04-20]
      BC7E AEDC EE00 0EC7 DDD8  0E09 4C4F E542 89DD 470B
sub   rsa4096 2020-04-20 [E] [verfällt: 2021-04-20]
      CE86 9C63 F148 EF80 B3F1  0F34 2C45 D2DE BC6F 037A
```

To export an key we use the last 8 characters of its key-id, ie `261D0362` for the authentication key.
Exporting an key to an file is pritty straight forward, we just `gpg --export-secret-subkeys -a 261D0362 > export/test2_261D0362.private-auth-subkey.txt`.
Same for the sign Key `gpg --export-secret-subkeys -a 89DD470B > export/test2_89DD470B.private-sign-subkey.txt`. And for the encryption key `gpg --export-secret-subkeys -a BC6F037A > export/test2_BC6F037A.private-enc-subkey.txt`.

After that is taken care of, we export the Master-Key as well. To do so we `gpg --export-secret-keys F974167F -a > export/test2_F974167F.private-master-key.txt` and again without the `-a`-flag `gpg --export-secret-keys F974167F > export/test2_F974167F.private-master-key`


## Put the Keys onto the Yubikey
To achive that we have to `gpg --edit-key --expert $(echo $keyid)` again. This time we select the authentication Key with `key 2`. The authentication key should now be highlighted with an asterics.
```
ssb* rsa4096/112D178E261D0362
     erzeugt: 2020-04-20  verfällt: 2021-04-20  Nutzung: A   
```

Then we can put the key to into the yubikey with `keytocard`. After that we have to select what the key should be used for, we select `3` for authentication. And then `y` to override an existing key.

Repeat for the remaining 2 keys, sign and encrypt.

After we are done with this procedure we do NOT `save` but `quit` without saving changes. Otherwise the keys are deleted from the computer.

## Delete the Master Keys
If the Keys were generated on the machine we actually use we should delete the master-key from the keyring and only leave the backups in secure places to be retrieved in 365 days to renew the subkeys.
To do so we `gpg --delete-secret-keys $(echo $keyid)`.


# Do stuff with this setup
After all that efford I want to see some action with this fancy USB thingy.
## Sign and Encrypt
Now we do have the power to use the Yubikey in order to sign and encrypt stuff.
- `gpg -u $(echo $keyid) --clearsign testfile.txt` will sign an file in clear text. Then we can verify that file with `gpg --verify testfile.txt.asc`.
- To encrypt data we do `gpg --encrypt testfile.txt` and select the keyid of the key in the Yubikey (aka $keyid). Now only the Yubikey can decrypt the data with `gpg --decrypt testfile.txt.gpg`.



# Nice to Know
- It is possible to add multiple E-Mail addresses (or uids) to one key. To do so just `gpg --edit-key --expert $(echo keyid)` and then `adduid` in the gpg shell and answer the questions about name, mail and comment.
- With the Program `paperkey` it is possible to export an key in printable format. This is nice for backup reasons.
