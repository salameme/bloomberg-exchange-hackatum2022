# Tasks

- order book server (OBS) starts
  - OBS subscribes to trade channel
  - -"- subscribes to order channel
  - -"- listens on socket
- matching server (MS) starts
  - MS subscribes to order channel
  - on new order event, check if trade is possible, and if so, update DB and publish new event on trading channel
- client connects to socket
  - server sends unique user ID maintained for the socket session (makes it simpler for us so we don't need to implement authentication)
  - -"- SELECTs and forwards latest aggregated order book and last n trades from relational DB (e.g. Postgres with Timescale extension; throttled/cached to 1 req/X ms)
  - -"- forwards all events from the trade channel
  - -"- on event in order channel, SELECT latest aggregated order book and forward (throttled/cached to 1 req/X ms)
  - -"- listens for order commands (add immediate or limit order; delete order)
- client with ID=X sends order command
  - OBS updates DB
  - -"- publishes event in order channel (which the matching server and all clients are subscribed to)

# Stack

## Server
- Rust, Axum for WebSocket handling
- PostgreSQL with Timescale extension
- Redis as our Pub/Sub broker

## Client
- Svelte web app with WebSockets

# Tables

## Orders
```
id string (uuid or nano id)
timestamp int (unix timestamp)
user_id string (random maybe in pattern user#123)
type enum ('bid', 'ask')
exec_type enum ('market', 'limit')
amount float
price float (optional, only for limit orders)
```

## Trades
```
id string (uuid or nano id)
timestamp int (unix timestamp)
user_id string
amount float
avg_price float (when multiple entries from order book needed to fulfil trade)
```

## Queries

**addOrder(args[])** (executed by OBS)
```sql
INSERT INTO orders VALUES (id, timestamp, user_id, type, exec_type, amount, price);
```

**addTrade(args[])** (executed by MS)
```sql
INSERT INTO trades VALUES (id, timestamp, user_id, amount, avg_price);
```

**getLatestTrades(number int)** (only queried on client init and trade event by OBS)
```sql
SELECT timestamp, user_id, amount, avg_price FROM trades ORDER BY timestamp DESC LIMIT $number;
```

**getAggOrderBook(number int)** (only queried on client init and order event by OBS)
```sql
SELECT price, amount, price * amount AS total FROM orders WHERE type = 'ask' GROUP BY ...;TODO
```

**TODO()**
```sql

```

# API

- Single websocket endpoint (server view)
  - onOpen: OBS, initializes connection with new user id, sends aggregated order book and last n trades
  - onMessage: OBS checks and executes command (add, delete)
  - onClose: clean up connection, delete user ID

## Structs

```
aggOrderBook: {
  timestamp int,
  ask: [{
      price float
      amount float
      total float
  }],
  bid: [{
      price float
      amount float
      total float
  }]
}

trades: [{
    userId string
    security string
    timestamp int
    price float
    amount float
    total float
}]

```