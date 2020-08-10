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

### Migrations

```javascript
exports.up = function up(knex) {
  return knex.schema.raw(`
    CREATE TABLE IF NOT EXISTS customer_reviews
    (
      id SERIAL PRIMARY KEY,
      business_id INTEGER REFERENCES users(id) ON DELETE CASCADE NOT NULL,
      customer_id INTEGER REFERENCES users(id) ON DELETE CASCADE NOT NULL,
      order_id INTEGER REFERENCES orders(id) ON DELETE CASCADE NOT NULL,
      review TEXT,
      rating INTEGER NOT NULL,
      added_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
    );

    CREATE UNIQUE INDEX business_id_customer_id_order_id_customer_reviews_uindex
    ON customer_reviews (business_id, customer_id, order_id);

    CREATE INDEX customer_id_customer_reviews_uindex
	  ON customer_reviews (customer_id);
  `);
};

exports.down = function down(knex) {
  return knex.schema.raw(`
    DROP TABLE IF EXISTS customer_reviews;
    DROP INDEX business_id_customer_id_order_id_customer_reviews_uindex;
    DROP INDEX customer_id_customer_reviews_uindex
  `);
};
```

```javascript
exports.up = function up(knex) {
  return knex.schema.raw(`
    ALTER TABLE vehicle_manufacturers ADD COLUMN IF NOT EXISTS fts tsvector;

    CREATE OR REPLACE FUNCTION fts_update()
    RETURNS trigger
    LANGUAGE plpgsql
    AS $function$
    BEGIN
      IF (TG_OP = 'UPDATE') THEN
          IF (OLD.name <> NEW.name) THEN
            NEW.fts = setweight(coalesce(to_tsvector('english', NEW.name),''),'A');
            RETURN NEW;
          ELSE
            RETURN NEW;
          END IF;
      ELSIF (TG_OP = 'INSERT') THEN
        NEW.fts = setweight(coalesce(to_tsvector('english', NEW.name),''),'A');
        RETURN NEW;
      END IF;
    END;
    $function$;

    CREATE INDEX manufacturers_fts on vehicle_manufacturers using GIN(fts);

    CREATE TRIGGER manufacturers_fts_update BEFORE INSERT OR UPDATE ON vehicle_manufacturers FOR EACH ROW EXECUTE PROCEDURE fts_update();

    UPDATE vehicle_manufacturers
      SET "fts" = setweight( coalesce( to_tsvector('english', "name"),''),'A');
  `);
};

exports.down = function down(knex) {
  return knex.schema.raw(`
    ALTER TABLE manufacturers DROP COLUMN fts;
    DROP FUNCTION IF EXISTS fts_update();
    DROP TRIGGER fts_update;
    DROP INDEX manufacturers_name_trgm_idx;
    DROP INDEX manufacturers_fts;
  `);
};
```
