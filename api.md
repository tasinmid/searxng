# Custom SearXNG API Endpoint Documentation

A custom, streamlined API endpoint has been implemented in SearXNG to allow programmatic retrieval of search result source URLs.

---

## Configuration (`.env`)

A `.env` file has been added in the project root containing the environment variables.

* **For Docker (default)**:
  ```env
  SEARXNG_PORT=8080
  SEARXNG_HOST=127.0.0.1
  SEARXNG_URL=http://127.0.0.1:8080
  SEARXNG_API_URL=http://127.0.0.1:8080/api
  ```
* **For Local Python (`make run`)**:
  ```env
  SEARXNG_URL=http://127.0.0.1:8888
  SEARXNG_API_URL=http://127.0.0.1:8888/api
  ```

---

## Endpoint Specification

### `GET /api`

Retrieves a raw list of search result source URLs from the **general** category.

### Authentication

To secure this endpoint, requests must include the API key. You can authenticate using any of the following methods:

1. **HTTP Headers**:
   - `X-API-KEY: <your_api_key>`
   - `Authorization: Bearer <your_api_key>`
2. **Query Parameters**:
   - `key=<your_api_key>`
   - `api_key=<your_api_key>`

The default value is `ultrasecretapikey` (configured in `.env` via `SEARXNG_API_KEY`).

### Query Parameters

| Parameter | Type | Required | Description |
| :--- | :--- | :--- | :--- |
| `q` | `string` | **Yes** | The search query string. |
| `rqs` | `integer` | No | Request Quantity Size. Limits the returned results list. (E.g., `rqs=5` returns the first 5 URLs; `rqs=10` returns the first 10 URLs). Must be a positive integer. |
| `key` | `string` | No | API key authentication parameter. |
| `api_key` | `string` | No | API key authentication parameter. |

---

## Output Format

The endpoint returns a **pure JSON array of strings** containing only the source URLs. No wrappers or extra metadata are included.

### Example Response (`rqs=5`)

```json
[
  "https://www.apple.com/",
  "https://en.wikipedia.org/wiki/Apple",
  "https://apple.com/iphone/",
  "https://appleinsider.com/",
  "https://www.reddit.com/r/apple/"
]
```

---

## Example Usage

### 1. Requesting 5 URLs (using query parameter key)
```bash
curl -s "http://127.0.0.1:8080/api?q=apple&rqs=5&key=ultrasecretapikey"
```

### 2. Requesting 10 URLs (using HTTP header key)
```bash
curl -s -H "X-API-KEY: ultrasecretapikey" "http://127.0.0.1:8080/api?q=apple&rqs=10"
```

### 3. Requesting All Available URLs (using Bearer token)
```bash
curl -s -H "Authorization: Bearer ultrasecretapikey" "http://127.0.0.1:8080/api?q=apple"
```

---

## Programmatic Examples

### 1. Bash / Curl (with URL Encoding & Header Key)
To automatically URL-encode space characters and special symbols in a query (e.g. `"how is the ceo of tesla?"`):
```bash
curl -G --data-urlencode "q=how is the ceo of tesla?" \
     -H "X-API-KEY: ultrasecretapikey" \
     "http://127.0.0.1:8080/api" --data "rqs=5"
```

### 2. TypeScript (using fetch & header authentication)
```typescript
async function fetchSourceUrls(query: string, limit: number, apiKey: string): Promise<string[]> {
  const url = new URL('http://127.0.0.1:8080/api');
  url.searchParams.append('q', query);
  url.searchParams.append('rqs', limit.toString());

  const response = await fetch(url.toString(), {
    headers: {
      'X-API-KEY': apiKey
    }
  });
  if (!response.ok) {
    throw new Error(`HTTP error! Status: ${response.status}`);
  }
  const urls: string[] = await response.json();
  return urls;
}

// Example usage:
fetchSourceUrls('how is the ceo of tesla?', 5, 'ultrasecretapikey')
  .then(console.log)
  .catch(console.error);
```

### 3. TypeScript (using Axios & header authentication)
```typescript
import axios from 'axios';

async function fetchSourceUrlsAxios(query: string, limit: number, apiKey: string): Promise<string[]> {
  const response = await axios.get<string[]>('http://127.0.0.1:8080/api', {
    params: {
      q: query,
      rqs: limit
    },
    headers: {
      'X-API-KEY': apiKey
    }
  });
  return response.data;
}
```

### 4. Python (using requests & header authentication)
```python
import requests

params = {
    'q': 'how is the ceo of tesla?',
    'rqs': 5
}
headers = {
    'X-API-KEY': 'ultrasecretapikey'
}
response = requests.get('http://127.0.0.1:8080/api', params=params, headers=headers)
urls = response.json()
print(urls)
```

---

## Technical Implementation Details

* **Category Restriction**: The endpoint automatically restricts queries to the `'general'` category (text-based web search) to keep responses focused and fast.
* **Backend Processing**: It uses the same engine routing and query execution path (`searx.search.SearchWithPlugins`) as standard HTML searches but filters out all UI elements, layout templates, infoboxes, suggestions, and engine timings.
* **Raw Extraction**: The server extracts the `url` attribute from every matched search result dictionary and truncates the resulting array using Python's slice syntax based on the `rqs` parameter limit before sending it to the client.
