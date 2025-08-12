# ehr-project
LLM-enhanced Applicant Tracking System (ATS)
# PDF Resume & Job Description Matching Pipeline
![Databricks Workflow](./databricks%20workflow.png)

## Overview
This project implements an end-to-end **resume–job description matching pipeline** using **Databricks** and the **Medallion Architecture** (Bronze → Silver → Gold).

The system ingests binary PDF documents (candidate resumes and job descriptions) from a cloud data storage (GitHub Repo) into a **Databricks Volume**, processes them through structured data transformation layers, extracts key insights using **Databricks AI Query**, and creates a **vector search index** for matching. The ingestion is incremental and all databricks persistent objects are **CDC enabled**.

At each run, the job description is queried against the profile vector index to produce a **Gold table** of candidates ranked by a fitness score based on matching criteria.

---

## Architecture
The pipeline follows Databricks’ **Medallion Architecture**:

1. **Bronze Layer** – Raw ingestion of PDF data from cloud storage.  
2. **Silver Layer** – Cleaned, structured, and enriched data after AI-based extraction.  
3. **Gold Layer** – Final matching results with ranked candidates based on semantic similarity and criteria weights.

---

## Workflow

### 1. Data Ingestion
- **Ingest_Profile_to_Src**: Loads candidate resume PDFs from cloud storage into Databricks Volumes.  
- **Ingest_JD_to_Src**: Loads job description PDFs from cloud storage into Databricks Volumes.

### 2. Bronze Layer
- **Profile_Source_to_Bronze**: Converts raw profile PDF binary into structured Bronze tables.  
- **JD_Source_to_Bronze**: Converts raw job description PDF binary into structured Bronze tables.

### 3. Silver Layer
- **Profile_Bronze_to_Silver**:  
  - Uses **Databricks AI Query** to extract structured insights from resumes (skills, experience, education, certifications, etc.).  
  - Stores enriched profile data in Silver tables.
- **JD_Bronze_to_Silver**:  
  - Uses **Databricks AI Query** to extract job requirements, responsibilities, and qualifications.  
  - Stores enriched JD data in Silver tables.

### 4. Indexing
- **Profile_Index_Sync**:  
  - Creates/updates a **vector search index** for all profiles using embeddings from AI-extracted fields.  
  - Index syncs on every pipeline run to ensure latest profiles are searchable.

### 5. Matching & Gold Layer
- **Profile_JD_Matching_Summary**:  
  - Performs semantic search of the job description against the profile vector index.  
  - Calculates a **fitness score** for each candidate based on skills overlap, experience match, and other criteria.  
  - Outputs a Gold table of top-matching candidates.

---

## Technologies Used
- **Databricks Workflows** – Orchestration of ETL and AI processes.  
- **Databricks Volumes** – Storage of ingested PDFs.  
- **Medallion Architecture** – Bronze/Silver/Gold data layers.  
- **Databricks AI Query** – PDF text extraction and insight generation.  
- **Vector Search Index** – Candidate embedding storage and semantic search.  
- **Delta Tables** – Persistent data storage at each stage.

---

## Folder Structure
```plaintext ehr-project-bundle
.
├── ehr-project
│	├── ehr-project-bundle
│	│	  ├── source/
│	│   	    ├── Profile_Ingestion.ipynb
│	│   	    ├── JD_Ingestion.ipynb
│	│   	    ├── Profile_Source_to_Bronze.ipynb
│	│   	    ├── JD_Source_to_Bronze.ipynb
│	│   	    ├── Profile_Bronze_to_Silver.ipynb
│	│   	    ├── JD_Bronze_to_Silver.ipynb
│	│   	    ├── Profile_Silver_Indexing.ipynb
│	│   	    └── Profile_JD_Matching_gl.ipynb
│	│	  ├── envsetup/
│	│   	    ├── Configs.ipynb
│	│   	    ├── Setup.ipynb
│	│   	    ├── ModelEndpoint.ipynb
│	│   	    ├── VectorSearch.ipynb
│	│   	    └── Cleanup.ipynb
│	│	  ├── resource/
│	│   	    ├── ats_data_processing.job.yml
│	│   	    ├── cleanup.job.yml
│	│   	    └── setup.job.yml
│	├── databricks workflow.png
│	├── databricks.yml
├── README.md
```

