# PubMedley

Single call pipeline to:
1. Query PubMed
2. Filter for relevant hits if GCS Gemini or OpenAI API available
3. Download full text PDF

## Purpose
* Automatically search pubmed for articles of interest, filter results, and download full source PDFs

## Setup
* Install requirements.txt
* Install headless browser with
  * `python -m playwright install chromium`

## Parameters
### Basic information and config
* `--email` - specify an email that will be used with NCBI queries. No need to pre-register or anything just helps.
* `--gemini-auth` - location of service account JSON file, default is 'gemini_service_account.json'
* `--gemini-model` - only accepts gemini models 3.1 and later, default 'gemini-3.6-flash'
* `--openai-model` - Overrides Gemini setting to use OpenAI API instead
* `--pmc-only` - restrict results to articles available in PubMed Central
* `--retries 3` - number of times will retry downloading PDF
* `--timeout 60` - seconds
* `--ncbi-api-key <NCBI_API_KEY>` - not required but allows for faster throughput
* `--max-articles 20` - maximum number of files you want downloaded at the end
* `--max-tries 100` - Total number of articles that fit inclusion criteria and attempt to download.
  * For example, say you want 20 files, to prevent it from running forever if it keeps failing to download, set this for how many it will try before giving up.
* `--max-rounds 10` - The absolute maximum number of search-filter-download rounds that will be performed before it quits.
  * For example, say every round LLM filtered out all but a couple articles, if after max-rounds you still haven't hit the max tries or max articles it will give up
    * When it gives up, it will print the current query so you can relaunch and specify the exact query to pick up where you left off.
* `--continuation-state PubMedley_continuation.json` - atomically checkpoint query and completed PMIDs after every round
* `--resume-from <state.json>` - continue without re-screening or downloading completed PMIDs

### Output
* Pipeline writes the following files that can be configured
  * `--failure-list failed_to_download.s` - list of which articles failed to download, one article per line
  * `--metadata article_metadata.jsonl` - information about article metadata as list of JSON for articles
  * `--success-list retrieved_articles.ls` - List of articles that were successfully downloaded, one article per line
  * `--llm-report llm_screening.json` - info about the LLM call results

### Query
* Provide information that you want to query in several ways
  * Most basic is just to give a query phrase with `--query "phrase"`
  * Advanced queries can be specified in YAML file `--query-yaml example_query.yaml`
    * See 'example_query.yaml' for an example and all information about how to set up an advanced query
  * You can specify exclusion terms outside of YAML with comma separated string `--exclude "TERM1, TERM2, etc"`
* You can also supply filters
  * `--min-length 30` - Minimum number of pages the articles must have to be counted 
    * Note: it's at 30 because I was initially using it to download good review articles
  * `--max-age 10` - Maximum age of article in years

### LLM Filter
* `--prompt-filter <text or filename>` - Instructions for what to filter that will get inserted into LLM prompt
  * You may also provide this information within the query YAML under the 'screening' block
  * If nothing is provided in either location, it will try to insert phrases based on the query

## Credentials
* Credentials are only necessary for LLM relevance filter
  * For Gemini:
    * Expects a JSON file service account that contains project information and appropriate roles
  * For OpenAI:
    * Expects OPENAI_API_KEY env variable and `--openai-model <model_name>` specified
* If no credentials are supplied, LLM-based filtering will not be performed and all results will be assumed relevant

## Pipeline
1. Construct the search and LLM screening plan
   * Use the YAML format for advanced querying otherwise use the `--query` argument
     * Script automatically adds 'free full text[sb]' when using `--query`
2. Search pubmed by relevance using PubMed ESearch
3. Preliminary page count filter
4. [ONLY WITH LLM CREDENTIALS] Send candidate titles to LLM and ask to filter for unrelated hits
  * Search will often result in hits you don't want, the purpose of this step is to filter all those out
  * LLM will also return a list of exclusion phrases to use if you're get a lot of unrelated hits
5. Automatically refine noisy query
  * When LLM rejects at least 75% of candidates, exclusion terms are automatically appended to query
6. Check for existing PDF (If article has already been downloaded skip)
7. For PMC articles, try the current anonymous PMC AWS Open Data PDF route
8. Try legacy PMC OA, PMC page/canonical, and publisher HTTP routes
9. If unable to download PDF, use headless Chromium browser
  * Will visit up to 12 pages in an attempt to download the PDF
10. Repeat search-download steps until max is reached

## PDF retrieval
* PMC is removing legacy FTP/OA PDF structure in August 2026
  * New method is AWS Open Data service for programmatic PDF retrieval
* Code currently supports both and uses headless Chromium as a final backup