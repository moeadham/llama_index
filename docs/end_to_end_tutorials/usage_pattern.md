# Basic Usage Pattern

The general usage pattern of LlamaIndex is as follows:

1. Load in documents (either manually, or through a data loader)
2. Parse the Documents into Nodes
3. Construct Index (from Nodes or Documents)
4. [Optional, Advanced] Building indices on top of other indices
5. Query the index

## 1. Load in Documents

The first step is to load in data. This data is represented in the form of `Document` objects.
We provide a variety of [data loaders](/core_modules/data_modules/connector/root.md) which will load in Documents
through the `load_data` function, e.g.:

```python
from llama_index import SimpleDirectoryReader

documents = SimpleDirectoryReader('./data').load_data()
```

You can also choose to construct documents manually. LlamaIndex exposes the `Document` struct.

```python
from llama_index import Document

text_list = [text1, text2, ...]
documents = [Document(text=t) for t in text_list]
```

A Document represents a lightweight container around the data source. You can now choose to proceed with one of the
following steps:

1. Feed the Document object directly into the index (see section 3).
2. First convert the Document into Node objects (see section 2).

## 2. Parse the Documents into Nodes

The next step is to parse these Document objects into Node objects. Nodes represent "chunks" of source Documents,
whether that is a text chunk, an image, or more. They also contain metadata and relationship information
with other nodes and index structures.

Nodes are a first-class citizen in LlamaIndex. You can choose to define Nodes and all its attributes directly. You may also choose to "parse" source Documents into Nodes through our `NodeParser` classes.

For instance, you can do

```python
from llama_index.node_parser import SimpleNodeParser

parser = SimpleNodeParser()

nodes = parser.get_nodes_from_documents(documents)
```

You can also choose to construct Node objects manually and skip the first section. For instance,

```python
from llama_index.schema import TextNode, NodeRelationship, RelatedNodeInfo

node1 = TextNode(text="<text_chunk>", id_="<node_id>")
node2 = TextNode(text="<text_chunk>", id_="<node_id>")
# set relationships
node1.relationships[NodeRelationship.NEXT] = RelatedNodeInfo(node_id=node2.node_id)
node2.relationships[NodeRelationship.PREVIOUS] = RelatedNodeInfo(node_id=node1.node_id)
nodes = [node1, node2]
```

The `RelatedNodeInfo` class can also store additional `metadata` if needed:

```python
node2.relationships[NodeRelationship.PARENT] = RelatedNodeInfo(node_id=node1.node_id, metadata={"key": "val"})
```

## 3. Index Construction

We can now build an index over these Document objects. The simplest high-level abstraction is to load-in the Document objects during index initialization (this is relevant if you came directly from step 1 and skipped step 2).

`from_documents` also takes an optional argument `show_progress`. Set it to `True` to display a progress bar during index construction.

```python
from llama_index import VectorStoreIndex

index = VectorStoreIndex.from_documents(documents)
```

You can also choose to build an index over a set of Node objects directly (this is a continuation of step 2).

```python
from llama_index import VectorStoreIndex

index = VectorStoreIndex(nodes)
```

Depending on which index you use, LlamaIndex may make LLM calls in order to build the index.

### Reusing Nodes across Index Structures

If you have multiple Node objects defined, and wish to share these Node
objects across multiple index structures, you can do that.
Simply instantiate a StorageContext object,
add the Node objects to the underlying DocumentStore,
and pass the StorageContext around.

```python
from llama_index import StorageContext

storage_context = StorageContext.from_defaults()
storage_context.docstore.add_documents(nodes)

index1 = VectorStoreIndex(nodes, storage_context=storage_context)
index2 = ListIndex(nodes, storage_context=storage_context)
```

**NOTE**: If the `storage_context` argument isn't specified, then it is implicitly
created for each index during index construction. You can access the docstore
associated with a given index through `index.storage_context`.

### Inserting Documents or Nodes

You can also take advantage of the `insert` capability of indices to insert Document objects
one at a time instead of during index construction.

```python
from llama_index import VectorStoreIndex

index = VectorStoreIndex([])
for doc in documents:
    index.insert(doc)
```

If you want to insert nodes on directly you can use `insert_nodes` function
instead.

```python
from llama_index import VectorStoreIndex

# nodes: Sequence[Node]
index = VectorStoreIndex([])
index.insert_nodes(nodes)
```

See the [Document Management How-To](/core_modules/data_modules/index/document_management.md) for more details on managing documents and an example notebook.

### Customizing Documents

When creating documents, you can also attach useful metadata. Any metadata added to a document will be copied to the nodes that get created from their respective source document.

```python
document = Document(
    text='text',
    metadata={
        'filename': '<doc_file_name>',
        'category': '<category>'
    }
)
```

