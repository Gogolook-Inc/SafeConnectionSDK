# Block Feature API Documentation

## Overview

The Block feature provides comprehensive call blocking functionality within the SafeConnection SDK. It enables applications to automatically block unwanted calls based on spam detection, user-defined block lists, and configurable spam thresholds. The feature combines number lookup with intelligent spam evaluation to determine whether incoming calls should be blocked.

## Architecture

### Core Components

#### 1. Public API Entry Point
- **`SafeConnection`** - Main SDK object providing public-facing Block methods
  - Delegates to `CallBlockManager` for all blocking operations
  - Ensures proper initialization and authentication before operations

#### 2. Block Manager Layer
- **`IBlockManager`** - Interface defining the blocking contract
- **`CallBlockManager`** - Implementation orchestrating block logic
  - Coordinates between block list repository and spam evaluator
  - Applies business rules for blocking decisions

#### 3. Repository Layer
- **`IBlockListRepository`** - Interface for block list persistence
- **`DefaultBlockListRepository`** - Room-based implementation
  - Handles CRUD operations for blocked numbers
  - Uses `BlockListDatabase` for local storage
  - Normalizes phone numbers for consistent matching

#### 4. Spam Evaluation
- **`IBlockSpamEvaluator`** - Interface for spam detection logic
- **`DefaultBlockSpamEvaluator`** - Implementation with configurable threshold
  - Default spam threshold: 2 (spamLevel >= 2 triggers block)
  - Evaluates `NumberInfo.spamLevel` from search results

#### 5. Data Models
- **`IncomingEvent`** - Represents incoming call/SMS/VoIP events
- **`BlockedInfo`** - Represents blocked number entry
- **`BlockSource`** - Enum for communication channel types
- **`NumberInfo`** - Search result containing spam information

### Database Schema

**Table**: `blocked_info`

| Column | Type | Description |
|--------|------|-------------|
| `phoneNumber` | TEXT PRIMARY KEY | Normalized phone number |
| `blockSource` | TEXT | Source type (PHONE_CALL, SMS, VOIP) |
| `addTime` | INTEGER | Unix timestamp when added |

## Integration Guide

This section provides a step-by-step guide for integrating the Block feature into your application, based on the canonical usage pattern.

### Step 1: Initialize SafeConnection SDK

Before using any Block features, ensure the SDK is properly initialized and authenticated:

```kotlin
// Initialize SDK
val initResult = SafeConnection.initialization(
    app = application,
    memberId = "optional_member_id",
    licenseId = "your_license_id",
    environment = Environment.RELEASE
)

// Check initialization status
when (initResult) {
    is InitializationResult.InitializedAndAuthenticated -> {
        // Ready to use Block features
    }
    is InitializationResult.AuthenticationFailed -> {
        // Handle auth failure
    }
    is InitializationResult.InitializationFailed -> {
        // Handle init failure
    }
}
```

### Step 2: Standard Call Blocking Flow

The recommended workflow combines number search with block checking:

#### Complete Integration Example

```kotlin
import com.gogolook.safeconnection.SafeConnection
import com.gogolook.safeconnection.block.data.IncomingEvent
import com.gogolook.safeconnection.block.data.BlockSource
import kotlinx.coroutines.launch
import kotlinx.coroutines.flow.first

// Register call callback to intercept incoming calls
MyCallsCallbackManager.registerCallsCallback(object : CallsCallback {
    override fun onScreenCall(screenCallEvent: CallEvent.ScreenCall) {
        applicationScope.launch {
            // Step 1: Get user's spam blocking preference
            val spamOn = RepositoryProvider.provideSettingsRepository()
                .observeSpamEnabled()
                .first()
            
            // Step 2: Search for number information
            val numberInfoGroup = SafeConnection.search(screenCallEvent.number)
            
            // Step 3: Check if call should be blocked
            val shouldBlock = if (numberInfoGroup.numberInfoList.isNotEmpty()) {
                val numberInfo = numberInfoGroup.numberInfoList.firstOrNull()
                
                // Create IncomingEvent with search results
                numberInfo?.let {
                    val incomingEvent = IncomingEvent(
                        phoneNumber = it.number,
                        source = BlockSource.PHONE_CALL,
                        numberInfo = it
                    )
                    
                    // Evaluate block decision
                    SafeConnection.shouldBlockNumber(
                        incomingEvent = incomingEvent,
                        isAutoSpamOn = spamOn
                    )
                } ?: false
            } else {
                false
            }
            
            // Step 4: Take action based on block decision
            val callResponse = CustomCallResponse(
                disallowCall = shouldBlock,
                rejectCall = shouldBlock,
                skipNotification = shouldBlock,
                // ... other parameters
            )
            screenCallEvent.updateCall(callResponse)
        }
    }
    
    // ... other callback methods
})
```

