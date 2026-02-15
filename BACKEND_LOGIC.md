# Backend Logic - Bharat AI Text Simplifier
## Conceptual Implementation with Google Gemini API

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    BACKEND LAYER                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │         FLASK WEB APPLICATION                    │  │
│  │                                                  │  │
│  │  ┌────────────────────────────────────────────┐ │  │
│  │  │  Route Handlers                            │ │  │
│  │  │  • GET  /                                  │ │  │
│  │  │  • POST /simplify                          │ │  │
│  │  │  • POST /translate                         │ │  │
│  │  │  • GET  /health                            │ │  │
│  │  └────────────────────────────────────────────┘ │  │
│  │                                                  │  │
│  │  ┌────────────────────────────────────────────┐ │  │
│  │  │  Middleware                                │ │  │
│  │  │  • Request Validation                      │ │  │
│  │  │  • Input Sanitization                      │ │  │
│  │  │  • Rate Limiting                           │ │  │
│  │  │  • Error Handling                          │ │  │
│  │  └────────────────────────────────────────────┘ │  │
│  │                                                  │  │
│  │  ┌────────────────────────────────────────────┐ │  │
│  │  │  Business Logic                            │ │  │
│  │  │  • Text Processing                         │ │  │
│  │  │  • API Integration                         │ │  │
│  │  │  • Response Formatting                     │ │  │
│  │  └────────────────────────────────────────────┘ │  │
│  │                                                  │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
│  ┌──────────────────────────────────────────────────┐  │
│  │         EXTERNAL API CLIENT                      │  │
│  │                                                  │  │
│  │  • Google Gemini API Integration                │  │
│  │  • Request Builder                              │  │
│  │  • Response Parser                              │  │
│  │  • Error Handler                                │  │
│  └──────────────────────────────────────────────────┘  │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Pseudocode Implementation

### 1. Main Application Setup

```python
# app.py - Main Flask Application

# Import required libraries
IMPORT Flask, request, jsonify, render_template
IMPORT requests  # For API calls
IMPORT os  # For environment variables
IMPORT time  # For timing
IMPORT logging  # For error logging

# Initialize Flask app
app = Flask(__name__)

# Configuration
CONFIG = {
    'GEMINI_API_KEY': os.getenv('GEMINI_API_KEY'),
    'GEMINI_API_URL': 'https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent',
    'MAX_TEXT_LENGTH': 5000,
    'API_TIMEOUT': 30,
    'RATE_LIMIT': 10  # requests per minute
}

# Setup logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)
```

### 2. Route Handlers

```python
# ===== HOME PAGE ROUTE =====
@app.route('/')
FUNCTION home():
    """
    Serve the main landing page
    """
    TRY:
        RETURN render_template('index.html')
    CATCH Exception as e:
        logger.error(f"Error serving home page: {e}")
        RETURN "Service temporarily unavailable", 503


# ===== SIMPLIFY TEXT ROUTE =====
@app.route('/simplify', methods=['POST'])
FUNCTION simplify_text():
    """
    Handle text simplification requests
    
    Input: Form data with 'text' field
    Output: HTML page with original and simplified text
    """
    
    # Step 1: Extract input
    user_text = request.form.get('text', '').strip()
    
    # Step 2: Validate input
    is_valid, error_message = validate_input(user_text)
    IF NOT is_valid:
        RETURN render_template('error.html', 
                             error=error_message), 400
    
    # Step 3: Sanitize input
    clean_text = sanitize_input(user_text)
    
    # Step 4: Build API request
    prompt = build_simplification_prompt(clean_text)
    
    # Step 5: Call Gemini API
    TRY:
        start_time = time.time()
        simplified_text = call_gemini_api(prompt)
        processing_time = time.time() - start_time
        
        # Step 6: Log success
        logger.info(f"Simplification successful. Time: {processing_time}s")
        
        # Step 7: Return result
        RETURN render_template('result.html',
                             original=user_text,
                             processed=simplified_text,
                             mode='simplified',
                             stats={
                                 'original_length': len(user_text),
                                 'processed_length': len(simplified_text),
                                 'processing_time': round(processing_time, 2)
                             })
    
    CATCH APIError as e:
        logger.error(f"API Error: {e}")
        RETURN render_template('error.html',
                             error="Processing failed. Please try again."), 500
    
    CATCH TimeoutError:
        logger.error("API Timeout")
        RETURN render_template('error.html',
                             error="Service is busy. Please try again."), 503


# ===== TRANSLATE TEXT ROUTE =====
@app.route('/translate', methods=['POST'])
FUNCTION translate_text():
    """
    Handle Hindi translation requests
    
    Input: Form data with 'text' field
    Output: HTML page with original and translated text
    """
    
    # Step 1: Extract input
    user_text = request.form.get('text', '').strip()
    
    # Step 2: Validate input
    is_valid, error_message = validate_input(user_text)
    IF NOT is_valid:
        RETURN render_template('error.html',
                             error=error_message), 400
    
    # Step 3: Sanitize input
    clean_text = sanitize_input(user_text)
    
    # Step 4: Build API request
    prompt = build_translation_prompt(clean_text)
    
    # Step 5: Call Gemini API
    TRY:
        start_time = time.time()
        translated_text = call_gemini_api(prompt)
        processing_time = time.time() - start_time
        
        # Step 6: Log success
        logger.info(f"Translation successful. Time: {processing_time}s")
        
        # Step 7: Return result
        RETURN render_template('result.html',
                             original=user_text,
                             processed=translated_text,
                             mode='translated',
                             stats={
                                 'original_length': len(user_text),
                                 'processed_length': len(translated_text),
                                 'processing_time': round(processing_time, 2)
                             })
    
    CATCH APIError as e:
        logger.error(f"API Error: {e}")
        RETURN render_template('error.html',
                             error="Translation failed. Please try again."), 500
    
    CATCH TimeoutError:
        logger.error("API Timeout")
        RETURN render_template('error.html',
                             error="Service is busy. Please try again."), 503


# ===== HEALTH CHECK ROUTE =====
@app.route('/health')
FUNCTION health_check():
    """
    Health check endpoint for monitoring
    """
    RETURN jsonify({
        'status': 'healthy',
        'timestamp': time.time(),
        'service': 'Bharat AI Text Simplifier'
    }), 200
```

