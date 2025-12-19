# SafeConnection SDK

SafeConnection is a comprehensive SDK for phone number lookup, spam detection, call blocking, and URL checking.

## Table of Contents

- [Quick Start](#quick-start)
- [API Reference](#api-reference)
  - [Initialization](#initialization)
  - [Number Search](#number-search)
  - [Call Blocking](#call-blocking)
  - [GGA (Gogolook Global API)](#gga-gogolook-global-api)
  - [URL Checker](#url-checker)
- [Data Structures](#data-structures)
- [Response Data Definitions](#response-data-definitions)
  - [Business Categories](#business-categories)
  - [Spam Categories](#spam-categories)
- [Error Handling](#error-handling)
- [Examples](#examples)

## Quick Start

```kotlin
import com.gogolook.safeconnection.SafeConnection
import com.gogolook.safeconnection.Environment
import com.gogolook.safeconnection.data.InitializationResult

// Initialize the SDK
val result = SafeConnection.initialization(
    app = application,
    memberId = "your-member-id", // Optional
    licenseId = "your-license-id",
    environment = Environment.RELEASE
)

when (result) {
    is InitializationResult.InitializedAndAuthenticated -> {
        // SDK is ready to use
        println("Initialized successfully with auth code: ${result.authCode}")
    }
    is InitializationResult.AuthenticationFailed -> {
        // Handle authentication failure
        println("Authentication failed: ${result.authError}")
    }
    is InitializationResult.InitializationFailed -> {
        // Handle initialization failure
        println("Initialization failed: ${result.error}")
    }
    else -> {}
}

// Search for a phone number (E.164 format like "+886..." recommended, or local format like "0912...")
val numberInfo = SafeConnection.search("+886912345678")
```

## API Reference

### Initialization

#### `initialization()`

Initialize the SafeConnection SDK with automatic authentication. This is the primary method for setting up the SDK.

**Method Signature:**
```kotlin
suspend fun initialization(
    app: Application,
    memberId: String? = null,
    licenseId: String,
    environment: Environment = Environment.RELEASE
): InitializationResult
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `app` | `Application` | Yes | Android Application instance |
| `memberId` | `String?` | No | Optional member identifier for your application |
| `licenseId` | `String` | Yes | Your SDK license key |
| `environment` | `Environment` | No | Target environment (default: `RELEASE`) |

**Environment Enum Values:**

| Value | Description |
|-------|-------------|
| `SANDBOX` | Development environment for testing |
| `STAGING` | Pre-production environment for final testing |
| `RELEASE` | Production environment |

**Return Type:** `InitializationResult`

The initialization result is a sealed class with the following possible values:

| Result Type | Description |
|-------------|-------------|
| `InitializationResult.InitializedAndAuthenticated(authCode: Int)` | Both initialization and authentication succeeded. `authCode` is typically 200 for success. |
| `InitializationResult.AuthenticationFailed(authError: Throwable, authCode: Int?)` | SDK initialized but authentication failed. Contains the authentication error. |
| `InitializationResult.InitializationFailed(error: Throwable)` | Initialization failed. Contains the initialization error. |
| `InitializationResult.InitializedOnly` | Initialization completed without authentication (legacy support). |

**Example:**
```kotlin
val result = SafeConnection.initialization(
    app = application,
    memberId = "user123",
    licenseId = "your-license-key",
    environment = Environment.RELEASE
)

when (result) {
    is InitializationResult.InitializedAndAuthenticated -> {
        Log.d("SafeConnection", "Success! Auth code: ${result.authCode}")
    }
    is InitializationResult.AuthenticationFailed -> {
        Log.e("SafeConnection", "Auth failed", result.authError)
    }
    is InitializationResult.InitializationFailed -> {
        Log.e("SafeConnection", "Init failed", result.error)
    }
    else -> {}
}
```

### Number Search

#### `search(e164: String)`

Search for information about a phone number.

**Method Signature:**
```kotlin
suspend fun search(e164: String): NumberInfoGroup
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `e164` | `String` | Yes | Phone number (supports both E.164 format with country code like "+886912345678" or local format like "0912345678") |

**Phone Number Format:**

The API accepts phone numbers in multiple formats:
- **E.164 format** (recommended): Starts with `+` followed by country code and number
  - Examples: `+886912345678`, `+14155552671`, `+447911123456`
- **Local format**: National number without country code (country is determined by device/network settings)
  - Examples: `0912345678`, `4155552671`
- Numbers should not contain spaces, dashes, or other separators

**Return Type:** `NumberInfoGroup`

Returns a `NumberInfoGroup` object containing:
- `infoType`: Source of the information (CONTACT, NETWORK, OFFLINE_DB, NONE)
- `numberInfoList`: List of `NumberInfo` objects

**Example:**
```kotlin
try {
    val result = SafeConnection.search("+886912345678")
    result.numberInfoList.forEach { info ->
        println("Name: ${info.name}")
        println("Business Category: ${info.businessCategory}")
        println("Spam Category: ${info.spamCategory}")
        println("Spam Level: ${info.spamLevel}")
    }
} catch (e: IllegalStateException) {
    // Handle not initialized or not authenticated
}
```

#### `search(e164: String, source: InfoType)`

Search for information about a phone number from a specific source.

**Method Signature:**
```kotlin
suspend fun search(e164: String, source: InfoType): NumberInfoGroup
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `e164` | `String` | Yes | Phone number (E.164 format or local format) |
| `source` | `InfoType` | Yes | Specific information source to query |

**InfoType Values:**

| Value | Description |
|-------|-------------|
| `NONE` | No specific source |
| `CONTACT` | Search in device contacts |
| `NETWORK` | Search via network API |
| `OFFLINE_DB` | Search in offline database |

**Return Type:** `NumberInfoGroup`

**Example:**
```kotlin
// Search only in offline database
val result = SafeConnection.search("+886912345678", InfoType.OFFLINE_DB)
```

### Call Blocking

Methods for managing call blocking functionality.

| Method | Parameters | Return Type | Description |
|--------|------------|-------------|-------------|
| `shouldBlockNumber()` | `incomingEvent: IncomingEvent`, `isAutoSpamOn: Boolean` | `Boolean` | Determine if an incoming call should be blocked |
| `getBlockedList()` | None | `Flow<List<BlockedInfo>>` | Get the current blocked phone numbers list |
| `addBlockedNumber()` | `phoneNumber: String` | `Unit` | Add a phone number to the block list |
| `isNumberBlocked()` | `phoneNumber: String` | `Boolean` | Check if a phone number is blocked |
| `removeBlockedNumber()` | `phoneNumber: String` | `Unit` | Remove a phone number from the block list |

### GGA (Gogolook Global API)

Access the GGA client for advanced features:

```kotlin
val ggaClient = SafeConnection.gga
```

See [GGA_IMPLEMENTATION.md](GGA_IMPLEMENTATION.md) for detailed documentation.

### URL Checker

Access the URL Checker for scanning URLs:

```kotlin
val urlChecker = SafeConnection.urlChecker
```

See [URL_CHECKER_README.md](URL_CHECKER_README.md) for detailed documentation.

## Data Structures

### NumberInfoGroup

```kotlin
data class NumberInfoGroup(
    val infoType: InfoType,
    val numberInfoList: List<NumberInfo>
)
```

### NumberInfo

```kotlin
data class NumberInfo(
    val number: String,
    val name: String,
    val businessCategory: String,
    val spamCategory: String,
    val spamLevel: Int
)
```

**Example JSON Response:**
```json
{
  "infoType": "NETWORK",
  "numberInfoList": [
    {
      "number": "+886912345678",
      "name": "Example Restaurant",
      "businessCategory": "food",
      "spamCategory": "",
      "spamLevel": 0
    },
    {
      "number": "+886987654321",
      "name": "Suspected Fraud",
      "businessCategory": "",
      "spamCategory": "FRAUD",
      "spamLevel": 3
    }
  ]
}
```

## Response Data Definitions

### Business Categories

The `businessCategory` field in `NumberInfo` can contain the following values:

| Business Category | Definition |
|:------------------|:-----------|
| `publicperson` | Individuals known for their public influence or celebrity status. |
| `food` | Establishments offering prepared food services, groceries, or beverages. |
| `shopping` | Retail and wholesale outlets selling goods ranging from apparel to electronics. |
| `beauty` | Services and products related to personal care, aesthetics, and wellness. |
| `education` | Institutions and services offering learning experiences and certifications. |
| `entertainment` | Activities and venues providing recreational and cultural experiences. |
| `life` | Services aimed at improving daily living through personal and home care. |
| `health` | Medical, wellness, and fitness services and institutions for physical and mental health. |
| `travel` | Services related to the planning and facilitation of travel and accommodations. |
| `automobile` | Businesses involved in the sale, maintenance, and repair of vehicles. |
| `traffic` | Services and solutions for managing and facilitating the flow of vehicles and pedestrians. |
| `professional` | Professionals offering specialized skills and services in various fields. |
| `bank` | Financial institutions offering monetary transactions, loans, and financial advice. |
| `activity` | Organized recreational or educational events and programs. |
| `government` | Public sector entities providing civic services and administration. |
| `politics` | Activities associated with governance, campaigning, and political consultation. |
| `organization` | Entities organized for a collective purpose, both non-profit and for-profit. |
| `pet` | Services and products related to the care and management of pets. |
| `logistic` | The management and coordination of the transportation, warehousing, and distribution of goods. |
| `media` | Platforms and channels for the distribution of information, news, and entertainment. |
| `others` | Categories that do not fit into the other defined areas, including miscellaneous services. |

### Spam Categories

The `spamCategory` field in `NumberInfo` can contain the following values:

| Spam Category | Definition |
|:--------------|:-----------|
| `FRAUD` | The number is used for any type of scam including investment scam, delivery scam, job scam, impersonation scam, etc. |
| `HARASSMENT` | The user of this number has some harassing behaviors including hanging up the call immediately after pick-up, no response, etc. |
| `TELEMARKETING` | The number is used for any type of telemarketing including insurance, loan, etc. |
| `HFB` | High Frequency Block. Derived from "OTHERS" reports; if most-reported, labeled as HFB. |

**Note:** An empty string (`""`) in `spamCategory` indicates that the number is not classified as spam.

## Error Handling

### InitializationResult Types

The `initialization()` method returns different result types based on success or failure:

#### 1. InitializedAndAuthenticated

```kotlin
InitializationResult.InitializedAndAuthenticated(authCode: Int)
```

**Meaning:** Both initialization and authentication completed successfully.

**authCode:** Typically `200` for successful authentication.

**Action:** The SDK is ready to use. Proceed with number searches or other operations.

#### 2. AuthenticationFailed

```kotlin
InitializationResult.AuthenticationFailed(
    authError: Throwable,
    authCode: Int? = null
)
```

**Meaning:** The SDK was initialized successfully, but authentication failed.

**Possible Causes:**
- Invalid license key
- Network connectivity issues
- Server-side authentication errors
- Expired or revoked credentials

**Action:** 
- Check your license key
- Verify network connectivity
- Review the `authError` exception for specific details
- Contact support if the license key should be valid

#### 3. InitializationFailed

```kotlin
InitializationResult.InitializationFailed(error: Throwable)
```

**Meaning:** The SDK initialization process failed before authentication could be attempted.

**Possible Causes:**
- Missing required parameters
- Invalid Application context
- Internal SDK configuration errors
- Missing Android permissions

**Action:**
- Verify all required parameters are provided
- Check that the Application instance is valid
- Review the `error` exception for specific details
- Ensure your app has necessary Android permissions

### Exception Handling

All SDK methods may throw `IllegalStateException` if called before proper initialization:

```kotlin
try {
    val result = SafeConnection.search("+886912345678")
} catch (e: IllegalStateException) {
    when (e.message) {
        "SafeConnection is not initialized" -> {
            // Call initialization() first
        }
        "SafeConnection is not authenticated" -> {
            // Authentication failed or not completed
        }
    }
}
```

## Examples

### Complete Initialization and Search

```kotlin
import android.app.Application
import com.gogolook.safeconnection.SafeConnection
import com.gogolook.safeconnection.Environment
import com.gogolook.safeconnection.data.InitializationResult
import kotlinx.coroutines.GlobalScope
import kotlinx.coroutines.launch

class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        
        GlobalScope.launch {
            // Initialize the SDK
            val result = SafeConnection.initialization(
                app = this@MyApplication,
                memberId = null,
                licenseId = "your-license-key",
                environment = Environment.RELEASE
            )
            
            when (result) {
                is InitializationResult.InitializedAndAuthenticated -> {
                    // SDK is ready - perform a search
                    searchPhoneNumber("+886912345678")
                }
                is InitializationResult.AuthenticationFailed -> {
                    // Handle auth failure
                    println("Authentication failed: ${result.authError.message}")
                }
                is InitializationResult.InitializationFailed -> {
                    // Handle init failure
                    println("Initialization failed: ${result.error.message}")
                }
                else -> {}
            }
        }
    }
    
    private suspend fun searchPhoneNumber(phoneNumber: String) {
        try {
            val result = SafeConnection.search(phoneNumber)
            
            result.numberInfoList.forEach { info ->
                println("========================================")
                println("Number: ${info.number}")
                println("Name: ${info.name}")
                
                if (info.businessCategory.isNotEmpty()) {
                    println("Business: ${info.businessCategory}")
                }
                
                if (info.spamCategory.isNotEmpty()) {
                    println("⚠️ SPAM: ${info.spamCategory}")
                    println("Spam Level: ${info.spamLevel}")
                }
            }
        } catch (e: Exception) {
            println("Search failed: ${e.message}")
        }
    }
}
```

### Different Environment Configurations

```kotlin
// Development/Testing
val devResult = SafeConnection.initialization(
    app = application,
    licenseId = "dev-license-key",
    environment = Environment.SANDBOX
)

// Staging
val stagingResult = SafeConnection.initialization(
    app = application,
    licenseId = "staging-license-key",
    environment = Environment.STAGING
)

// Production
val prodResult = SafeConnection.initialization(
    app = application,
    licenseId = "prod-license-key",
    environment = Environment.RELEASE
)
```

### Handling Different Number Formats

```kotlin
// ✅ Correct - E.164 format (recommended)
SafeConnection.search("+886912345678")
SafeConnection.search("+14155552671")
SafeConnection.search("+447911123456")

// ✅ Also correct - Local format
SafeConnection.search("0912345678")
SafeConnection.search("4155552671")

// ❌ Incorrect - Contains separators
// SafeConnection.search("+886 912 345 678") // Has spaces
// SafeConnection.search("+886-912-345-678") // Has dashes
// SafeConnection.search("091-234-5678")     // Has dashes
```

### Working with Different Information Sources

```kotlin
// Search in all sources
val allResults = SafeConnection.search("+886912345678")

// Search only in offline database (faster, works offline)
val offlineResults = SafeConnection.search("+886912345678", InfoType.OFFLINE_DB)

// Search only via network (most up-to-date)
val networkResults = SafeConnection.search("+886912345678", InfoType.NETWORK)

// Search only in contacts
val contactResults = SafeConnection.search("+886912345678", InfoType.CONTACT)
```

---

## Additional Resources

- [GGA Implementation Guide](GGA_IMPLEMENTATION.md)
- [URL Checker Documentation](URL_CHECKER_README.md)

## Support

For issues, questions, or support, please contact Gogolook support or refer to your license agreement.