#### Workflow Breakdown

**2.1. Search Number Information**

Before making a block decision, retrieve spam and business information:

```kotlin
val numberInfoGroup = SafeConnection.search(phoneNumber)
```

- **Purpose**: Obtains spam rating, business category, and other metadata
- **Returns**: `NumberInfoGroup` containing list of `NumberInfo` objects
- **Sources**: Contact, Network API, Offline DB (in priority order)

**2.2. Construct IncomingEvent**

Create an event object combining the number and search results:

```kotlin
val incomingEvent = IncomingEvent(
    phoneNumber = numberInfo.number,       // E.164 format recommended
    source = BlockSource.PHONE_CALL,       // PHONE_CALL, SMS, or VOIP
    numberInfo = numberInfo                 // Optional: from search result
)
```

- **phoneNumber**: The incoming number (E.164 format recommended for international numbers)
- **source**: Communication channel type
- **numberInfo**: Optional search result enabling spam evaluation

**2.3. Check Block Decision**

Evaluate whether to block the call:

```kotlin
val shouldBlock = SafeConnection.shouldBlockNumber(
    incomingEvent = incomingEvent,
    isAutoSpamOn = userSpamSetting
)
```

- **Returns**: `true` if number should be blocked, `false` otherwise
- **Logic**:
  1. First checks if number is in user's manual block list
  2. If `isAutoSpamOn = true` and `numberInfo` provided, evaluates spam level
  3. Default spam threshold: `spamLevel >= 2`

**2.4. Handle Block Decision**

Take appropriate action based on the result:

```kotlin
if (shouldBlock) {
    // Reject the call
    callEvent.updateCall(
        CustomCallResponse(
            disallowCall = true,
            rejectCall = true,
            skipNotification = true
        )
    )
} else {
    // Allow the call to proceed normally
    callEvent.updateCall(CustomCallResponse())
}
```

### Step 3: Manual Block List Management

Users can manually add/remove numbers from their block list:

#### Add Number to Block List

```kotlin
suspend fun blockNumber(phoneNumber: String) {
    SafeConnection.addBlockedNumber(phoneNumber)
}
```

#### Remove Number from Block List

```kotlin
suspend fun unblockNumber(phoneNumber: String) {
    SafeConnection.removeBlockedNumber(phoneNumber)
}
```

#### Check if Number is Blocked

```kotlin
suspend fun isBlocked(phoneNumber: String): Boolean {
    return SafeConnection.isNumberBlocked(phoneNumber)
}
```

#### Observe Block List Changes

```kotlin
import kotlinx.coroutines.flow.collect

lifecycleScope.launch {
    SafeConnection.getBlockedList().collect { blockedList ->
        // Update UI with current block list
        updateBlockListUI(blockedList)
    }
}
```

## API Reference

### Public Methods (SafeConnection)

All Block-related methods are exposed through the `SafeConnection` object. These are convenience methods that delegate to the internal `CallBlockManager`.

#### shouldBlockNumber

Determines if an incoming event should be blocked based on block list and spam evaluation.

```kotlin
@JvmStatic
suspend fun shouldBlockNumber(
    incomingEvent: IncomingEvent,
    isAutoSpamOn: Boolean
): Boolean
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `incomingEvent` | `IncomingEvent` | Yes | Event to evaluate containing number and optional spam info |
| `isAutoSpamOn` | `Boolean` | Yes | User setting for automatic spam blocking |

**Returns:**
- `Boolean` - `true` if should block, `false` otherwise

**Throws:**
- `IllegalStateException` - If SDK not initialized or not authenticated

**Business Logic:**
1. Returns `true` immediately if number exists in manual block list
2. If `isAutoSpamOn = true` and `numberInfo` provided:
   - Evaluates `numberInfo.spamLevel >= threshold` (default threshold: 2)
   - Returns `true` if spam threshold exceeded
3. Returns `false` otherwise

**Example:**
```kotlin
val incomingEvent = IncomingEvent(
    phoneNumber = "+886912345678",
    source = BlockSource.PHONE_CALL,
    numberInfo = searchResult
)

