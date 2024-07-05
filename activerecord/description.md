### Motivation / Background

There are cases where `group` inside `merge` generates unexpected queries.

Grouping by a column that has the same name in both the main and the merged model

```rb
Product.joins(:items).merge(Item.group(:id)).count
# SELECT COUNT(*) AS "count_all", "products"."id" AS "products_id"
# FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "products"."id"
```
Using `group` with a column that only exists in the merged model generates a GROUP BY with the raw column name instead of properly scoping the column to the table:

```rb
Product.joins(:items).merge(Item.group(:rank)).count
# SELECT COUNT(*) AS "count_all", "rank" AS "rank"
# FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "rank"
```

Expanding on the previous example, it leads to ambiguity errors as well, when the other joined table or tables have a column with the same name as the column of the merged group. In the example below, both `Item` and `Comment` has a column named `rank`:

```rb
Product.joins(:items, :comments).merge(Item.group(:rank)).count
# SELECT COUNT(*) AS "count_all", "rank" AS "rank"
# FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "rank"
#
# PG::AmbiguousColumn: ERROR:  column reference "rank" is ambiguous (ActiveRecord::StatementInvalid)
```

Furthermore, `regroup` seems not to be allowed inside `merge`:

```rb
Product.joins(:items).merge(Item.regroup(:rank)).count
# undefined method `regroup' for Item:Class (NoMethodError)
# Did you mean?  group
```

### Detail

Now, `group` inside `merge` is applied to the merged relation, as intended. The behavior was changed for `regroup` as well, and documented.

Before:

```rb
Product.joins(:items).group(:id).merge(Item.group(:title))
# SELECT "products".* FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "products"."id", "products"."title"

Product.joins(:items).group(:id).merge(Item.group(:title).regroup(:id))
# SELECT "products".* FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "products"."id"

Product.joins(:items).group(:id).merge(Item.group(:title).regroup(:id)).regroup(:title)
# SELECT "products".* FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "products"."title"
```

After:
```rb
Product.joins(:items).group(:id).merge(Item.group(:title))
# SELECT "products".* FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "products"."id", "items"."title"

Product.joins(:items).group(:id).merge(Item.group(:title).regroup(:id))
# SELECT "products".* FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "products"."id", "items"."id"

Product.joins(:items).group(:id).merge(Item.group(:title).regroup(:id)).regroup(:title)
# SELECT "products".* FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "products"."title"
```

### Checklist

Before submitting the PR make sure the following are checked:

* [x] This Pull Request is related to one change. Changes that are unrelated should be opened in separate PRs.
* [x] Commit message has a detailed description of what changed and why. If this PR fixes a related issue include it in the commit message. Ex: `[Fix #issue-number]`
* [x] Tests are added or updated if you fix a bug or add a feature.
* [x] CHANGELOG files are updated for the changed libraries if there is a behavior change or additional feature. Minor bug fixes and documentation changes should not be included.
