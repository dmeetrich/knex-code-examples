## KnexJS code examples

```javascript
function getUnpaidOrdersHourAgo() {
  const hourBeforeNow = moment().subtract(1, "hour").toISOString();
  const twoHourBeforeNow = moment().subtract(2, "hour").toISOString();

  return knex("orders")
    .select("orders.*")
    .leftJoin("order_payments AS op", "op.order_id", "orders.id")
    .whereBetween("orders.addedAt", [twoHourBeforeNow, hourBeforeNow])
    .andWhere({ isDeleted: false })
    .andWhereRaw(
      `(op.status <> '${SUCCESS_PAYMENT_STATUS}' OR op.status IS NULL)`
    );
}
```

```javascript
function getOrderAudios(userId: number, orderId: number) {
  return knex("order_audios")
    .select([
      ...getSelectFields("order_audios", ["id", "audioUri"]),
      knex.raw(`json_build_object(
          'id', pt.id,
          'name', pt.name) as "productType"`),
    ])
    .innerJoin("order_items AS oi", "order_audios.order_item_id", "oi.id")
    .leftJoin("products as p", "p.id", "oi.product_id")
    .leftJoin("product_types as pt", "pt.id", "p.type_id")
    .where("oi.order_id", orderId)
    .andWhere("user_id", userId)
    .orderBy(["order_audios.addedAt", "pt.id"]);
}
```