val shouldBlock = SafeConnection.shouldBlockNumber(
    incomingEvent = incomingEvent,
    isAutoSpamOn = true
)
```

---

#### addBlockedNumber

Adds a phone number to the user's manual block list.

```kotlin
@JvmStatic
suspend fun addBlockedNumber(phoneNumber: String)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `phoneNumber` | `String` | Yes | Phone number to block (any format, will be normalized) |

**Returns:**
- `Unit` - No return value

**Behavior:**
- Normalizes phone number before storing (removes spaces, dashes, etc.)
- If number already exists, operation is idempotent (no error)
- Timestamp is automatically set to current system time
- Block source is set to `PHONE_CALL` by default

**Example:**
```kotlin
SafeConnection.addBlockedNumber("+886-912-345-678")
// Stored as normalized: "+886912345678"
```

---

#### removeBlockedNumber

Removes a phone number from the user's manual block list.

```kotlin
@JvmStatic
suspend fun removeBlockedNumber(phoneNumber: String)
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `phoneNumber` | `String` | Yes | Phone number to unblock (any format) |

**Returns:**
- `Unit` - No return value

**Behavior:**
- Normalizes phone number before lookup
- If number doesn't exist in list, operation is idempotent (no error)
- Removes entry from local database

**Example:**
```kotlin
SafeConnection.removeBlockedNumber("+886912345678")
```

---

#### isNumberBlocked

Checks if a phone number is currently in the block list.

```kotlin
@JvmStatic
suspend fun isNumberBlocked(phoneNumber: String): Boolean
```

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `phoneNumber` | `String` | Yes | Phone number to check |

**Returns:**
- `Boolean` - `true` if blocked, `false` otherwise

**Behavior:**
- Normalizes phone number before lookup
- Only checks manual block list (does not evaluate spam)
- Fast database query with indexed lookup

**Example:**
```kotlin
val isBlocked = SafeConnection.isNumberBlocked("+886912345678")
if (isBlocked) {
    // Show "Unblock" button
} else {
    // Show "Block" button
}
```

---

#### getBlockedList

Returns a Flow of the current blocked numbers list that updates automatically when changes occur.

```kotlin
@JvmStatic
fun getBlockedList(): Flow<List<BlockedInfo>>
```

**Parameters:**
- None

**Returns:**
- `Flow<List<BlockedInfo>>` - Reactive stream of blocked numbers

**Behavior:**
- Returns Flow that emits whenever block list changes
- Filters to `PHONE_CALL` source only
- List is ordered by `addTime` (most recent first)
- Flow is cold - starts observation when collected

**Example:**
```kotlin
// Collect in coroutine
lifecycleScope.launch {
    SafeConnection.getBlockedList().collect { blockedList ->
        adapter.submitList(blockedList)
    }
}

