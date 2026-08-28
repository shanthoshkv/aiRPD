# aiRPD

### AI-Powered Research Paper Downloader and Relevance Filter

**aiRPD** is a local-first research paper discovery and filtering tool designed to automate one of the most tedious parts of technical research: finding the papers that are actually relevant to a specific research question.

Instead of simply searching for papers and downloading everything returned by a keyword query, aiRPD adds an AI-based relevance filtering layer.

The workflow is:

```text
Research Query
      ↓
Multi-Source Search
      ↓
Paper Discovery
      ↓
PDF Download
      ↓
Text Extraction
      ↓
Local AI Relevance Analysis
      ↓
Relevant / Irrelevant
      ↓
Organized Local Library
```

The relevance analysis is performed locally using **Ollama with Qwen3:8B**, so the downloaded research material does not need to be sent to an external LLM service.

## Live Project

**Website:** https://aboutkvs.vercel.app/airpd.html

---

# Why I Built This

Literature review becomes surprisingly time-consuming once a research topic becomes even slightly specific.

A typical workflow looks something like:

```text
Search Google Scholar / arXiv
        ↓
Open paper
        ↓
Read abstract
        ↓
Download PDF
        ↓
Check whether it is actually useful
        ↓
Repeat
```

Doing this for hundreds of papers quickly becomes repetitive.

Keyword-based search also has an obvious limitation.

A paper can contain all the right keywords while still being irrelevant to the actual research question.

For example, a search involving:

```text
rocket combustion
```

could return papers about:

* Combustion chemistry
* Internal ballistics
* Solid propellants
* Combustion instability
* Injector design
* Turbomachinery
* Experimental diagnostics
* Numerical combustion

Only a fraction of those papers may actually be useful for a particular project.

aiRPD adds a semantic filtering stage that allows a local language model to inspect the extracted paper content and decide whether the paper is relevant to the user's query.

The goal is simple:

> **Download fewer papers, but make the papers you keep more useful.**

---

# Core Architecture

The system is designed as a modular research-processing pipeline.

```text
┌─────────────────────────────┐
│       Research Query        │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│     Multi-Source Search     │
│                             │
│  arXiv                      │
│  DOAJ                       │
│  PubMed Central             │
│  PLOS ONE                   │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       Paper Discovery       │
│                             │
│ Metadata + PDF URL          │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       PDF Download          │
│                             │
│ Requests + Error Handling   │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│       Text Extraction       │
│                             │
│ PyPDF2                      │
│ Page-limited processing     │
└──────────────┬──────────────┘
               │
               ▼
┌─────────────────────────────┐
│      Local AI Analysis      │
│                             │
│      Ollama Qwen3:8B        │
└──────────────┬──────────────┘
               │
        ┌──────┴──────┐
        ▼             ▼
   Relevant       Irrelevant
        │             │
        ▼             ▼
     Save          Reject
        │
        ▼
┌─────────────────────────────┐
│     Organized Paper         │
│          Library            │
└─────────────────────────────┘
```

The architecture separates paper discovery, document processing, AI analysis, and storage so that individual components can be improved independently.

---

# Multi-Source Research Discovery

aiRPD searches across multiple open academic repositories.

Currently supported sources include:

### arXiv

A large open repository covering areas such as:

* Physics
* Mathematics
* Computer science
* Statistics
* Engineering-related research

### DOAJ

The Directory of Open Access Journals provides access to a broad range of open-access academic literature.

### PubMed Central

Useful for biomedical and life-science literature where full-text papers are available.

### PLOS ONE

Provides access to open scientific publications across a wide range of disciplines.

The multi-source approach prevents the system from depending on a single academic repository.

---

# Query Processing

The user provides a research query describing what they are looking for.

For example:

```text
Machine learning based diagnostics for combustion instability
```

or:

```text
Regenerative cooling of liquid rocket engines
```

The query is then sent to the supported academic sources.

Each source returns metadata and links to available paper documents.

