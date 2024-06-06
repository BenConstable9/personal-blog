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

With the recent rise of Retrieval Augmented Generation (RAG) alongside Large Language Model (LLMs), we can build AI powered systems that can search and sumarise indexed documents to enhance the knowledge of chatbots. This works great for unstructured data stored in documents, but what if you want to access information stored in a SQL database and summarise accordingly?

By combining existing RAG techniques, with natural language to SQL processing, we can create enterprise chatbot systems that can fetch and sumarise both unstructured, and structured data.

Below, a proof of concept implementation is shown in Python using [Semantic Kernel](https://github.com/microsoft/semantic-kernel) for orchestration. 

**This code is not production ready, but simply a proof of concept. Care should be taken to improve the SQL security to prevent attacks on the database, such as SQL injection.**

### 1. Pre-requisites

For this example, we have the following deployed in Azure:
- SQL Server with the [Adventure Works](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver16&tabs=ssms) sample database
- GPT-4

### 2. Setting Up GPT Chat Completition

```python
from semantic_kernel.kernel import Kernel
from semantic_kernel.connectors.ai.open_ai import     AzureChatCompletion

kernel = Kernel()
chat_service = AzureChatCompletion(
    service_id="chat-gpt",
    deployment_name="gpt-4",
    endpoint=os.environ["GPT-ENDPOINT"],
    api_key=os.environ["GPT-API-KEY"],
)
kernel.add_service(chat_service)
```

### 3. SQL Schema Plugin

To enable SQL plugin, a custom [Semantic Kernel Plugin](https://learn.microsoft.com/en-us/semantic-kernel/agents/plugins/?tabs=python) is developed. The SQL Schemas Plugin holds information on the schemas available within the database that the LLM can use to form a SQL query. 

Adding all the table / view definitions, enables the LLM to have in-depth knowledge about the contents of the database can be used. 

```python
from semantic_kernel.functions import kernel_function
import aioodbc
import os
from typing import Annotated

class SQLPlugin:
    @kernel_function(description="Returns SQL schema for the databases.", name="GetSQLSchemas")
    def get_sql_schemas(self):
        """Get the schemas for the database
        
        Returns:
            A description of all the tables and views within the database."""

        schema = """Use the following SQL Schema Views and their associated definitions when you need to fetch information from a database:

        CREATE VIEW [SalesLT].[vGetAllCategories] WITH SCHEMABINDING AS -- Returns the CustomerID, first name, and last name for the specified customer. WITH CategoryCTE([ParentProductCategoryID], [ProductCategoryID], [Name]) AS ( SELECT [ParentProductCategoryID], [ProductCategoryID], [Name] FROM SalesLT.ProductCategory WHERE ParentProductCategoryID IS NULL UNION ALL SELECT C.[ParentProductCategoryID], C.[ProductCategoryID], C.[Name] FROM SalesLT.ProductCategory AS C INNER JOIN CategoryCTE AS BC ON BC.ProductCategoryID = C.ParentProductCategoryID ) SELECT PC.[Name] AS [ParentProductCategoryName], CCTE.[Name] as [ProductCategoryName], CCTE.[ProductCategoryID] FROM CategoryCTE AS CCTE JOIN SalesLT.ProductCategory AS PC ON PC.[ProductCategoryID] = CCTE.[ParentProductCategoryID]
        
        Do not use any other tables / views, other than those defined above."""

        return schema
```

Semantic Kernel uses *kernel_function* to provide information to the LLM on what actions the function is capable of completing. This is used in the planning stage to determine whether to invoke the function or not.

### 4. SQL Query Plugin

To implement querying of the database, another function is added to the SQLPlugin to run the generated query against the database. The *aioodbc* library is used for asynchronous connections to the database.

```python
@kernel_function(description="Runs an SQL query against the SQL Database to extract information.", name="RunSQLQuery")
async def run_sql_query(self, query: Annotated[str, "The SQL query to run against the DB"]):
    """Sends an SQL Query to the SQL Databases and returns to the result.
    
    Args:
        query: The query to run against the DB.
        
    Returns:
        The response as a dictionary of the column and value for each column and row in the database."""

    connection_string = os.environ["SQL-CONNECTION-STRING"]
    async with await aioodbc.connect(dsn=connection_string) as sql_db_client:
        async with sql_db_client.cursor() as cursor:
            await cursor.execute(statement)

            columns = [column[0] for column in cursor.description]

            rows = await cursor.fetchall()
            results = [
                dict(zip(columns, returned_row)) for returned_row in rows
            ]

    return results
```

### 5. Adding SQLPlugin to Kernel

Our custom SQL plugin needs to be registered with Semantic Kernel.
```python
from sql_plugin.SQLPlugin import SQLPlugin
sql_db_plugin = kernel.add_plugin(SQLPlugin(), plugin_name="SQLDB")
```

### 6. Setting Up The Planner

To allow the LLM to "think through" the steps needed to answer a query, a [Semantic Kernel Planner](https://learn.microsoft.com/en-us/semantic-kernel/agents/planners/?tabs=python) is used. A planner takes the prompt, and returns an corresponding plan of how to fulfill the request. If plugins are registered with the kernel, these plugins can be used within the plan.

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

The planner can be invoked with:

```python
question = "Give me an example of one the categories."
response = await planner.invoke(kernel, question)
print(f"Q: {question}\nA: {response.final_answer}\n")
```

The response from the LLM is:

```
Q: Give me an example of one the categories.
A: An example of one of the categories from the database is 'Bike Racks'.
```

Our LLM has automatically worked out from the prompt, what the categories relate to, formulated an SQL query from natural language and ran the formulated query directly on the database to produce a result.

The full plan executed by the LLM can be viewed with:

```python
print(f"Chat history: {response.chat_history}\n")
```

Resulting in a plan of:

``````