// Or get single value
val currentList = SafeConnection.getBlockedList().first()
```

---

## Data Structures

### IncomingEvent

Represents an incoming communication event (call, SMS, VoIP) to be evaluated for blocking.

```kotlin
data class IncomingEvent(
    val phoneNumber: String,
    val source: BlockSource,
    val numberInfo: NumberInfo? = null
)
```

**Fields:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `phoneNumber` | `String` | Yes | The incoming number (E.164 format recommended) |
| `source` | `BlockSource` | Yes | Communication channel type |
| `numberInfo` | `NumberInfo?` | No | Optional search result for spam evaluation |

**Usage Notes:**
- **phoneNumber**: While any format is accepted, E.164 format (`+[country][number]`) is recommended for international numbers
- **source**: Must match the communication type for accurate blocking
- **numberInfo**: Required for automatic spam evaluation; obtain via `SafeConnection.search()`

**Example:**
```kotlin
val event = IncomingEvent(
    phoneNumber = "+886912345678",
    source = BlockSource.PHONE_CALL,
    numberInfo = SafeConnection.search("+886912345678").numberInfoList.firstOrNull()
)
```

---

### BlockedInfo

Represents a blocked number entry in the user's manual block list.

```kotlin
data class BlockedInfo(
    val phoneNumber: String,
    val blockSource: BlockSource,
    val addTime: Long
)
```

**Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `phoneNumber` | `String` | Normalized phone number |
| `blockSource` | `BlockSource` | Source type: PHONE_CALL, SMS, or VOIP |
| `addTime` | `Long` | Unix timestamp (milliseconds) when added to block list |

**Usage Notes:**
- **phoneNumber**: Automatically normalized by repository (spaces/dashes removed)
- **addTime**: Set automatically when calling `addBlockedNumber()`
- Used primarily for displaying block list in UI

**Example:**
```kotlin
SafeConnection.getBlockedList().first().forEach { blockedInfo ->
    println("Blocked: ${blockedInfo.phoneNumber}")
    println("Since: ${Date(blockedInfo.addTime)}")
}
```

---

### BlockSource (Enum)

Defines the communication channel types that can be blocked.

```kotlin
enum class BlockSource {
    PHONE_CALL,
    SMS,
    VOIP
}
```

**Values:**

| Value | Description | Usage |
|-------|-------------|-------|
| `PHONE_CALL` | Traditional phone calls | Most common - use for standard call blocking |
| `SMS` | Text messages | For SMS blocking features |
| `VOIP` | Voice over IP calls | For internet-based call blocking (e.g., WhatsApp, Telegram) |

**Current Support:**
- The SDK currently focuses on `PHONE_CALL` blocking
- `SMS` and `VOIP` are supported in data structures for future extension
- All public methods default to `PHONE_CALL` source

**Example:**
```kotlin
val event = IncomingEvent(
    phoneNumber = number,
    source = BlockSource.PHONE_CALL,
    numberInfo = searchResult
)
```

---

### NumberInfo

Contains spam and business information for a phone number, obtained from `SafeConnection.search()`.

```kotlin
data class NumberInfo(
    val number: String,
    val name: String,
    val businessCategory: String,
    val spamCategory: String,
    val spamLevel: Int
)
```

**Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `number` | `String` | The phone number |
| `name` | `String` | Contact name or business name (empty if unknown) |
| `businessCategory` | `String` | Business type (e.g., "Restaurant", "Bank") |
| `spamCategory` | `String` | Spam classification (e.g., "Scam", "Telemarketing") |
| `spamLevel` | `Int` | Spam severity: 0 = safe, 1 = suspicious, 2+ = spam |

**Spam Level Values:**

| Level | Meaning | Block Behavior (Default) |
|-------|---------|--------------------------|
| 0 | Safe - verified legitimate number | Not blocked |
| 1 | Suspicious - potential spam | Not blocked |
| 2+ | Spam - confirmed unwanted calls | Blocked if `isAutoSpamOn = true` |

**Usage in Blocking:**
- The `spamLevel` field is the primary input for automatic spam blocking
- Default threshold is 2 (configurable in `DefaultBlockSpamEvaluator`)
- Must be provided in `IncomingEvent.numberInfo` for spam evaluation

**Example:**
```kotlin
val numberInfoGroup = SafeConnection.search("+886912345678")
val numberInfo = numberInfoGroup.numberInfoList.firstOrNull()

numberInfo?.let {
    println("Name: ${it.name}")
    println("Business: ${it.businessCategory}")
    println("Spam Level: ${it.spamLevel}")
    
    if (it.spamLevel >= 2) {
        println("⚠️ This is a spam number!")
    }
}
```

---

### NumberInfoGroup

Wrapper containing search results from `SafeConnection.search()`.

```kotlin
data class NumberInfoGroup(
    val infoType: InfoType,
    val numberInfoList: List<NumberInfo>
)
```

**Fields:**

| Field | Type | Description |
|-------|------|-------------|
| `infoType` | `InfoType` | Source of information (CONTACT, NETWORK, OFFLINE_DB) |
| `numberInfoList` | `List<NumberInfo>` | List of number info results (usually 0 or 1 entry) |

**InfoType Values:**
- `CONTACT` - Found in device contacts
- `NETWORK` - Retrieved from API
- `OFFLINE_DB` - Found in offline database
- `NONE` - No information found

**Example:**
```kotlin
val result = SafeConnection.search(phoneNumber)
when (result.infoType) {
    InfoType.CONTACT -> println("Known contact")
    InfoType.NETWORK -> println("Spam database match")
    InfoType.OFFLINE_DB -> println("Offline database match")
    InfoType.NONE -> println("Unknown number")
}
```

---

## Usage Constraints

### Prerequisites

#### 1. SDK Initialization Required

All Block methods require the SDK to be initialized:

```kotlin
if (!SafeConnection.isInitialized) {
    throw IllegalStateException("SafeConnection is not initialized")
}
```

**How to Initialize:**
```kotlin
SafeConnection.initialization(
    app = application,
    licenseId = "your_license_id",
    environment = Environment.RELEASE
)
```

#### 2. Authentication Required

Block features require active authentication:

```kotlin
if (!SafeConnection.isAuthInitialized) {
    throw IllegalStateException("SafeConnection is not authenticated")
}
```

**Checking Status:**
```kotlin
val isReady = SafeConnection.isInitialized && SafeConnection.isAuthInitialized
if (isReady) {
    // Safe to use Block features
}
```

### Phone Number Format

#### Recommended Format: E.164

While the SDK accepts various phone number formats, **E.164 is strongly recommended** for international numbers:

```kotlin
// ✅ Recommended - E.164 format
"+886912345678"   // Taiwan
"+14155552671"    // USA
"+442071838750"   // UK

