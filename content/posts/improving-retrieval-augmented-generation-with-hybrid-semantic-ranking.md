---
title: "Improving Retrieval Augmented Generation (RAG) with Hybrid Semantic Ranking"
date: 2024-06-12T19:00:00+01:00
tags: ["Semantic Kernel", "Python", "Semantic Ranking", "Azure AI Search", "Vector Search", "OpenAI", "GPT"]
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

*This post is a follow up to my previous one on [Querying SQL Server With Natural Language In Semantic Kernel](https://www.benconstable.dev/posts/querying-sql-server-with-natural-language-in-semantic-kernel/), if you haven't read it yet, go take a look for a brief introduction to Semantic Kernel and the plugin architecture.*

---

The majority of Retrieval Augmented Generation (RAG) systems utilise [Vector Databases](https://learn.microsoft.com/en-us/semantic-kernel/memories/vector-db), such as Azure AI Search, to store chunks of documents that can be accessed via Large Language Models (LLMs). Semantic similarity, powered via a vector search, is then used to retrieve the most relevant documents to answer the user's question. 

Whilst this works well, important documents that provide good context for the question, are often missed as individual words can have multiple meanings which is not known when performing a vector-based similarity match. Instead, a hybrid approach that re-ranks the original query results, can lead to a [significant relevancy increase](https://techcommunity.microsoft.com/t5/ai-azure-ai-services-blog/azure-ai-search-outperforming-vector-search-with-hybrid/ba-p/3929167) in the documents returned, boosting RAG performance and user satisfaction.

In this post, an implementation for Hybrid Semantic Ranking within Semantic Kernel is demonstrated, this approach uses a planner for Chain of Thought reasoning.

### 1. Prerequisites

For this example, we have the following deployed in Azure:
- Azure AI Search deployed in a [Semantic Ranking supported region](https://azure.microsoft.com/global-infrastructure/services/?products=search)
- [Documents ingested](https://learn.microsoft.com/en-us/azure/search/search-what-is-data-import) in Azure AI Search with a Semantic Config.
- GPT-4
- Semantic Kernel Planner Setup. See [Querying SQL Server With Natural Language In Semantic Kernel](https://www.benconstable.dev/posts/querying-sql-server-with-natural-language-in-semantic-kernel/) for a brief introduction on how to setup a basic planner.

### 2. Why A Custom Plugin?

Semantic Kernel includes basic support for [On Your Data](https://techcommunity.microsoft.com/t5/ai-azure-ai-services-blog/on-your-data-is-now-generally-available-in-azure-openai-service/ba-p/4059514) as shown in this [sample](https://github.com/microsoft/semantic-kernel/blob/01e826ddc1a35ae53a7134e46642297919ed79aa/python/samples/concepts/on_your_data/azure_chat_gpt_with_data_api_vector_search.py). This provides a great starting point for basic RAG applications, however combining the On Your Data functionality, alongside planners to give Chain of Thought reasoning, is incompatible at the time of writing.

If you want to combine the power of a planner with indexed documents, you need to develop a custom functions. This is essential for allowing the LLM to think through the steps, formulate a plan and use the two data sources as it thinks is necessary.

### 3. Semantic Config

If you haven't already, go and setup a [Semantic Config](https://learn.microsoft.com/en-us/azure/search/semantic-how-to-configure?tabs=portal) on your index. You should setup the following:

- Title field (Optional)
- Content fields. These fields contain chunks of text, often the original body of a document or a summary.
- Keyword fields. These fields contain keywords such as tags.

### 4. System Prompt

Similarly to the last Semantic Kernel plugin developed, a system prompt is setup to give the LLM knowledge about this plugin. Unlike the SQL system prompt, we don't need to provide any information on the contents of AI Search as it only needs to provide a plain text query.

```python
from semantic_kernel.functions import kernel_function
from typing import Annotated
from azure.core.credentials import AzureKeyCredential

from azure.search.documents.models import VectorizableTextQuery
from azure.search.documents.aio import SearchClient
import os


class AISearchPlugin:
    """A plugin that allows for the execution of AI Search queries against a text input."""

    @staticmethod
    def system_prompt() -> str:
        return """Use the AI Search to return documents that have been indexed, that might be relevant for a piece of text to aid understanding. Documents ingested here could be relevant to anything, so AI Search should always be used. Always provide references to any documents you use."""
```

Basic prompt engineering is used to guide the LLM to always use AI Search; without this, the LLM may decide to skip it if it is not considered relevant. Unlike the SQL DB example, the LLM doesn't know the structure of the documents, or the type of data that is stored here, so it should always consider it in our example.

### 5. AI Search Query Function

To implement the Hybrid Semantic Ranking Search, a function is added to the plugin as follows. The search is against the specified Semantic Config and the search vectors; you can experiment with different configs to improve relevancy. Here, [embedded vectorisation](https://learn.microsoft.com/en-gb/azure/search/vector-search-how-to-configure-vectorizer) is used to allow AI Search to automatically run the vectorisation for you; this reduces complexity in your API calls and makes it easier to setup searches. Instead of passing a set of vectors, we pass the text to vectorise and AI Search handles the rest.

*At the time of writing embedded vectorisation is still in preview within the SDK. Version **11.6.0b3** is used in this plugin.*

Returning the data in a dictionary format, allows the LLM to understand the different fields, and extract the relevant ones to fulfill the user request.

```python
@kernel_function(
    description="Runs an semantic search against some text to return relevant documents that are indexed within AI Search.",
    name="RunAISearchOnText",
)
async def run_ai_search_on_text(
    self, text: Annotated[str, "The text to run a semantic search against."]
) -> list[dict]:
    """Sends an text query to AI Search and uses Semantic Ranking to return a result.

    Args:
    ----
        text (str): The text to run the search against.

    Returns:
    ----
        list[dict]: The response from the search that is ranked according to relevancy.
    """

    credential = AzureKeyCredential(os.environ["AI_SEARCH_API_KEY"])

    search_client = SearchClient(
        endpoint=os.environ["AI_SEARCH_ENDPOINT"],
        index_name="< YOUR INDEX NAME >",
        credential=credential,
    )

    vector_query = VectorizableTextQuery(
        text=text,
        k_nearest_neighbors=5,
        fields="< YOUR VECTOR FIELDS e.g. title_vector,chunk_vector >",
    )

    results = await search_client.search(
        top=2,
        query_type="semantic",
        semantic_configuration_name="< YOUR SEMANTIC CONFIG NAME >",
        search_text=text,
        select="< FIELDS TO RETURN e.g. title,chunk,categories>",
        vector_queries=[vector_query],
    )

    documents = [
        document async for result in results.by_page() async for document in result
    ]
    return documents
```

Semantic Kernel uses *kernel_function* to provide information to the LLM on what actions the function is capable of completing. This is used in the planning stage to determine whether to invoke the function or not.

In this case, the title, original content chunk and category tags are returned to the LLM. This functionality can be extended further with the use of [captions and highlights](https://learn.microsoft.com/en-gb/azure/search/semantic-answers) which provide key extracts from the chunks.

Depending on your content window size (token limit), you can return a differing number of results.

### 6. Integrating With An Existing Planner

The developed plugin can be integrated with the existing Planner and the full prompt updated with the AI Search information.

```python
from ai_search_plugin.ai_search_plugin import AISearchPlugin

kernel.add_plugin(AISearchPlugin(), plugin_name="AISearch")
```

For a given question, we can update the system prompt to include additional information on how we want to use AI Search.
```python
question = "Select 1 product and any documents related to them?"
full_prompt = f"""Here is some additional information that you might find useful in determining which functions to call to fulfill the user question. 

AI Search Information:
{AISearchPlugin.system_prompt()}

SQL Database Information:
{SQLPlugin.system_prompt()}

User Question:
{question}"""

response = await planner.invoke(kernel, full_prompt)
```

The planner gives us a plan of:

```
Original request: Here is some additional information that you might find useful in determining which functions to call to fulfill the user question. 

AI Search Information:
Use the AI Search to return documents that have been indexed, that might be relevant for a piece of text to aid understanding. Documents ingested here could be relevant to anything, so AI Search should always be used. Always provide references to any documents you use.

SQL Database Information:
Use the following SQL Schema Views and their associated definitions when you need to fetch information from a database:

        vGetAllCategories View. Use this to get details about the categories available.
        CREATE VIEW [SalesLT].[vGetAllCategories] WITH SCHEMABINDING AS -- Returns the CustomerID, first name, and last name for the specified customer. WITH CategoryCTE([ParentProductCategoryID], [ProductCategoryID], [Name]) AS ( SELECT [ParentProductCategoryID], [ProductCategoryID], [Name] FROM SalesLT.ProductCategory WHERE ParentProductCategoryID IS NULL UNION ALL SELECT C.[ParentProductCategoryID], C.[ProductCategoryID], C.[Name] FROM SalesLT.ProductCategory AS C INNER JOIN CategoryCTE AS BC ON BC.ProductCategoryID = C.ParentProductCategoryID ) SELECT PC.[Name] AS [ParentProductCategoryName], CCTE.[Name] as [ProductCategoryName], CCTE.[ProductCategoryID] FROM CategoryCTE AS CCTE JOIN SalesLT.ProductCategory AS PC ON PC.[ProductCategoryID] = CCTE.[ParentProductCategoryID]

        vProductAndDescription View. Use this to get details about the products and their associated descriptions.
        CREATE VIEW [SalesLT].[vProductAndDescription] WITH SCHEMABINDING AS -- View (indexed or standard) to display products and product descriptions by language. SELECT p.[ProductID] ,p.[Name] ,pm.[Name] AS [ProductModel] ,pmx.[Culture] ,pd.[Description] FROM [SalesLT].[Product] p INNER JOIN [SalesLT].[ProductModel] pm ON p.[ProductModelID] = pm.[ProductModelID] INNER JOIN [SalesLT].[ProductModelProductDescription] pmx ON pm.[ProductModelID] = pmx.[ProductModelID] INNER JOIN [SalesLT].[ProductDescription] pd ON pmx.[ProductDescriptionID] = pd.[ProductDescriptionID]
        
        Do not use any other tables / views, other than those defined above.

User Question:
Select 1 product and any documents related to them?

You are in the process of helping the user fulfill this request using the following plan:
To achieve the goal of summarizing 5 products and any documents related to them, we will follow a structured plan that leverages the provided functions. Here's how we can proceed:

### Step 1: Fetch Product Information
- **Action**: Use the `SQLDB-RunSQLQuery` function.
- **Input**: 
  - **Query**: `"SELECT TOP 1 p.[ProductID], p.[Name], pm.[Name] AS [ProductModel], pd.[Description] FROM [SalesLT].[vProductAndDescription] p ORDER BY p.[ProductID]"`
- **Purpose**: This query fetches the top 5 products from the `vProductAndDescription` view, including their ID, name, model, and description. This step is crucial for obtaining the basic information about the products we want to summarize.

### Step 2: Identify Relevant Documents
After selecting a product and obtaining its description, we will use this description to find related documents using AI Search. This will help us identify any indexed documents that are relevant to the selected product.

- **Action**: Use the `AISearch-RunAISearchOnText` function.

The user will ask you for help with each step.
```

With the use of a custom plugin, our planner can now decide the steps it needs to take to fullfil the user request. Here, it chooses to query the database first to find the relevant categories, followed by choosing to use the AI Search to retrieve the relevant documents. Combining multiple plugins with a planner, provides a great basis for Chain of Thought reasoning which can be further enhanced with prompt engineering.

In a use case like this, a Hybrid Semantic Ranking based search will excel over a vector based search on the chunk, avoiding content that is similar in a vector based search, but different in semantic meaning. The vector search is used as one of the base searches, ensuring that similar semantic content is surfaced, but the Semantic Ranking ensures it is relevant to the question and re-orders it as necessary.

If the documents are appropriately tagged with the category, relevant documents will be surfaced and ranked accordingly as well. Experimentation is needed to determine whether Semantic Ranking is good for an individual use case, but overall it shows a significant improve on a purely vector-based search. 

### 7. Production Considerations

If using this technique in production, the following should be considered amongst others:

- Proper management of secrets within the Semantic Kernel script.
- Role based access controls for the documents to ensure only those a user is authorised to view, can so. Find out more [here](https://learn.microsoft.com/en-us/azure/search/search-security-overview#restricting-access-to-documents).
- Prompting for references within the response, to surface as clickable references to the user.

### 8. Full Code

You can find the full plugin and example notebook in this [repository](https://github.com/BenConstable9/semantic-kernel-experiments).