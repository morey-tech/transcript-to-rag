# Transcript to RAG

A **RAG data pipeline project** to transform transcripts to hierarchical knowledge base.

## **Project Overview**

This project aims to build an automated, end-to-end data pipeline that ingests raw transcripts from a GitHub repository, processes them into a Hierarchical Summary / Document Tree structure using a local LLM hosted on OpenShift AI (utilizing an NVIDIA RTX 3090 24GB), and injects the structured data into AnythingLLM for user-facing Retrieval-Augmented Generation (RAG).

## **Architecture Flow**

1. **Source:** GitHub Actions scrapes transcripts weekly and commits them as Markdown.  
2. **Trigger:** GitHub Action webhook triggers an OpenShift AI Pipeline.  
3. **Processing (OpenShift AI):** A Jupyter/Elyra-backed pipeline pulls the Markdown, chunks it, and uses a locally hosted LLM to generate hierarchical summaries (Chunk level \-\> Chapter level \-\> Document level).  
4. **Destination:** The pipeline generates embeddings using the embedding model and stores them in OpenShift AI's vector database (Milvus). The structured documents are also pushed to AnythingLLM for the chat interface.
5. **Serving:** AnythingLLM serves the data to users via chat interface, querying the OpenShift AI vector database for retrieval.

## **Phase 1: Infrastructure & Model Serving (OpenShift AI)**

**Objective:** Stand up the LLM and Embedding models on the 3090 GPU to support the summarization and generation tasks, and deploy the vector database infrastructure.