---

## Gold Layer Output

The **Gold Layer** stores the final ranked matches of job descriptions against candidate profiles in a **Delta table**.

**Schema:**

| Column           | Description |
|------------------|-------------|
| `jd_id`          | Unique identifier for the job description. |
| `jd_source`      | Source path to the original job description PDF. |
| `jd_extract`     | AI-extracted JSON containing job title, requirements, responsibilities, and qualifications. |
| `profile_id`     | Unique identifier for the candidate profile. |
| `profile_source` | Source path to the original resume PDF. |
| `profile_extract`| AI-extracted JSON containing candidate name, skills, education, and experience. |
| `summary`        | AI-generated summary of the match including fitness score and key matching criteria. |
| `generated_date` | Timestamp when the match record was generated. |

**Sample Output:**
```json
{
  "jd_id": 2,
  "jd_source": "dbfs:/Volumes/ehr_dev/ats_db/data_ingestion/jd/job_desc.pdf",
  "jd_extract": {
    "job_title": "Decision Scientist",
    "required_skills": ["Python", "SQL", "Machine Learning"],
    "experience": "3+ years"
  },
  "profile_id": 2,
  "profile_source": "dbfs:/Volumes/ehr_dev/ats_db/data_ingestion/profile/candidate_resume.pdf",
  "profile_extract": {
    "name": "Cynthia Dwayne",
    "skills": ["Python", "SQL", "Databricks", "Machine Learning"],
    "experience": "4 years"
  },
  "summary": {
    "fitness_score": 0.91,
    "matching_skills": ["Python", "SQL", "Machine Learning"],
    "notes": "Strong match based on skills and experience"
  },
  "generated_date": "2025-08-12T20:07:59.04247Z"
}
```

The **fitness score** and **matching details** in the summary enable recruiters to quickly shortlist top candidates for a given job description.

---

## Example Output Table

| Candidate Name | Fitness Score | Matching Skills                  | Years Experience | Education Match |
|----------------|--------------|-----------------------------------|------------------|-----------------|
| John Doe       | 0.92         | Python, SQL, Databricks, Spark    | 5                | Yes             |
| Jane Smith     | 0.87         | Machine Learning, NLP, MLOps      | 4                | Yes             |

---

## Setup & Deployment

### **Prerequisites**
- Databricks workspace with cluster permissions  
- Access to Databricks Volumes for storage  
- Databricks Vector Search enabled  
- Source cloud storage (e.g., AWS S3, Azure Blob, GCS) or github repo containing PDF resumes and job descriptions

### **Steps**

1. **Clone/Upload Source Code**
   - Upload all Python scripts in `source/` to your Databricks repo or workspace.

2. **Create Databricks Volumes**
   ```sql
   CREATE VOLUME ehr_dev.ats_db.data_ingestion;
   ```

3. **Configure Workflow**
   - In Databricks Workflows, create tasks corresponding to each Python script:
     1. Ingest_Profile_to_Src  
     2. Ingest_JD_to_Src  
     3. Profile_Source_to_Bronze  
     4. JD_Source_to_Bronze  
     5. Profile_Bronze_to_Silver  
     6. JD_Bronze_to_Silver  
     7. Profile_Index_Sync  
     8. Profile_JD_Matching_Summary  

4. **Set Job Parameters**
   - Source paths for resumes and job descriptions  
   - Vector search index name  
   - Output table location for Gold Layer results

5. **Run the Workflow**
   - Trigger manually or schedule periodic runs.

6. **Access Results**
   - Query the Gold table:
     ```sql
     SELECT * FROM ehr_dev.ats_db.gold_profile_jd_matches ORDER BY fitness_score DESC;
     ```

---

## Use Cases
- Automated recruitment candidate shortlisting.  
- Resume database semantic search.  
- Skill-gap analysis between job postings and available talent.

---

## Possible Future Improvements
- Multi-language resume parsing.  
- Weighted scoring per matching criterion.  
- Real-time resume ingestion and indexing.  
- Integration with applicant tracking systems (ATS).
