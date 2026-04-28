# RAGFlow Tools Available To The Agent

This markdown describes the MCP tools exposed by `ragflow_mcp`.

Server name: `ragflow-mcp`

## What The Agent Can Do

The agent can use this server to manage knowledge-base content and run semantic retrieval against RAGFlow.

Typical uses:

* create or inspect datasets
* upload and manage documents
* inspect parsed chunks
* search knowledge bases semantically
* query the knowledge graph
* manage tagging metadata

## Accessible Tools

| Tool                        | Purpose                                            |
| --------------------------- | -------------------------------------------------- |
| `create_dataset`            | Create a knowledge base                            |
| `list_datasets`             | List or search datasets                            |
| `get_dataset_detail`        | Fetch full configuration and stats for one dataset |
| `update_dataset`            | Update dataset settings                            |
| `delete_datasets`           | Delete dataset records                             |
| `upload_with_metadata`      | Upload a file and assign metadata                  |
| `list_docs`                 | List documents in a dataset                        |
| `doc_infos`                 | Fetch document details                             |
| `delete_docs`               | Delete documents by ID                             |
| `rename_doc`                | Rename a document                                  |
| `set_doc_metadata`          | Set metadata fields on a document                  |
| `batch_update_doc_metadata` | Update metadata across multiple documents          |
| `list_chunks`               | List parsed chunks for a document                  |
| `retrieval`                 | Run semantic retrieval across one or more datasets |
| `retrieve_knowledge_graph`  | Query the knowledge graph for a dataset            |
| `download_attachment`       | Download an attachment by chunk ID                 |
| `list_tags`                 | List tags across datasets                          |
| `set_tags`                  | Assign or update dataset tags                      |

## Tool Families

### Dataset management

* `create_dataset`
* `list_datasets`
* `get_dataset_detail`
* `update_dataset`
* `delete_datasets`

### Document management

* `upload_with_metadata`
* `list_docs`
* `doc_infos`
* `delete_docs`
* `rename_doc`
* `set_doc_metadata`
* `batch_update_doc_metadata`

### Chunk inspection

* `list_chunks`

### Retrieval and graph operations

* `retrieval`
* `retrieve_knowledge_graph`
* `download_attachment`

### Tag operations

* `list_tags`
* `set_tags`

