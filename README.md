# Dots and Boxes

A prototype multiplayer game, specially made for a talk at O'Reilly's FluentJS conference, 12 March 2014. Should run on your laptop with node.js, socket.io, and many modern handheld devices, such as Apple's iPhone, iPad, and iPod Touch.  (Compatiblity note: the Web browser on each handset must be able to handle sockets.)

## Intro

Some time ago I was fortunate enough to play [Renga](http://wallfour.co.uk/announce/renga/), at the Palo Alto International Film Festival.  Renga is a 100-player game, shown in a movie theater or other large space.  Players are handed laser pointers on the way in; they are the only I/O the game requires. In 90 minutes, this diverse crowd of strangers became a well-trained spaceship crew: we explored, mined asteroids, build a colony, defended it against marauders, and defeated the Big Boss at the end, which was a very near thing that does not happen for most audiences.

Naturally I thought "Hey! I should steal this idea!"

Fast-forward to Yahoo's Open Hack Day in Sunnyvale, last September. For the first time ever I was making something for Hack Day instead of running the thing, so I figured I'd give it a try.

So I installed node.js, got socket.io, made the demo by [Rob Hawkes](http://rawkes.com/articles/creating-a-real-time-multiplayer-game-with-websockets-and-node.html) come up and run.  Presto, dots on a screen!  But then:

* managing players
* death and dying
* collision detection
* scorekeeping
* addressing individual handsets from the server
* other stuff

Turns out this is hard.  Really, really, really freaking hard.  I've made games before, but only single-player things.  As soon as you introduce the second player to the screen, things get super-hairy.

## What I Made for Fluent

Dots and boxes is a game for up to 16 active and many inactive players.  There's a big red box in the middle; your mission is to push it over the goal line.  You can also harass and bother the other players, by overtaking them from the rear.  First team to 7 points wins, and then the game resets.

In the repo are three files.  In descending order of specialization, they are:

server.js - serves socket.io plus the client and console files.  Also directs traffic.  This could be used for many applications, once many horrible bugs are fixed.

client.html - starts socket.io, puts a touchpad on the screen, sends new position to server.js if the player makes a new move or stops moving.

console.html - specially written to do nothing but run Dots and Boxes.

Important note: much (most) of this code is crap.  Please a) don't judge me by it and b) feel free to fix and ask for a merge!

The JavaScript inside the client and the console is case-hardened, meaning that it would (theoretically) run as third-party JavaScript inside any other Web page.  See my [De-Suckifying Third-Party JavaScript](https://github.com/kentbrew/desuckify) talk for more about all that.

## Installing

You'll need node.js and socket.io.  Install [node.js](http://nodejs.org/), fork [the GitHub repo](https://github.com/kentbrew/dotsnboxes) to your local machine, and run `npm install socket.io` from the repo directory.

## Running

Everyone must be on the same network to play.  

You'll need to figure out your IP address. I open up a terminal window and run netstat. 

Don't try to serve this from 127.0.0.1 or localhost; connected devices need to know the IP address running server.js or they won't be able to see it.

Once you know your IP address, start the server.  To do this, open up a terminal window, navigate to your repo directory, and enter `node server.js`.

Server running? Bring up a fresh window (Chrome = best as of now) and enter http://[youripaddress]:8001.  If all is good, the console will come up and show the right address for clients that wish to connect.  

Don't run more than one console window; it would be Bad.

When there's no wi-fi I've been able to create a network on my Macbook, have other devices join, and use my 169.XXX network.  If your network starts with 169, the connect URL will also show a network name; currently this is hardcoded to mine, which is usually `kentbrew`.  You'll need to fix that if you're creating your own disconnected network. 

iDevices will complain bitterly about connecting to this network; they will tell you that it isn't connected to the Internet, and then they will helpfully append www to your IP address if you don't start it with http://, but not actually SHOW YOU THAT THEY ARE DOING THIS, so it will just mysteriously fail.

## The Breakdown

### server.js

On first init, all this does is fire up three servers: one for `socket.io` on 8000, one for `console.html` on 8001, and one for `client.html` on 8080.  

We use `fs.readFile` to suck down the HTML and prepare to serve it with `http`, which we have required up top.  

Since some browsers will ask for a favicon, we have function `fakeIcon` set up to send back a blank response instead of littering the airwaves with 404s.

When a socket connects, we attache three event listeners, `move`, `join`, and `clientDisconnect`.  These uses `broadcast.emit` to keep all clients appraised of what's happening in the game.

Big problem I hit using socket.io: I can't figure out how to address a single client. When you die in the console, I should be able to send your handset a signal to let you know.

### client.html

The client is a simple Web page, with various `viewport` and `web-app-capable` header tags to keep it the right size in Apple devices.  Hopefully it will be fine in Android devices as well but I have not tested yet.

Although the client is not animating anything on the screen I am using Paul Irish's [requestAnimationFrame polyfill](http://www.paulirish.com/2011/requestanimationframe-for-smart-animating/) for my timing loop and it seems to be doing all the right things.  Battery life is not unreasonable, even on my crusty old zero-generation iPad.

All the client does is project a square, in the color of the team (blue or yellow) that you are on.  The client itself makes this decision by modding the ID that socket.io assigns; if you want to be on the other team, reload the page.  (Crude, right? I warned you it was crude.)

The pad is divided into a grid with an odd number of squares in each row and column. If you're pressing in the middle, you're at {x:0, y:0} and you won't move. If you go up a bit you'll be at  {x:0, y:-1} and your dot will start to move up.  All we're doing is communicating that pair of numbers from the client to the console, and if you or your vehicle hasn't hit a wall, the console will update on the next refresh with your location shifted in the right direction.  

This means there's no tapping, no flicking, and none of the usual drag-down-go-back up stuff; it's just a joystick.

Unfortunate bug: if you're driving a box and you aren't moving, your box will face right. This gives an advantage to the yellow team, whose goal line is to the right.

If you're trying it out on a laptop, mousemove on your trackpad should work. Click to stop on the screen, but remember when you mouse off you will start moving again.

### console.html

Like the client, the console is a simple load-once page that starts a socket connection and listens for input. It's the source of truth about where things live and what they're doing on the screen, not the game-runner at server.js.

There's some genuinely horrible stuff in here, particularly my collision detection and player handling.  

Players live in an object keyed by their socket ID; they should be in an array for much faster access. 

Collision detection is kept simple by the fact that everything on the screen is a square; all you need is this:

    collide: function (a, b) {
      return !(
        (b.y > (a.y + a.h)) ||
        (a.y > (b.y + b.h)) ||
        (b.x > (a.x + a.w)) ||
        (a.x > (b.x + b.w))
      );
    }
    
Bugs abound if you use this scheme.  Players wind up entangling with each other, penetrating the big red box, and getting stuck in the wall. 
    
There are no physics in this game; if you aren't driving a box, you are a ghost; if you are driving and you hit a wall, you die; and if you collide with another player or the big box in the middle, you stop moving and your momentum goes to the thing you hit.

You die when you hit a wall instead of bouncing in order to encourage speeds to be kept slow.  Otherwise crap would be bouncing like crazy all over the screen.

You score a point for your team by pushing the big red box in the middle over your goal line.  

I stumbled on the simple idea of right-left orientation according to your X vector, so if you are facing left and you hit someone on the other team who is also facing left, you are overtaking and hitting from behind, so you win.  (It's kind of like Joust, from the old days.)

Kills on overtake allow teams to have some players push while some players try to circle around behind and take out opposing players.  End zones are tight, to help with last-ditch defenses.

A limited number of drivable boxes (16) are available; this also helps keep it from getting too crazy, and extra players can simply park where boxes show up in order to be next up.

I implemented trails by overlaying the entire screen with a black box at 10% opacity and then printing everything else on top of that at 100%. If you're moving, your previous nine impressions will show as a gradually fading trail.

## Please help!

If Dots and Boxes appeals, help would be very welcome.  Games are hard, socket.io is underdocumented, and this was a much more complicated project than I thought it was at first.  Fork it, fix it, and make it your own!  

Thanks for listening, and special thanks to GitHub for their convert-your-readme-to-a-beautifully-formatted-page functionality, which made this presentation possible.