More information and approaches to this are discussed in the section [Customizing Documents](/core_modules/data_modules/documents_and_nodes/usage_documents.md).

### Customizing LLM's

By default, we use OpenAI's `text-davinci-003` model. You may choose to use another LLM when constructing
an index.

```python
from llama_index import VectorStoreIndex, ServiceContext, set_global_service_context
from llama_index.llms import OpenAI

...

# define LLM
llm = OpenAI(model="gpt-4", temperature=0, max_tokens=256)

# configure service context
service_context = ServiceContext.from_defaults(llm=llm)
set_global_service_context(service_context)

# build index
index = VectorStoreIndex.from_documents(
    documents
)
```

To save costs, you may want to use a local model.

```python
from llama_index import ServiceContext
service_context = ServiceContext.from_defaults(llm="local")
```

This will use llama2-chat-13B from with LlamaCPP, and assumes you have `llama-cpp-python` installed. Full LlamaCPP usage guide is available in a [notebook here](/examples/llm/llama_2_llama_cpp.ipynb).

See the [Custom LLM's How-To](/core_modules/model_modules/llms/usage_custom.md) for more details.

### Global ServiceContext

If you wanted the service context from the last section to always be the default, you can configure one like so:

```python
from llama_index import set_global_service_context
set_global_service_context(service_context)
```

This service context will always be used as the default if not specified as a keyword argument in LlamaIndex functions.

For more details on the service context, including how to create a global service context, see the page [Customizing the ServiceContext](/core_modules/supporting_modules/service_context.md).

### Customizing Prompts

Depending on the index used, we used default prompt templates for constructing the index (and also insertion/querying).
See [Custom Prompts How-To](/core_modules/model_modules/prompts.md) for more details on how to customize your prompt.

### Customizing embeddings

For embedding-based indices, you can choose to pass in a custom embedding model. See
[Custom Embeddings How-To](/core_modules/model_modules/embeddings/usage_pattern.md) for more details.

### Cost Analysis 

Creating an index, inserting to an index, and querying an index may use tokens. We can track
token usage through the outputs of these operations. When running operations,
the token usage will be printed.

You can also fetch the token usage through `TokenCountingCallback` handler.
See [Cost Analysis How-To](/core_modules/supporting_modules/cost_analysis/usage_pattern.md) for more details.

### [Optional] Save the index for future use

By default, data is stored in-memory.
To persist to disk:

```python
index.storage_context.persist(persist_dir="<persist_dir>")
```

You may omit persist_dir to persist to `./storage` by default.

To reload from disk:

```python
from llama_index import StorageContext, load_index_from_storage

# rebuild storage context
storage_context = StorageContext.from_defaults(persist_dir="<persist_dir>")

# load index
index = load_index_from_storage(storage_context)
```

**NOTE**: If you had initialized the index with a custom
`ServiceContext` object, you will also need to pass in the same
ServiceContext during `load_index_from_storage` or ensure you have a global sevice context.

```python

service_context = ServiceContext.from_defaults(llm=llm)
set_global_service_context(service_context)

# when first building the index
index = VectorStoreIndex.from_documents(
    documents, # service_context=service_context -> optional if not using global
)

...

# when loading the index from disk
index = load_index_from_storage(
    StorageContext.from_defaults(persist_dir="<persist_dir>")
    # service_context=service_context -> optional if not using global
)

```

## 4. [Optional, Advanced] Building indices on top of other indices

You can build indices on top of other indices!
Composability gives you greater power in indexing your heterogeneous sources of data. For a discussion on relevant use cases,
see our [Query Use Cases](/end_to_end_tutorials/question_and_answer.md). For technical details and examples, see our [Composability How-To](/core_modules/data_modules/index/composability.md).

## 5. Query the index.

After building the index, you can now query it with a `QueryEngine`. Note that a "query" is simply an input to an LLM -
this means that you can use the index for question-answering, but you can also do more than that!

### High-level API

To start, you can query an index with the default `QueryEngine` (i.e., using default configs), as follows:

```python
query_engine = index.as_query_engine()
response = query_engine.query("What did the author do growing up?")
print(response)

response = query_engine.query("Write an email to the user given their background information.")
print(response)
```

### Low-level API

We also support a low-level composition API that gives you more granular control over the query logic.
Below we highlight a few of the possible customizations.

```python
from llama_index import (
    VectorStoreIndex,
    get_response_synthesizer,
)
from llama_index.retrievers import VectorIndexRetriever
from llama_index.query_engine import RetrieverQueryEngine
from llama_index.indices.postprocessor import SimilarityPostprocessor

# build index
index = VectorStoreIndex.from_documents(documents)

# configure retriever
retriever = VectorIndexRetriever(
    index=index,
    similarity_top_k=2,
)

# configure response synthesizer
response_synthesizer = get_response_synthesizer()

# assemble query engine
query_engine = RetrieverQueryEngine(
    retriever=retriever,
    response_synthesizer=response_synthesizer,
    node_postprocessors=[
        SimilarityPostprocessor(similarity_cutoff=0.7)
    ]

)

# query
response = query_engine.query("What did the author do growing up?")
print(response)
```

You may also add your own retrieval, response synthesis, and overall query logic, by implementing the corresponding interfaces.

For a full list of implemented components and the supported configurations, please see the detailed [reference docs](/api_reference/query.rst).

In the following, we discuss some commonly used configurations in detail.

### Configuring retriever

An index can have a variety of index-specific retrieval modes.
For instance, a list index supports the default `ListIndexRetriever` that retrieves all nodes, and
`ListIndexEmbeddingRetriever` that retrieves the top-k nodes by embedding similarity.

For convienience, you can also use the following shorthand:

```python
    # ListIndexRetriever
    retriever = index.as_retriever(retriever_mode='default')
    # ListIndexEmbeddingRetriever
    retriever = index.as_retriever(retriever_mode='embedding')
```

After choosing your desired retriever, you can construct your query engine:

```python
query_engine = RetrieverQueryEngine(retriever)
response = query_engine.query("What did the author do growing up?")
```

The full list of retrievers for each index (and their shorthand) is documented in the [Query Reference](/api_reference/query.rst).

(setting-response-mode)=

### Configuring response synthesis
After a retriever fetches relevant nodes, a `BaseSynthesizer` synthesizes the final response by combining the information.

You can configure it via

```python
query_engine = RetrieverQueryEngine.from_args(retriever, response_mode=<response_mode>)
```

Right now, we support the following options:

- `default`: "create and refine" an answer by sequentially going through each retrieved `Node`;
  This makes a separate LLM call per Node. Good for more detailed answers.
- `compact`: "compact" the prompt during each LLM call by stuffing as
  many `Node` text chunks that can fit within the maximum prompt size. If there are
  too many chunks to stuff in one prompt, "create and refine" an answer by going through
  multiple prompts.
- `tree_summarize`: Given a set of `Node` objects and the query, recursively construct a tree
  and return the root node as the response. Good for summarization purposes.
- `no_text`: Only runs the retriever to fetch the nodes that would have been sent to the LLM,
  without actually sending them. Then can be inspected by checking `response.source_nodes`.
  The response object is covered in more detail in Section 5.
- `accumulate`: Given a set of `Node` objects and the query, apply the query to each `Node` text
  chunk while accumulating the responses into an array. Returns a concatenated string of all
  responses. Good for when you need to run the same query separately against each text
  chunk.

```python
index = ListIndex.from_documents(documents)
retriever = index.as_retriever()

# default
query_engine = RetrieverQueryEngine.from_args(retriever, response_mode='default')
response = query_engine.query("What did the author do growing up?")

# compact
query_engine = RetrieverQueryEngine.from_args(retriever, response_mode='compact')
response = query_engine.query("What did the author do growing up?")

# tree summarize
query_engine = RetrieverQueryEngine.from_args(retriever, response_mode='tree_summarize')
response = query_engine.query("What did the author do growing up?")

# no text
query_engine = RetrieverQueryEngine.from_args(retriever, response_mode='no_text')
response = query_engine.query("What did the author do growing up?")
```

### Configuring node postprocessors (i.e. filtering and augmentation)

We also support advanced `Node` filtering and augmentation that can further improve the relevancy of the retrieved `Node` objects.
This can help reduce the time/number of LLM calls/cost or improve response quality.

For example:

- `KeywordNodePostprocessor`: filters nodes by `required_keywords` and `exclude_keywords`.
- `SimilarityPostprocessor`: filters nodes by setting a threshold on the similarity score (thus only supported by embedding-based retrievers)
- `PrevNextNodePostprocessor`: augments retrieved `Node` objects with additional relevant context based on `Node` relationships.

The full list of node postprocessors is documented in the [Node Postprocessor Reference](/api_reference/node_postprocessor.rst).

To configure the desired node postprocessors:

```python
node_postprocessors = [
    KeywordNodePostprocessor(
        required_keywords=["Combinator"],
        exclude_keywords=["Italy"]
    )
]
query_engine = RetrieverQueryEngine.from_args(
    retriever, node_postprocessors=node_postprocessors
)
response = query_engine.query("What did the author do growing up?")
```

## 5. Parsing the response

The object returned is a [`Response` object](/api_reference/response.rst).
The object contains both the response text as well as the "sources" of the response:

```python
response = query_engine.query("<query_str>")

# get response
# response.response
str(response)

# get sources
response.source_nodes
# formatted sources
response.get_formatted_sources()
```

An example is shown below.
![](/_static/response/response_1.jpeg)
