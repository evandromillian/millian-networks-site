---
title: "Portfolio"
date: 2022-06-28T09:33:29-03:00
draft: false
---
 
### Myridian: The Last Stand

{{<figure src="/images/myridian_wallpaper.png" width="600" >}}

**Myridian: The Last Stand** is an old school mobile platformer featuring stunning 2D pixelart, an orchestral soundtrack and intense action.

Driven by a story that requires players’ interaction in order to evolve, **The Last Stand** is the first chapter of Nettor’s mission to awaken the Verge and bring legendary heroes to life so they can face their greatest challenge.

{{<figure src="/images/portfolio/Carlo_Slash_AttackA.gif" caption="Carlo attacking">}}

At the end of every chapter, players’ progresses achieving their goals will dictate how new heroes, missions and events will be inserted in the storyline. The most experienced players will be invited to join solo or co-op quests and their results will lead the story on.

#### Lessons Learned

The project required extensive research in many areas. Despite of all resources and funcionality available, **Unreal Engine** is not optimized for 2D games, missing some essencial tools. 

###### 1. Animation

{{<figure src="/images/portfolio/Neeno_Carlo_Donna_RunCycle.gif" alt="Heroes running" title="Heroes running">}}

Due to the lack of a version of the animation controller for **Paper2D**, we found a plugin called **PaperZD**, exposing a state machine to control the transition among animations in the same way that it's available for skeleton meshes. 

###### 2. Color Palette & Skins

{{<figure src="/images/Neeno-Alternate_Skins.png" alt="Neeno skins" title="Neeno skins" width="450">}}

Color palettes were implemented using a combination of material calculations and texture preparations. The texture's colors acts as indexes to select colors from a palette in the materials.

Skins were implemented overlapping sprites containing the weapons and armor pieces, using the same palette mechanism for the characters.

###### 3. Navigation & AI

As the **Unreal Engine** lacks 2D navigation, the team implemented a custom  navigation logic using path points linked among them, with additional instructions in the character should jump or drop to the next point.

To control the characters, the project used AI controllers using goal score, which is much simpler than existing Behavior Trees funcionality.

###### 4. Multiplayer

{{<figure src="/images/Fay-KillingSpree.gif" alt="Multiplayer fight" title="Multiplayer fight">}}

Multiplayer logic were used for heroes and enemies movement, projectiles and moving platforms, as well as to replicate damage dealt and received among all characters. 

We implemented advanced techniques to reduce data transmission between the peers and the server, and to increase the perceived responsiveness for the players.

The game used dedicated servers hosted in **AWS** **GameLift**, and it was implemented using the minimum amount of code possible, receiving all information about the match from the matchmaker.

###### 5. Matchmaking and Backend

{{<figure src="/images/Myridian_teste_escolha_de_time.webp" alt="Team selection" title="Team selection">}}

The matchmaker required an application using cloud services for fast communication, great availablility and reduced cost. It uses a combination of **DynamoDB**, **Simple Queue Service** and **WebSockets** controlled by **API Gateway**.

To manage player accounts, we used ready-to-use services with unified data storage, single sign on and leaderboards, allowing clients from different platforms to access the same data. 


#### Future roadmap

Currently we are working on a single player history mode, allowing other players to experience the game without the need to connect to a multiplayer server and find people to play.