# Block Feature API Documentation

## Overview

The Block feature provides call blocking functionality within the SafeConnection SDK. It enables applications to automatically block unwanted calls based on spam detection and user-defined block lists. The feature combines number lookup with spam evaluation to determine whether incoming calls should be blocked.

## Prerequisites

Before using the Block feature, ensure the SafeConnection SDK is properly initialized and authenticated.

### SDK Initialization

```kotlin
import com.gogolook.safeconnection.SafeConnection
import com.gogolook.safeconnection.Environment

// Initialize and authenticate in one call
val initResult = SafeConnection.initialization(
    app = application,
    memberId = "optional_member_id",  // Optional: your user ID
    licenseId = "your_license_id",     // Required: your SDK license
    environment = Environment.RELEASE   // RELEASE, STAGING, or SANDBOX
)

// Check initialization status
when (initResult) {
    is InitializationResult.InitializedAndAuthenticated -> {
        // SDK is ready to use
    }
    is InitializationResult.AuthenticationFailed -> {
        // Handle authentication failure
    }
    is InitializationResult.InitializationFailed -> {
        // Handle initialization failure
    }
}
```

**Note**: All Block feature methods require `SafeConnection.isInitialized` and `SafeConnection.isAuthInitialized` to be `true`. If not initialized, methods will throw `IllegalStateException`.

## API Reference

### Public Methods

All Block-related methods are exposed through the `SafeConnection` object.

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

Represents an incoming call event to be evaluated for blocking.

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
| `source` | `BlockSource` | Yes | Always use `BlockSource.PHONE_CALL` (only supported value currently) |
| `numberInfo` | `NumberInfo?` | No | Optional search result for spam evaluation |

**Usage Notes:**
- **phoneNumber**: While any format is accepted, E.164 format (`+[country][number]`) is recommended for international numbers
- **source**: Currently only `BlockSource.PHONE_CALL` is supported. Always use this value.
- **numberInfo**: Recommended for automatic spam evaluation; obtain via `SafeConnection.search()`

**Example:**
```kotlin
val event = IncomingEvent(
    phoneNumber = "+886912345678",
    source = BlockSource.PHONE_CALL,  // Always use PHONE_CALL
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

Defines the communication channel types.

```kotlin
enum class BlockSource {
    PHONE_CALL,
    SMS,
    VOIP
}
```

**Current Support:**
- **Only `PHONE_CALL` is currently supported** for the Block feature
- Always use `BlockSource.PHONE_CALL` when creating `IncomingEvent` objects
- `SMS` and `VOIP` values exist for future functionality but are not yet supported

**Example:**
```kotlin
val event = IncomingEvent(
    phoneNumber = number,
    source = BlockSource.PHONE_CALL,  // Always use PHONE_CALL
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
| `businessCategory` | `String` | Business type - see Business Categories table below |
| `spamCategory` | `String` | Spam classification - see Spam Categories table below |
| `spamLevel` | `Int` | Spam severity: 0 = safe, 1 = suspicious, 2+ = spam |

**Business Categories:**

| Category | Description |
|----------|-------------|
| `publicperson` | Individuals known for their public influence or celebrity status |
| `food` | Establishments offering prepared food services, groceries, or beverages |
| `shopping` | Retail and wholesale outlets selling goods ranging from apparel to electronics |
| `beauty` | Services and products related to personal care, aesthetics, and wellness |
| `education` | Institutions and services offering learning experiences and certifications |
| `entertainment` | Activities and venues providing recreational and cultural experiences |
| `life` | Services aimed at improving daily living through personal and home care |
| `health` | Medical, wellness, and fitness services and institutions for physical and mental health |
| `travel` | Services related to the planning and facilitation of travel and accommodations |
| `automobile` | Businesses involved in the sale, maintenance, and repair of vehicles |
| `traffic` | Services and solutions for managing and facilitating the flow of vehicles and pedestrians |
| `professional` | Professionals offering specialized skills and services in various fields |
| `bank` | Financial institutions offering monetary transactions, loans, and financial advice |
| `activity` | Organized recreational or educational events and programs |
| `government` | Public sector entities providing civic services and administration |
| `politics` | Activities associated with governance, campaigning, and political consultation |
| `organization` | Entities organized for a collective purpose, both non-profit and for-profit |
| `pet` | Services and products related to the care and management of pets |
| `logistic` | The management and coordination of the transportation, warehousing, and distribution of goods |
| `media` | Platforms and channels for the distribution of information, news, and entertainment |
| `others` | Categories that do not fit into the other defined areas, including miscellaneous services |

**Spam Categories:**

| Category | Description |
|----------|-------------|
| `FRAUD` | The number is used for any type of scam including investment scam, delivery scam, job scam, impersonation scam, etc. |
| `HARASSMENT` | The user of this number has some harassing behaviors including hanging up the call immediately after the pick-up, no response after the phone is picked up, etc. Sometimes the poll is also reported as "HARASSMENT" by the end users. |
| `TELEMARKETING` | The number is used for any type of telemarketing including but not limited to insurance telemarketing, loan telemarketing, etc. |
| `HFB` | HFB means High Frequency Block. Whoscall provides the option "OTHERS" in its number report interface. If "OTHERS" is the most-reported, the number will be labelled as "HFB". |

**Spam Level Values:**

| Level | Meaning | Block Behavior (Default) |
|-------|---------|--------------------------|
| 0 | Safe - verified legitimate number | Not blocked |
| 1 | Suspicious - potential spam | Not blocked |
| 2+ | Spam - confirmed unwanted calls | Blocked if `isAutoSpamOn = true` |

**Usage in Blocking:**
- The `spamLevel` field is the primary input for automatic spam blocking
- Default threshold is 2 (numbers with `spamLevel >= 2` are considered spam)
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

## Summary

The Block feature provides call blocking functionality with:

- **Dual blocking modes**: Manual block list + automatic spam detection (threshold: spamLevel >= 2)
- **Simple integration**: Initialize SDK → Search number → Check block → Handle result
- **Reactive updates**: Flow-based block list observation for UI
- **Format flexibility**: Accepts various number formats with automatic normalization

For questions or support, contact the SafeConnection SDK team.
