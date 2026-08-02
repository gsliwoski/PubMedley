# PubMedley

Single call pipeline to:
1. Query PubMed
2. Filter for relevant hits if GCS Gemini or OpenAI API available
3. Download full text PDF
4. Repeatedly search with improved queries until desired number of articles are downloaded

## Purpose
* Automatically and iteratively search pubmed for articles of interest, filter results, and download full source PDFs

## Important
* Complete pipeline requires API credentials for OpenAI or service account credentials for GCS Gemini
* To run the limited pipeline without LLM support you must set `--no-llm`

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
* `--no-llm` - explicitly disable LLM screening and query rewriting
  * Without this flag, missing credentials or failed/incomplete LLM screening is fatal; PubMedley will not silently approve everything.
* `--pmc-only` - restrict results to articles available in PubMed Central
* `--retries 3` - HTTP/PDF retries after the initial attempt
* `--llm-retries 3` - LLM retries after the initial call, with 1, 2, and 4 second waits by default
* `--timeout 60` - seconds
* `--ncbi-api-key <NCBI_API_KEY>` - not required but allows for faster throughput
* `--max-articles 20` - maximum number of files you want downloaded at the end
* `--max-tries 100` - Maximum number of qualifying-length download outcomes.
  * For example, say you want 20 files, to prevent it from running forever if it keeps failing to download, set this for how many it will try before giving up.
  * A PDF verified below `--min-length` is written to the failure list with its page-count reason, but does not consume a try.
* `--max-rounds 10` - The absolute maximum number of search/filter/download or exhausted-query-refinement rounds that will be performed before it quits.
  * For example, say every round LLM filtered out all but a couple articles, if after max-rounds you still haven't hit the max tries or max articles it will give up
    * When it gives up, it will print the current query so you can relaunch and specify the exact query to pick up where you left off.
* `--continuation-state PubMedley_continuation.json` - atomically checkpoint the exact current query, explanation, and completed PMIDs after every round
* `--resume-from <state.json>` - continue without re-screening or downloading completed PMIDs

### Output
* Pipeline writes the following files that can be configured
  * `--failure-list failed_to_download.ls` - list of which articles failed to download, one article per line
  * `--metadata article_metadata.jsonl` - information about article metadata as list of JSON for articles; browser failures include each attempted page, HTTP status, PDF-looking link/request, matching control, and rejection reason under `retrieval.browser.attempts`
  * `--success-list retrieved_articles.ls` - List of articles that were successfully downloaded, one article per line
  * `--llm-report llm_screening.json` - info about the LLM call results
* Interactive terminal output uses color for quick scanning:
  * query text is cyan, successful downloads are green, browser failure/retry markers are maroon, and raw browser diagnostics are gray
  * every completed round ends with a purple per-round summary of PubMed hits, LLM and length rejections, download failures/successes, remaining requested articles, and remaining rounds
  * ANSI colors are automatically omitted when output is redirected to a file

### Query
* Provide information that you want to query in several ways
  * With no custom query, the built-in intelligence search starts with strong phrases such as `human intelligence`, `general intelligence`, `general cognitive ability`, `theory of intelligence`, `g factor`, and `Cattell-Horn-Carroll`
    * It uses the broader non-exploding `Intelligence[MeSH:noexp]` heading only after the focused search is exhausted
    * It does not use naked `intelligence`, generic `model/framework` matches, or `Humans[MeSH]`
    * Multiword Title/Abstract phrases use PubMed proximity-zero syntax (for example, `"human intelligence"[Title/Abstract:~0]`) so PubMed cannot silently split an unrecognized quoted phrase into unrelated individual words
    * The ambiguous `g factor` phrase must also have intelligence, cognitive, or psychometric context; generic `synthesis[Title]` is not treated as proof that an article is a review
  * Most basic is just to give a query phrase with `--query "phrase"`
  * Advanced queries can be specified in YAML file `--query-yaml example_query.yaml`
    * See 'example_query.yaml' for an example and all information about how to set up an advanced query
  * You can specify exclusion terms outside of YAML with comma separated string `--exclude "TERM1, TERM2, etc"`
* You can also supply filters
  * `--min-length 20` - Minimum number of pages the articles must have to be counted
    * Note: it's at 20 because I was initially using it to download good review articles
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
* Credentials are required unless `--no-llm` is explicitly supplied
  * For Gemini:
    * Expects a JSON file service account that contains project information and appropriate roles
  * For OpenAI:
    * Expects OPENAI_API_KEY env variable and `--openai-model <model_name>` specified
* If credentials are missing, PubMedley stops before searching and tells you to provide credentials or pass `--no-llm`
* The default Gemini credential lookup supports both the current `sources/pubmedley/` directory and the former parent `sources/` location

## Pipeline
1. Construct the search and LLM screening plan
   * Use the YAML format for advanced querying otherwise use the `--query` argument
     * Script automatically adds 'free full text[sb]' when using `--query`
2. Print the exact effective query, then search PubMed by relevance using PubMed ESearch
3. Preliminary page count filter
4. [ONLY WITH LLM CREDENTIALS] Send candidate metadata, the exact active PubMed query, the research explanation, screening instructions, prior decisions, and live task progress to the LLM
  * The LLM returns a relevance decision for every PMID, one complete improved PubMed query, and a short reason for the rewrite
  * Task progress includes requested/completed downloads, tries used/remaining, short PDFs, failures, and search rounds remaining so the LLM can balance recall against precision
5. Validate and preflight the proposed query
  * PubMedley rejects malformed/oversized rewrites and rewrites that drop free-full-text, date, review/publication-type, PMC-only, or explicit title-exclusion constraints
  * A valid rewrite is preflighted against PubMed and is accepted only if it returns at least one PMID that has not already been seen; then it becomes the exact query for the next round and continuation checkpoint
  * If a query is exhausted, a query-refinement-only LLM round asks for broader terminology instead of stopping or re-screening the same records
  * An API error, invalid JSON, or two or more omitted PMIDs is retried according to `--llm-retries`; one omitted PMID is ignored/rejected without repeating the whole LLM call
  * After retry exhaustion PubMedley stops before downloading that batch and explicitly suggests `--no-llm`
  * With explicit `--no-llm`, hard title exclusions still apply locally but semantic relevance filtering and adaptive query rewrites are disabled
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
