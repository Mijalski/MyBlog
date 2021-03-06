---
author: "kyrcooler"
title: "EF Core Registering Model Configurations By Convention"
date: "2021-06-08"
description: "Is your ApplicationDbContext getting to large? See how you can organise and divide the configurations neatly."
tags: ["efcore", "dependency-injection"]
keywords: ["efcore", "entity framework core", "EntityTypeBuilder", "ef core", "ef core configuration"]
hideMeta: true
searchHidden: true
ShowBreadCrumbs: false
---

When our model configuration is stored all in one file it becomes tedious to edit it as the application grows.
Even, if developers agree to stick to some kind of ordering it is bound to blur and fade away as one of the developers inevitably adds one configuration in the wrong place. How can we prevent it?

The answer to our problem is: `IEntityTypeConfiguration<T>`. By implementing the interface you can configure the model for each class by implementing the interface's only method `void Configure(EntityTypeBuilder<T> builder)`. With it, we can decentralize our model configuration to separate the configuration across multiple files thus following the Open/Closed Principle (as adding a new configuration becomes as easy as adding a new file without altering the existing code). 

We now have to register all our Configurations from given assembly in our `ApplicationDbContext.cs`, like so:

{{< highlight csharp >}}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.ApplyConfigurationsFromAssembly(
        Assembly.GetExecutingAssembly());
}

{{< / highlight >}}

And create the configuration classes in our project (preferably under a separate directory called `Configurations`) like so:

{{< highlight csharp >}}

public class AppUserEntityTypeConfiguration : IEntityTypeConfiguration<AppUser>
{
    public void Configure(EntityTypeBuilder<AppUser> builder)
    {
        builder.ToTable("AppUsers");
        builder.HasKey(e => e.Id);
    }
}

{{< / highlight >}}

I have found that following the suffix `EntityTypeConfiguration` or `Configuration` makes it easy to navigate the project. Editing an entity and wanting to edit the configuration as well? Easy, just hit `Ctrl`+`T` and type in the name of the entity with the suffix.