### 3. Validation Functions

```python
# ===== INPUT VALIDATION =====
FUNCTION validate_input(text):
    """
    Validate user input
    
    Args:
        text (str): User input text
    
    Returns:
        tuple: (is_valid, error_message)
    """
    
    # Check if empty
    IF text is None OR len(text) == 0:
        RETURN (False, "Please enter some text to process")
    
    # Check length
    IF len(text) > CONFIG['MAX_TEXT_LENGTH']:
        RETURN (False, f"Text too long. Maximum {CONFIG['MAX_TEXT_LENGTH']} characters allowed")
    
    # Check for minimum length
    IF len(text) < 10:
        RETURN (False, "Text too short. Please enter at least 10 characters")
    
    # All checks passed
    RETURN (True, None)


# ===== INPUT SANITIZATION =====
FUNCTION sanitize_input(text):
    """
    Sanitize user input to prevent injection attacks
    
    Args:
        text (str): Raw user input
    
    Returns:
        str: Sanitized text
    """
    
    # Remove HTML tags
    text = remove_html_tags(text)
    
    # Escape special characters
    text = escape_special_chars(text)
    
    # Normalize whitespace
    text = normalize_whitespace(text)
    
    # Remove control characters
    text = remove_control_chars(text)
    
    RETURN text
```

### 4. Prompt Engineering

```python
# ===== SIMPLIFICATION PROMPT =====
FUNCTION build_simplification_prompt(text):
    """
    Build prompt for text simplification
    
    Args:
        text (str): Text to simplify
    
    Returns:
        str: Complete prompt for Gemini API
    """
    
    prompt = f"""You are an expert at simplifying complex English text for people with basic English skills.

Your task: Simplify the following text while keeping the same meaning.

Rules:
1. Use simple, common words (avoid jargon and technical terms)
2. Use short sentences (maximum 15-20 words per sentence)
3. Break complex ideas into smaller parts
4. Keep the same meaning and key information
5. Use active voice instead of passive voice
6. Explain difficult concepts in simple terms

Original Text:
{text}

Simplified Text:"""
    
    RETURN prompt


# ===== TRANSLATION PROMPT =====
FUNCTION build_translation_prompt(text):
    """
    Build prompt for Hindi translation
    
    Args:
        text (str): Text to translate
    
    Returns:
        str: Complete prompt for Gemini API
    """
    
    prompt = f"""You are an expert translator specializing in English to Hindi translation.

Your task: Translate the following English text into simple, easy-to-understand Hindi.

Rules:
1. Use simple Hindi words that common people understand
2. Use Devanagari script
3. Keep sentences short and clear
4. Preserve the original meaning
5. Use everyday language, not formal or literary Hindi
6. Make it accessible to people with basic Hindi literacy

English Text:
{text}

Hindi Translation (हिंदी अनुवाद):"""
    
    RETURN prompt
```

### 5. API Integration

