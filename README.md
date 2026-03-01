# Transcript to RAG

A **RAG data pipeline project** to transform transcripts to hierarchical knowledge base.

## **Project Overview**

This project aims to build an automated, end-to-end data pipeline that ingests raw transcripts from a GitHub repository, processes them into a Hierarchical Summary / Document Tree structure using a local LLM hosted on OpenShift AI (utilizing an NVIDIA RTX 3090 24GB), and injects the structured data into AnythingLLM for user-facing Retrieval-Augmented Generation (RAG).

## **Architecture Flow**

1. **Source:** GitHub Actions scrapes transcripts weekly and commits them as Markdown.  
2. **Trigger:** GitHub Action webhook triggers an OpenShift AI Pipeline.  
3. **Processing (OpenShift AI):** A Jupyter/Elyra-backed pipeline pulls the Markdown, chunks it, and uses a locally hosted LLM to generate hierarchical summaries (Chunk level \-\> Chapter level \-\> Document level).  
4. **Destination:** The pipeline pushes the linked hierarchical Markdown documents to AnythingLLM via its Developer API and triggers an embedding update.  
5. **Serving:** AnythingLLM serves the data to users via chat interface.

## **Phase 1: Infrastructure & Model Serving (OpenShift AI)**

**Objective:** Stand up the LLM and Embedding models on the 3090 GPU to support the summarization and generation tasks.

* **Task 1.1:** Deploy a lightweight LLM (e.g., Llama-3-8B-Instruct or Mistral-7B-Instruct) using OpenShift AI Single-model serving (vLLM/TGI).  
* **Task 1.2:** Expose the LLM via an OpenAI-compatible API endpoint within the cluster.  
* **Task 1.3:** Configure AnythingLLM to point to this internal OpenShift AI endpoint for chat inference.  
* **Task 1.4:** Deploy an embedding model (e.g., nomic-embed-text) on CPU or remaining GPU VRAM and connect it to AnythingLLM.

## **Phase 2: Hierarchical Summarization Engine (Jupyter Notebook)**

**Objective:** Develop the core Python logic to transform raw transcripts into a Document Tree.

* **Task 2.1:** Create a JupyterLab Workbench in OpenShift AI with required libraries (langchain, requests, openai).  
* **Task 2.2:** Write ingestion script to pull the latest Markdown commits from the local/cloned repo.  
* **Task 2.3 (Base Level):** Implement chunking logic to split transcripts into readable segments (e.g., 5-10 minute blocks of dialogue).  
* **Task 2.4 (Level 1 Summaries):** Write prompts and LangChain logic to pass each chunk to the local LLM to generate a "Chapter Summary" with key quotes.  
* **Task 2.5 (Level 2 Summary):** Write logic to pass all "Chapter Summaries" to the LLM to generate a "Global Executive Summary" for the entire transcript.  
* **Task 2.6 (Formatting):** Stitch the Global Summary, Chapter Summaries, and Raw Chunks into a nested Markdown document with internal links/references.

## **Phase 3: AnythingLLM API Integration**

**Objective:** Automate the ingestion of the structured Document Trees into the RAG vector database.

* **Task 3.1:** Generate an AnythingLLM Developer API key.  
* **Task 3.2:** Write Python code in the Jupyter Notebook to POST the final hierarchical Markdown files to the /api/v1/document/add endpoint.  
* **Task 3.3:** Add logic to move the uploaded documents into the specific target Workspace.  
* **Task 3.4:** Trigger the /api/v1/workspace/{slug}/update-embeddings endpoint to vectorize the new data.

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
* **US 2.4:** As a Data Engineer, I want to output a formatted Markdown file that contains the Global Summary at the top, followed by Chapter Summaries, followed by the raw chunks, so that AnythingLLM can index the relationships hierarchically.

### **Epic 3: AnythingLLM Integration**

* **US 3.1:** As a RAG Administrator, I want the Python script to push the finalized hierarchical markdown files directly to AnythingLLM via API so that I don't have to manually upload files through the UI.  
* **US 3.2:** As a RAG Administrator, I want the script to automatically trigger an embedding update in AnythingLLM so that the new transcripts are instantly searchable by end-users.

### **Epic 4: Automation**

* **US 4.1:** As a DevOps Engineer, I want the weekly GitHub Action scraper to trigger the OpenShift AI Data Science Pipeline automatically so that new transcripts are processed and vectorized without human intervention.  
  * *Acceptance Criteria:* A new commit to the transcripts folder in GitHub results in the data appearing in AnythingLLM within 15 minutes.