---
layout: post
title:  "Game Hacking Part I"
summary: "Hack a Unity game by patching the source"
author: trigleos
date: '2020-10-12 20:00:00 +0200'
category: ['Csharp','Reversing', 'Windows_Reversing']
tags: Windows Reversing, Reversing, C#
thumbnail: /assets/img/posts/game-hacking-part-1/thumbnail.png
keywords: .NET Reversing, Unity game patching, Reversing, C# Reversing
usemathjax: false
permalink: /blog/game-hacking-part-1/
---
# Follow the white rabbit

## TL;DR
Hacking a Unity game to access hidden areas and patch new content in
## Description
Since I’ve encountered a hacker for the first time in a game, I’ve always wondered how they manage to exploit the mechanics in a way to fly or become unvincible. After I started participating in CTFs, I soon encountered the Windows Reversing category, where the executable is often a .NET executable or a Unity game. Both of these can be easily reversed, due to the fact that they are interpreted languages and thus produce a bytecode that is interpreted by the CLR (Common Language Runtime). I started loving this category and by hunger for challenges got satisfied when Stackoverflow released his Follow the white rabbit challenge, a game written in Unity that was part of the CSCG 2020. This post will explore several hacks and explain how I got the two flags hidden in the game. So let’s dive into the wonderful and fun world of game hacking

![game](/assets/img/posts/game-hacking-part-1/thumbnail.png)

## Exploring the game
First of all, what stackoverflow has created here just for a CTF challenge is amazing and I have seldomly seen challenges that are this much fleshed out. We got thrown into a beautiful medieval/futuristic world (weird combination I know). After looking around for a bit, we see a white rabbit. Following him like the challenge indicates us to, we get to a hole in the ground. The rabbit, without even thinking about it, just jumps into the hole

![rabbit suicide](/assets/img/posts/game-hacking-part-1/rabbit_suicide.gif)

However, if we try to follow him, this happens:

![death](/assets/img/posts/game-hacking-part-1/death.gif)

We clearly need to do something to survive this fall. So let’s code our first hack

## No Fall damage hack

Diving into the game’s assemblies, we soon find the PlayerController class which has a checkFallDeath function.

![csharp](/assets/img/posts/game-hacking-part-1/check_fall_death.png)

It’s a fairly easy function, and we clearly need to remove the call to Die() so our character isn’t killed when he falls to the bottom of the hole. So let’s just replace it by a nice and simple print(“Hello”)
After making this change, we no longer die at the bottom of the hole and we can easily walk to the first flag

![flag](/assets/img/posts/game-hacking-part-1/flag.png)

But now that we have it, how can we exit the cave? It’s time for another hack

## Fly hack

In order to leave the cave again, we need to implement a fly hack, Let’s take a look into the PlayerController class again, this time however into the CalculateVerticalMovement() function.
The important part of the function looks like this:

```csharp
else  if  (this.m_IsGrounded)  
{  
    this.m_VerticalSpeed  =  -this.gravity  *  0.3f;  
    if  (this.m_Input.JumpInput  &&  this.m_ReadyToJump)  
    {  
        this.m_VerticalSpeed  =  this.jumpSpeed;  
        this.m_ReadyToJump  =  false;  
    }  
}
```
There are two problems in this code that prevent us from continously jumping after we’re up in the air. First of all, the function checks if the player is grounded before it increases the player’s vertical speed so we need to remove that check. Second, the ReadyToJump variable is set to false after we’ve jumped once so let’s change that to true. Our updated code will look like this:
```csharp
 else  
{  
    this.m_VerticalSpeed  =  -this.gravity  *  0.3f;  
    if  (this.m_Input.JumpInput  &&  this.m_ReadyToJump)  
    {  
        this.m_VerticalSpeed  =  this.jumpSpeed;  
        this.m_ReadyToJump  =  true;  
    }  
}
```
After making these changes, we can easily fly out of the cave and explore the island more easily. Here, you can see the hack in action:

![fly hack](/assets/img/posts/game-hacking-part-1/flyhack.gif)

I will follow up this post with a second one where I implement a speed hack as well as patching new content into the game to get the second flag.

-Trigleos