```python
# ===== GEMINI API CALL =====
FUNCTION call_gemini_api(prompt):
    """
    Call Google Gemini API with the given prompt
    
    Args:
        prompt (str): Complete prompt for the AI
    
    Returns:
        str: AI-generated response text
    
    Raises:
        APIError: If API call fails
        TimeoutError: If API call times out
    """
    
    # Build request payload
    payload = {
        "contents": [
            {
                "parts": [
                    {"text": prompt}
                ]
            }
        ],
        "generationConfig": {
            "temperature": 0.7,
            "maxOutputTokens": 2048,
            "topP": 0.8,
            "topK": 40
        }
    }
    
    # Build request URL with API key
    url = f"{CONFIG['GEMINI_API_URL']}?key={CONFIG['GEMINI_API_KEY']}"
    
    # Set headers
    headers = {
        "Content-Type": "application/json"
    }
    
    # Make API call with timeout
    TRY:
        response = requests.post(
            url,
            json=payload,
            headers=headers,
            timeout=CONFIG['API_TIMEOUT']
        )
        
        # Check response status
        IF response.status_code == 200:
            # Parse response
            result = parse_gemini_response(response.json())
            RETURN result
        
        ELSE IF response.status_code == 429:
            RAISE APIError("Rate limit exceeded. Please try again later.")
        
        ELSE IF response.status_code >= 500:
            RAISE APIError("Service temporarily unavailable.")
        
        ELSE:
            RAISE APIError(f"API error: {response.status_code}")
    
    CATCH requests.Timeout:
        RAISE TimeoutError("Request timed out")
    
    CATCH requests.RequestException as e:
        logger.error(f"Request failed: {e}")
        RAISE APIError("Network error occurred")


# ===== RESPONSE PARSER =====
FUNCTION parse_gemini_response(response_json):
    """
    Parse Gemini API response and extract text
    
    Args:
        response_json (dict): JSON response from API
    
    Returns:
        str: Extracted text
    
    Raises:
        APIError: If response format is invalid
    """
    
    TRY:
        # Navigate response structure
        candidates = response_json.get('candidates', [])
        
        IF len(candidates) == 0:
            RAISE APIError("No response generated")
        
        content = candidates[0].get('content', {})
        parts = content.get('parts', [])
        
        IF len(parts) == 0:
            RAISE APIError("Empty response")
        
        text = parts[0].get('text', '')
        
        IF len(text) == 0:
            RAISE APIError("No text in response")
        
        RETURN text.strip()
    
    CATCH (KeyError, IndexError, TypeError) as e:
        logger.error(f"Response parsing error: {e}")
        RAISE APIError("Invalid response format")
```

### 6. Error Handling

```python
# ===== CUSTOM EXCEPTIONS =====
CLASS APIError(Exception):
    """Custom exception for API-related errors"""
    PASS


# ===== GLOBAL ERROR HANDLER =====
@app.errorhandler(404)
FUNCTION not_found(error):
    RETURN render_template('error.html',
                         error="Page not found"), 404


@app.errorhandler(500)
FUNCTION internal_error(error):
    logger.error(f"Internal error: {error}")
    RETURN render_template('error.html',
                         error="Internal server error"), 500


@app.errorhandler(Exception)
FUNCTION handle_exception(error):
    logger.error(f"Unhandled exception: {error}")
    RETURN render_template('error.html',
                         error="An unexpected error occurred"), 500
```

### 7. Application Entry Point

```python
# ===== MAIN ENTRY POINT =====
IF __name__ == '__main__':
    # Check if API key is configured
    IF NOT CONFIG['GEMINI_API_KEY']:
        logger.error("GEMINI_API_KEY not configured")
        EXIT(1)
    
    # Run application
    app.run(
        host='0.0.0.0',
        port=5000,
        debug=True  # Set to False in production
    )
```

---

## API Request/Response Examples

### Simplification Request
```json
{
  "contents": [
    {
      "parts": [
        {
          "text": "Simplify: The implementation of the aforementioned policy..."
        }
      ]
    }
  ],
  "generationConfig": {
    "temperature": 0.7,
    "maxOutputTokens": 2048
  }
}
```

### Simplification Response
```json
{
  "candidates": [
    {
      "content": {
        "parts": [
          {
            "text": "To use this policy, we need to carefully check..."
          }
        ]
      },
      "finishReason": "STOP"
    }
  ]
}
```

---

## Performance Optimization

### Caching Strategy (Future Enhancement)
```python
# Use Redis for caching frequent requests
FUNCTION get_cached_result(text, mode):
    cache_key = generate_cache_key(text, mode)
    cached = redis.get(cache_key)
    IF cached:
        RETURN cached
    RETURN None

FUNCTION cache_result(text, mode, result):
    cache_key = generate_cache_key(text, mode)
    redis.setex(cache_key, 3600, result)  # Cache for 1 hour
```

### Rate Limiting
```python
# Simple in-memory rate limiting
rate_limit_store = {}

FUNCTION check_rate_limit(ip_address):
    current_time = time.time()
    
    IF ip_address NOT IN rate_limit_store:
        rate_limit_store[ip_address] = []
    
    # Remove old requests (older than 1 minute)
    rate_limit_store[ip_address] = [
        t FOR t IN rate_limit_store[ip_address]
        IF current_time - t < 60
    ]
    
    # Check if limit exceeded
    IF len(rate_limit_store[ip_address]) >= CONFIG['RATE_LIMIT']:
        RETURN False
    
    # Add current request
    rate_limit_store[ip_address].append(current_time)
    RETURN True
```

---

*Complete backend logic specification for implementation.*
