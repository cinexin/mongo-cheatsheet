# MongoDB useful queries cheatsheet

## Regular expressions

```sql
SELECT *
FROM Products p
WHERE p.description LIKE '%BOOK%'
```

ie: Fetch the products which description contains "BOOK" string

```javascript
db.getCollection('Products').find({"description": /.*BOOK.*/})
```

## Get elements that are "in" / "not in" a nested collection

```sql
SELECT *
FROM ShoppingCarts s, ShoppingCartsProducts sp
WHERE s.id = sp.shoppingCartId
    AND sp.productId = "5cd1a0475334fe0009133102"
```

ie: Fetch the shopping carts in which list of products "5cd1a0475334fe0009133102" is / is not contained

```javascript
db.getCollection('ShoppingCarts').find({"shoppingCartProducts": { $in: ["5cd1a0475334fe0009133102"]}})
```

```javascript
db.getCollection('ShoppingCarts').find({"shoppingCartProducts": { $nin: ["5cd1a0475334fe0009133102"]}})
```

ie: Fetch the wallets in which list of owners "5cd1a0475334fe0009133102" is not contained

## Filter by a field in a nested object

```javascript
db.getCollection('Accounts').find({'mobile.number': '652272058'})
```

ie: Fetch the accounts which "mobile -> number" is "652272058"

## Using boolean operators (and, or...)

```sql
SELECT *
FROM Accounts
WHERE "firstName" = "Migue" OR "firstName" = "Angel"
```

ie: Get the accounts which firstName 'Migue' or 'Angel'

```javascript
db.getCollection('Accounts').find({$or: [
    {'firstName': 'Migue'},
    {'firstName':'Angel'}
]})
```

```sql
SELECT *
FROM ShoppingCarts sp,
WHERE EXISTS (
    SELECT *
    FROM ShoppingCartsOwners spo
    WHERE sp.id = spo.shoppingCartId
        AND spo.ownerId IN (SELECT "5cd1a0475334fe0009133102" FROM DUAL)
) AND EXISTS (
    SELECT *
    FROM ShoppingCartsOwners spo2
    WHERE sp.id = spo2.shoppingCartId
    GROUP BY spo2.shoppingCartId
    HAVING COUNT(*) = 2
)
```

```javascript
db.getCollection("ShoppingCarts").find({$and: [
    {"shoppingCartsOwners": { $in: ["5cd1a0475334fe0009133102"]}},
    {"shoppingCartsOwners": {$size: 2}}
]})
```

ie: Get the shopping carts which "5cd1a0475334fe0009133102" is on its owners list, and its owners list is size 2

## Select only a set of fields (projection)

```sql
SELECT createdAt, updatedAt
FROM Accounts
```

```javascript
db.getCollection('Accounts').find(
    {},
    {"_id": 0, "createdAt": 1, "updatedAt": 1}
)
```

ie: Display only the "createdAt" and "updatedAt" fields of "appData" document

```sql
SELECT s.id, sp.*
FROM ShoppingCarts s, ShoppingCartProducts sp
WHERE s.id = sp.shoppingCartId
```

```javascript
db.ShoppingCarts.find({}, {shoppingCartProducts: 1})
```

ie: Display only the "storedProducts" field of "inventory" document

_Note that mongo "\_id" attribute will be always in the list of documents unless we decide not to include it. Ie: {\_id: 0}_

## Select records which some collection is empty / not empty

```javascript
db.ShoppingCarts.find({ shoppingCartProducts: { $exists: true, $not: {$size: 0} } })
```

ie: Find inventories which products list is not empty

```javascript
db.ShoppingCarts.find({ shoppingCartProducts: { $exists: true, $size: 0 } })
```

ie: Find inventories which products list is empty

## Select "Distinct"

```sql
SELECT DISTINCT(sp.productId)
FROM ShoppingCartProducts sp
WHERE s.id = sp.shoppingCartId AND s.status = "CONFIRMED"

```

ie: Fetch the distinct product Id's of shopping carts which status is "CONFIRMED"

```javascript
db.ShoppingCarts.distinct("productId", {status: "CONFIRMED"})
```

## Group by / Having clauses (Aggregations in Mongo)

```sql
SELECT *
FROM ShoppingCarts
GROUP BY "month", "year"
HAVING COUNT(*) > 1

```

