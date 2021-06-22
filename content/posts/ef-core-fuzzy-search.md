+++
title = "EF Core Implementing Fuzzy Search"
date = "2021-06-11T22:36:23+02:00"
author = ""
authorTwitter = "kyrcooler"
cover = ""
tags = ["efcore", "postgresql"]
keywords = ["efcore", "entity framework core", "fuzzysearch", "ef core", "ef core fuzzy search", "fuzzy search", "ef core postgresql fuzzysearch", "postgresql fuzzysearch", "ef core postgre fuzzy search", "ef core postgre fuzzysearch"]
description = ""
showFullContent = false
draft=false
+++

![EF Core + Postgre + Fuzzy Search](https://i.imgur.com/W861qmy.png)

Fuzzy search allows finding strings that match the pattern approximately. It is a feature needed in almost every app, but it can be a little problematic to implement. We will focus on EFCore PostgreSql provider as it is free and seems to be the most popular choice nowadays (for me at least).

First, our EFCore project needs this package: [Npgsql.EntityFrameworkCore.PostgreSQL.FuzzyStringMatch](https://www.nuget.org/packages/Npgsql.EntityFrameworkCore.PostgreSQL.FuzzyStringMatch/)

Then, we need to register it with our `DbContext`, so let's navigate to `Startup.cs` and edit our registration like so:

{{< highlight csharp >}}

public void ConfigureServices(IServiceCollection services)
{
    services
        .AddEntityFrameworkNpgsql()
        .AddDbContext<ApplicationDbContext>(opt =>
            opt.UseNpgsql(Configuration.GetConnectionString(
                "DefaultConnection"), 
            n => n.UseFuzzyStringMatch()));

    // ...
}

{{< / highlight >}}

After that, we need to create a new database migration with our new extension. In `ApplicationDbContext.cs` edit `OnModelCreating` method and register the extension:

{{< highlight csharp >}}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.HasPostgresExtension("fuzzystrmatch");

    // ...
}

{{< / highlight >}}

With that out of the way, we can create new migration as usual with `Package Manager Console`:

{{< highlight bash >}}

PM> add-migration AddFuzzySearchExtension
PM> update-database

{{< / highlight >}}

That's it! We can now use `EF.Functions` to query our database, just like on the example:

{{< highlight csharp >}}

public IQueryable<Example> GetAllWithMatchingName(string name)
{
    return _dbSet
        .Where(_ => EF.Functions.FuzzyStringMatchLevenshtein(
            _.Name, name) <= 5)
        .OrderBy(_ => EF.Functions.FuzzyStringMatchLevenshtein(
            _.Name, name));
}

{{< / highlight >}}

This code will output an `IQueryable` of our `Example` entity with a matching name, ordered from the most similar to the least. The method also accepts a threshold acting as a cutoff point for matching, so that it won't return the whole database for you. You will need to experiment with the values and of course, there are different types of functions to perform the search, but I'll leave the research up to you as it might depend on your case.

With that you are ready to create a better search bar for your users!
![What is Fuzzy Search](https://i.imgflip.com/5d0km4.jpg)