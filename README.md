# cdc neo4j

## Json message schema example from kafka topic

```
{
  "before": null,
  "after": {
    "mysql_connector.demo.customers.Value": {
      "customerNumber": 103,
      "customerName": "Atelier graphique",
      "contactLastName": "Schmitt",
      "contactFirstName": "Carine ",
      "phone": "40.32.2555",
      "addressLine1": "54, rue Royale",
      "addressLine2": null,
      "city": "Nantes",
      "state": null,
      "postalCode": {
        "string": "44000"
      },
      "country": "France",
      "salesRepEmployeeNumber": {
        "int": 1370
      },
      "creditLimit": {
        "double": 21000.0
      }
    }
  },
  "source": {
    "version": "1.9.3.Final",
    "connector": "mysql",
    "name": "mysql_connector",
    "ts_ms": 1656420989704,
    "snapshot": {
      "string": "true"
    },
    "db": "demo",
    "sequence": null,
    "table": {
      "string": "customers"
    },
    "server_id": 0,
    "gtid": null,
    "file": "binlog.000002",
    "pos": 32307,
    "row": 0,
    "thread": null,
    "query": null
  },
  "op": "r",
  "ts_ms": {
    "long": 1656420989707
  },
  "transaction": null
}
```
The information useful for the injection are, most of the time, defined inside the fields before, after and op. This last field is really important because is what tells us the type of event that happened in the database (in this case, op is equal to r, meaning that the operation is of type read (snapshot of the database)).

## And the cypher query from message
```
WITH event CALL {
  WITH event
  WITH event
  WHERE event.op IN ["c", "u", "r"]
  WITH event["after"] AS evData
  MERGE (c:Customer{customerId:evData.customerNumber})
  ON CREATE SET c.name = evData.customerName
  ON MATCH SET c.name = evData.customerName
  UNION
  WITH event
  WITH event
  WHERE event.op IN ["d"]
  WITH event["before"] AS evData
  MATCH (c:Customer{customerId:evData.customerNumber})
  WITH c
  OPTIONAL MATCH (c)-[:PLACED_ORDER]->(o)
  DETACH DELETE c, o }
  ```
Because we are handling the sink ingestion using the Cypher strategy, we need to manage, for each topic, each type of operation (CRUD). To understand better this concept, here the Cypher query defined for the messages coming from the topic customers.

# Example of event handling
Insert to database 
```
INSERT INTO `customers`(`customerNumber`,`customerName`,`contactLastName`,`contactFirstName`,`phone`,`addressLine1`,`addressLine2`,`city`,`state`,`postalCode`,`country`,`salesRepEmployeeNumber`,`creditLimit`) VALUE
(500,'foo','bar','uml','40.32.2555','54, rue Royale',NULL,'Nantes',NULL,'44000','France',1370,'21000.00');
```
After the creation, message will create a new event of type c and will submit it to the topic customers. We can see the message from the Confluent Control Center instance:

<img width="733" alt="image" src="https://user-images.githubusercontent.com/77326619/229351946-24ba7ed3-173a-4df0-b130-63881200e8d1.png">

## see node in neo4j
 ```
neo4j@neo4j> MATCH (c:Customer{customerId:500}) RETURN c;
+--------------------------------------------+
| c                                          |
+--------------------------------------------+
| (:Customer {name: "foo", customerId: 500}) |
+--------------------------------------------+
```

Let’s create an order for this customer, to see if the query can handle correctly its creation. Like before, we need to add a new row to a table in the demo database, in this case the table orders.
```
INSERT INTO `orders`(`orderNumber`,`orderDate`,`requiredDate`,`shippedDate`,`status`,`comments`,`customerNumber`) VALUE
(20000,'2003-01-06','2003-01-13','2003-01-10','Created',NULL,500);
```
Like earlier, we can see (now from Neo4j Browser for clarity) that the query created both the node with label Order and the relationship of type PLACED_ORDER with its customer, using the foreign key contained in the message.

<img width="521" alt="image" src="https://user-images.githubusercontent.com/77326619/229352035-e7b8f638-672b-4364-9177-a5d468ee724f.png">

```
Figure 2. The customer and its order
The order we just created has status Created, but suppose that, at some point, the order is shipped, and we want to change its status to Shipped. We can update the order from MySql with this query:
```
UPDATE `orders` SET status='Shipped' WHERE orderNumber=20000;
```
We can see that the node related to the order with orderNumber=20000 has now changed status to Shipped.
```
neo4j@neo4j> MATCH (o:Order{orderId:20000}) RETURN o;
+----------------------------------------------+
| o                                            |
+----------------------------------------------+
| (:Order {orderId: 20000, status: "Shipped"}) |
+----------------------------------------------+
```
To show the last possible operation, the deletion of a row, let’s delete the customer we created earlier:
```
DELETE FROM `customers` WHERE customerNumber=500;
```
We can see that now, inside our graph, the customer and its order are not present, because the connector managed the deletion of the nodes related to the deleted rows.
```
neo4j@neo4j> MATCH (c:Customer{customerId:500}) RETURN c;
+---+
| c |
+---+
+---+

0 rows
ready to start consuming query after 2 ms, results consumed after another 1 ms
neo4j@neo4j> MATCH (o:Order{orderId:20000}) RETURN o;
+---+
| o |
+---+
+---+

0 rows
ready to start consuming query after 2 ms, results consumed after another 1 ms
```


