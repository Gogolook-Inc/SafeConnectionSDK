# URL Checker Implementation

## Overview

The URL Checker functionality provides a complete solution for checking URL safety within the SafeConnection SDK. It implements a dedicated URL checker API with proper architectural separation, Room-based local caching, and a clean interface following the repository pattern.

#### Data Models
- **`UrlCheckResult`** - Sealed class representing check results (Success/Error)

### Status Values
- **`safe`** - URL is safe to visit
- **`suspicious`** - URL may be suspicious, proceed with caution
- **`malicious`** - URL is known to be malicious, avoid visiting  
- **`unknown`** - Status could not be determined

### Response Codes
- **200**: Success - URL check completed
- **400**: Bad Request - Invalid URL format
- **402**: Authentication failed - Invalid user agent or access token
- **403**: JWT validation errors
- **500**: Internal server error

## Local Caching (Room Database)

### Cache Features
- **Unique URL indexing** for efficient lookups
- **Timestamp indexing** for cache expiration queries
- **24-hour cache duration** by default
- **REPLACE strategy** to update existing entries
- **Automatic cleanup** of expired entries

### Cache Operations
- `getCachedResult(url)` - Retrieve cached result
- `cacheResult(url, result)` - Store result in cache
- `clearExpiredCache(timestamp)` - Remove expired entries
- `clearCacheForUrl(url)` - Remove specific URL cache
- `clearAllCache()` - Clear entire cache

## Usage

### From SafeConnection SDK

```kotlin
// After SDK initialization
val result = SafeConnection.urlChecker.checkUrl("https://example.com")
when (result) {
    is UrlCheckResult.Success -> {
        println("URL: ${result.url}")
        println("Status: ${result.status}") // safe, suspicious, malicious, unknown
        println("Cached: ${result.timestamp}")
    }
    is UrlCheckResult.Error -> {
        println("URL: ${result.url}")
        println("Error: ${result.errorMessage}")
    }
}
```

### Status Levels

- **SAFE**: URL is safe to visit
- **SUSPICIOUS**: URL may be suspicious, proceed with caution  
- **MALICIOUS**: URL is known to be malicious, avoid visiting
- **UNKNOWN**: Status could not be determined

## Caching

- Local caching using Realm database
- Default cache duration: 24 hours
- Cache policy can be customized via `CachePolicy` parameters
=======
## Error Handling

### Network Errors
- Connection timeouts (30-second default)
- Network unavailability
- DNS resolution failures

### API Errors
- Invalid URL format validation
- Authentication failures
- Server-side errors (500, 502, 503)
- Rate limiting responses

### Validation Errors
- Empty URL input
- Malformed URL strings
- Unsupported URL schemes

### Cache Errors
- Database corruption recovery
- Storage space limitations
- Cache expiration handling

## Configuration

### Timeouts
- **API Timeout**: 30 seconds (configurable)
- **Cache Duration**: 24 hours (configurable)