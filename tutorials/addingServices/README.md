---
sort: 1
---

# Creating services

UxAS is a configurable collection of services that interact with other services through a publish-subscribe message passing architecture. In other words, a UxAS service subscribes to particular types of messages, and it automatically receives messages of those types when other services publish them. When a service receives a subscribed message, it performs some action based on the type and content of the message and usually publishes a message in response.

There are two categories of services in UxAS: task services and general (non-task) services. Task services have a specific sequence of messages that they should receive and send in order to correctly interact with other key services in UxAS. General services do not. 

Let's start by considering how to add a general service to UxAS. Since UxAS is written in C++, let's  start by considering a general service in C++.
