### Motivation / Background

There are cases where `group` inside `merge` generates unexpected queries.

It generates a GROUP BY for the column of the source table instead of the joined table:

```rb
Product.joins(:items).group(:id).merge(Item.group(:id)).count
# SELECT COUNT(*) AS "count_all", "products"."id" AS "products_id"
# FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "products"."id"
```

As a corollary, using `regroup` inside a `merge` has the same effect:

```rb
Product.joins(:items).group(:id, :type).merge(Item.group(:type).regroup(:id)).count
# SELECT COUNT(*) AS "count_all", "products"."id" AS "products_id", "products"."title" AS "products_title"
# FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "products"."id", "products"."title"
```

It also generates a GROUP BY with the raw column name without specifying the table (when the source table has no column with that name), instead of properly scoping it to the joined table:

```rb
Product.joins(:items).merge(Item.group(:rank)).count
# SELECT COUNT(*) AS "count_all", "rank" AS "rank"
# FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "rank"
```

Expanding on the previous example, it leads to ambiguity errors as well, when the joined tables have a column with the same name as the column of the merged group. Suppose both `Item` and `Comment` has a column named `rank` but `Product` does not:

```rb
Product.joins(:items, :comments).merge(Item.group(:rank)).count
# SELECT COUNT(*) AS "count_all", "rank" AS "rank"
# FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# INNER JOIN "comments" ON "comments"."product_id" = "products"."id"
# GROUP BY "rank"
#
# PG::AmbiguousColumn: ERROR:  column reference "rank" is ambiguous (ActiveRecord::StatementInvalid)
```

### Detail

Now, `group` inside `merge` is applied to the merged relation as intended. The behavior was changed for `regroup` as well:

```rb
Product.joins(:items).merge(Item.group(:id)).count
# SELECT COUNT(*) AS "count_all", "products"."id" AS "products_id"
# FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "products"."id"

Product.joins(:items).group(:id, :type).merge(Item.group(:type).regroup(:id)).count
# SELECT COUNT(*) AS "count_all", "products"."type" AS "products_type", "products"."id" AS "products_id"
# FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "products"."id", "products"."type"

Product.joins(:items).merge(Item.group(:rank)).count
# SELECT COUNT(*) AS "count_all", "rank" AS "rank"
# FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "rank"

Product.joins(:items, :comments).merge(Item.group(:rank)).count
# SELECT COUNT(*) AS "count_all", "rank" AS "rank"
# FROM "products"
# INNER JOIN "items" ON "items"."product_id" = "products"."id"
# GROUP BY "rank"
#
# PG::AmbiguousColumn: ERROR:  column reference "rank" is ambiguous (ActiveRecord::StatementInvalid)
```

### Checklist

Before submitting the PR make sure the following are checked:

* [x] This Pull Request is related to one change. Changes that are unrelated should be opened in separate PRs.
* [x] Commit message has a detailed description of what changed and why. If this PR fixes a related issue include it in the commit message. Ex: `[Fix #issue-number]`
* [x] Tests are added or updated if you fix a bug or add a feature.
* [x] CHANGELOG files are updated for the changed libraries if there is a behavior change or additional feature. Minor bug fixes and documentation changes should not be included.
