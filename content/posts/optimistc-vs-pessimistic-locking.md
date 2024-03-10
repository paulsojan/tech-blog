---
author: Paul Elias Sojan
title: Optimistic vs Pessimistic locking in Rails
date: 2024-03-10
tags:
  - ruby on rails

readingSpeed: 20
readingSpeedMin: 50
readingSpeedMax: 100
---

Concurrency handling is an essential aspect of any multi-user application. For example, suppose two users are trying to update the same data fields in an application concurrently. In such instances, you must decide which update is valid and which one should be discarded, which is a pretty challenging task. This is where database-locking mechanisms come into play. Locking ensures that only one user can modify a resource at a time, preventing conflicts and maintaining data integrity.

In this article, let's see two primary locking strategies provided by Rails: Optimistic locking and Pessimistic locking.

### Optimistic locking

It is a method of locking transactions that allows multiple transactions to access the same data concurrently assuming that conflicts in the database are rare. If a conflict is detected, such as when multiple transactions are trying to commit at the same time, and if one of the transactions commits successfully, it will reject all other transactions. To detect the conflict optimistic locking uses either a timestamp or a version number that is associated with each record. The transaction with the older version, the change is rolled back.

Rails supports optimistic locking by adding a `lock_version` to the model that needs to be locked. Every time a record is successfully updated `lock_version` increases. When a user tries to update the record, with a version number that doesn't match the current version in the database, Rails raises an `ActiveRecord::StaleObjectError` exception and the update is rejected.

For example, we have a table called `products` with columns `title`, `stock_quantity` and `lock_version`.

The first user wants to update the `stock_quantity` and do several operations at the same time in the same transaction
but the second user just wanted to update the `stock_quantity`, and did it in another request simultaneously.

![Optimistic locking](https://ik.imagekit.io/eapzn8piu/optimistic-locking.gif)

In this case, the second user will end up with an error `ActiveRecord::StaleObjectError` because `lock_version` got updated.

The downside of optimistic locking is that a rollback will be triggered during the conflict, therefore losing all the work done previously by the currently executing transaction. Rollbacks can be costly for the system as it needs to revert all current pending changes. This is suitable when database transaction conflicts due to concurrent record updates are infrequent and efficiency is a priority.

### Pessimistic locking

It assumes that conflicts between database transactions can happen often and locks the data record before reading or writing. This prevents other transactions from accessing the data until the current transaction is either committed or rolled back. In Rails, pessimistic Lock is based upon row-level locking. There are several [locking clauses](https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE) while `FOR UPDATE` is the default behavior in Rails. It causes the rows retrieved by the `SELECT` statement to be locked as though for update. This prevents them from being locked, modified, or deleted by other transactions until the current transaction ends.

Acquiring a database-level lock on a record within a transaction can be achieved using the `lock` method.

Let's take the same example where the first user is trying to update the `stock_quantity` of a record while the second user is trying to fetch the same record.

![Pessimistic locking](https://ik.imagekit.io/eapzn8piu/pessimistic-locking.gif?updatedAt=1710090099902)

While the first user's transaction was executing, it locked the record, preventing the second user from fetching it from the database. Only after the transaction was finished the second user able to retrieve the record.

This approach might be more suitable when conflicts happen frequently, as it reduces the chance of rolling back transactions.
