# Vulnerabilities in screenshot-to-code Project

## 1. Path Traversal in Evals Routes

### Vulnerability Name
Path Traversal in Evals API Endpoints

### Description
The application's evaluation routes in `routes/evals.py` accept arbitrary folder paths from users and use them directly to access files on the server's filesystem without proper path sanitization. An attacker can exploit this by supplying paths with directory traversal sequences (e.g., "../") to access sensitive files outside the intended directory structure.

To trigger this vulnerability:
1. Send a GET request to the `/evals` endpoint with a `folder` parameter containing path traversal sequences
2. The application will read files from the specified directory without proper validation
3. This allows reading arbitrary files from the filesystem that the application has access to

### Impact
An attacker could access sensitive configuration files, credentials, or source code of the application. They might be able to read API keys (OpenAI, Anthropic, etc.) from environment files, access user data, or obtain other sensitive information stored on the server filesystem.

### Vulnerability Rank
High

### Currently Implemented Mitigations
The code checks if the folder exists but does not restrict the path to safe directories:
```python
folder_path = Path(folder)
if not folder_path.exists():
    raise HTTPException(status_code=404, detail=f"Folder not found: {folder}")
```

### Missing Mitigations
1. Path sanitization to remove traversal sequences
2. Path normalization to resolve paths before use
3. Restricting file access to specific allowed directories using a whitelist approach
4. Using a safe API for file access that prevents directory traversal

### Preconditions
- The attacker must have access to the API endpoints
- The application must be running with file system access permissions

### Source Code Analysis
In `routes/evals.py`, several endpoints take a folder path directly from user input:

```python
@router.get("/evals", response_model=list[Eval])
async def get_evals(folder: str):
    if not folder:
        raise HTTPException(status_code=400, detail="Folder path is required")

    folder_path = Path(folder)
    if not folder_path.exists():
        raise HTTPException(status_code=404, detail=f"Folder not found: {folder}")
```

The application uses this path directly without sanitization:
```python
files = {
    f: os.path.join(folder, f)
    for f in os.listdir(folder)
    if f.endswith(".html")
}
```

Similar vulnerabilities exist in other routes like `/pairwise-evals` and `/best-of-n-evals` where folder paths are accepted as parameters without proper validation.

### Security Test Case
1. Start the application and ensure it's accessible
2. Send a GET request to the `/evals` endpoint with a path traversal sequence:
   ```
   GET /evals?folder=../../../etc
   ```
3. Observe if the application returns directory contents outside of the intended directory
4. Try accessing a known configuration file like `/evals?folder=../../../etc/passwd` (on Linux) or `/evals?folder=../../../Windows/win.ini` (on Windows)
5. Verify that the contents of the file are returned in the response

## 2. Server-Side Request Forgery (SSRF) in Screenshot API

### Vulnerability Name
Server-Side Request Forgery (SSRF) in Screenshot API

### Description
The application's screenshot functionality in `routes/screenshot.py` accepts arbitrary URLs from users and makes HTTP requests to those URLs without adequate validation. This allows an attacker to make the server initiate connections to arbitrary hosts, including internal network resources that might not be directly accessible to the attacker.

To trigger this vulnerability:
1. Send a POST request to the `/api/screenshot` endpoint with a URL pointing to an internal network service
2. The server will attempt to capture a screenshot by connecting to that URL
3. This can be used to scan internal networks, access internal services, or exfiltrate data

### Impact
An attacker could:
- Scan and enumerate internal network services
- Access and interact with internal services that are not exposed to the internet
- Bypass network security controls like firewalls
- Potentially exploit vulnerabilities in internal services
- Exfiltrate data from internal systems

### Vulnerability Rank
High

### Currently Implemented Mitigations
The code has a TODO comment acknowledging the need for error handling but doesn't implement any URL validation:
```python
# TODO: Add error handling
image_bytes = await capture_screenshot(url, api_key=api_key)
```

### Missing Mitigations
1. URL validation to ensure only allowed domains can be accessed
2. Blocking requests to private IP ranges and local addresses
3. Implementing a URL allowlist approach
4. Rate limiting to prevent abuse
5. Network-level controls to restrict outbound connections

### Preconditions
- The attacker must have access to the API endpoint
- The server must be able to make outbound network connections

### Source Code Analysis
In `routes/screenshot.py`, the application accepts a URL from the user without validation:

