---
author: "kyrcooler"
title: "EF Core Implementing Soft Delete"
date: "2021-06-07"
description: "Deleting your entites completly is dangerous and without global filters may be dangerous."
tags: ["efcore"]
keywords: ["efcore", "entity framework core", "soft delete", "ef core", "ef core soft delete", "soft deleting"]
hideMeta: true
searchHidden: true
ShowBreadCrumbs: false
---

Soft deleting is an easy way of deleting data without actually removing it from the database. Instead of performing `DELETE` on the database we mark it as deleted and filter it out by default on the application side. We want our soft delete mechanism to function seamlessly with EFCore, after all, it's just the implementation detail, so we need to intercept all `DbContext` `Remove` calls.

So, to achieve this we need two parts: 
- filtering out all data that has the deleted flag set, so that it is impossible for our Application to read deleted data
- intercepting all `SaveChanges` or `SaveChangesAsync` calls and replacing the usual delete with our custom Soft Delete mechanism

First, we need our `ISoftDelete.cs` interface, which all our entities will implement. It's a good practice to put in the time stamp as well.

{{< highlight csharp >}}

public interface ISoftDelete
{
    bool IsDeleted { get; set; }
    DateTimeOffset? DeletionDateTime { get; set; }
}

{{< / highlight >}}

After that in our `ApplicationDbContext.cs` we need to create the filter with the help of some reflection magic. We need to set the filter on each entity, so we iterate over them and then check if they implement the `ISoftDelte` interface to see if they are a candidate for the filter. After that, we set the filtering by getting the property and creating a lambda expression for it.

{{< highlight csharp >}}

protected void SetGlobalSoftDeleteQueryFilter(ModelBuilder modelBuilder)
{
    foreach (var entityType in modelBuilder.Model.GetEntityTypes())
    {
        var isDeletedProperty = entityType.FindProperty("IsDeleted");
        if (isDeletedProperty != null 
            && isDeletedProperty.ClrType == typeof(bool))
        {
            var parameter = Expression.Parameter(
                entityType.ClrType, "p");
            var prop = Expression.Property(parameter, 
                isDeletedProperty.PropertyInfo);
            var filter = Expression.Lambda(Expression.Not(prop),
                parameter);
            entityType.SetQueryFilter(filter);
        }
    }
}

{{< / highlight >}}

Now we need to register it in our `OnModelCreating` method:

{{< highlight csharp >}}

protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    SetGlobalQueryForSoftDelete(modelBuilder);

    // ...
}

{{< / highlight >}}

We are halfway there, now we need to set those columns by intercepting the `SaveChanges` calls. For this purpose let's create a method, that will check the EFCore internal `ChangeTracker` for any candidates implementing `ISoftDelete` interface to be removed and replacing their state with `EntityState.Modified` and setting the appropriate columns.

{{< highlight csharp >}}

private void SetSoftDeleteColumns()
{
    var entriesDeleted = ChangeTracker
        .Entries()
        .Where(e => e.Entity is ISoftDelete 
                && e.State == EntityState.Deleted);

    foreach (var entityEntry in entriesDeleted)
    {
        ((ISoftDelete)entityEntry.Entity).IsDeleted = true;
        ((ISoftDelete)entityEntry.Entity).DeletionDateTime = 
                DateTimeOffset.Now;
        entityEntry.State = EntityState.Modified;
    }
}

{{< / highlight >}}

Now we need to override `SaveChanges` and `SaveChangesAsync` methods in our `ApplicationDbContext.cs` to perform our custom logic and that's it.

{{< highlight csharp >}}

public override int SaveChanges()
{
    SetSoftDeleteColumns();

    return base.SaveChanges();
}

public override Task<int> SaveChangesAsync(
    CancellationToken cancellationToken = default)
{
    SetSoftDeleteColumns();

    return base.SaveChangesAsync(cancellationToken);
}

{{< / highlight >}}