* **Task 1.0:** Deploy OpenShift AI via Red Hat operator to the [ocp-gpu cluster](https://github.com/morey-tech/homelab/blob/main/kubernetes/ocp-gpu).
* **Task 1.1:** Deploy and configure OpenShift AI's vector database (Milvus) to store document embeddings. Ensure proper indexing and connection pooling for efficient vector similarity search.
* **Task 1.2:** Migrate the existing [inference server deployment](https://github.com/morey-tech/homelab/blob/main/kubernetes/ocp-gpu/applications/inference-server) to use OpenShift AI constructs. The LLM model [RedHatAI/Qwen3-30B-A3B-quantized.w4a16](https://huggingface.co/RedHatAI/Qwen3-30B-A3B-quantized.w4a16/tree/main) is currently running as a plain pod and needs to be refactored to use OpenShift AI Single-model serving (vLLM/TGI). This task should be completed after OpenShift AI is deployed in Task 1.0.
* **Task 1.3:** Expose the LLM via an OpenAI-compatible API endpoint within the cluster.
* **Task 1.4:** Configure AnythingLLM to point to this internal OpenShift AI endpoint for chat inference.
* **Task 1.5:** Deploy [nomic-embed-text-v1.5](https://huggingface.co/RedHatAI/nomic-embed-text-v1.5/tree/main) embedding model (5.84 GB) on CPU and expose it via API endpoint for the data pipeline to generate embeddings that will be stored in the OpenShift AI vector database.

## **Phase 2: Hierarchical Summarization Engine (Jupyter Notebook)**

**Objective:** Develop the core Python logic to transform raw transcripts into a Document Tree.

* **Task 2.1:** Create a JupyterLab Workbench in OpenShift AI with required libraries (langchain, requests, openai).  
* **Task 2.2:** Write ingestion script to pull the latest Markdown commits from the local/cloned repo.  
* **Task 2.3 (Base Level):** Implement chunking logic to split transcripts into readable segments (e.g., 5-10 minute blocks of dialogue).  
* **Task 2.4 (Level 1 Summaries):** Write prompts and LangChain logic to pass each chunk to the local LLM to generate a "Chapter Summary" with key quotes.  
* **Task 2.5 (Level 2 Summary):** Write logic to pass all "Chapter Summaries" to the LLM to generate a "Global Executive Summary" for the entire transcript.  
* **Task 2.6 (Formatting):** Stitch the Global Summary, Chapter Summaries, and Raw Chunks into a nested Markdown document with internal links/references.

## **Phase 3: Vector Database & AnythingLLM Integration**

**Objective:** Automate the embedding generation and storage in OpenShift AI's vector database, and integrate structured documents with AnythingLLM for chat interface.

* **Task 3.1:** Write Python code in the Jupyter Notebook to call the embedding model API endpoint to generate embeddings for the hierarchical Markdown chunks.
* **Task 3.2:** Implement database connection logic to store the generated embeddings along with document metadata in the OpenShift AI vector database (Milvus).
* **Task 3.3:** Configure vector similarity search queries and test retrieval performance from the vector database.
* **Task 3.4:** Generate an AnythingLLM Developer API key for chat interface integration.
* **Task 3.5:** Write Python code to POST the final hierarchical Markdown files to the AnythingLLM /api/v1/document/add endpoint for serving via chat interface.
* **Task 3.6:** Configure AnythingLLM to connect to the external OpenShift AI vector database for document retrieval instead of using an embedded vector database.

## **Phase 4: Pipeline Automation (CI/CD & Elyra)**

**Objective:** Remove manual intervention by linking the weekly GitHub scrape to the OpenShift AI pipeline execution.

* **Task 4.1:** Use Elyra within the OpenShift AI Workbench to convert the Jupyter Notebook into a Kubeflow/Data Science Pipeline node.  
* **Task 4.2:** Publish the pipeline and expose a webhook trigger URL.  
* **Task 4.3:** Update the existing GitHub Action workflow (that scrapes the transcripts) to include a final curl step that hits the OpenShift AI pipeline webhook upon a successful commit.

## **User Stories (Agile Backlog)**

### **Epic 1: Infrastructure**

* **US 1.1:** As a System Admin, I want to deploy an 8B parameter LLM on my 3090 GPU in OpenShift AI so that I have a fast, local, and private engine for both chat and data summarization.  
  * *Acceptance Criteria:* Model is running, VRAM usage is under 20GB, and the endpoint responds successfully to an OpenAI-compatible API request.

### **Epic 2: Data Transformation (The Document Tree)**

* **US 2.1:** As a Data Engineer, I want the Jupyter Notebook to automatically chunk raw transcript markdown into smaller, logical dialogue blocks so that the LLM can process them without losing context.  
* **US 2.2:** As a Data Engineer, I want the local LLM to generate a summary for each individual chunk so that granular details are preserved alongside a high-level overview.  
* **US 2.3:** As a Data Engineer, I want the local LLM to read all chunk summaries and generate a single "Global Summary" for the transcript so that users asking broad questions get immediate, accurate answers without retrieving the whole document.  
* **US 2.4:** As a Data Engineer, I want to output a formatted Markdown file that contains the Global Summary at the top, followed by Chapter Summaries, followed by the raw chunks, so that the vector database can store the hierarchical relationships and AnythingLLM can serve them via chat interface.

### **Epic 3: Vector Database & AnythingLLM Integration**

* **US 3.1:** As a RAG Administrator, I want the Python script to automatically generate embeddings using the embedding model and store them in the OpenShift AI vector database so that I have full control over the vector storage infrastructure.
* **US 3.2:** As a RAG Administrator, I want the script to push the finalized hierarchical markdown files to AnythingLLM via API so that users can interact with the data through a chat interface.
* **US 3.3:** As a RAG Administrator, I want AnythingLLM to query the external OpenShift AI vector database for document retrieval so that all vector data is centralized and not duplicated in embedded databases.

### **Epic 4: Automation**

* **US 4.1:** As a DevOps Engineer, I want the weekly GitHub Action scraper to trigger the OpenShift AI Data Science Pipeline automatically so that new transcripts are processed, embedded, and stored in the vector database without human intervention.
  * *Acceptance Criteria:* A new commit to the transcripts folder in GitHub results in the embeddings being stored in the OpenShift AI vector database and the data appearing in AnythingLLM within 15 minutes.