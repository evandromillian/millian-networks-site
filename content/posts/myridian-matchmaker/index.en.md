---
title: "Developing the matchmaker system for Myridian: The Last Stand - Part 1"
date: 2022-08-11T18:30:46-03:00
description: ""
draft: false
author: "Evandro Millian"
resources:
- name: "featured-image"
  src: "featured-image.webp"
- name: "featured-image-preview"
  src: "featured-image-preview.webp"
  
tags: ["AWS", "Serverless", "Matchmaker"]
categories: ["System Development"]
lightgallery: true
---

One of the tricky points in the development of our game was the **matchmaker**. For those who don't know, this is the system that puts users in rooms, based on characters, skill and other information.

There are several **matchmaker** systems on the market, however they are very simple, they just receive information and organize players based on skill levels, number of wins, etc. We wanted a system that would coordinate the entire flow of information, prepare the rooms, then allocate the server and receive the match information. Usually in games like **League of Legend** or **Fortnite**, among others, the **matchmaker** just connects players to the game server, which in turn interacts with players to create teams and configure characters, among other things.

Our plan was for the game server to be as simple as possible, so all the described logic has been implemented in **matchmaker**. In addition, it is much simpler, easier and cheaper to hire people to work with cloud providers and develop in **Typescript** than to work with **Blueprints** in **Unreal Engine**. So that was our first high-level architecture decision.

We then divided the development into two parts, one dealing with the client's connection to the system (let's call it the **front-end**) and the other processing player requests (let's call it the **back-end**).

### Front-End

We developed three versions of the **front-end**, the first being created as a **REST** API called by the client, with **REST** endpoints in **API Gateway** invoking Lambda functions. This version worked very well, but the client accessed the system via **HTTP** call pooling, so there was a delay of a few seconds for the information to reach the client, in addition to multiple system calls during the entire time the player was connected. Although the **API** was very simple, there was this performance issue, but what motivated the evolution was the fact that the **matchmaker** could not actively communicate with the client.

In the development of the second version, we used a publish-subscribe system from a company called **PubNub**, but during the tests we had delay and performance problems, with messages taking too long to reach customers, so this version was quickly discarded.

Then we thought about the current version, which uses WebSocket also managed by **API Gateway**. In this way, the customer only sends a message requesting a departure, and the system actively sends the following messages to the customers:

1. Match created by the system
2. Team chosen by each player
3. Hero chosen by each player
4. Confirmation from each player
5. Game server connection information

In this way, there is no more pooling, reducing the number of calls to the system, reducing the cost, in addition to simplifying the logic on the client side.

### Back-End

The backend logic had two versions. The first was a **Typescript** application executing complex queries against **DynamoDB**, running periodically on a physical server also allocated on AWS. While it worked flawlessly, having a physical machine running such a small application was a waste of money.

We wanted all the logic to run in the **Lambda** environment, but there was an important technical issue: the minimum time between runs for scheduled tasks in **AWS Lambda** is 1 minute, which is unacceptable, as users would have to wait well over 1 minute. minute to enter a match, considering all the time of player choices, server allocation and client connection.

We had three options to reduce the time between matchmaking runs:

* A **Lambda** function executing logic multiple times during the execution minute, waiting a few seconds between each execution;
* A **Lambda** function sending scheduled messages to a **Simple Queue Service** queue, with the logic performed by the queue handler.
* **DynamoDB Stream** handling elements with **DynamoDB** TTL (time-to-live)

We decided to use the second option due to the lower cost (due to the shorter execution time), simplicity and also because it is an option not yet analyzed by other blogs specialized in the subject. We configured the scheduled function to send 6 messages with a delay of 10 seconds between them, so the matchmaking logic would run every 10 seconds.

Another task was used to handle the behavior of the room after its creation, waiting for the players' choices or the configured timeout, in which random choices are forced on the player. This task is performed periodically per room created, and only lasts between the time the players make their choices for the match until they connect to the match server.

### Call Flow

With the system described, the application flow is as follows:

{{<figure src="/images/myridian-matchmaker/Myridian_Matchmaker.webp" >}}

1. **Client** logs into **PlayFab** and receives a session ticket
2. Using the session ticket, the client connects to the **WebSocket API**, which then verifies the ticket
3. **Client** requests a new match; the system stores the request in the database
4. Periodically, the **Matchmaking Task** queries requests from customers, and groups them into a room
5. **Matchmaking Task** notifies the **client** that the room has been created
6. **Matchmaking Task** creates the task that monitors the room and handles player choices
7. **Client** sends team, character and build choices; the system broadcasts information to other players
8. After receiving all data from the players (or after a certain timeout), the **Waiting Room Task** asks **GameLift** to allocate a server passing the players' choices to it
9. **Waiting Room Task** sends the game server's connection information to the **Clients**
10. **Clients** connect to the server and the match starts

In this article, we focus on the high-level architecture and the AWS services used. In the next post we will talk a little bit about code, optimizations of **DynamoDB** queries and the team selection functionality, which was the last addition to the system. See ya.