// ⚠️ Acceptable but may cause issues
"0912-345-678"    // Local format (country-dependent)
"(02) 1234-5678"  // Formatted local
```

**E.164 Format Rules:**
- Starts with `+` (plus sign)
- Followed by country code (1-3 digits)
- Followed by national number (up to 15 digits total)
- No spaces, dashes, or parentheses

#### Number Normalization

The SDK automatically normalizes numbers before storage:

```kotlin
// All these are normalized to the same value
SafeConnection.addBlockedNumber("+886-912-345-678")
SafeConnection.addBlockedNumber("+886 912 345 678")
SafeConnection.addBlockedNumber("+886912345678")
// All stored as: "+886912345678"
```

**Normalization Process:**
- Removes spaces, dashes, parentheses
- Preserves `+` prefix and digits
- Uses `PhoneNumberUtils.normalizeNumber()` internally

### Spam Evaluation Requirements

For automatic spam blocking to work, both conditions must be met:

#### 1. User Setting Enabled

```kotlin
val isAutoSpamOn = true  // User has enabled auto-spam blocking
```

#### 2. NumberInfo Provided

```kotlin
// ❌ Won't evaluate spam (numberInfo is null)
val event1 = IncomingEvent(
    phoneNumber = number,
    source = BlockSource.PHONE_CALL,
    numberInfo = null
)

// ✅ Can evaluate spam
val searchResult = SafeConnection.search(number)
val event2 = IncomingEvent(
    phoneNumber = number,
    source = BlockSource.PHONE_CALL,
    numberInfo = searchResult.numberInfoList.firstOrNull()
)
```

**Best Practice:**
```kotlin
// Always search before checking block
val numberInfoGroup = SafeConnection.search(phoneNumber)
val numberInfo = numberInfoGroup.numberInfoList.firstOrNull()

val shouldBlock = SafeConnection.shouldBlockNumber(
    IncomingEvent(phoneNumber, BlockSource.PHONE_CALL, numberInfo),
    isAutoSpamOn = userSetting
)
```

### Coroutine Context

All suspend functions should be called from appropriate coroutine scopes:

```kotlin
// ✅ Correct - Using appropriate scope
lifecycleScope.launch {
    SafeConnection.addBlockedNumber(number)
}

applicationScope.launch {
    val isBlocked = SafeConnection.isNumberBlocked(number)
}

// ❌ Wrong - Blocking main thread
runBlocking {
    SafeConnection.addBlockedNumber(number)  // Don't do this on main thread
}
```

**Recommended Scopes:**
- `lifecycleScope` - For UI-related operations
- `viewModelScope` - For ViewModel operations
- `applicationScope` - For background operations

### Threading

- All suspend methods use `Dispatchers.IO` internally for database operations
- Flow collection should be done on Main dispatcher for UI updates:

```kotlin
SafeConnection.getBlockedList()
    .flowOn(Dispatchers.IO)           // Data operations on IO
    .collect {                          // Collection on Main (default)
        updateUI(it)
    }
