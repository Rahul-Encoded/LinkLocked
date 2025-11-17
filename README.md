# LinkLocked
# 🤖 LinkedIn Job Scraper and AI Matcher (n8n Workflow)

This repository contains an n8n workflow designed to automate the process of finding, scraping, and matching job opportunities from LinkedIn against a candidate's resume using a Large Language Model (LLM) agent (Google Gemini).

The workflow automatically calculates a **matching score** and generates a **customized cover letter** for jobs that meet a minimum score threshold. Just download the .json file and import it into your n8n workflow.

## ✨ Workflow Overview

The core of this automation is a multi-step n8n workflow that performs two main functions:

1.  **Job Search & Scraping (Top Flow):** Searches LinkedIn based on filter criteria, scrapes job links, and stores them in a Google Sheet ("Scraped").
2.  **Job Matching & Application Prep (Bottom Flow):** Iterates over the scraped jobs, extracts the job description, compares it to a candidate's resume using an AI Agent, and saves high-scoring matches (Score >= 60) and their generated cover letters to another Google Sheet ("Jobs for me").

## 🌊 Detailed Workflow Flow

The automation runs in two main parallel pathways after initialization:

### Pathway 1: LinkedIn Scraping and Link Management

This path uses your spreadsheet filters to find new job links.

1.  **`When clicking ‘Execute workflow’`** $\rightarrow$ **`Get row(s) in sheet`** (Filters)
    * Triggers the workflow and fetches search criteria (Keywords, Location, Experience) from the **`Filters`** sheet.
2.  **`Code in JavaScript`**
    * Constructs the specific LinkedIn job search URL using the criteria from the `Filters` sheet.
      
       ```js

      let url = "https://www.linkedin.com/jobs/search/?f_TPR=r86400"
      
      const keyword = $json.Keyword
      const location = $json.Location
      const experienceLevel = $json["Experience level"]
      const remote = $json.Remote
      const jobType = $json["Job type"]
      const easyApply = $json["Easy Apply"]
      
      if (keyword != ""){
        url += `&keywords=${keyword}`
      }
      if (location != ""){
        url += `&location=${location}`
      }
      
      // Experience
      if (experienceLevel.length){
        // Corelate experience levels to their respective linkedIn codes
        const codesExp = experienceLevel.split(",").map((exp) => {
           switch(exp.trim()){
             case "Internship": return "1";
             case "Entry Level": return "2";
             case "Associate": return "3";
             case "Mid-Senior Level": return "4";
             case "Director": return "5";
             case "Executive": return "6";
             default: return "";  
           }
        }).filter(Boolean);  // for the default return
        url += `&f_E=${codesExp.join(",")}`;
      }
      
      //Work Type
      
      if (remote.length){
        // Corelate Remote values to their respective linkedIn Codes
        const codesRem = remote.split(",").map((rem) => {
          switch(rem.trim()){
            case "On-Site": return "";
            case "Remote": return "";
            case "Hybrid": return "";
            default: return "";
          }
        }).filter(Boolean);
        url += `&f_WT=${codesRem.join(",")}`;
      }
      
      if (jobType.length){
        // Corelate Job Types to their respective linkedIn Codes
         // Full-time -> F, Part-time -> P, Contract -> C, etc.
        const codeJob = jobType.split(",").map((job) => {
          job.trim().charAt(0).toUpperCase()
        });
        url += `&f_JT=${codeJob}`;
      }
      
      if (easyApply != ""){
        url += `&f_EA = ${true}`
      }
      else{
        url += `&f_EA = ${false}`
      }
      
      return {json:{...$json, url:url}}
      ```
       

3.  **`Fetch Jobs`** $\rightarrow$ **`HTML`**
    * Performs an HTTP Request using the generated URL and scrapes all job links (`a.base-card__full-link`) from the resulting HTML.
4.  **`Split Out`** $\rightarrow$ **`Loop Over Items`**
    * Splits the array of scraped links into individual items for processing.
5.  **`Wait`** $\rightarrow$ **`Get row(s) in sheet1`** (Scraped)
    * Fetches the list of links already processed from the **`Scraped`** sheet to check for duplicates.
6.  **`Code in JavaScript1`** $\rightarrow$ **`If1`**
    * Checks if the current job link has already been scraped (by comparing it to the links from the `Scraped` sheet). If it's a duplicate, the flow stops for that link.
      
      ```js
      const currentJobLink = $('Wait').first().json.links.split('?')[0];

      console.log("Current Job Link (Normalized)", currentJobLink);  // Please check browser console.logs for this log
      
      const scrapedLinksArray = $input.all().map(item => item.json["Scraped Links"]);
      
      console.log("Scraped Job links", scrapedLinksArray); 
      
      let bool = 0;
      
      if(scrapedLinksArray.includes(currentJobLink)){
        bool = 1
      }
      
      return [{json:{link: currentJobLink, bool: bool}}];
      ```
      
7.  **`Append or update row in sheet1`** $\rightarrow$ **`HTTP Request`** $\rightarrow$ **`HTML1`**
    * If the link is new, it is added to the **`Scraped`** sheet and the job page is scraped for detailed information (Title, Company, Location, Description).
8.  **`Edit Fields`**
    * Cleans up the raw scraped HTML description (removes extra whitespace).

### Pathway 2: AI Matching and Cover Letter Generation