The system can therefore work as a lightweight research-discovery layer before the papers enter the local analysis pipeline.

---

# Paper Download

Once candidate papers are identified, aiRPD retrieves the available PDF files.

The downloader uses Python's `requests` library for HTTP communication and file retrieval.

The implementation includes error handling so that an unavailable or malformed paper does not necessarily terminate the entire processing run.

This is important when processing large search results where individual repositories may occasionally return:

* Missing PDFs
* Invalid URLs
* HTTP errors
* Incomplete documents
* Unexpected response formats

The pipeline can continue processing the remaining candidates instead of treating every failed download as a fatal error.

---

# PDF Text Extraction

Downloaded papers are processed locally.

The current implementation uses **PyPDF2** to extract text from PDFs.

The extraction process is intentionally limited so that extremely large papers do not unnecessarily consume the entire context window of the local language model.

The current configuration processes up to:

```text
5 pages
```

per PDF and limits the extracted content supplied to the LLM to approximately:

```text
5000 characters
```

These parameters can be adjusted depending on the hardware and model being used.

---

# Local AI Relevance Filtering

This is the core feature of aiRPD.

Instead of assuming that a paper is relevant simply because its metadata matches a keyword query, the extracted paper content is passed to:

**Ollama Qwen3:8B**

The model evaluates the paper against the original research query.

The basic process is:

```text
User Query
      +
Extracted Paper Content
      ↓
Qwen3:8B
      ↓
Relevance Decision
      ↓
Relevant / Rejected
```

This introduces a semantic filtering layer between paper discovery and permanent storage.

The model can reason over the content rather than relying solely on title or keyword matching.

---

# Why Local AI?

The local inference architecture was an intentional design decision.

Research documents can contain:

* Unpublished research
* Proprietary technical information
* Experimental results
* Internal reports
* Sensitive datasets
* Early-stage research ideas

Sending every downloaded document to an external AI API is not always desirable.

With Ollama, the relevance analysis can be performed directly on the user's machine.

```text
                    LOCAL MACHINE

Research Query
      ↓
Paper Search
      ↓
PDF Download
      ↓
Text Extraction
      ↓
Qwen3:8B via Ollama
      ↓
Relevance Decision
      ↓
Local Storage
```

The project therefore follows a **privacy-first architecture** where the AI processing stage does not inherently require a cloud API.

---

# Intelligent Filtering

The AI filter is designed to answer a practical question:

> "Is this paper actually useful for the research query?"

This is different from asking:

> "Does this paper contain the search keywords?"

That distinction becomes increasingly important for interdisciplinary research.

For example, a query involving:

```text
machine learning + turbomachinery vibration
```

could retrieve papers about machine learning, vibration, or turbomachinery individually.

The AI filtering stage can examine whether the paper actually discusses the intersection of those topics.

---

# Rejected Paper Logging

A useful research tool should also explain what it rejected.

aiRPD maintains detailed logging for rejected papers, including the reason for rejection.

This creates an audit trail such as:

```text
Paper
   ↓
Rejected
   ↓
Reason recorded
```

This is useful when refining a research query because the researcher can identify whether the filter is rejecting papers due to:

* Wrong domain
* Insufficient relevance
* Unrelated methodology
* Different application
* Weak connection to the research question

The logging system also makes debugging the AI filtering process easier.

---

# Organized Paper Storage

Relevant papers are automatically organized into directories based on the research query.

The intended structure is approximately:

```text
papers/
│
├── query_1/
│   ├── paper_01.pdf
│   ├── paper_02.pdf
│   └── paper_03.pdf
│
├── query_2/
│   ├── paper_01.pdf
│   └── paper_02.pdf
│
└── query_3/
    └── paper_01.pdf
```

This makes the resulting collection easier to browse and prevents hundreds of unrelated PDFs from ending up in one directory.

The project also uses clear naming conventions to make the downloaded library easier to navigate.

---

# Processing Pipeline

The complete workflow can be summarized as:

