# How to Solve CAPTCHA in Browser-use with CapSolver API

## Introduction
 Browser-use is a powerful open-source Python library that enables AI agents to control web browsers for automating tasks such as data scraping, form filling, and repetitive online activities. By leveraging Playwright for browser automation and integrating with large language models (LLMs) like OpenAI’s GPT models, Browser-use allows users to issue natural language commands, making it accessible even for those without extensive coding skills. However, a common challenge in web automation is encountering CAPTCHAs, which are designed to block automated scripts and can disrupt Browser-use’s workflows.

[CapSolver](https://www.capsolver.com/?utm_source=blog&utm_medium=integration&utm_campaign=browser-use) is an AI-powered service that specializes in solving various types of CAPTCHAs, including reCAPTCHA,and Cloudflare Turnstile. By integrating CapSolver with Browser-use, you can ensure that your automation tasks proceed smoothly without requiring manual intervention to solve CAPTCHAs.

This article provides a step-by-step guide on how to integrate CapSolver with Browser-use to handle CAPTCHAs effectively. We’ll cover the necessary setup, provide a complete code example, and share best practices to help you get started.

## Browser-use Overview & Use Cases
[Browser-use](https://github.com/browser-use/browser-use) is a Python library that simplifies web automation by allowing AI agents to interact with websites through natural language instructions. It uses Playwright under the hood to control browsers like Chromium, Firefox, and WebKit, and integrates with LLMs to interpret and execute user commands. This makes Browser-use ideal for automating complex tasks without writing extensive code.

### Use Cases
Browser-use supports a variety of automation tasks, including:

- **Data Scraping**: Extracting data from websites for market research, price monitoring, or content aggregation.
- **Form Filling**: Automating the process of filling out online forms with data from various sources, such as job applications or account registrations.
- **Task Automation**: Performing repetitive tasks like logging into accounts, navigating websites, or clicking buttons.

These tasks often involve interacting with websites that deploy CAPTCHAs to prevent automated access, making a reliable CAPTCHA-solving solution essential for uninterrupted automation.

## Why CAPTCHA Solving is Needed
Websites often deploy anti-bot defenses like CAPTCHAs to block automated access, spam, and malicious activities. These CAPTCHAs—designed to differentiate humans from bots with challenges like clicking checkboxes or solving image puzzles—pose a significant obstacle for web scraping. When automating tasks with Browser-use, encountering a CAPTCHA can stop the process dead in its tracks, preventing the tool from scraping the desired data without manual intervention.

Common CAPTCHA types include:

| CAPTCHA Type         | Description                                                                 |
|----------------------|-----------------------------------------------------------------------------|
| **reCAPTCHA v2**     | Requires users to check a box or select images based on a prompt.            |
| **reCAPTCHA v3**     | Uses a scoring system to assess user behavior, often invisible to users.     |                    |
| **Cloudflare Turnstile** | A privacy-focused CAPTCHA alternative that minimizes user interaction.   |

For web scraping, this is a critical issue: CAPTCHAs are specifically intended to thwart the kind of automation Browser-use relies on to extract data from websites. Without a way to bypass these barriers, scraping efforts are stalled, rendering the automation ineffective. Fortunately, integrating CapSolver’s API with Browser-use provides a powerful solution. CapSolver automatically solves these CAPTCHAs, enabling Browser-use to pass through anti-bot defenses and successfully scrape data without interruption. Whether it’s handling reCAPTCHA v2, or Cloudflare Turnstile, CapSolver ensures that Browser-use can tackle a wide range of CAPTCHA challenges, making it an essential tool for seamless and efficient data extraction from protected websites.

This integration is a game-changer for anyone looking to scrape data from sites that use CAPTCHAs, as it eliminates the need for manual input and keeps the web scraping process running smoothly.

## How to Use CapSolver to Handle CAPTCHAs
CapSolver offers an API that can solve various CAPTCHAs using advanced AI algorithms. To integrate CapSolver with Browser-use, you can define a custom action using the `@controller.action` decorator. This action will detect CAPTCHAs on a webpage, extract necessary information (e.g., the site key for reCAPTCHA), call CapSolver’s API to obtain a solution, and inject the solution into the page.

### Steps to Integrate CapSolver with Browser-use
1. **Sign Up for CapSolver**: Create an account at [CapSolver](https://www.capsolver.com?utm_source=blog&utm_medium=integration&utm_campaign=browser-use), add funds, and obtain your API key.
2. **Set Up Browser-use**: Install Browser-use and its dependencies, and configure your environment with API keys for an LLM provider (e.g., OpenAI).
3. **Install Dependencies**: Use Python and install the required packages: `browser-use`, `playwright`, and `requests`.
4. **Define a Custom Action**: Create a custom action in your Browser-use script to handle CAPTCHAs using CapSolver’s API.
5. **Run the Agent**: Instruct the AI agent to call the custom action when a CAPTCHA is encountered during task execution.

### Key Code Snippet
Below is an example of a custom action to solve a reCAPTCHA v2 using CapSolver’s API:

```python
import requests
import time
from browser_use import Controller, ActionResult
from playwright.async_api import Page

CAPSOLVER_API_KEY = 'YOUR_CAPSOLVER_API_KEY'

@controller.action('Solve CAPTCHA', domains=['*'])
async def solve_captcha(page: Page) -> ActionResult:
    if await page.query_selector('.g-recaptcha'):
        site_key = await page.evaluate("document.querySelector('.g-recaptcha').getAttribute('data-sitekey')")
        page_url = page.url

        # Create task with CapSolver
        response = requests.post('https://api.capsolver.com/createTask', json={
            'clientKey': CAPSOLVER_API_KEY,
            'task': {
                'type': 'ReCaptchaV2TaskProxyLess',
                'websiteURL': page_url,
                'websiteKey': site_key,
            }
        })
        task_id = response.json().get('taskId')
        if not task_id:
            return ActionResult(success=False, message='Failed to create CapSolver task')

        # Poll for solution
        while True:
            time.sleep(5)
            result_response = requests.post('https://api.capsolver.com/getTaskResult', json={
                'clientKey': CAPSOLVER_API_KEY,
                'taskId': task_id
            })
            result = result_response.json()
            if result.get('status') == 'ready':
                solution = result.get('solution', {}).get('gRecaptchaResponse')
                if solution:
                    await page.evaluate(f"document.getElementById('g-recaptcha-response').innerHTML = '{solution}';")
                    return ActionResult(success=True, message='CAPTCHA solved')
                else:
                    return ActionResult(success=False, message='No solution found')
            elif result.get('status') == 'failed':
                return ActionResult(success=False, message='CapSolver failed to solve CAPTCHA')
    return ActionResult(success=False, message='No CAPTCHA found')
```

This snippet defines a custom action that checks for a reCAPTCHA v2 element, extracts the site key, creates a task with CapSolver, polls for the solution, and injects the token into the page.

## Complete Code Example + Step-by-Step Explanation
Below is a complete code example that demonstrates how to integrate CapSolver with Browser-use to solve CAPTCHAs.

### Prerequisites
Ensure you have the necessary packages installed:

```bash
pip install browser-use playwright requests
playwright install
```

Set up your environment with the required API keys. Create a `.env` file with your OpenAI and CapSolver API keys:

```env
OPENAI_API_KEY=your_openai_api_key
CAPSOLVER_API_KEY=your_capsolver_api_key
```

### Complete Code Example
Create a Python script with the following content:

```python
import os
import asyncio
import requests
from dotenv import load_dotenv
from browser_use import Agent, Controller, ActionResult
from browser_use.browser import BrowserSession
from browser_use.llm import ChatOpenAI
from playwright.async_api import Page

# Load environment variables from .env file
load_dotenv()

CAPSOLVER_API_KEY = os.getenv('CAPSOLVER_API_KEY')

controller = Controller()

@controller.action('Solve CAPTCHA', domains=['*'])
async def solve_captcha(page) -> ActionResult:
    if await page.query_selector('.g-recaptcha'):
        site_key = await page.evaluate("document.querySelector('.g-recaptcha').getAttribute('data-sitekey')")
        page_url = page.url

        response = requests.post('https://api.capsolver.com/createTask', json={
            'clientKey': CAPSOLVER_API_KEY,
            'task': {
                'type': 'ReCaptchaV2TaskProxyLess',
                'websiteURL': page_url,
                'websiteKey': site_key,
            }
        })

        task_id = response.json().get('taskId')
        print(task_id)
        if not task_id:
            return ActionResult(success=False, message='Failed to create CapSolver task')

        while True:
            await asyncio.sleep(5)
            result_response = requests.post('https://api.capsolver.com/getTaskResult', json={
                'clientKey': CAPSOLVER_API_KEY,
                'taskId': task_id
            })
            result = result_response.json()
            print(f"CAPTCHA result status: {result.get('status')}")
            if result.get('status') == 'ready':
                solution = result.get('solution', {}).get('gRecaptchaResponse')
                print(f"CAPTCHA solution: {solution}")
                if solution:
                    print("Submitting CAPTCHA solution...")
                    # Try both possible input fields for the CAPTCHA token
                    await page.evaluate(f"""
                        // Try the standard g-recaptcha-response field
                        var gRecaptchaResponse = document.getElementById('g-recaptcha-response');
                        if (gRecaptchaResponse) {{
                            gRecaptchaResponse.innerHTML = '{solution}';
                            var event = new Event('input', {{ bubbles: true }});
                            gRecaptchaResponse.dispatchEvent(event);
                        }}
                        
                        // Also try the recaptcha-token field
                        var recaptchaToken = document.getElementById('recaptcha-token');
                        if (recaptchaToken) {{
                            recaptchaToken.value = '{solution}';
                            var event = new Event('input', {{ bubbles: true }});
                            recaptchaToken.dispatchEvent(event);
                        }}
                    """)
                    
                    # Wait a moment for the token to be processed
                    await asyncio.sleep(2)
                    print("Token injected successfully! CAPTCHA solved.")
                    
                    # Method 2: Click submit button directly using the correct selector
                    print("Now clicking submit button...")
                    try:
                        # Use the specific button selector you provided
                        submit_button = await page.query_selector("body > main > form > fieldset > button")
                        if submit_button:
                            await submit_button.click()
                            print("✅ Submit button clicked successfully!")
                        else:
                            print("❌ Submit button not found!")
                            return ActionResult(success=False, message='Submit button not found')
                    except Exception as e:
                        print(f"❌ Error clicking submit button: {e}")
                        return ActionResult(success=False, message=f'Error clicking submit: {e}')
                    
                    print("CAPTCHA solved and form submitted successfully!")
                    return ActionResult(success=True, message='CAPTCHA solved and form submitted')
                   
                else:
                    return ActionResult(success=False, message='No solution found')
            elif result.get('status') == 'failed':
                return ActionResult(success=False, message='CapSolver failed to solve CAPTCHA')
    return ActionResult(success=False, message='No CAPTCHA found')

llm = ChatOpenAI(model="gpt-4o-mini")

async def main():
    try:
        print("🚀 Starting browser-use CAPTCHA solver agent...")
        
        # Simple task instruction for CAPTCHA solving and form submission
        task = """Navigate to https://recaptcha-demo.appspot.com/recaptcha-v2-checkbox.php and solve the CAPTCHA, then submit the form.

STEP 1: Navigate to the reCAPTCHA demo page: https://recaptcha-demo.appspot.com/recaptcha-v2-checkbox.php

STEP 2: Wait for the page to fully load. You should see a form with input fields and a reCAPTCHA checkbox.

STEP 3: Look for a reCAPTCHA element (usually a checkbox that says "I'm not a robot" or similar).

STEP 4: Use the "solve_captcha" action to automatically solve the CAPTCHA and submit the form.

STEP 5: Report the final result.

Note: The solve_captcha action will handle both solving the CAPTCHA and submitting the form automatically."""
        
        # Create browser session first
        browser_session = BrowserSession()
        
        # Create agent with the browser session
        agent = Agent(
            task=task, 
            llm=llm, 
            controller=controller,
            browser_session=browser_session
        )
        
        print("📱 Running CAPTCHA solver agent...")
        result = await agent.run()
        print(f"✅ Agent completed: {result}")
        
        # Keep browser open to see results
        input('Press Enter to close the browser...')
        await browser_session.close()
        
    except Exception as e:
        print(f"❌ Error: {e}")

if __name__ == "__main__":
    asyncio.run(main())

```

### Step-by-Step Explanation
| Step | Description |
|------|-------------|
| **1. Install Dependencies** | Install `browser-use`, `playwright`, and `requests` using `pip install browser-use playwright requests`. Run `playwright install` to install the necessary browsers. |
| **2. Configure Environment** | Create a `.env` file with your OpenAI and CapSolver API keys to securely store credentials. |
| **3. Define Custom Action** | Use the `@controller.action` decorator to define `solve_captcha`, which checks for a reCAPTCHA v2 element, extracts the site key, calls CapSolver’s API, and injects the solution into the page. |
| **4. Initialize Controller and Agent** | Create a `Controller` instance, define the custom action, initialize the LLM (e.g., ChatOpenAI with GPT-4o-mini), and create the `BrowserUse` agent with the controller. |
| **5. Run the Agent** | Provide a task that includes instructions to solve CAPTCHAs using the custom action if encountered. The agent navigates to the specified URL, detects the CAPTCHA, calls the custom action, and submits the form. |
| **6. Error Handling** | The custom action includes error handling for cases where the CapSolver task fails or no solution is found, returning appropriate `ActionResult` objects. |
| **7. Clean Up** | The agent automatically manages browser resources, closing the browser when the task is complete. |

This example focuses on reCAPTCHA v2, but you can adapt it for other CAPTCHA types by modifying the task type (e.g., `AntiTurnstileTaskProxyLess` for Turnstile).

## Demo Walkthrough
This section describes how the integration works using a sample task to navigate to a demo page with a reCAPTCHA v2 checkbox and submit the form.

1. **Task Setup**: The task instructs the AI agent to visit `https://recaptcha-demo.appspot.com/recaptcha-v2-checkbox.php`, submit the form, and solve any CAPTCHAs using the `solve_captcha` action.
2. **Agent Execution**: The Browser-use agent launches a Playwright-controlled browser and navigates to the specified URL.
3. **CAPTCHA Detection**: The agent checks for a CAPTCHA by looking for the `.g-recaptcha` element. If found, it triggers the `solve_captcha` action.
4. **Custom Action Execution**: The `solve_captcha` action extracts the site key and page URL, creates a task with CapSolver’s API, and polls for the solution.
5. **Solution Injection**: Once the solution is received, the action injects the token into the `g-recaptcha-response` field.
6. **Form Submission**: The agent submits the form by clicking the submit button, completing the task.
7. **Task Completion**: The agent returns the result, indicating successful form submission.

Visually, you would see the browser navigate to the demo page, the reCAPTCHA checkbox being marked automatically after the solution is injected, and the form being submitted successfully.

## FAQ Section
| Question | Answer |
|----------|--------|
| **What types of CAPTCHAs can CapSolver solve?** | CapSolver supports reCAPTCHA v2/v3, Cloudflare Turnstile, and more. Refer to the [CapSolver documentation](https://docs.capsolver.com) for a complete list. |
| **How do I handle different CAPTCHA types?** | Modify the custom action to detect the CAPTCHA type (e.g., check for specific elements or attributes) and use the appropriate CapSolver task type, such as `AntiTurnstileTaskProxyLess` for Turnstile. |
| **What if CapSolver fails to solve the CAPTCHA?** | Implement retry logic in the custom action or notify the user of the failure. Log errors for debugging and consider fallback strategies. |
| **Can I use CapSolver with other automation tools?** | Yes, CapSolver’s API is compatible with any tool that supports HTTP requests, including Selenium, Puppeteer, and Playwright. |
| **Do I need proxies with CapSolver?** | Proxies may be required for region-specific or IP-bound CAPTCHAs. CapSolver supports proxy usage; see their documentation for details. |

## Conclusion & Call to Action
Integrating CapSolver with Browser-use provides a robust solution for handling CAPTCHAs in web automation tasks. By defining a custom action to solve CAPTCHAs, you can ensure that your AI agents navigate websites seamlessly, even when faced with anti-bot measures. This combination leverages Browser-use’s ease of use and CapSolver’s powerful CAPTCHA-solving capabilities to create efficient automation workflows.

To get started, sign up for [CapSolver](https://www.capsolver.com?utm_source=blog&utm_medium=integration&utm_campaign=browser-use) and explore [Browser-use](https://github.com/browser-use/browser-use). Follow the setup instructions and implement the code example provided. For more details, visit the [CapSolver documentation](https://docs.capsolver.com?utm_source=blog&utm_medium=integration&utm_campaign=browser-use) and [Browser-use documentation](https://docs.browser-use.com). Try this integration in your next automation project and experience the ease of bypassing CAPTCHAs automatically!

Bonus for Browser-use Users: Use the promo code BROWSERUSE when recharging your CapSolver account and receive an exclusive 6% bonus credit—no limits, no expiration.
<img width="533" height="246" alt="image" src="https://github.com/user-attachments/assets/00c3b7c7-d4cd-4229-8cc9-abf067341dcc" />

### Supported Browsers and Tools
- **Browser-use**: Uses Playwright, supporting Chromium, Firefox, and WebKit browsers.
- **CapSolver**: Compatible with any HTTP-capable client, including browser extensions for Chrome and Firefox.

### Learn More and Explore Other Types of Frameworks
- [Browser-use GitHub](https://github.com/browser-use/browser-use)
- [CapSolver Official Website](https://www.capsolver.com?utm_source=blog&utm_medium=integration&utm_campaign=browser-use)
- [Playwright Documentation](https://playwright.dev)
- [CapSolver Documentation](https://docs.capsolver.com?utm_source=blog&utm_medium=integration&utm_campaign=browser-use)
- [Browser-use Documentation](https://docs.browser-use.com)