```

### Error Handling

Implement proper error handling for all operations:

```kotlin
try {
    SafeConnection.addBlockedNumber(phoneNumber)
} catch (e: IllegalStateException) {
    // SDK not initialized or not authenticated
    showError("Please initialize SDK first")
} catch (e: Exception) {
    // Database or other errors
    showError("Failed to block number: ${e.message}")
}
```

## Configuration

### Spam Threshold

The default spam threshold is **2**, meaning numbers with `spamLevel >= 2` are considered spam.

This is configured in `SafeConnectionDependencyFactory`:

```kotlin
private fun provideBlockSpamEvaluator(): IBlockSpamEvaluator {
    return DefaultBlockSpamEvaluator(threshold = 2)
}
```

**To customize** (requires SDK source modification):
```kotlin
// In SafeConnectionDependencyFactory.kt
DefaultBlockSpamEvaluator(threshold = 3)  // More lenient
DefaultBlockSpamEvaluator(threshold = 1)  // More aggressive
```

### Database Location

Block list database is stored at:
```
{app_data_dir}/databases/block_list.db
```

Managed automatically by Room database.

## Best Practices

### 1. Always Search Before Blocking

Combine search with block checking for optimal results:

```kotlin
// ✅ Recommended pattern
val searchResult = SafeConnection.search(phoneNumber)
val numberInfo = searchResult.numberInfoList.firstOrNull()
val event = IncomingEvent(phoneNumber, BlockSource.PHONE_CALL, numberInfo)
val shouldBlock = SafeConnection.shouldBlockNumber(event, isAutoSpamOn)

// ❌ Less effective (no spam evaluation)
val event = IncomingEvent(phoneNumber, BlockSource.PHONE_CALL, null)
val shouldBlock = SafeConnection.shouldBlockNumber(event, false)
```

### 2. Cache Search Results

Avoid redundant searches by caching results:

```kotlin
private val searchCache = mutableMapOf<String, NumberInfo?>()

suspend fun shouldBlockWithCache(phoneNumber: String): Boolean {
    val numberInfo = searchCache.getOrPut(phoneNumber) {
        SafeConnection.search(phoneNumber).numberInfoList.firstOrNull()
    }
    
    val event = IncomingEvent(phoneNumber, BlockSource.PHONE_CALL, numberInfo)
    return SafeConnection.shouldBlockNumber(event, isAutoSpamOn)
}
```

### 3. Observe Block List Changes

Use Flow for reactive UI updates:

```kotlin
class BlockListViewModel : ViewModel() {
    val blockedNumbers: StateFlow<List<BlockedInfo>> = 
        SafeConnection.getBlockedList()
            .stateIn(
                scope = viewModelScope,
                started = SharingStarted.WhileSubscribed(5000),
                initialValue = emptyList()
            )
}
```

### 4. Provide User Feedback

Always inform users about block actions:

```kotlin
suspend fun blockNumberWithFeedback(phoneNumber: String) {
    try {
        SafeConnection.addBlockedNumber(phoneNumber)
        showToast("Number blocked successfully")
    } catch (e: Exception) {
        showToast("Failed to block number: ${e.message}")
    }
}
```

### 5. Handle Edge Cases

```kotlin
suspend fun safeBlockCheck(phoneNumber: String?): Boolean {
    // Check for null or empty
    if (phoneNumber.isNullOrBlank()) return false
    
    // Check SDK status
    if (!SafeConnection.isInitialized || !SafeConnection.isAuthInitialized) {
        return false
    }
    
    // Perform actual check
    return try {
        val searchResult = SafeConnection.search(phoneNumber)
        val numberInfo = searchResult.numberInfoList.firstOrNull()
        val event = IncomingEvent(phoneNumber, BlockSource.PHONE_CALL, numberInfo)
        SafeConnection.shouldBlockNumber(event, isAutoSpamOn = true)
    } catch (e: Exception) {
        Log.e(TAG, "Block check failed", e)
        false  // Fail safe - don't block on error
    }
}
```

## Performance Considerations

### Database Queries

- Block list queries are indexed for O(1) lookup performance
- Phone numbers are normalized and indexed
- Recommended for lists up to 10,000 entries

### Search Performance

- Network searches may take 200-500ms
- Offline DB searches are faster (~50-100ms)
- Cache search results to avoid redundant API calls

### Memory Usage

- Block list is loaded into memory via Flow
- Each `BlockedInfo` entry is ~100 bytes
- 1,000 blocked numbers ≈ 100KB memory

## Troubleshooting

### Common Issues

#### Issue: IllegalStateException on method call

**Cause:** SDK not initialized or not authenticated

**Solution:**
```kotlin
// Check status before operations
if (SafeConnection.isInitialized && SafeConnection.isAuthInitialized) {
    SafeConnection.addBlockedNumber(phoneNumber)
} else {
    // Initialize first
    SafeConnection.initialization(app, licenseId = "...", ...)
}
```

---

#### Issue: Number not blocked despite being in list

**Cause:** Phone number format mismatch

**Solution:**
```kotlin
// Use E.164 format consistently
val normalizedNumber = "+886912345678"  // Not "0912-345-678"
SafeConnection.addBlockedNumber(normalizedNumber)
```

---

#### Issue: Spam numbers not auto-blocked

**Cause:** Missing `numberInfo` or `isAutoSpamOn = false`

**Solution:**
```kotlin
// Ensure both conditions are met
val searchResult = SafeConnection.search(phoneNumber)
val numberInfo = searchResult.numberInfoList.firstOrNull()  // Must not be null

