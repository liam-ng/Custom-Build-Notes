# Custom-Build-Notes
Imported Snippet of my Obsidian Markdown Notes, an analysis of Azure OpenAI Demo design provided by Microsoft

Azure OpenAI Demo with Enterprise-owned Data
![image](https://github.com/liam-ng/Custom-Build-Notes/assets/90180576/fd897428-a9be-406a-bf28-5cdf41b4692d)


## Repo preparation
https://github.com/Azure-Samples/azure-search-openai-demo/tree/main

Open `./app/frontend/vite.config.ts`
Change `outDir` from `../backend/static` to `../static` 

Copy `./app/frontend` to `./`

Git push to the Git repo

## Azure

1. Create storage account
2. Create blob container and upload documents

3. Create Document intelligence
	1. Select *S0 Standard tier* under Pricing Tier
4. Create Cognitive Search
	1. Select at least *Basic* tier
	2. Select open *BOTH authentication* (RBAC and Key) under API access control
	3. Enable *free plan* in semantic search under setting

5. Create OpenAI
6. Create 2 x Deployment ("gpt-35-turbo" and "embedding")

7. Create App service
	1. create App Service Plan Basic B1
	2. create Application Insight
8. Create *system managed identity* inside *web app* and assign following details 
	1. Scope:  Resource Group
	2. Roles:
		1. Storage Blob Data Reader
		2. Search Index Data Reader
		3. Cognitive Services OpenAI User

9. Populate *appSettings*  
![image](https://github.com/liam-ng/Custom-Build-Notes/assets/90180576/aadaf987-443b-4bca-a35f-a34ea96e48c6)


10. Go Configuration > General Setting > Platform settings, 
- set *always on* to true
- set *appCommandLine*  `python3 -m gunicorn main:app`
> [!note]  Note: Can add application insight

```bash
azd auth login --tenant-id "e301f0d0-7bca-4922-a781-253924f32cf7"

# install once only
python3.10 -m venv .venv
.venv/bin/python3.10 -m pip install -r requirements.txt

.venv/bin/python3.10 ./prepdocs.py './data/*' \
	--storageaccount "stkstest01" \
	--container "files" \
	--storagekey "0A+AbypG5voEjMyw6sXeHINS6PLhqdlL7N5S/THbuFRx561NCdKkW61ZH6dVh87gUct496QWF7SF+AStTxTBwg==" \
	--searchservice "search-kstest" \
	--searchkey "a3Jie7l2aYzql7bA4SqJwlSaLl3SLFCvGGWgOEA3pVAzSeCx6hVl" \
	--index "index" \
	--openaiservice "open-kstest" \
	--openaideployment "embedding" \
	--openaimodelname "text-embedding-ada-002" \
	--openaikey "c15ff43ec508403e83f572b42fac37d6" \
	--formrecognizerservice "doc-kstest" \
	--formrecognizerkey "c32fcc4080b94718a797830cf5a08b7f" \
	--tenantid "e301f0d0-7bca-4922-a781-253924f32cf7" \
	-v
```

11. Link  GitHub in Deployment Center
12. Update `main_apptgkstest.yaml`
```yaml
name: Build and deploy Python app to Azure Web App - apptgkstest

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    environment:
      name: 'Production'
      url: ${{ steps.deploy-to-webapp.outputs.webapp-url }}
      
    steps:
      - uses: actions/checkout@v2

      - name: Set up Python version
        uses: actions/setup-python@v1
        with:
          python-version: '3.10'

      - name: Create and start virtual environment
        run: |
          python -m venv venv
          source venv/bin/activate
      
      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Compile React by Vite
        run: |
          cd ./frontend
          npm install
          npm run build
          cd ..
          
      - name: 'Deploy to Azure Web App'
        uses: azure/webapps-deploy@v2
        id: deploy-to-webapp
        with:
          app-name: 'apptgkstest'
          slot-name: 'Production'
          publish-profile: ${{ secrets.AZUREAPPSERVICE_PUBLISHPROFILE_4E458739341B417CA72A00D5BE054484 }}
```

## Prepdoc
Step 1-4
![image](https://github.com/liam-ng/Custom-Build-Notes/assets/90180576/d458efd7-7b06-4ab7-869f-220a6f81a6f3)

**Step 1: Structure your internal documents  **
**Step 2: Embed the text corpus  **
**Step 3: Store vector embeddings  **
**Step 4: Save text representations  **

### scripts/prepdocs.py

The purpose of the script `prepdocs.py` is to prepare documents for indexing and search using Azure services, specifically Azure Search and Azure Form Recognizer. It performs various tasks such as uploading documents to Azure Blob Storage, extracting text from documents, splitting text into sections, and creating sections for indexing.

Let's go through the purpose of each function in the script:

1. `calculate_tokens_emb_aoai(input: str)`: This function calculates the number of tokens in an input string based on the chosen OpenAI language model. It uses the `tiktoken` library to determine the token count.

2. `blob_name_from_file_page(filename, page=0)`: This function generates a unique name for a blob (file) based on the input filename and page number. It appends the page number to the filename if the file is a PDF.

3. `upload_blobs(filename)`: This function uploads a file (or its pages, in the case of PDF) to Azure Blob Storage. It uses the `BlobServiceClient` to connect to the storage account and upload the file as a blob.

4. `remove_blobs(filename)`: This function removes blobs from Azure Blob Storage. It deletes the specified blobs or all blobs associated with a given filename.

5. `table_to_html(table)`: This function converts a table object into an HTML representation. It iterates over the cells of the table and generates the HTML markup for the table structure.

6. `get_document_text(filename)`: This function extracts the text content from a document. If the document is a PDF, it uses the `PdfReader` library to extract text from each page. Otherwise, it uses Azure Form Recognizer to perform layout analysis and extract text.

7. `split_text(page_map, filename)`: This function splits the extracted text into sections. It ensures that each section has a maximum length (`MAX_SECTION_LENGTH`) and attempts to split the text at sentence boundaries. It returns a generator that yields the sections along with their corresponding page numbers.

8. `filename_to_id(filename)`: This function generates a unique ID for a filename. It removes special characters from the filename, replaces them with underscores, and appends a hash of the filename for uniqueness.

9. `create_sections(filename, page_map, use_vectors, embedding_deployment: str = None, embedding_model: str = None)`: This function creates sections from the extracted text. It takes the filename, page map, and optional parameters for vector search using embeddings. The resulting sections are used for indexing and search operations.


These functions, along with other code in the script, provide the necessary functionality to prepare documents for indexing and searching in Azure Search.

## Frontend (App Service)
Step 5-9
![image](https://github.com/liam-ng/Custom-Build-Notes/assets/90180576/a602589d-c5a7-4834-87cb-a546f506b7b5)

**Step 5: Embed the question  **
**Step 6: Run a query  **
**Step 7: Retrieve similar vectors  **
**Step 8: Map vectors to text chunks  **
**Step 9: Generate the answer  **

### API flow
![image](https://github.com/liam-ng/Custom-Build-Notes/assets/90180576/20f25dd3-9bd0-4945-90ce-0cb54b2651f1)


### app/backend/app.py

- `ask()`: This function handles the "/ask" endpoint and performs question-answering using the specified approach. It receives a JSON payload containing the question, approach, and optional overrides, and returns the answer generated by the OpenAI model.
    
- `chat()`: This function handles the "/chat" endpoint and performs chat-based interactions using the specified approach. It receives a JSON payload containing the chat history, approach, and optional overrides, and returns the responses generated by the OpenAI model.
    
- `format_as_ndjson(r)`: This is a helper function that takes an asynchronous generator of dictionaries and converts each dictionary to a newline-delimited JSON string.
    
- `chat_stream()`: This function handles the "/chat_stream" endpoint and performs chat-based interactions with streaming responses. It receives a JSON payload containing the chat history, approach, and optional overrides. Instead of returning a single response, it returns a stream of responses in newline-delimited JSON format.
    
- `setup_clients()`: This is a before-app-serving function that sets up the clients for Azure Cognitive Search and Azure Blob Storage. It initializes the necessary client instances and configures the OpenAI SDK based on the deployment environment. Refers to the *appSetting* of App Service

