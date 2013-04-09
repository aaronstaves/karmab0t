var sqlite3 = require('sqlite3').verbose();
var db = new sqlite3.Database('karma');
db.run('CREATE TABLE IF NOT EXISTS "karma" ("amount" INTEGER, "who" TEXT, "channel" TEXT)',[],function(err){
	db.run('CREATE UNIQUE INDEX IF NOT EXISTS "whochannel" ON "karma" ("who","channel")');
});
var request = require('request');
var channels=[
	"#groupcard",
	"#mkedev"
];

var names=[];

var irc = require('irc');
var client = new irc.Client('chat.freenode.net','KarmaB0t',{
	userName: 'KarmaB0t',
	realName: 'KarmaB0t',
	floodProtection: true,
	floodProtectionDelay: 250
});

var giveKarma = function(who,channel,amount){
	who = who.toLowerCase();
	channel = channel.toLowerCase();
	db.run('INSERT OR IGNORE INTO "karma" (who,channel,amount) VALUES (?,?,0)',[who,channel],function(){
		db.run('UPDATE "karma" SET amount = amount + ? WHERE who = ? AND channel = ?',[amount,who,channel]);
	});
};

var getKarma = function(who,channel,callback){
	who = who.toLowerCase();
	channel = channel.toLowerCase();
	db.get('SELECT amount FROM karma WHERE who = ? AND channel = ?', [who, channel], function(err,row){
		if(row && row.amount)
			callback(row.amount);
		else
			callback(0);
	});			
};

var statsKarma = function(channel, callback){
	channel = channel.toLowerCase();
	db.get('SELECT who,amount FROM karma WHERE channel=? and who!=?', [channel,'karmab0t'], function(err,row) {

	});
};

client.addListener('notice', function (nick, to, text, message){
	if(nick && text && nick.toLowerCase()=="nickserv" && text.match(/This nickname is registered/)){
		client.say('NickServ','identify PASSWORD');
	} else if (nick && text && nick.toLowerCase()=="nickserv" && text.match(/You are now identified/)){
		for(var i = 0; i<channels.length; i++){
			client.join(channels[i]);
		}
	}
});

client.addListener('names', function(channel, nicks){
	var tmp = [];
	for(var k in nicks){
		tmp.push(k);
	}
	names[channel]=tmp;
});

client.addListener('join', function(channel, nick, message){
	if(typeof names[channel]=="object" && names[channel].push && names[channel].indexOf(nick)==-1){
		names[channel].push(nick);
	}
});

client.addListener('part', function(channel, nick, reason, message){
	if(typeof names[channel]=="object" && names[channel].push){
		var l = names[channel].indexOf(nick);
		if(l > -1){
			names[channel].splice(l,1);
		}
	}
});

client.addListener('nick', function(oldnick, newnick, channels, message){
	for(var i = 0; i<channels.length; i++){
		var channel = channels[i];
		if(typeof names[channel]=="object" && names[channel].push){
			var l = names[channel].indexOf(oldnick);
			if(l > -1){
				names[channel][l]=newnick;
			}
		}
	}
});

client.addListener('message', function(from, to, message) {
	var re = new RegExp("^\s*karma( for)? ([a-z0-9_-]+)","i");
	var match = re.exec(message);
	if(to.substr(0,1)=="#"){
		if(match){
			(function(){
				var theChan=to;
				var theUser=match[2];
				if(theUser.toLowerCase()=="me"){
					theUser=from;
				}
				if(theUser.toLowerCase()=="karmab0t"){
					client.say(theChan,theUser+" has infinite karma");
					return;
				}
				if(theUser.toLowerCase()=="chameleon"){
					client.say(theChan,"http://www.youtube.com/watch?v=JmcA9LIIXWw");
					return;
				}
				getKarma(theUser,theChan,function(amount){
					if(amount===0)
						amount="neutral";
					client.say(theChan,theUser+" has "+amount+" karma");
				});
			})();
		}
		if(names[to]){
			re = new RegExp("([a-z0-9_-]+):?\\s*([+-]{2})","i");
			match = re.exec(message);
			if(match && match[1].toLowerCase()==from.toLowerCase()){
				client.say(from,"Try as you might, I will not allow any karma-masturbation");
			} else if(match && (match[2]=="++" || match[2]=="--")){
				if(names[to].map(function(e){return e.toLowerCase();}).indexOf(match[1].toLowerCase())>-1){
					switch(match[2]){
						case "++":
							giveKarma(match[1],to,1);
						break;
						case "--":
							giveKarma(match[1],to,-1);
						break;
					}
				}
			}
		}
	}
});