```text
01. Query Input
       ↓
02. Search Academic Sources
       ↓
03. Collect Paper Metadata
       ↓
04. Retrieve PDF
       ↓
05. Extract Text
       ↓
06. Limit Context
       ↓
07. Send to Local Qwen3:8B
       ↓
08. Evaluate Relevance
       ↓
09. Reject or Save
       ↓
10. Log Result
       ↓
11. Organize Local Library
```

Each paper moves independently through the pipeline.

This makes the system suitable for batch processing.

---

# Processing Controls

The application exposes several parameters that can be tuned depending on the use case.

### Page Extraction Limit

Controls how many PDF pages are extracted for AI analysis.

Current default:

```text
5 pages
```

### Character Limit

Controls the amount of extracted text sent to the local model.

Current default:

```text
5000 characters
```

### Paper Processing Delay

A delay of approximately:

```text
2 seconds
```

is used between paper-processing operations.

### Query Delay

A delay of approximately:

```text
5 seconds
```

is used between queries.

These controls help make the downloader more predictable when interacting with external repositories and when processing large batches.

---

# Technology Stack

## Programming Language

**Python**

Python was chosen because of its mature ecosystem for:

* HTTP requests
* XML/HTML parsing
* PDF processing
* AI integration
* Data processing
* Automation

---

## HTTP

### Requests

Used for:

* API requests
* Metadata retrieval
* PDF downloads
* HTTP error handling

---

## Parsing

### BeautifulSoup

Used for HTML/XML parsing and extracting structured information from repository responses.

---

## AI

### Ollama

Provides the local model runtime.

### Qwen3:8B

Used for:

* Relevance checking
* Technical document analysis
* Context-aware filtering

The project specifically uses Qwen3:8B because it provides a useful balance between local inference requirements and reasoning capability for this type of task.

---

## PDF Processing

### PyPDF2

Used for extracting text from downloaded research papers.

---

# Repository APIs

The system interacts with multiple academic sources through their available interfaces.

```text
┌───────────────┐
│    arXiv      │
└───────┬───────┘
        │
┌───────▼───────┐
│     DOAJ      │
└───────┬───────┘
        │
┌───────▼──────────┐
│ PubMed Central   │
└───────┬──────────┘
        │
┌───────▼───────┐
│   PLOS ONE    │
└───────┬───────┘
        │
        ▼
   aiRPD Pipeline
```

The result is a unified research-discovery workflow rather than four separate search scripts.

---

# Performance Considerations

The pipeline intentionally balances retrieval volume with local processing cost.

If hundreds of papers are returned, sending every complete PDF to an LLM would be unnecessarily expensive computationally.

Instead, aiRPD limits the amount of document content used during the relevance stage.

```text
Large PDF
    ↓
Selected Pages
    ↓
Character Limit
    ↓
Qwen3:8B
```

This reduces local inference requirements and makes batch processing more practical.

The current website configuration specifies a five-page extraction limit, a 5000-character LLM context limit, a two-second processing delay, and a five-second query delay.

---

# Example Workflow

Suppose the research question is:

```text
Machine learning based detection of combustion instability in liquid rocket engines
```

A typical run would look like:

```text
Search repositories
       ↓
Find candidate papers
       ↓
Download PDFs
       ↓
Extract first 5 pages
       ↓
Send extracted content + query
       ↓
Qwen3:8B relevance analysis
       ↓
       ┌───────────────┐
       │               │
       ▼               ▼
   Relevant        Irrelevant
       │               │
       ▼               ▼
    Save PDF        Log reason
       │
       ▼
Query-specific folder
```

The result is a smaller collection of papers that is more closely aligned with the actual research objective.

---

# Research Workflow Integration

aiRPD is particularly useful as the first stage of a larger research automation pipeline.

For example:

```text
                 aiRPD
                   │
                   ▼
          Relevant Papers
                   │
                   ▼
             PDF Library
                   │
          ┌────────┴────────┐
          ▼                 ▼
       STEM RAG         Dataset Pipeline
          │                 │
          ▼                 ▼
   Research Assistant    Training Data
```

