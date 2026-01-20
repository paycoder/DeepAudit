# SSRF Vulnerability Fix Summary

## Issue
GitHub Issue: https://github.com/lintsinghua/DeepAudit/issues/137

A Server-Side Request Forgery (SSRF) vulnerability was identified in the `/api/v1/config/test-llm` endpoint. The endpoint accepted user-controlled URLs without proper validation, allowing attackers to probe internal network services.

## Fix Applied

### 1. URL Validation Function
Added a comprehensive URL validation function `is_safe_url()` that:
- Blocks localhost and loopback addresses (127.0.0.0/8, ::1)
- Blocks private IP ranges (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16)
- Blocks link-local addresses (169.254.0.0/16)
- Blocks IPv6 private ranges (fc00::/7, fe80::/10)
- Resolves domain names and checks if they point to internal IPs
- Handles DNS resolution failures securely

### 2. Protected Endpoints

#### backend/app/api/v1/endpoints/config.py
- **test_llm_connection** (POST `/api/v1/config/test-llm`): Validates `request.baseUrl` before making LLM API calls
- **update_my_config** (PUT `/api/v1/config/me`): Validates `llmBaseUrl` and `ollamaBaseUrl` before saving to database

#### backend/app/api/v1/endpoints/embedding_config.py
- **test_embedding** (POST `/api/v1/embedding-config/test`): Validates `request.base_url` before making embedding API calls
- **update_config** (PUT `/api/v1/embedding-config/config`): Validates `config.base_url` before saving to database

### 3. Defense in Depth
The fix implements multiple layers of protection:
1. **Input validation**: URLs are validated at the API endpoint level before any processing
2. **Configuration validation**: URLs are validated when users save their configuration
3. **Runtime protection**: The LLM and embedding services use validated URLs from the database

## Testing
Created `backend/test_ssrf_protection.py` to verify the fix:
- ✅ All internal IP addresses are correctly blocked
- ✅ All public API endpoints are correctly allowed
- ✅ Edge cases (empty URLs, localhost variants) are handled correctly

## Files Modified
1. `backend/app/api/v1/endpoints/config.py`
   - Added imports: `ipaddress`, `socket`, `urlparse`
   - Added `INTERNAL_IP_RANGES` constant
   - Added `is_safe_url()` function
   - Added validation in `test_llm_connection()` endpoint
   - Added validation in `update_my_config()` endpoint

2. `backend/app/api/v1/endpoints/embedding_config.py`
   - Added imports: `ipaddress`, `socket`, `urlparse`
   - Added `INTERNAL_IP_RANGES` constant
   - Added `is_safe_url()` function
   - Added validation in `test_embedding()` endpoint
   - Added validation in `update_config()` endpoint

3. `backend/test_ssrf_protection.py` (new file)
   - Comprehensive test suite for SSRF protection

## Security Impact
- **Before**: Attackers could probe internal network services, map network topology, and potentially access sensitive internal APIs
- **After**: All attempts to access internal network addresses are blocked with a clear error message

## Error Messages
When an internal URL is detected, users receive:
- "不允许访问内网地址，请使用公网API地址" (Do not allow access to internal network addresses, please use public API addresses)

## Recommendations
1. Consider adding rate limiting to these endpoints to prevent abuse
2. Consider logging blocked SSRF attempts for security monitoring
3. Review other endpoints that accept URLs for similar vulnerabilities
