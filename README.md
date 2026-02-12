
## Workflow Architecture
The workflow follows a 12step logical sequence to ensure data integrity and bypass basic anti-bot measures.

### 1. Basic configuration
Node 1: You need to use your headers of your browser, for example User-agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/121.0.0.0 Safari/537.36 and Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8  , you can get this running this command:
curl 'https://www.forsalebyowner.com/search/list/Virginia' \
  -H 'User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/144.0.0.0 Safari/537.36' \
  -H 'Accept: text/html' \
  -H 'Accept-Language: en-US,en;q=0.9'

Node 2: Here the JavaScript processor extracts the "form_key" and session cookies. It then generates a sequence for the first 10 pages of results to drive the loop, thanks to this is possible to scrap in all the web.

### 2. Intelligent Pagination Loop
Node 3: A batch controller manages the execution flow, ensuring the server is not overwhelmed and the session remains active.
Node 4: The workflow executes an asynchronous POST request. It injects the previously captured cookies and security tokens to mimic a legitimate browser-side AJAX call.

### 3. Multi-Layer Extraction, and Data normalization
Node 5: The system parses the raw response to isolate individual property listing blocks.
Node 6: A secondary extraction layer captures high-level data: property URL, price, and raw address strings.
Node 7: The JavaScript node performs advanced Regex cleaning. It splits raw address strings into structured fields: Street, City, State, and Zip Code.
Node 8: The workflow performs a "Deep Crawl" by visiting each unique property URL.
Node 9:The system extracts the data for contact the seller it gets Owner Names, Phones, and Emails,property descriptions and photo links.

### 4. Memory Persistence & save to csv
Node 10: Since standard automation loops can lose data between iterations, this node uses a global accumulator to persist all the pages of data in the workflow static memory.
Node 11: To ensure long-term stability and avoid IP flagging, a 6-minute wait period is enforced between page batches.
Node 12: Upon completion of all cycles, the workflow retrieves the accumulated data, converts it into a standardized CSV file, and prepares it for final storage or upload.

