+++
title = "EF Core Automatic Creation and Modification Timestamp"
date = "2021-06-11T22:24:55+02:00"
author = ""
authorTwitter = "kyrcooler"
cover = ""
tags = ["efcore"]
keywords = ["efcore", "entity framework core", "timestamp", "ef core", "ef core creation date", "creation timestamp"]
description = ""
showFullContent = false
+++

For auditing purposes, we might want our entities to have creation and/or modification timestamps (we may also extend it with some additional data, like user ID). It can be a pain to keep in mind to always set the appropriate columns, but we can make EFCore do it for us.

First we need to create some sort of interface for it, like: `ICreationAudited.cs` or `IAuditedEntity.cs`.

{{< code language=csharp >}}

public interface IAuditedEntity
{
    DateTimeOffset CreationDateTime { get; set; }
    DateTimeOffset? ModificationDateTime { get; set; }
}

{{< /code >}}

Now we need to intercept the `SaveChanges` calls. For this purpose let's create a method, that will check the EFCore internal `ChangeTracker` for any new entities that are implementing our `IAuditedEntity` interface. We can do that by checking their state against the enums `EntityState.Added` or `EntityState.Modified` and setting the appropriate columns with dates.

{{< code language=csharp >}}

private void SetAuditedColumns()
{
    var entriesCreated = ChangeTracker
        .Entries()
        .Where(e => e.Entity is ICreationAuditedEntity 
                    && e.State == EntityState.Added);

    var entriesModified = ChangeTracker
        .Entries()
        .Where(e => e.Entity is IModificationAuditedEntity 
                    && e.State == EntityState.Modified);

    foreach (var entityEntry in entriesCreated)
    {
        ((ICreationAuditedEntity)entityEntry.Entity).CreationDateTime = 
            DateTimeOffset.Now;
    }

    foreach (var entityEntry in entriesModified)
    {
        ((IModificationAuditedEntity)entityEntry.Entity).LastModificationDateTime = 
            DateTimeOffset.Now;
    }
}

{{< /code >}}

Now we need to override `SaveChanges` and `SaveChangesAsync` methods in our `ApplicationDbContext.cs` to perform our custom logic and that's it.

{{< code language=csharp >}}

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

{{< /code >}}