ie: Fetch all records from "ShoppingCartProducts" document grouping by fields: "productId", "month" and "year" having its count > 1

```javascript
db.ShoppingCarts
    .aggregate(
        [
            {
                $group: {_id: {
                    month: "$month",
                    year: "$year"
                },
                count:{
                    $sum:1
                    }
                }
            }
            ,  
            {
                $match: {
                    count: { $gt: 1 }
                }
            }
        ])
```

ie: Group shopping carts by last updated timestamp date and status and calculate the count

```sql
SELECT *
FROM ShoppingCarts
GROUP BY TRUNC(lastUpdatedTS)
HAVING COUNT(*) > 1

```

```javascript
db.ShoppingCarts.aggregate([
    {
        "$group": {
            _id: {
                lastUpdatedUTC: {
                    $dateToString: { format: "%Y-%m-%d", date: "$lastUpdatedTimestamp" }
                },
                status: "$status"
            },
            count: { $sum : 1}
        }
    }
]);
```

## Get all records which elements in an embedded collection match some condition

ie: Give me the ACTIVE Subscriptions that have a promo with code = "PROMO 300" on its list of promos

```sql
    SELECT *
    FROM Subscriptions s
    WHERE s.status = 'ACTIVE'
        AND
        EXISTS (SELECT *
                FROM PROMOTIONS p
                WHERE
                    s.id = p.subscriptionId
                    AND
                    p.code = "PROMO 300"
                )
```

```javascript
db.Subscriptions.find(
    {
        status: 'ACTIVE',
        promotions: {$elemMatch: {"promotionCode": "PROMO 300"}}
    }
);
```

ie: Give me the ACTIVE subscriptions that have a promo with priority > 20 on its list of promos

```sql
    SELECT *
    FROM Subscriptions s
    WHERE s.status = 'ACTIVE'
        AND
        EXISTS (SELECT *
                FROM PROMOTIONS p
                WHERE
                    s.id = p.subscriptionId
                    AND
                    p.priority > 20
                )
```

```javascript
db.Subscriptions.find(
    {
        status: 'ACTIVE',
        promotions: {$elemMatch: {"priority": {$gt: 20}}}
    }
);
```

## Update collection based on some criteria

ie: Update the Products which name contains _Extended_:

```sql
    UPDATE Products
    SET hidden = FALSE
    WHERE name LIKE '%EXTENDED%';
```

```javascript
db.Products.updateMany(
    {name: /Extended/},
    {$set: {hidden: true}}
);
```

## Update fields of embedded object(s) based on some criteria

ie: Update benefit status of promotions that match some criteria

```sql
UPDATE Subscriptions s, PROMOTIONS p
SET p.status = 'BENEFIT_FULFILLED_OK'
WHERE s.promoId = promos.id
    AND s.status = 'ACTIVE'
    AND p.status = 'PENDING_CONDITION_COMPLETION';
```

````javascript
db.Products.updateMany(
    {
        status: 'ACTIVE',
        promotions: {
            $elemMatch: {
                "status": "PENDING_CONDITION_COMPLETION"
            }
        }
    },
    {
        $set: {
            "promotions.$.status": "BENEFIT_FULFILLED_OK"
        }
    }
);
````

## Update fields of all embedded document(s) (no criteria)

ie: Update all benefit status of promos

```sql
UPDATE SUBSCRIPTIONS s, PROMOTIONS p
SET p.benefit.status = 'BENEFIT_FULFILLED_OK'
WHERE s.id = p.subscriptionId
    AND s.status = 'ACTIVE'
```

```javascript
db.subscriptions.updateMany(
    {
        status: 'ACTIVE'
    },
    {
        $set: {
            "promotions.$[].benefit.status": "BENEFIT_FULFILLED_OK"
        }
    }
);
```

ie: Set end date of all promos of subscriptions of accountId = ...

```sql
UPDATE SUBSCRIPTIONS s, PROMOTIONS p
SET p.endDate = DATE()
WHERE
    s.accountId = '5d4e6cb4e015f20001999e46'
    AND
    s.subscriptionId = p.subscriptionId
    AND s.status = 'ACTIVE'
```

```javascript
db.subscriptions.update(
    {
        accountId: "5d4e6cb4e015f20001999e46",
        status: 'ACTIVE'
    },
    {
        $set: {
            "promotions.$[].endDate": new Date()
        }
    }
);
```
