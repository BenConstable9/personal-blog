---
title: "Querying SQL Server With Natural Language In Semantic Kernel"
date: 2024-06-06T15:31:44+01:00
tags: ["Semantic Kernel", "Python", "SQL Server", "OpenAI", "GPT"]
draft: false
author: "Me"
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
---

With the recent rise of Retrieval Augmented Generation (RAG) alongside Large Language Models (LLMs), we can build AI-powered systems that can search and summarize indexed documents to enhance the knowledge of chatbots. This works great for unstructured data stored in documents, but what if you want to access information stored in a SQL database and summarize accordingly?

By combining existing RAG techniques with natural language to SQL processing, we can create enterprise chatbot systems that can fetch and summarize both unstructured and structured data.

Below, a proof of concept implementation is shown in Python using [Semantic Kernel](https://github.com/microsoft/semantic-kernel) for orchestration with natural language to SQL conversion. Semantic Kernel is a Microsoft provided SDK that automatically orchestrates plugins to fulfill user requests using an LLM. 

![Semantic Kernel Image](/Semantic%20Kernel.png)
*AI orchestration with Semantic Kernel. Image Source: [GitHub](https://github.com/microsoft/semantic-kernel)*

**This code is not production-ready but simply a proof of concept. Care should be taken to improve the SQL security to prevent attacks on the database, such as SQL injection.**

### 1. Prerequisites

For this example, we have the following deployed in Azure:
- SQL Server with the [Adventure Works](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver16&tabs=ssms) sample database
- GPT-4

Below is a sample of the data available in the Adventure Works database. In an enterprise scenario, this data may change in near-real time, so querying the database directly, enables users to get the most up to date information available, rather than ingesting and indexing it.

![Sample Data](/Adventure%20Works%20Sample%20Data.png)

Whilst we are using SQL server in this scenario, any database could be used.

### 2. Setting Up GPT Chat Completion

```python
from semantic_kernel.kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import AzureChatCompletion

kernel = Kernel()
chat_service = AzureChatCompletion(
    service_id="chat-gpt",
    deployment_name="gpt-4",
    endpoint=os.environ["GPT-ENDPOINT"],
    api_key=os.environ["GPT-API-KEY"],
)
kernel.add_service(chat_service)
```

### 3. SQL Schema Knowledge

To enable the SQL plugin, a custom [Semantic Kernel Plugin](https://learn.microsoft.com/en-us/semantic-kernel/agents/plugins/?tabs=python) is developed. The SQL Plugin holds information on the schemas available within the database that the LLM can use to form an SQL query, and provides a method to run a given SQL query.

Adding all the table/view definitions enables the LLM to have in-depth knowledge about the contents of the database that can be used.

```python
from semantic_kernel.functions import kernel_function
import aioodbc
import os
from typing import Annotated

class SQLPlugin:
    @staticmethod
    def get_system_prompt():
        """Get the schemas for the database
        
        Returns:
            A description of all the tables and views within the database."""

        schema = """Use the following SQL Schema Views and their associated definitions when you need to fetch information from a database:

        vGetAllCategories View. Use this to get details about the categories available.
        CREATE VIEW [SalesLT].[vGetAllCategories] WITH SCHEMABINDING AS -- Returns the CustomerID, first name, and last name for the specified customer. WITH CategoryCTE([ParentProductCategoryID], [ProductCategoryID], [Name]) AS ( SELECT [ParentProductCategoryID], [ProductCategoryID], [Name] FROM SalesLT.ProductCategory WHERE ParentProductCategoryID IS NULL UNION ALL SELECT C.[ParentProductCategoryID], C.[ProductCategoryID], C.[Name] FROM SalesLT.ProductCategory AS C INNER JOIN CategoryCTE AS BC ON BC.ProductCategoryID = C.ParentProductCategoryID ) SELECT PC.[Name] AS [ParentProductCategoryName], CCTE.[Name] as [ProductCategoryName], CCTE.[ProductCategoryID] FROM CategoryCTE AS CCTE JOIN SalesLT.ProductCategory AS PC ON PC.[ProductCategoryID] = CCTE.[ParentProductCategoryID]

        vProductAndDescription View. Use this to get details about the products and their associated descriptions.
        CREATE VIEW [SalesLT].[vProductAndDescription] WITH SCHEMABINDING AS -- View (indexed or standard) to display products and product descriptions by language. SELECT p.[ProductID] ,p.[Name] ,pm.[Name] AS [ProductModel] ,pmx.[Culture] ,pd.[Description] FROM [SalesLT].[Product] p INNER JOIN [SalesLT].[ProductModel] pm ON p.[ProductModelID] = pm.[ProductModelID] INNER JOIN [SalesLT].[ProductModelProductDescription] pmx ON pm.[ProductModelID] = pmx.[ProductModelID] INNER JOIN [SalesLT].[ProductDescription] pd ON pmx.[ProductDescriptionID] = pd.[ProductDescriptionID]
        
        Do not use any other tables/views other than those defined above."""

        return schema
```

This method could be implemented as a dynamic method similarly to the query function below if you have a large number of tables for instance. However, this adds latency into the system as LLM needs to run this step in the orchestration process, and the orchestrator needs to query the DB for the tables. By hardcoding the schema, we can significantly speed up the retrieval.

### 4. SQL Query Function

To implement querying of the database, another function is added to the SQLPlugin to run the generated query against the database. The *aioodbc* library is used for asynchronous connections to the database.

```python
@kernel_function(description="Runs an SQL query against the SQL Database to extract information.", name="RunSQLQuery")
async def run_sql_query(self, query: Annotated[str, "The SQL query to run against the DB"]):
    """Sends an SQL Query to the SQL Databases and returns the result.
    
    Args:
        query: The query to run against the DB.
        
    Returns:
        The response as a dictionary of the column and value for each column and row in the database."""

    connection_string = os.environ["SQL-CONNECTION-STRING"]
    async with await aioodbc.connect(dsn=connection_string) as sql_db_client:
        async with sql_db_client.cursor() as cursor:
            await cursor.execute(query)

            columns = [column[0] for column in cursor.description]

            rows = await cursor.fetchall()
            results = [
                dict(zip(columns, returned_row)) for returned_row in rows
            ]

    return results
```

Semantic Kernel uses *kernel_function* to provide information to the LLM on what actions the function is capable of completing. This is used in the planning stage to determine whether to invoke the function or not.

### 5. Adding SQLPlugin to Kernel

Our custom SQL plugin needs to be registered with Semantic Kernel.

```python
from sql_plugin.SQLPlugin import SQLPlugin
sql_db_plugin = kernel.add_plugin(SQLPlugin(), plugin_name="SQLDB")
```

### 6. Setting Up The Planner

To allow the LLM to "think through" the steps needed to answer a query, a [Semantic Kernel Planner](https://learn.microsoft.com/en-us/semantic-kernel/agents/planners/?tabs=python) is used. A planner takes the prompt and returns a corresponding plan of how to fulfill the request. If plugins are registered with the kernel, these plugins can be used within the plan.

```python
from semantic_kernel.planners.function_calling_stepwise_planner import (
    FunctionCallingStepwisePlanner,
    FunctionCallingStepwisePlannerOptions,
)
options = FunctionCallingStepwisePlannerOptions(
    max_iterations=10,
    max_tokens=4000
)

planner = FunctionCallingStepwisePlanner(service_id="chat-gpt", options=options)
```

To provide information about the database schemas, we use the system prompt that we defined in our plugin, along with some basic prompt engineering.

```python
question = "Give me an example of one of the categories."
full_prompt = f"""Here is some additional information that you might find useful in determining which functions to call to fulfill the user question.

SQL Database Information:
{SQLPlugin.system_prompt()}

User Question:
{question}"""
```

Finally, planner can be invoked with:

```python
response = await planner.invoke(kernel, full_prompt)
print(f"Q: {question}\nA: {response.final_answer}\n")
```

The response from the LLM is:

```
Q: Find 5 categories for sales data.
A: The 5 categories for sales data are: 1. Bike Racks, 2. Bike Stands, 3. Bottles and Cages, 4. Cleaners, 5. Fenders.
```

Our LLM has automatically worked out from the prompt what the categories relate to, formulated an SQL query from natural language, and ran the formulated query directly on the database to produce a result without the user needing to know about the database structure or any SQL. Super cool!

**This has great potential in expanding AI powered services, to leverage the vast amount of structured data available in enterprises.**

The full plan executed by the LLM can be viewed with:

```python
print(f"Chat history: {response.chat_history}\n")
```

Resulting in a plan of:

```
Chat history: <chat_history><message role="user"><text>Original request: Here is some additional information that you might find useful in determining which functions to call to fulfill the user question.

SQL Database Information:
Use the following SQL Schema Views and their associated definitions when you need to fetch information from a database:

        vGetAllCategories View. Use this to get details about the categories available.
        CREATE VIEW [SalesLT].[vGetAllCategories] WITH SCHEMABINDING AS -- Returns the CustomerID, first name, and last name for the specified customer. WITH CategoryCTE([ParentProductCategoryID], [ProductCategoryID], [Name]) AS ( SELECT [ParentProductCategoryID], [ProductCategoryID], [Name] FROM SalesLT.ProductCategory WHERE ParentProductCategoryID IS NULL UNION ALL SELECT C.[ParentProductCategoryID], C.[ProductCategoryID], C.[Name] FROM SalesLT.ProductCategory AS C INNER JOIN CategoryCTE AS BC ON BC.ProductCategoryID = C.ParentProductCategoryID ) SELECT PC.[Name] AS [ParentProductCategoryName], CCTE.[Name] as [ProductCategoryName], CCTE.[ProductCategoryID] FROM CategoryCTE AS CCTE JOIN SalesLT.ProductCategory AS PC ON PC.[ProductCategoryID] = CCTE.[ParentProductCategoryID]

        vProductAndDescription View. Use this to get details about the products and their associated descriptions.
        CREATE VIEW [SalesLT].[vProductAndDescription] WITH SCHEMABINDING AS -- View (indexed or standard) to display products and product descriptions by language. SELECT p.[ProductID] ,p.[Name] ,pm.[Name] AS [ProductModel] ,pmx.[Culture] ,pd.[Description] FROM [SalesLT].[Product] p INNER JOIN [SalesLT].[ProductModel] pm ON p.[ProductModelID] = pm.[ProductModelID] INNER JOIN [SalesLT].[ProductModelProductDescription] pmx ON pm.[ProductModelID] = pmx.[ProductModelID] INNER JOIN [SalesLT].[ProductDescription] pd ON pmx.[ProductDescriptionID] = pd.[ProductDescriptionID]
        
        Do not use any other tables / views, other than those defined above.

User Question:
Find 5 categories for sales data.

You are in the process of helping the user fulfill this request using the following plan:
To find 5 categories for sales data, we will use the SQL Database Information provided and execute an SQL query against the database using the `SQLDB-RunSQLQuery` function. The query will select from the `vGetAllCategories` view, which gives us the details about the categories available. Since the goal is to find 5 categories, we will limit the query to return only 5 categories.

Plan:

1. Use the `SQLDB-RunSQLQuery` function to run an SQL query against the SQL Database to extract information about 5 categories. The SQL query to use will be:
   ```
   SELECT TOP 5 ProductCategoryName FROM SalesLT.vGetAllCategories;
   ```
   This query selects the top 5 product category names from the `vGetAllCategories` view.

2. Once we have the result from the SQL query, we will format the information into a readable format for the user.

3. Use the `UserInteraction-SendFinalAnswer` function to send the final answer back to the user. The answer will include the names of the 5 categories found in step 1.

This plan will effectively use the provided functions to fulfill the user's request for finding 5 categories for sales data.

The user will ask you for help with each step.</text></message><message role="user"><text>Perform the next step of the plan if there is more work to do. When you have reached a final answer, use the UserInteraction-SendFinalAnswer function to communicate this back to the user.</text></message><message ai_model_id="gpt-4-0125" finish_reason="tool_calls" role="assistant"><function_call id="call_O3nGMvvB9VqQQBFCZNSL8KuG" name="SQLDB-RunSQLQuery">{"query":"SELECT TOP 5 ProductCategoryName FROM SalesLT.vGetAllCategories;"}</function_call></message><message role="tool"><function_result id="call_O3nGMvvB9VqQQBFCZNSL8KuG" name="SQLDB-RunSQLQuery">{'ProductCategoryName': 'Bike Racks'},{'ProductCategoryName': 'Bike Stands'},{'ProductCategoryName': 'Bottles and Cages'},{'ProductCategoryName': 'Cleaners'},{'ProductCategoryName': 'Fenders'}</function_result></message><message role="user"><text>Perform the next step of the plan if there is more work to do. When you have reached a final answer, use the UserInteraction-SendFinalAnswer function to communicate this back to the user.</text></message><message ai_model_id="gpt-4-0125" finish_reason="tool_calls" role="assistant"><function_call id="call_nDr6u1yMLRJaNWpJynExe2s6" name="UserInteraction-SendFinalAnswer">{"answer":"The 5 categories for sales data are: 1. Bike Racks, 2. Bike Stands, 3. Bottles and Cages, 4. Cleaners, 5. Fenders."}</function_call></message></chat_history>
```

### 7. Production Considerations

If using this technique in production, the following should be considered amongst others:

- Proper management of secrets within the Semantic Kernel script.
- SQL Data Access Management to ensure that the LLM cannot query data it does not have access to.
- Validation of SQL query to ensure it does not invoke an SQL injection for instance.