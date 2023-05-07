Here's an updated `README.md` file for your repository:

---

# Vector Database with OpenAI Embeddings using Milvus

This repository contains a Docker container setup for Milvus, an open-source vector similarity search engine, and a Python script to generate and store OpenAI embeddings in Milvus.

## Requirements

- Docker
- Docker Compose
- Python 3.6+
- pip

## Getting Started

### 1. Set up Milvus with Docker Compose

#### a. Create a docker-compose.yml file

Create a new directory for your project and inside it, create a file named `docker-compose.yml` with the following content:

```yaml
version: '3.5'
services:
  milvus:
    image: milvusdb/milvus:2.0.0-rc5-hotfix1-cpu-d050721-5e559c
    container_name: milvus_cpu
    ports:
      - "19530:19530"
      - "19121:19121"
    environment:
      - "TZ=UTC"
    volumes:
      - /var/lib/milvus/db:/var/lib/milvus/db
      - /var/lib/milvus/conf:/var/lib/milvus/conf
      - /var/lib/milvus/logs:/var/lib/milvus/logs
      - /var/lib/milvus/wal:/var/lib/milvus/wal
```

#### b. Start the Milvus container

Navigate to your project directory in the terminal and run the following command to start the Milvus container:

```bash
docker-compose up -d
```

### 2. Install required Python libraries

Install the required libraries using pip:

```bash
pip install pymilvus==2.0.0rc5 torch transformers
```

### 3. Generate embeddings and populate the Milvus database

#### a. Create a Python script `openai_embeddings_milvus.py`:

```python
from pymilvus import DataType, CollectionSchema, FieldSchema, Collection, connections, utility
import torch
from transformers import GPT2Model, GPT2Tokenizer

def get_embedding(text):
    tokenizer = GPT2Tokenizer.from_pretrained("gpt2")
    model = GPT2Model.from_pretrained("gpt2")

    input_ids = tokenizer.encode(text, return_tensors="pt")
    outputs = model(input_ids)
    embeddings = outputs.last_hidden_state.mean(dim=1).detach().numpy()

    return embeddings

def main():
    # Connect to Milvus
    connections.connect("default", host="127.0.0.1", port="19530")

    # Define the collection schema
    collection_name = "openai_embeddings"
    dimension = 768  # Assuming embeddings have 768 dimensions
    schema = CollectionSchema(
        fields=[
            FieldSchema(name="id", dtype=DataType.INT64, is_primary=True, auto_id=True),
            FieldSchema(name="embedding", dtype=DataType.FLOAT_VECTOR, dim=dimension)
        ],
        description="OpenAI embeddings collection"
    )

    # Create the collection
    collection = Collection(name=collection_name, schema=schema)

    # Insert embeddings
    example_embedding = get_embedding("This is an example sentence.")
    example_data = [{"embedding": example_embedding.tolist()}]
    insert_result = collection.insert(example_data)

    # Search for similar embeddings
    search_params = {"metric_type": "L2", "params": {"nprobe": 10}}
    top_k = 5
    search_embedding = example_embedding
    search_results = collection.search(
        [search_embedding],
        anns_field="embedding",
        param=search_params,
        limit=top_k,
    )

    # Print search results
    for result in search_results:
        print(f"Top {top_k} most similar embeddings:")
        for idx, match in enumerate(result):
            print(f"{idx + 1}. ID: {match.id}, Distance: {match.distance}")

    # Disconnect from Milvus
    connections.remove("default")

if __name__ == "__main__":
    main()
    
```

#### b. Run the Python script

With the Milvus container running, execute the Python script to generate embeddings, populate the database, and perform a search:

```python
python openai_embeddings_milvus.py
```

This will connect to the Milvus database, create a collection for storing embeddings, insert an example embedding, and perform a search for similar embeddings.

---

With this setup, you can use Milvus, a specialized vector database, to efficiently store and search your OpenAI embeddings in a Docker container.
