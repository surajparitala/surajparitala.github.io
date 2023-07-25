---
title: AWS Serverless
date: 2023-07-21 05:40:00 +0800
categories: [aws, serverless, sns, sqs, dynamodb]
tags: [aws, serverless, sns, sqs, dynamodb]
layout: post
---

# AWS Serverless

This post explains about how to configure AWS Lambda Serverless with different scenarios in JAVA. We would first need to configure SAM (Serverless Application Model) CLI In our machine follow the [link](https://docs.aws.amazon.com/serverless-application-model/latest/developerguide/install-sam-cli.html#install-sam-cli-instructions).

- **NOTE: Before doing all this sam commands create a user role in AWS environment and use it configure your local AWS permissions. `aws configure` can be used and can input api key and value as prompted which will provide access AWS to deploy from local.**

### SAM CLI Commands

- SAM CLI is build on top of AWS CLI for easier deployment of serveless functions to AWS. To do that we need some basic commands which are listed below:

  1. To Create a New Project we first need to open terminal in desired folder and type `sam init` which will prompt for some selections first choose `AWS Quick Start Templates` and then choose any quickstart template such as `Hello World Example` to start with first.
  2. When asked `Use the most popular runtime and package type? (Python and zip) [y/N]:` choose **N** and later when prompted choose desired **Java Platform** such as `java11` and choose package type `Zip` and as a last step choose your build system `Maven`.
  3. Give your project a desired name `test-app`. For any other steps you can provide input as **N**.

- After we add required code we may need to build the project so that SAM moves all required Java classes, Library files and template yaml file to `.aws-sam` folder. For that we can use `sam build`

- Once build is successful run `sam deploy --guided` for the first time to setup `samconfig.toml` file once this is setup there is no need to run **--guided** the next time. Since required configuration for deployment has already been configured initially.

  1. First input stack-name `test-app` when prompted.
  2. Choose Region `us-east-1` or any other preferred location.
  3. `Confirm changes before deploy` select `N`
  4. `Allow SAM CLI IAM role creation` select `Y` for IAM Role creation.
  5. `Disable rollback` select as `N`
  6. `HelloWorldFunction has no authentication. Is this okay?` select `Y` for now since we don't need authntication for now.
  7. When prompted `Save arguments to configuration file` select `Y` at this step it will save all the configuration to **samconfig.toml** file. You can name your file as you desire but I would leave it at default.
  8. SAM Configuration environment choose as `default`

### POM Dependencies

- Lambda Core and Java Events is required for lambda functioning as well as to retrieve event and process accordingly.
- FasterXML is needed for data parsing and mapping to JAVA POJO.
- DynamoDB SDK dependency is needed for interacting with DynamoDB Table.

```xml
    <dependencies>
        <dependency>
            <groupId>com.amazonaws</groupId>
            <artifactId>aws-lambda-java-core</artifactId>
            <version>1.2.2</version>
        </dependency>
        <dependency>
          <groupId>com.amazonaws</groupId>
          <artifactId>aws-lambda-java-events</artifactId>
          <version>3.11.0</version>
        </dependency>
      <dependency>
        <groupId>com.amazonaws</groupId>
        <artifactId>aws-java-sdk-dynamodb</artifactId>
        <version>1.12.511</version>
      </dependency>
      <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-core</artifactId>
        <version>2.15.2</version>
      </dependency>
      <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.15.2</version>
      </dependency>
    </dependencies>
```

- Also we need build plugin `Maven Shade` whose primary purpose is to create a self-contained JAR that includes all the required dependencies, classes, and resources from the project and its dependencies. This is often referred to as a "shaded" or "fat" JAR because it contains everything needed to run the application in a single, standalone package.

```xml
<build>
      <plugins>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-shade-plugin</artifactId>
          <version>3.2.4</version>
          <configuration>
          </configuration>
          <executions>
            <execution>
              <phase>package</phase>
              <goals>
                <goal>shade</goal>
              </goals>
            </execution>
          </executions>
        </plugin>
      </plugins>
    </build>
```

### Java Project

- First step is to create a JAVA POJO for transferring required data.
  Here's an example:

```java
public class Order {
  public int id;
  public String itemName;

  public Order() {
  }

  public Order(int id, String itemName) {
    this.id = id;
    this.itemName = itemName;
  }
}
```

- Next Step is to create `CreateOrderLambda` function handler itself for which method argument would be `APIGatewayProxyRequestEvent` which would have all the related data regarding event from which we get request body and parse it with Order class.
- Later process data into dynamodb document. we will see how we created the table and association to lambda in next section.

```java
public class CreateOrderLambda {
  private final ObjectMapper objectMapper = new ObjectMapper();
  private final DynamoDB dynamoDB = new DynamoDB(AmazonDynamoDBClientBuilder.defaultClient());
  public APIGatewayProxyResponseEvent createOrder(APIGatewayProxyRequestEvent request)
      throws JsonProcessingException {
    Order order = objectMapper.readValue(request.getBody(), Order.class);
    Table ordersTable = dynamoDB.getTable(System.getenv("ORDERS_TABLE"));
    ordersTable.putItem(new Item().withPrimaryKey("id", order.id)
                                  .withString("itemName", order.itemName));
    return new APIGatewayProxyResponseEvent().withStatusCode(200).withBody("Order ID:"+ order.id);
  }
}
```

- Next Step is to create `ReadOrdersLambda` function handler which is triggered when we make a GET call and we are scanning entire table here and streaming through the records for mapping it back into list of order objects and send it back as response.

```java
public class ReadOrdersLambda {
  private final ObjectMapper objectMapper = new ObjectMapper();
  private final AmazonDynamoDB dynamoDB = AmazonDynamoDBClientBuilder.defaultClient();
  public APIGatewayProxyResponseEvent getOrders(APIGatewayProxyRequestEvent request)
      throws JsonProcessingException {
    ScanResult ordersTable = dynamoDB.scan(
        new ScanRequest().withTableName(System.getenv("ORDERS_TABLE")));
    List<Order> orders = ordersTable.getItems().stream().map(order ->
                                         new Order(Integer.parseInt(order.get("id").getN()), order.get("itemName").getS()))
                                     .collect(Collectors.toList());
    String jsonOutput = objectMapper.writeValueAsString(orders);
    return new APIGatewayProxyResponseEvent().withStatusCode(200).withBody(jsonOutput);
  }
}
```

### SAM Template

- And final part here is creating a YAML configuration file for deployment and create associations between functions and dynamodb. As well as grant necessary policy permissions for read and write.
- First thing in the template is `AWSTemplateFormatVersion` which is auto generated by SAM while doing sam init. `Transform` value is `AWS::Serverless` with version.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Transform: AWS::Serverless-2016-10-31
Description: >
  ordersapi

  Sample SAM Template for ordersapi
```

- Next step is under resources create orders table which is required for both functions to read & write. And we specify type here as `AWS::Serverless::SimpleTable` and specify Primary key name and type which is Number.

```yaml
Resources:
  OrdersTable:
    Type: AWS::Serverless::SimpleTable
    Properties:
      PrimaryKey:
        Name: id
        Type: Number
```

- Similarly create two functions for create and read as below:

  1. Type here is `Type: AWS::Serverless::Function` and give the project name in CodeUri `ordersapi` and Handler directing it to method `Handler: com.suraj.api.CreateOrderLambda::createOrder` under this also have policies `DynamoDBCrudPolicy` which is provided only for **OrdersTable** by using intrisic reference `!Ref OrdersTable` and also **Events** would configure api gateway for us with path `/orders` and accepts `POST`.

  ```yaml
  CreateOrderFunction:
    Type: AWS::Serverless::Function
    Properties:
    CodeUri: ordersapi
    Handler: com.suraj.api.CreateOrderLambda::createOrder
    Policies:
      - DynamoDBCrudPolicy:
          TableName: !Ref OrdersTable
    Events:
      OrderEvents:
      Type: Api
      Properties:
        Path: /orders
        Method: POST
  ```

  2. ReadOrderLambda is pretty much similar previous however here we are only providing a role which has read policy `DynamoDBReadPolicy` and Event type is `GET`.

  ```yaml
  ReadOrdersFunction:
  Type: AWS::Serverless::Function
  Properties:
    CodeUri: ordersapi
    Handler: com.suraj.api.ReadOrdersLambda::getOrders
    Policies:
      - DynamoDBReadPolicy:
          TableName: !Ref OrdersTable
    Events:
      OrderEvents:
        Type: Api
        Properties:
          Path: /orders
          Method: GET
  ```

  - Complete template file:

  ```yaml
  AWSTemplateFormatVersion: '2010-09-09'
    Transform: AWS::Serverless-2016-10-31
    Description: >
    ordersapi

    Sample SAM Template for ordersapi

    Globals:
        Function:
            Runtime: java11
            MemorySize: 512
            Timeout: 30
            Environment:
                Variables:
                    ORDERS_TABLE: !Ref OrdersTable

    Resources:
    OrdersTable:
        Type: AWS::Serverless::SimpleTable
        Properties:
        PrimaryKey:
            Name: id
            Type: Number
    CreateOrderFunction:
        Type: AWS::Serverless::Function
        Properties:
        CodeUri: ordersapi
        Handler: com.suraj.api.CreateOrderLambda::createOrder
        Policies:
            - DynamoDBCrudPolicy:
                TableName: !Ref OrdersTable
        Events:
            OrderEvents:
            Type: Api
            Properties:
                Path: /orders
                Method: POST
    ReadOrdersFunction:
        Type: AWS::Serverless::Function
        Properties:
        CodeUri: ordersapi
        Handler: com.suraj.api.ReadOrdersLambda::getOrders
        Policies:
            - DynamoDBReadPolicy:
                TableName: !Ref OrdersTable
        Events:
            OrderEvents:
            Type: Api
            Properties:
                Path: /orders
                Method: GET
  ```

* After this process we can head to AWS to verify if the lambda service is created successfully and get the endpoint from API Gateway service. Once we get endpoint you can hit with sample body from postman client with context path and it should work fine.

Hope you enjoyed this 🙂❤️
