---
layout: post
title:  "Game Hacking Part II"
summary: "Hack a Unity game by patching the source"
author: trigleos
date: '2020-10-16 20:00:00 +0200'
category: ['Csharp','Reversing', 'Windows_Reversing']
tags: Windows Reversing, Reversing, C#
thumbnail: /assets/img/posts/game-hacking-part-1/thumbnail.png
keywords: .NET Reversing, Unity game patching, Reversing, C# Reversing
usemathjax: false
permalink: /blog/game-hacking-part-2/
---
# Follow the white rabbit
## TL;DR

Hacking a Unity game to access hidden areas and patch new content in
## Description
This is the second part of a two part series. In this part, we'll try to implement more hacks and patch more content into the game
## Recap
In part 1, we discovered the first flag as well as implement a No Fall Damage Hack and a Fly Hack.
## Speed Hack
Let's try to create a speed hack so we can explore the island more easily. Again, let's focus mostly on the Player Movement class because that's where most of the calculations take place. To increase running speed, we'll take a look at the CalculateForwardMovement() function where the important part looks like this:
```csharp
this.m_DesiredForwardSpeed  =  moveInput.magnitude  *  this.maxForwardSpeed;  
.
.
.
float  num  =  this.IsMoveInput  ?  5f  :  40f;  
this.m_ForwardSpeed  =  Mathf.MoveTowards(this.m_ForwardSpeed,  this.m_DesiredForwardSpeed,  num  *  Time.deltaTime);
```

What we need to change here is first of all somehow increase this.maxForwardSpeed which isn't set by default. By searching through the code, we can find the following reference:
```csharp
this.maxForwardSpeed  =  this.maxSprintSpeed;
```
So we now know that we need to change this.maxSprintSpeed which in turn is set by default to 10 so let's increase it to 100. Second, let's multiply this.m_DesiredForwardSpeed by 10f so it also increases after the program has started. The code now looks like this:
```csharp
this.m_DesiredForwardSpeed  =  moveInput.magnitude  *  this.maxForwardSpeed  *  10f;  
.
.
.
float  num  =  this.IsMoveInput  ?  5f  :  40f;  
this.m_ForwardSpeed  =  Mathf.MoveTowards(this.m_ForwardSpeed,  this.m_DesiredForwardSpeed,  num  *  Time.deltaTime);
```
And the speed hack works perfectly
## Searching for content
After implementing these hacks, we can easily explore the island and soon discover another rabbit that has a message for us:

![rabbit](/assets/img/posts/game-hacking-part-2/rabbit.png)

It seems that we have to wait for an update so we can find the second flag. But what if the update is already included in the executable. Let's investigate with utinyRipper that can extract Unity object files from finished Unity games. After doing that, we can open the extracted project in Unity and have a look around. We can find a scene that's not part of the finished game. So let's change the code a bit to patch it in. Because one of the only inputs we can control in the game is the Jump input, we'll change the behaviour of the calculateVerticalMovement function and add an if clause:
```csharp
if  (this.m_Input.JumpInput  &&  SceneManager.GetActiveScene().name  ==  "FlagLand"  &&  this.load)  
	{  
	this.load  =  false;  
	Scene  activeScene  =  SceneManager.GetActiveScene();  
	SceneManager.LoadScene(5,  LoadSceneMode.Additive);  
	SceneManager.MergeScenes(SceneManager.GetSceneByBuildIndex(5),  activeScene);  
}
```
This code checks for Jump input and then loads the formerly unimplemented scene and merges it with our current active scene. Let's give this a try and see if we can find the new scene.
![update](/assets/img/posts/game-hacking-part-2/update.png)
And there it is. Now the only question that remains is how to get into the house. It's time for our last hack.
## Teleport Hack
We will again change the calculateVerticalMovement function by adding an if clause so we get teleported into the house once we click the space bar long enough. Following is the added code:
```csharp
if  (this.m_Input.JumpInput)  
{  
	this.timer  +=  Time.deltaTime;  
	if  (this.timer  >  3f)  
	{  
		this.timer  =  0f;  
		GameObject  gameObject  =  GameObject.Find("Player");  
		gameObject.SetActive(false);  
		gameObject.transform.position  =  new  Vector3(-82f,  217f,  25f);  
		gameObject.SetActive(true);    
	}  
}
```
The timer checks if we hold down the button for three seconds before we get teleported. The rest of the code just deals with moving the player to the correct position. Finding the correct coordinates involved some trial and error for me but there might be a way to do it faster. So let's see the hack in action.
![teleport hack](/assets/img/posts/game-hacking-part-2/teleport.gif)
And finally, let's get a good look at the final flag
![flag](/assets/img/posts/game-hacking-part-2/flag.png)
So that's it for this short series about Windows Game Hacking, I hope you liked it and probs to LiveOverflow for creating this awesome challenge. He created another harder one and I might tackle it at some point in the future and will of course also post it here. Thank you for visiting my blog and have a good day

-Trigleos 
