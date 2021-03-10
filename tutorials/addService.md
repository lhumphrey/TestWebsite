---
sort: 2
---

# Adding a non-task service

Services in UxAS interact with other services through a publish-subscribe message passing 
architecture. In other words, a UxAS service subscribes to particular types of messages. The service then automatically receives messages of those types when they are published by other services. Generally when a service receives a subscribed message, it performs some action based on the content of the message. Often, it publishes a different type of message in response.

There are two categories of services in UxAS: task services and non-task services. Task services have a specific sequence of messages that they should send and receive. Non-task services do not.

Let us start by considering how to add a non-task service to UxAS. Since UxAS is written in C++, let us start by considering a non-task service in C++.


## In C++

A template for new services is provided in directory `OpenUxAS/src/cpp/Services`. This includes a header file `00_ServiceTemplate.h` and a source file `00_ServiceTemplate.cpp`.

In general, creating a new non-task service consists of the following steps:

- Choose a name `<ServiceName>` for your service
- In directory `OpenUxAS/src/cpp/Services`, copy `00_ServiceTemplate.h` to `<ServiceName>.h` and `00_ServiceTemplate.cpp` to `<ServiceName>.cpp`
- In header file `<ServiceName>.h`
  - Change all instances of the string `00_ServiceTemplate` to `<ServiceName>`
  - Change the string `UXAS_00_SERVICE_TEMPLATE_H` to a unique string appropriate for the new service
  - If you expect the service to generate output files, update function `s_directoryName()` to return an appropriate directory name
  - Include any necessary header files
  - Declare any necessary private member data and functions
  - Update the doxygen comments
- In source file `<ServiceName>.cpp`
  - Change all instances of the string `00_ServiceTemplate` to `<ServiceName>`
  - Update the logic of the member function `configure`
  - Include headers for any published or subscribed messages
  - Include any other necessary header files
  - Update the logic for member function `processLmcpMessage` to handle different types of subscribed message types
  - Define any necessary private member functions
- Modify the build process to include any 3rd party dependencies


## In SPARK/Ada

## In other languages

## Adding a task service
