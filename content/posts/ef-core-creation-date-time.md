---
author: "kyrcooler"
title: "EF Core Automatic Creation and Modification Timestamp"
date: "2021-06-10"
description: "How to timestamp the creation of all entites for auditing purposes."
tags: ["efcore"]
keywords: ["efcore", "entity framework core", "timestamp", "ef core", "ef core creation date", "creation timestamp"]
hideMeta: true
searchHidden: true
ShowBreadCrumbs: false
---
For auditing purposes, we might want our entities to have creation and/or modification timestamps (we may also extend it with some additional data, like user ID). It can be a pain to keep in mind to always set the appropriate columns, but we can make EFCore do it for us.

First we need to create some sort of interface for it, like: `ICreationAudited.cs` or `IAuditedEntity.cs`.

{{< highlight csharp >}}

public interface IAuditedEntity
{
    DateTimeOffset CreationDateTime { get; set; }
    DateTimeOffset? LastModificationDateTime { get; set; }
}

{{< / highlight >}}

Now we need to intercept the `SaveChanges` calls. For this purpose let's create a method, that will check the EFCore internal `ChangeTracker` for any new entities that are implementing our `IAuditedEntity` interface. We can do that by checking their state against the enums `EntityState.Added` or `EntityState.Modified` and setting the appropriate columns with dates.

{{< highlight csharp >}}

private void SetAuditedColumns()
{
    var entitiesCreated = ChangeTracker
        .Entries()
        .Where(e => e.Entity is IAuditedEntity 
                    && e.State == EntityState.Added)
        .Select(x => x.Entity as IAuditedEntity);

    var entitiesModified = ChangeTracker
        .Entries()
        .Where(e => e.Entity is IAuditedEntity 
                    && e.State == EntityState.Modified)
        .Select(x => x.Entity as IAuditedEntity);

    foreach (var entity in entitiesCreated)
    {
        entity.CreationDateTime = DateTimeOffset.Now;
    }

    foreach (var entity in entitiesModified)
    {
        entity.LastModificationDateTime = DateTimeOffset.Now;
    }
}

{{< / highlight >}}

Now we need to override `SaveChanges` and `SaveChangesAsync` methods in our `ApplicationDbContext.cs` to perform our custom logic and that's it.

{{< highlight csharp >}}

public override int SaveChanges()
{
    SetAuditedColumns();

    return base.SaveChanges();
}

public override Task<int> SaveChangesAsync(
    CancellationToken cancellationToken = default)
{
    SetAuditedColumns();

    return base.SaveChangesAsync(cancellationToken);
}

{{< / highlight >}}