### Setup
1. Add GraphQL Api
    1. Update name to `Trades Api`
    1. Update schema location to `trades/schema.graphql`
    1. Paste the following schema:
        ```graphql
        type Mutation {
          createTrade(type: TransactionType!, symbol: String!): Trade
        }

        type Price {
          symbol: String!
          price: Float!
        }

        type Query {
          getPrice(symbol: String!): Price
          getTrade(id: ID!): [Trade]
          listTrades(limit: Int, nextToken: String): TradesConnection
        }

        type Subscription {
          onCreateTrade(
            id: ID,
            symbol: String,
            timestamp: AWSDateTime,
            price: Float,
            type: String
          ): Trade
            @aws_subscribe(mutations: ["createTrade"])
        }

        type Trade {
          id: ID!
          type: TransactionType!
          symbol: String!
          timestamp: AWSDateTime!
        }

        type TradesConnection {
          items: [Trade]
          nextToken: String
        }

        enum TransactionType {
          BUY
          SELL
        }
        ```
    1. Attach resolvers to:
        * Query getPrice
        * Query getTrade
        * Query listTrades
        * Mutation createTrade
1. Add an HTTP Proxy Endpoint
1. Connect the Query getPrice resolver to the HTTP Proxy Endpoint
1. Update the HTTP Proxy Endpoint Host to `https://www.alphavantage.co`
1. Add a DynamoDB Table
1. Connect the rest of the resolvers to it
1. Update the DynamoDB Table Logical ID to `Trades`
1. Edit Query getPrice resolver
    * Request Template:
        ```vtl
        {
          "version": "2018-05-29",
          "method": "GET",
          "resourcePath": "/query",
          "params":{
              "query": {
                  "function": "GLOBAL_QUOTE",
                    "symbol": $util.toJson($ctx.args.symbol),
                    "apikey": "5X9X1G1HBV4VA6B1"
              }
          }
        }
        ```
    * Response Template:
        ```vtl
        #if($ctx.error)
          $util.error($ctx.error.message, $ctx.error.type)
        #end
        #if($ctx.result.statusCode == 200)
          {
            "symbol": $util.toJson($ctx.args.symbol),
            "price": $util.parseJson($ctx.result.body)["Global Quote"]["05. price"]
          }
        #else
            $utils.appendError($ctx.result.body, $ctx.result.statusCode)
        #end
        ```
1. Edit Mutation createTrade
    * Request Template:
        ```vtl
        {
            "version" : "2017-02-28",
            "operation" : "PutItem",
            "key" : {
                ## If object "id" should come from GraphQL arguments, change to $util.dynamodb.toDynamoDBJson($ctx.args.id)
                "id": $util.dynamodb.toDynamoDBJson($util.autoId()),
                "timestamp": $util.dynamodb.toDynamoDBJson($util.time.nowISO8601())
            },
            "attributeValues" : {
              "type": $util.dynamodb.toDynamoDBJson($ctx.args.type),
              "symbol": $util.dynamodb.toDynamoDBJson($context.arguments.symbol)
            }
        }
        ```
1. Edit Query getTrade
    * Request Template:
        ```vtl
        {
            "version": "2017-02-28",
            "operation": "GetItem",
            "key": {
                "id": $util.dynamodb.toDynamoDBJson($ctx.args.id),
            }
        }
1. Edit Query listTrades
    * Request Template:
        ```vtl
        {
          "version": "2017-02-28",
          "operation": "Scan",
          "limit": $util.defaultIfNull($ctx.args.limit, 20),
          "nextToken": $util.toJson($util.defaultIfNullOrEmpty($ctx.args.nextToken, null)),
        }
        ```
1. Deploy!

### Queries
* Get Price
    ```graphql
    query getPrice {
      getPrice(symbol: "NTRS") {
        symbol
        price
      }
    }
    ```
* Trade Subscription (in separate window)
    ```graphql
    subscription onCreateTrade {
      onCreateTrade {
        id
        timestamp
        type
        symbol
      }
    }
    ```
* Create Trade
    ```graphql
    mutation createTrade {
      createTrade(type: "BUY", symbol: "NTRS") {
        id
        timestamp
        type
        symbol
      }
    }
    ```
* List Trades
    ```graphql
    query listTrades {
      listTrades {
        items {
          id
          timestamp
          type
          symbol
        }
      }
    }
    ```