This path takes the job details, matches them against your resume, and stores the results.

1.  **`Download file`** $\rightarrow$ **`Extract from File`**
    * Downloads your PDF resume from Google Drive and extracts the text content.
2.  **`AI Agent`**
    * Uses the **Google Gemini Chat Model** to compare the extracted resume text against the job description.
    * It strictly enforces the **1-year experience constraint** (score $\rightarrow$ 0 if violated).
    * It calculates a final score (out of 100) and generates a custom cover letter.
      
    ```md
    Prompt:
    You are a highly analytical assistant that specializes in matching candidates to job opportunities.
    
    Your task is to:
    1.  **Analyze the Candidate's Experience Level:** The candidate's total professional experience is **1 year**.
    2.  **Apply the Hard Constraint:** If the provided `job_description` explicitly or implicitly requires **more than 1 year of professional experience**, the resulting `score` **must be 0**. This is a non-negotiable hard stop.
    3.  **Calculate the Job Matching Score (Out of 100):**
        * **Experience Alignment (Most Important):** This should account for at least 50% of the score. If the job requires 1 year or less, this factor is high. If it requires more, the score is 0 (due to the constraint in step 2).
        * **Skill and Keyword Alignment (Secondary):** Assess how well the skills, technologies, and responsibilities mentioned in the `my_resume` align with those in the `job_description`.
    4.  **Generate a Cover Letter:** Create a compelling cover letter based on the candidate's `my_resume` and the `job_description`.
        * The cover letter must contain at least **two paragraphs**.
        * **Crucially, exclude** the candidate's name, address, date, hiring manager's name, company address, and the signature/closing lines (e.g., "Sincerely,"). Focus only on the body of the letter.
    5.  **Format the Output:** The entire response must be a single, valid JSON object that can be parsed without errors, following the structure: `{"score": X, "coverLetter": "Y"}`. Ensure all quotation marks within the cover letter are properly escaped with a backslash (e.g., use `\"` instead of `"`).
    
    ---
    **Job Data for Analysis:**
    
    job_description: {{ $('Edit Fields').item.json.Description }}
    my_resume: {{ $json.text }}
    ```
    
3.  **`Edit Fields1`** $\rightarrow$ **`If`**
    * Parses the JSON output from the AI Agent to extract the `score` and `coverLetter`.
    * The `If` node checks if the calculated `score` is **greater than or equal to 60** (`>= 60`).
4.  **`Append or update row in sheet`**
    * If the score is 60 or higher, the job details, final score, and cover letter are written to the **`Jobs for me`** output sheet.

---

## 🛠️ Prerequisites

Before importing and running the workflow, ensure you have the following:

* **n8n Instance:** A running instance of n8n (self-hosted or cloud).
* **Google Account:** For access to Google Sheets and Google Drive.
* **Google Gemini API Key:** For the `Google Gemini Chat Model` nodes.
* **Google Sheets:** A spreadsheet named **`Job Hunt`** with three required sheets:
    * **`Filters`**: To input your search criteria (Keyword, Location, Experience level, etc.).
    * **`Jobs for me`**: The output sheet for high-scoring jobs.
    * **`Scraped`**: The intermediary sheet for all scraped job links.
* **Google Drive:** Your resume file (e.g., `RahulResume (6).pdf`) must be accessible to the Google Drive node.

## ⚙️ Setup and Configuration

### 1. Credentials Setup

You need to set up credentials for the following services in your n8n instance:

* **Google Sheets:** Create a **Google Sheets OAuth2 API** credential.
* **Google Drive:** Create a **Google Drive OAuth2 API** credential.
* **Google Gemini:** Create a **Google Gemini(PaLM) API** credential using your API key.

### 2. Import the Workflow

1.  In your n8n instance, click **`Workflows`** and then **`Import from JSON`**.
2.  Upload the `Job Automation.json` file from this repository.

### 3. Customize Resume & Google Sheet IDs

The workflow is currently configured for specific file and sheet IDs. **You must update these in the node parameters:**

* **Google Drive Node (File ID):**
    * Update the **File ID** in the **`Download file`** and **`Download file1`** nodes to point to **your PDF resume file** in Google Drive.
* **Google Sheets Node (Document ID & Sheet Name):**
    * Ensure the **Document ID** in all Google Sheets nodes is set to your **`Job Hunt`** spreadsheet.
    * If you change the names of your Sheets, you may need to update the **Sheet Name (GID)** parameter in the respective nodes to match the GID of your sheets.

### 4. Adjust AI Agent Prompt

The `AI Agent` and `AI Agent1` nodes are crucial. They currently enforce a **hard constraint** of **1 year of professional experience**.

* If your experience is different, modify the following lines in the `promptType: define` field of the **AI Agent** and **AI Agent1** nodes:
    * `The candidate's total professional experience is **1 year**.`
    * `If the provided job_description explicitly or implicitly requires **more than 1 year of professional experience**, the resulting score **must be 0**.`

## ▶️ Running the Workflow

1.  **Populate Filters:** Ensure the `Filters` sheet in your `Job Hunt` spreadsheet has the keywords and locations you want to search.
2.  **Execute:** Run the workflow in n8n.
3.  **Check Results:** After the workflow completes, check the `Jobs for me` sheet. It will contain job details, the AI-generated match score, and a personalized cover letter for all jobs scoring 60 or higher.
