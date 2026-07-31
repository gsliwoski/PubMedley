# PubMedley

Single call pipeline to:
1. Query PubMed
2. Filter for relevant hits if GCS Gemini or OpenAI API available
3. Download full text PDF
4. Repeatedly search with improved queries until desired number of articles are downloaded

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
* `--max-tries 100` - Maximum number of qualifying-length download outcomes.
  * For example, say you want 20 files, to prevent it from running forever if it keeps failing to download, set this for how many it will try before giving up.
  * A PDF verified below `--min-length` is written to the failure list with its page-count reason, but does not consume a try.
* `--max-rounds 10` - The absolute maximum number of search-filter-download rounds that will be performed before it quits.
  * For example, say every round LLM filtered out all but a couple articles, if after max-rounds you still haven't hit the max tries or max articles it will give up
    * When it gives up, it will print the current query so you can relaunch and specify the exact query to pick up where you left off.
* `--continuation-state PubMedley_continuation.json` - atomically checkpoint the exact current query, explanation, and completed PMIDs after every round
* `--resume-from <state.json>` - continue without re-screening or downloading completed PMIDs

### Output
* Pipeline writes the following files that can be configured
  * `--failure-list failed_to_download.ls` - list of which articles failed to download, one article per line
  * `--metadata article_metadata.jsonl` - information about article metadata as list of JSON for articles
  * `--success-list retrieved_articles.ls` - List of articles that were successfully downloaded, one article per line
  * `--llm-report llm_screening.json` - info about the LLM call results

### Query
* Provide information that you want to query in several ways
  * With no custom query, the built-in intelligence search requires strong evidence such as `human intelligence`, `general intelligence`, `general cognitive ability`, `g factor`, `Cattell-Horn-Carroll`, or the exact non-exploding `Intelligence[MeSH:noexp]` heading
    * It does not use naked `intelligence`, generic `model/framework` matches, or `Humans[MeSH]`
  * Most basic is just to give a query phrase with `--query "phrase"`
  * Advanced queries can be specified in YAML file `--query-yaml example_query.yaml`
    * See 'example_query.yaml' for an example and all information about how to set up an advanced query
  * You can specify exclusion terms outside of YAML with comma separated string `--exclude "TERM1, TERM2, etc"`
* You can also supply filters
  * `--min-length 30` - Minimum number of pages the articles must have to be counted 
    * Note: it's at 30 because I was initially using it to download good review articles
  * `--max-age 10` - Maximum age of article in years

### LLM Filter
* `--explanation <free text>` - plain-language description of the corpus you want. The LLM receives it together with the exact query and uses it for both relevance decisions and next-round query improvement.
  * With the built-in query, the default is: "Theories of human intelligence, including what is required for intelligence, how intelligence is defined, its underlying components, what intelligence produces, and how intelligence works."
  * A YAML file may instead provide a top-level `explanation:` value. The CLI flag overrides YAML.
  * If neither is supplied for a custom query, PubMedley derives an objective from the compiled query.
* `--prompt-filter <text or @filename>` - Additional approve/reject instructions inserted into the LLM prompt
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
4. [ONLY WITH LLM CREDENTIALS] Send candidate metadata, the exact active PubMed query, the research explanation, and screening instructions to the LLM
  * The LLM returns a relevance decision for every PMID, one complete improved PubMed query, and a short reason for the rewrite
5. Validate and preflight the proposed query
  * PubMedley rejects malformed/oversized rewrites and rewrites that drop free-full-text, date, review/publication-type, PMC-only, or explicit title-exclusion constraints
  * A valid rewrite is preflighted against PubMed and becomes the exact query for the next round and continuation checkpoint
  * If the LLM fails, all candidates remain relevant and the query remains unchanged
6. Check for existing PDF (If article has already been downloaded skip)
7. For PMC articles, try the current anonymous PMC AWS Open Data PDF route
8. Try legacy PMC OA, PMC page/canonical, and publisher HTTP routes
9. If unable to download PDF, use headless Chromium browser
  * Will visit up to 12 pages in an attempt to download the PDF
10. Repeat search-download steps until a configured limit is reached or the expanded/configured PubMed search space is genuinely exhausted

## PDF retrieval
* PMC is removing legacy FTP/OA PDF structure in August 2026
  * New method is AWS Open Data service for programmatic PDF retrieval
* Code currently supports both and uses headless Chromium as a final backup