val event = IncomingEvent(phoneNumber, BlockSource.PHONE_CALL, numberInfo)
SafeConnection.shouldBlockNumber(event, isAutoSpamOn = true)  // Must be true
```

---

#### Issue: Flow not updating UI

**Cause:** Not collecting Flow or collected in wrong scope

**Solution:**
```kotlin
// Collect in lifecycle-aware scope
lifecycleScope.launch {
    SafeConnection.getBlockedList().collect { list ->
        updateUI(list)  // Will update automatically on changes
    }
}
```

## Testing

### Unit Testing Block Logic

```kotlin
@Test
fun `shouldBlockNumber returns true for manually blocked number`() = runTest {
    // Given
    SafeConnection.addBlockedNumber("+886912345678")
    
    // When
    val event = IncomingEvent("+886912345678", BlockSource.PHONE_CALL, null)
    val result = SafeConnection.shouldBlockNumber(event, isAutoSpamOn = false)
    
    // Then
    assertTrue(result)
}

@Test
fun `shouldBlockNumber returns true for high spam level when auto-spam on`() = runTest {
    // Given
    val numberInfo = NumberInfo(
        number = "+886912345678",
        name = "Spam Caller",
        businessCategory = "",
        spamCategory = "Telemarketing",
        spamLevel = 3
    )
    
    // When
    val event = IncomingEvent("+886912345678", BlockSource.PHONE_CALL, numberInfo)
    val result = SafeConnection.shouldBlockNumber(event, isAutoSpamOn = true)
    
    // Then
    assertTrue(result)
}
```

### Integration Testing

```kotlin
@Test
fun `full blocking flow - search, evaluate, and block`() = runTest {
    // Given: A known spam number
    val phoneNumber = "+886912345678"
    
    // When: Search for information
    val searchResult = SafeConnection.search(phoneNumber)
    val numberInfo = searchResult.numberInfoList.firstOrNull()
    
    // Then: Should have spam information
    assertNotNull(numberInfo)
    assertTrue(numberInfo!!.spamLevel >= 2)
    
    // When: Check if should block
    val event = IncomingEvent(phoneNumber, BlockSource.PHONE_CALL, numberInfo)
    val shouldBlock = SafeConnection.shouldBlockNumber(event, isAutoSpamOn = true)
    
    // Then: Should be blocked
    assertTrue(shouldBlock)
}
```

## Related Documentation

- **Number Search API**: For obtaining `NumberInfo` required for spam evaluation
- **GGA Integration**: For uploading call log data including block events
- **URL Checker**: For complementary SMS spam detection via URL scanning

## Summary

The Block feature provides a comprehensive solution for call blocking with:

- ✅ **Dual blocking modes**: Manual block list + automatic spam detection
- ✅ **Intelligent spam evaluation**: Configurable threshold with NumberInfo integration
- ✅ **Reactive updates**: Flow-based block list observation
- ✅ **Format flexibility**: Accepts various number formats with automatic normalization
- ✅ **Type safety**: Strong Kotlin types with sealed classes
- ✅ **Performance**: Indexed database queries with efficient caching
- ✅ **Extensibility**: Support for multiple source types (CALL, SMS, VOIP)

For implementation questions or issues, refer to the canonical example in `MyApplication.kt` or contact the SafeConnection SDK team.
