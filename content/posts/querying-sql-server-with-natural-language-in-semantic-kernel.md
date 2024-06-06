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

![Semantic Kernel Image](/static/Semantic%20Kernel.png)
*AI orchestration with Semantic Kernel. Image Source: [GitHub](https://github.com/microsoft/semantic-kernel)*

**This code is not production-ready but simply a proof of concept. Care should be taken to improve the SQL security to prevent attacks on the database, such as SQL injection.**

### 1. Prerequisites

For this example, we have the following deployed in Azure:
- SQL Server with the [Adventure Works](https://learn.microsoft.com/en-us/sql/samples/adventureworks-install-configure?view=sql-server-ver16&tabs=ssms) sample database
- GPT-4

Below is a sample of the data available in the Adventure Works database. In an enterprise scenario, this data may change in near-real time, so querying the database directly, enables users to get the most up to date information available, rather than ingesting and indexing it.

![Sample Data](/static/Adventure%20Works%20Sample%20Data.png)

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

### 3. SQL Schema Plugin

To enable the SQL plugin, a custom [Semantic Kernel Plugin](https://learn.microsoft.com/en-us/semantic-kernel/agents/plugins/?tabs=python) is developed. The SQL Schemas Plugin holds information on the schemas available within the database that the LLM can use to form an SQL query.

Adding all the table/view definitions enables the LLM to have in-depth knowledge about the contents of the database that can be used.

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
        
        Do not use any other tables/views other than those defined above."""

        return schema
```

Semantic Kernel uses *kernel_function* to provide information to the LLM on what actions the function is capable of completing. This is used in the planning stage to determine whether to invoke the function or not.

### 4. SQL Query Plugin

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

The planner can be invoked with:

```python
question = "Give me an example of one of the categories."
response = await planner.invoke(kernel, question)
print(f"Q: {question}\nA: {response.final_answer}\n")
```

The response from the LLM is:

```
Q: Give me an example of one of the categories.
A: An example of one of the categories from the database is 'Bike Racks'.
```

Our LLM has automatically worked out from the prompt what the categories relate to, formulated an SQL query from natural language, and ran the formulated query directly on the database to produce a result without the user needing to know about the database structure or any SQL. Super cool!

**This has great potential in expanding AI powered services, to leverage the vast amount of structured data available in enterprises.**

The full plan executed by the LLM can be viewed with:

```python
print(f"Chat history: {response.chat_history}\n")
```

Resulting in a plan of:

```
Chat history: <chat_history><message role="user"><text>Original request: Give me an example of one of the categories.

You are in the process of helping the user fulfill this request using the following plan:
Given the available functions, it seems like you're asking for an example related to interacting with an SQL database or perhaps generating a response based on SQL data. Since the specific goal is to provide an example of one of the categories, and assuming "categories" refers to types of information that can be extracted from an SQL database, here's a plan to achieve that goal:

### Plan to Provide an Example of a Category from an SQL Database

1. **Use SQLDB-GetSQLSchemas Function**: 
   - This step involves calling the `SQLDB-GetSQLSchemas` function to retrieve the schema of the databases. The schema will provide information on the tables available in the database and their structure, including columns that might represent different categories of data.

2. **Identify a Table and Column Representing a Category**:
   - Based on the schema information retrieved, manually identify a table and a specific column within that table that represents a "category" of data. For example, if there's a table named `Products`, a column within that table might be `Category` which could represent different categories of products like electronics, clothing, etc.

3. **Use SQLDB-RunSQLQuery Function to Extract a Category Example**:
   - Construct an SQL

The user will ask you for help with each step.</text></message><message role="user"><text>Perform the next step of the plan if there is more work to do. When you have reached a final answer, use the UserInteraction-SendFinalAnswer function to communicate this back to the user.</text></message><message ai_model_id="gpt-4" finish_reason="tool_calls" role="assistant"><function_call id="call_3gNCAyqiz4x8do5p9mrPOMX3" name="SQLDB-GetSQLSchemas">{}</function_call></message><message role="tool"><function_result id="call_3gNCAyqiz4x8do5p9mrPOMX3" name="SQLDB-GetSQLSchemas">Use the following SQL Schema Views and their associated definitions when you need to fetch information from a database:

        CREATE VIEW [SalesLT].[vGetAllCategories] WITH SCHEMABINDING AS -- Returns the CustomerID, first name, and last name for the specified customer. WITH CategoryCTE([ParentProductCategoryID], [ProductCategoryID], [Name]) AS ( SELECT [ParentProductCategoryID], [ProductCategoryID], [Name] FROM SalesLT.ProductCategory WHERE ParentProductCategoryID IS NULL UNION ALL SELECT C.[ParentProductCategoryID], C.[ProductCategoryID], C.[Name] FROM SalesLT.ProductCategory AS C INNER JOIN CategoryCTE AS BC ON BC.ProductCategoryID = C.ParentProductCategoryID ) SELECT PC.[Name] AS [ParentProductCategoryName], CCTE.[Name] as [ProductCategoryName], CCTE.[ProductCategoryID] FROM CategoryCTE AS CCTE JOIN SalesLT.ProductCategory AS PC ON PC.[ProductCategoryID] = CCTE.[ParentProductCategoryID]
        
        Do not use any other tables/views other than those defined above.</function_result></message><message role="user"><text>Perform the next step of the plan if there is more work to do. When you have reached a final answer, use the UserInteraction-SendFinalAnswer function to communicate this back to the user.</text></message><message ai_model_id="gpt-4" finish_reason="tool_calls" role="assistant"><function_call id="call_WyEofwvMLiKdD82TWC7lR57C" name="SQLDB-RunSQLQuery">{"query":"SELECT TOP 1 [ProductCategoryName] FROM [SalesLT].[vGetAllCategories]"}</function_call></message><message role="tool"><function_result id="call_WyEofwvMLiKdD82TWC7lR57C" name="SQLDB-RunSQLQuery">{'ProductCategoryName': 'Bike Racks'}</function_result></message><message role="user"><text>Perform the next step of the plan if there is more work to do. When you have reached a final answer, use the UserInteraction-SendFinalAnswer function to communicate this back to the user.</text></message><message ai_model_id="gpt-4" finish_reason="tool_calls" role="assistant"><function_call id="call_AxnIvxX36lJJwvfiUMKD5jz6" name="UserInteraction-SendFinalAnswer">{"answer":"An example of a category from the SQL database is 'Bike Racks'."}</function_call></message></chat_history>
```

### 7. Production Considerations

If using this technique in production, the following should be considered amongst others:

- Proper management of secrets within the Semantic Kernel script.
- SQL Data Access Management to ensure that the LLM cannot query data it does not have access to.
- Validation of SQL query to ensure it does not invoke an SQL injection for instance.