```python
@router.post("/api/screenshot")
async def app_screenshot(request: ScreenshotRequest):
    # Extract the URL from the request body
    url = request.url
    api_key = request.apiKey

    # TODO: Add error handling
    image_bytes = await capture_screenshot(url, api_key=api_key)
```

The `capture_screenshot` function then uses this URL to make an HTTP request:

```python
async def capture_screenshot(
    target_url: str, api_key: str, device: str = "desktop"
) -> bytes:
    api_base_url = "https://api.screenshotone.com/take"

    params = {
        "access_key": api_key,
        "url": target_url,
        # ... other parameters
    }

    async with httpx.AsyncClient(timeout=60) as client:
        response = await client.get(api_base_url, params=params)
        # ...
```

This allows an attacker to provide URLs that point to internal network services, such as:
- `http://localhost:8080`
- `http://internal-service.local`
- `http://10.0.0.1:3000`
- `http://192.168.1.1`

### Security Test Case
1. Start the application and ensure it's accessible
2. Send a POST request to the `/api/screenshot` endpoint with a URL pointing to an internal service:
   ```json
   {
     "url": "http://localhost:8080",
     "apiKey": "your-api-key"
   }
   ```
3. If the server has any services running on port 8080, it will attempt to take a screenshot of that service
4. Try different internal IP addresses and ports to map the internal network
5. Check if the responses contain error messages or screenshots that reveal information about internal services

## 3. Unsanitized User Input in WebSocket Communication

### Vulnerability Name
Unsanitized User Input in WebSocket Communication

### Description
The application's WebSocket endpoint in `routes/generate_code.py` accepts and processes user input without proper validation or sanitization. This can lead to various injection attacks when this input is passed to sensitive operations like prompting AI models or generating code.

To trigger this vulnerability:
1. Connect to the WebSocket endpoint `/generate-code`
2. Send specially crafted JSON data with malicious input in fields like `image`, `generatedCodeConfig`, or other parameters
3. The server will process this input without adequate validation

### Impact
An attacker could:
- Inject malicious content that gets processed by AI models
- Potentially influence the AI to generate harmful code
- Cause application crashes or resource exhaustion through malformed inputs
- Manipulate the generated code in ways that could lead to vulnerabilities in websites created using the tool

### Vulnerability Rank
High

### Currently Implemented Mitigations
The code has some basic validation for specific fields:
```python
if generated_code_config not in get_args(Stack):
    await throw_error(f"Invalid generated code config: {generated_code_config}")
```

However, this validation is insufficient for all the complex input fields.

### Missing Mitigations
1. Comprehensive input validation for all user-supplied parameters
2. Content sanitization for image data and text inputs
3. Input length and format restrictions
4. Rate limiting to prevent abuse
5. Proper error handling to avoid revealing sensitive information

### Preconditions
- The attacker must have access to the WebSocket endpoint
- The application must be processing user inputs for AI model prompting

### Source Code Analysis
In `routes/generate_code.py`, the WebSocket endpoint accepts JSON data without thorough validation:

```python
@router.websocket("/generate-code")
async def stream_code(websocket: WebSocket):
    await websocket.accept()
    print("Incoming websocket connection...")

    params: dict[str, str] = await websocket.receive_json()
    print("Received params")

    extracted_params = await extract_params(params, throw_error)
```

The `extract_params` function validates only a few specific fields:

```python
async def extract_params(...):
    # Read the code config settings (stack) from the request.
    generated_code_config = params.get("generatedCodeConfig", "")
    if generated_code_config not in get_args(Stack):
        await throw_error(f"Invalid generated code config: {generated_code_config}")

    # Validate the input mode
    input_mode = params.get("inputMode")
    if input_mode not in get_args(InputMode):
        await throw_error(f"Invalid input mode: {input_mode}")
```

However, many other fields are used without validation, such as:
- `image` (base64 data)
- `resultImage` (base64 data)
- `history` (array of strings)
- Custom API keys and URLs

### Security Test Case
1. Start the application and ensure it's accessible
2. Connect to the WebSocket endpoint `/generate-code`
3. Send a malformed JSON payload with excessively large data or malformed base64 strings:
   ```json
   {
     "image": "data:image/png;base64,MALFORMED_BASE64_DATA_HERE",
     "generatedCodeConfig": "html_tailwind",
     "inputMode": "image",
     "openAiApiKey": "sk-test"
   }
   ```
4. Observe how the application handles the malformed input
5. Try sending different types of malicious inputs, including extremely large payloads
6. Monitor for error messages, crashes, or unexpected behavior