This makes aiRPD useful as a document acquisition layer for other research-oriented systems.

The downloaded papers can subsequently be passed into a RAG system, citation manager, dataset-generation pipeline, or local knowledge base.

---

# Potential Applications

## Academic Literature Review

Automate the initial screening of large search results.

## Engineering Research

Build focused collections of papers around a particular engineering problem.

## Aerospace Research

Search and filter literature involving:

* Propulsion
* Combustion
* Turbomachinery
* CFD
* Aerodynamics
* Structural dynamics
* Experimental diagnostics

## Machine Learning Research

Build domain-specific research collections for:

* Model development
* Dataset construction
* Literature surveys
* Benchmarking

## Private Research Libraries

Use local AI to classify technical documents without sending their contents to an external inference service.

---

# Design Principles

## Local First

AI analysis runs locally through Ollama.

## Open Access First

The system focuses on repositories that provide open research access.

## Relevance Before Storage

A paper should be evaluated before becoming part of the permanent local library.

## Transparent Filtering

Rejected papers and reasons are logged.

## Modular Architecture

Search, download, extraction, AI analysis, and storage remain separate components.

## Configurable Processing

Page limits, character limits, and processing delays can be adjusted depending on the workload.

---

# What I Learned Building This

The interesting part of this project was realizing that **research automation is not really a search problem**.

Search engines are already very good at finding papers.

The difficult part is what happens after the search.

You can easily end up with hundreds of papers that:

* Look relevant from the title
* Match the keywords
* Have related terminology
* But are not actually useful for the research question

The AI filtering stage was designed to address that gap.

I also wanted the system to remain local because research workflows increasingly involve documents that researchers may not want to upload to third-party AI services.

That combination led to the architecture used in aiRPD:

```text
Open Research Sources
        +
Local Document Processing
        +
Local LLM Reasoning
        =
Private Research Discovery Pipeline
```

---

# Future Improvements

There are several directions that would make the system substantially more powerful.

### Full-Paper Analysis

Instead of analysing only the first few pages, the system could perform hierarchical analysis across the entire document.

### Abstract-First Filtering

Use title and abstract analysis as a cheap first-stage filter before downloading or processing the full paper.

### Multi-Stage Ranking

Combine:

```text
Keyword Search
      ↓
Semantic Retrieval
      ↓
LLM Relevance
      ↓
Final Ranking
```

### Metadata Enrichment

Automatically collect:

* Authors
* DOI
* Publication date
* Citation count
* Journal
* Keywords

### Duplicate Detection

Identify the same paper appearing across multiple repositories.

### Citation Graphs

Build relationships between downloaded papers using their references and citations.

### Automatic Summarization

Generate structured summaries for accepted papers.

### Research Knowledge Base

Connect aiRPD directly to a RAG system so newly accepted papers become searchable immediately.

### Multimodal Paper Analysis

Extend the system to analyse:

* Figures
* Tables
* Equations
* Graphs
* Experimental plots

rather than relying primarily on extracted text.

---

# Project Status

The current system provides:

* Multi-source academic search
* arXiv integration
* DOAJ integration
* PubMed Central integration
* PLOS ONE integration
* Automated PDF downloading
* Local PDF text extraction
* Ollama Qwen3:8B relevance analysis
* Relevance-based filtering
* Rejection logging
* Query-based paper organization
* Configurable processing limits
* Local-first AI processing

The system is designed as a foundation for more advanced automated literature-review workflows.

---

# Live Project

Explore the full architecture and project details:

**https://aboutkvs.vercel.app/airpd.html**

---

## Author

**Shanthosh K V**

Aerospace Engineering Student
RV College of Engineering, Bengaluru

Interested in aerospace engineering, propulsion, turbomachinery, CFD, computational engineering, machine learning, research automation, and building practical engineering tools.

---

## License

© 2026 Shanthosh K V. All rights reserved.
