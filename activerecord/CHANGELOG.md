*   Make `group`/`regroup` inside `merge` be applied to the merged relation instead of the outermost relation.

    ```ruby
    Product.joins(:items).group(:id).merge(Item.group(:id))
    # SELECT "products".* FROM "products"
    # INNER JOIN "items" ON "items"."product_id" = "products"."id"
    # GROUP BY "products"."id", "items"."id"

    Product.joins(:items).group(:id).merge(Item.group(:title).regroup(:id))
    # SELECT "products".* FROM "products"
    # INNER JOIN "items" ON "items"."product_id" = "products"."id"
    # GROUP BY "products"."id", "items"."id"
    ```

    *João Marcos S B de Moraes*

*   Fix single quote escapes on default generated MySQL columns

Please check [8-0-stable](https://github.com/rails/rails/blob/8-0-stable/activerecord/CHANGELOG.md) for previous changes.
