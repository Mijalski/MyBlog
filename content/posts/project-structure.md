+++
title = ".NET Project Structure"
date = "2021-06-12T13:26:47+02:00"
author = ""
authorTwitter = "kyrcooler"
cover = ""
tags = ["architecture"]
keywords = ["architecture", "project architecture", ".net architecture", ".net project structure", "project structure"]
description = ""
showFullContent = false
+++

![.NET Project structure](https://i.imgur.com/doHlU9l.png)

For almost all of my projects, I follow this simple project structure and it works great, so let's go over it from the bottom.

![Example of my project structure](https://i.imgur.com/e5py7sN.png)

## Host

The entry project for your solution, in most cases it will be an API Host (for HTTP communication), but with this approach adding a new way of hosting your app is as easy as adding a new project and hooking up into your application. It should reference the `Infrastructure` project and register all it's entry points as needed (`app.UseEndpoints(endpoints => endpoints.MapControllers());` in the case of HTTP API). An example name for the project would be: `ExampleApp.HttpApi.Host`.

## Infrastucture & Database Access

Infrastructure is implementing all of the non-domain logic of your application, that your domain abstracts from via interfaces. This layer should be replaceable if needed - it should be possible to change database ORM, dependency injection container, etc. If your infrastructure layer is mostly focused on data persistence it can be merged into one project. Possible names for this project would be `ExampleApp.Infrastructure or `ExampleApp.EntityFrameworkCore`.

## Domain

The domain layer should realize all your business ideas and nothing more, it should function as a separate entity. The domain should be written in pure C# (POCOs) without any unnecessary dependencies on other projects. Part of the Domains is `Outbound Ports` which dictate via `Interfaces` what outside dependencies the Domain requires without any concrete implementations (eg. `Repositories` abstracted by interfaces). It should be possible to move over the project to another application that would like to realize the domain, with clear information on what the Domain requires to function (`Outbound ports` to be fulfilled in the `Infrastructure layer). Domain layer could also be split into multiple projects depending on the number of subdomains your application may realize. 

## Application layer

This layer is responsible for handling all the requests that are specified in the `Contracts` project. Handling the requests means not only passing them to the appropriate services located in the `Domain` or `Infrastructure layer, but it should also validate them against non-business requirements, map the request/response objects from the one exposed to the user to the internal ones used by the domain (eg. `DTO` -> `Entity`), check for permissions. This layer is the contact to the outside world, so it may be responsible for throttling requests and/or caching them.

## Contracts

This layer specifies all DTOs used in the Application layer as well as the User Interface. It should contain all definitions of requests and responses for them. The Contract name stands for the contract that it establishes between the Client and Application, saying given those inputs it will produce the given output. This layer may be named `ExampleApp.Application.Contracts`.

## User Interface

This layer is the entry point to the application for the user, it may take a form of a regular web page, SPA, native application, and many more. If the interface shares the same language as the backend (eg. `Blazor`) it should reference the same contracts project, but in the other case, it may be a good idea to autogenerate it from the original `Contracts` project. 