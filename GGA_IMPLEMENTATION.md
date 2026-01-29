# GGA Call Log Upload Implementation

This implementation provides a complete architecture for uploading call logs to GGA (Global Graph Analytics) backend using MessagePack serialization format, as specified in the BSBU High Level Design Document.

## Architecture Overview

The implementation follows the existing SafeConnection SDK patterns and provides an extensible architecture that can accommodate future hitrate upload requirements.

## Configuration

The implementation uses the existing `Gateway` configuration for GGA URLs:
- Staging: `https://gga.staging.scamadvisor-sdk.com`
- Sandbox: `https://gga.sandbox.scamadvisor-sdk.com`
- Production: `https://gga.scamadvisor-sdk.com`

## Dependencies

The following dependency was added to support MessagePack:

```kotlin
implementation("org.msgpack:msgpack-core:0.9.8")
```

## Integration

The GGA module is designed to integrate seamlessly with the existing SafeConnection SDK:

1. Uses existing authentication system
2. Follows established repository and use case patterns  
3. Manual dependency injection in SafeConnection class (no Hilt required)
4. Uses existing logging and error handling patterns

### Usage Example

```kotlin
// Initialize SafeConnection as usual
SafeConnection.initialize(app, licenseId, Environment.STAGING)
SafeConnection.initAuth()

// Upload call logs to GGA
suspend fun uploadCallLogs(callLogs: List<CallLog>) {
    when (val result = SafeConnection.uploadCallLogsToGga(callLogs)) {
        is GgaUploadResult.Success -> {
            Logger.i("Uploaded ${result.response.processedCount} call logs")
        }
        is GgaUploadResult.Error -> {
            Logger.e("Upload failed: ${result.message}")
        }
        is GgaUploadResult.NetworkError -> {
            Logger.e("Network error: ${result.exception.message}")
        }
    }
}

data class CallLog(
    val id: Long, // android call log raw id
    val number: String,
    val type: CallType, // INCOMING, OUTGOING, MISSED, VOICEMAIL, REJECTED, BLOCKED, ANSWERED_EXTERNALLY, UNKNOWN
    val date: Long,
    val duration: Long,
    val region: String, // country
    val nrsName: String, // number search result name of NumberInfo
    val isNew: Boolean,
)
```

The implementation is ready for production use and can be extended to support additional GGA functionality as requirements evolve.
