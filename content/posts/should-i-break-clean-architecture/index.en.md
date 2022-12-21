---
title: "Should I Break Clean Architecture?"
date: 2022-09-10T16:44:34-03:00
draft: true
author: "Evandro Millian"
resources:
- name: "featured-image"
  src: "featured-image.webp"
- name: "featured-image-preview"
  src: "featured-image-preview.webp"
tags: ["AWS", "Design", "Matchmaker"]
categories: ["System Development"]
lightgallery: true
---

Currently I'm working on a new version of the matchmaker that we used with Myridian: The Last Stand. The idea is to use all the best practices and frameworks, as well as to reduce the vendor lock-in from cloud providers, in this case some AWS services like DynamoDB and SQS.

### DynamoDB vs Redis

We already have an application, then we're separating the code accordingly with Clean Architeture concepts:

{{<image src="https://cdn-media-1.freecodecamp.org/images/lbexLhWvRfpexSV0lSIWczkHd5KdszeDy9a3" caption="Diagram from [freeCodeCamp post](https://www.freecodecamp.org/news/a-quick-introduction-to-clean-architecture-990c014448d2/)" width="100%">}}

So, we are creating projects for adapter, repository, and application code. And the first objective is to allow [Redis](https://redis.io) to be an database alternative to [DynamoDB](https://aws.amazon.com/pt/dynamodb/).
 
### Why Redis?

Of course this is the first question anybody will ask. DynamoDB is a proprietary NoSQL database managed by AWS, while Redis is a key-value store, but it's so very versatile that it's currently used for caching, session management and message broker, among many other tasks.

And to be clear, I choose to work with Redis due to its features, documentation, and also to be widely available from many cloud and infrastructure providers. Of course there are well known options like MongoDB, and other not so known like ScyllaDB (that has many performance benefits).

With this explained, then we start the integration. To connect the database to the other systems, it's wrapped by a **Adapter** port, that was developed to wrap DynamoDB internal behavior: 

```
export interface DatabaseAdapterPort {

    getByKey(keys: Key, consistent: boolean): Promise<Item>;
    create(obj: Item): Promise<void>;
    update(key: Key, update: UpdateExpressionBuilder): Promise<Item>;
    deleteByKey(entityKey: Key): Promise<void>;
    batchCreate(items: Item[]): Promise<number>;
    batchDelete(entityKeys: Key[]): Promise<number>;
    query(query: QueryExpressionBuilder, index_name?: string): Promise<Item[]>;
    queryByField(field: string, value: any, index_name?: string): Promise<Item[]>;
    queryByFields(fields: any, index_name?: string): Promise<Item[]>;
    scan(): Promise<Item[]>;
    executeTransaction(transaction: {
        createKeys?: Item[];
        deleteKeys?: Item[];
        updateKeys?: { key: Key, builder: UpdateExpressionBuilder }[];
    }): Promise<void>;
    createQueryExpressionBuilder(): QueryExpressionBuilder;
    createUpdateExpressionBuilder(): UpdateExpressionBuilder;
}
```

The **Item** object is just a Typescript dictionary (a type definition for `{ [key: string]: string }`), and **Key** is an object with pk and sk fields, to mimic a DynamoDB Hash and Sort keys. 

For the Redis port, the **Item** object can be mapped to Redis **Hash** object, and **Key** fields can be concatenated to be the hash key, defining the basics for entity creation, fetching and deleting.

### First problem: Query

Then the first problems arises with the implementation of **query** functions. Being a key-value database, Redis doesn't have a query API. There's a package called [RediSearch](https://redis.io/docs/stack/search/), but it's not available from the versions available to the majority of the cloud providers, so I don't want to be dependant of a external package. 

The **QueryExpressionBuilder** interface is:

```
export interface QueryExpressionBuilder {

    conditionBetween(name: string, leftValue: string | number, rightValue: string | number): QueryExpressionBuilder;
    conditionAttributeExists(name: string): QueryExpressionBuilder;
    conditionAttributeNotExists(name: string): QueryExpressionBuilder;
    conditionBeginsWith(name: string, value: string): QueryExpressionBuilder;
    conditionAnd(): QueryExpressionBuilder;
    conditionOr(): QueryExpressionBuilder;
    compareWithValue(name: string, value: string | number, operator: ComparisonOperators): QueryExpressionBuilder;
    compareWithAttribute(name: string, attrName: string, operator: ComparisonOperators): QueryExpressionBuilder;
    parseOperator(operator: ComparisonOperators): string;
}
```



### Second problem: Indexes


### Third problem: Leaderboards





