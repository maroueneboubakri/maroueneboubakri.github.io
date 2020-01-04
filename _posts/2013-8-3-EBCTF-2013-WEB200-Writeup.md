---
title: EBCTF 2013 - WEB200 - Writeup
----

Hi !

Challenge WEB200 “Hipster NoSQL hangout”
We found a place where the hipster NoSQL admins hang out and share their secrets. We understood this new cutting edge technology is very secure, can you break in?

By googling these commands (listed in the login page) we can identify the database system used.

PSUBSCRIBE, PUBSUB, PUBLISH, PUNSUBSCRIBE, SUBSCRIBE, UNSUBSCRIBE, DISCARD, EXEC, UNWATCH, WATCH, MULTI, EVAL, EVALSHA, ECHO, BGREWRITEAOF, BGSAVE, CLIENT, CONFIG, DEBUG, MONITOR, SAVE, SHUTDOWN, SLAVEOF AND SLOWLOG

It is Redis (REmote DIctionary Server) a key store.

Now we inject some values in the login form, the couple foo/bar give "User not found"

<!--more-->

We try to inject some special chars, we put foo+bar as username and bar as password we get :

Our open source, BSD licensed, advanced key-value store returned an error: -ERR wrong number of arguments for 'get' command

at this point we reveal a good information(execution of a command) which make us think in a CRLF injection that we can append additional, malicious commands to queries.

lets try with "foo%0D%0Aping" as login and "bar" as password, http://54.217.3.87:5000/?&action=login&username=foo%0D%0Aping&password=bar

nice, here we get a new information : User found however the SHA-1 hash in the database does not match the SHA-1 hash of the password you provided.

Good :D we get a "valid login" but we miss a valid password(his SHA1 hash is equal the SHA stored in the db), so we have to try to inject more commands to disclose it.

the list of commands supported by redis is listed here : http://redis.io/commands

All commands wich seems being dangerous are disabled(EVAL, EVALSHA...) but there is an interresting command SCRIPT LOAD

Command description : Load a script into the scripts cache, without executing it. After the specified command is loaded into the script cache it will be callable using EVALSHA with the correct SHA1 digest of the script, exactly like after the first successful invocation of EVAL.

This command returns the SHA1 digest of the script added into the script cache.

lets try to inject it(the script as argument is a Lua script)

Login : %0D%0Ascript+load+return
password : bar

the result of the execution : Our open source, BSD licensed, advanced key-value store returned an error: -ERR wrong number of arguments for 'get' command 63143b6f8007b98c53ca2149822777b3566f9241

Yes :D we got a new good information useful for the injection and bypass the password test, since 63143b6f8007b98c53ca2149822777b3566f9241 is the SHA1 hash of "return"

now it still to organize our injections values like this :D

Login : foo%0D%0Ascript+load+return
Password : return

Congrats the flag is: ebCTF{de26ceb319c7636f09adf6238e7fd606}

;)


