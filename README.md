# health_bg_sync

A Flutter plugin for syncing HealthKit data from iOS devices to your backend server, with support for background sync and incremental updates.

## Features

- ✅ Automatic background synchronization of HealthKit data
- ✅ Incremental sync using HealthKit anchors (only new/changed data)
- ✅ Full export on first sync
- ✅ Chunked uploads to prevent timeouts and HTTP 413 errors
- ✅ Background delivery when health data changes
- ✅ Configurable chunk sizes for optimal performance
- ✅ Supports all HealthKit sample types (quantities, categories, workouts, correlations)

## Installation

Add this to your `pubspec.yaml`:

```yaml
dependencies:
  health_bg_sync: ^1.0.0
```

Then run:

```bash
flutter pub get
```

## iOS Setup

### 1. Add Background Modes

In your `ios/Runner/Info.plist`, add:

```xml
<key>UIBackgroundModes</key>
<array>
    <string>background-processing</string>
    <string>background-fetch</string>
    <string>remote-notification</string>
</array>

<key>BGTaskSchedulerPermittedIdentifiers</key>
<array>
    <string>com.healthbgsync.task.refresh</string>
    <string>com.healthbgsync.task.process</string>
</array>
```

### 2. Add HealthKit Capability

1. Open your project in Xcode
2. Go to **Signing & Capabilities**
3. Click **+ Capability**
4. Add **HealthKit**

### 3. Add HealthKit Usage Description

In your `ios/Runner/Info.plist`:

```xml
<key>NSHealthShareUsageDescription</key>
<string>This app needs access to your health data to sync it with your account.</string>
```

## Usage

### 1. Initialize the Plugin

```dart
import 'package:health_bg_sync/health_bg_sync.dart';
import 'package:health_bg_sync/health_data_type.dart';

await HealthBgSync.initialize(
  endpoint: 'https://your-api.com/health/sync',
  token: 'your-auth-token',
  types: [
    HealthDataType.steps,
    HealthDataType.heartRate,
    HealthDataType.activeEnergyBurned,
    // Add more types as needed
  ],
  chunkSize: 1000, // Optional: default is 1000
  recordsPerChunk: 10000, // Optional: default is 10000 (~2-3MB per request)
  listenToLogs: true, // Optional: automatically print logs to console (default: true)
);
```

**Parameters:**
- `endpoint` (required): Your backend API endpoint URL
- `token` (required): Authentication token (sent as `Authorization: Bearer {token}`)
- `types` (required): List of health data types to sync
- `chunkSize` (optional): Internal chunk size for processing (default: 1000)
- `recordsPerChunk` (optional): Maximum records per HTTP request to prevent timeouts (default: 10000)
- `listenToLogs` (optional): Automatically listen to and print logs from native plugin (default: true)

### 2. Request Authorization

Request HealthKit read permissions:

```dart
bool authorized = await HealthBgSync.requestAuthorization();
if (authorized) {
  print('HealthKit authorization granted');
} else {
  print('HealthKit authorization denied');
}
```

**Returns:** `true` if authorization was granted, `false` otherwise.

### 3. Start Background Sync

Start the background sync process. This will:
- Register observer queries for all configured types
- Perform a full export on first sync (if not already done)
- Perform incremental syncs for subsequent runs
- Schedule background tasks for catch-up syncing

```dart
bool started = await HealthBgSync.startBackgroundSync();
if (started) {
  print('Background sync started successfully');
} else {
  print('Failed to start sync - check HealthKit availability and configuration');
}
```

**Returns:** `true` if sync started successfully, `false` if:
- HealthKit is not available on the device
- Endpoint or token is not configured
- No health data types are being tracked

**Important:** This method returns immediately with the result. The actual sync happens asynchronously in the background.

### 4. Manual Sync

Trigger a manual incremental sync (uses existing anchors):

```dart
await HealthBgSync.syncNow();
```

### 5. Stop Background Sync

Stop all background observers and cancel scheduled tasks:

```dart
await HealthBgSync.stopBackgroundSync();
```

### 6. Reset Anchors

Reset all anchors for the current endpoint (forces full export on next sync):

```dart
await HealthBgSync.resetAnchors();
```

**Note:** Logs are automatically printed to the console by default when `listenToLogs: true` (default) is set in `initialize()`. Logs include:
- Sync progress and status updates
- Data collection and query results
- Upload progress and HTTP responses
- Error messages and warnings

## Complete Example

```dart
import 'package:health_bg_sync/health_bg_sync.dart';
import 'package:health_bg_sync/health_data_type.dart';

class HealthSyncService {
  Future<void> setupSync() async {
    try {
      // 1. Initialize
      await HealthBgSync.initialize(
        endpoint: 'https://api.example.com/health/sync',
        token: 'your-auth-token',
        types: [
          HealthDataType.steps,
          HealthDataType.heartRate,
          HealthDataType.distanceWalkingRunning,
          HealthDataType.activeEnergyBurned,
        ],
        recordsPerChunk: 10000,
      );

      // 2. Request authorization
      bool authorized = await HealthBgSync.requestAuthorization();
      if (!authorized) {
        print('User denied HealthKit access');
        return;
      }

      // 3. Start background sync
      bool started = await HealthBgSync.startBackgroundSync();
      if (started) {
        print('✅ Background sync started successfully!');
      } else {
        print('❌ Failed to start background sync');
      }
      
      // Note: Logs are automatically printed to console (enabled by default in initialize)
    } catch (e) {
      print('Error setting up sync: $e');
    }
  }
}
```

## API Reference

### `initialize()`

Initializes the plugin with configuration.

```dart
Future<void> initialize({
  required String endpoint,
  required String token,
  required List<HealthDataType> types,
  int chunkSize = 1000,
  int recordsPerChunk = 10000,
  bool listenToLogs = true,
})
```

### `requestAuthorization()`

Requests HealthKit read permissions for configured types.

```dart
Future<bool> requestAuthorization()
```

**Returns:** `true` if authorization granted, `false` otherwise.

### `startBackgroundSync()`

Starts background sync process.

```dart
Future<bool> startBackgroundSync()
```

**Returns:** `true` if sync started successfully, `false` if prerequisites are not met.

### `syncNow()`

Manually triggers an incremental sync.

```dart
Future<void> syncNow()
```

### `stopBackgroundSync()`

Stops all background sync operations.

```dart
Future<void> stopBackgroundSync()
```

### `resetAnchors()`

Resets all anchors for the current endpoint.

```dart
Future<void> resetAnchors()
```

## How It Works

### Full Export vs Incremental Sync

- **First Sync (Full Export):** On the first sync for a given endpoint, all available health data is exported.
- **Incremental Sync:** Subsequent syncs only send new or changed data since the last sync, using HealthKit anchors.

### Chunking

Large datasets are automatically split into chunks to:
- Prevent HTTP 413 "Payload Too Large" errors
- Prevent request timeouts
- Optimize memory usage

Data is sent in chunks sequentially, with each chunk containing up to `recordsPerChunk` records (default: 10,000).

### Background Delivery

The plugin uses HealthKit's observer queries to automatically detect when new health data is available and sync it in the background, even when the app is not running.

### Background Tasks

iOS background tasks are scheduled as fallbacks to ensure data is synced even if:
- The device is in low-power mode
- Background delivery is temporarily disabled
- The app hasn't been opened recently

## Data Format

The plugin sends data to your endpoint as a JSON POST request:

```json
{
  "data": {
    "HKQuantityTypeIdentifierStepCount": [
      {
        "value": 1000,
        "unit": "count",
        "startDate": "2024-01-01T10:00:00Z",
        "endDate": "2024-01-01T11:00:00Z",
        "metadata": {}
      }
    ],
    "HKQuantityTypeIdentifierHeartRate": [
      {
        "value": 72,
        "unit": "count/min",
        "startDate": "2024-01-01T10:00:00Z",
        "endDate": "2024-01-01T10:01:00Z",
        "metadata": {}
      }
    ]
  },
  "fullExport": false
}
```

**Request Headers:**
- `Content-Type: application/json`
- `Authorization: Bearer {token}`
- `Content-Length: {size}`

## Troubleshooting

### Sync Not Starting

- Ensure HealthKit is available on the device (not available on iPad without Apple Watch)
- Verify endpoint and token are correctly configured
- Check that at least one health data type is being tracked
- Ensure authorization was granted

### Duplicate Requests

The plugin includes safeguards to prevent duplicate requests:
- Only one sync runs at a time
- Initial sync blocks observer-triggered syncs
- Chunks are sent sequentially

If you see duplicates, check that:
- Only one instance of the plugin is initialized
- Multiple endpoints are using different endpoint keys (they're isolated automatically)

### Timeouts

If you experience timeouts:
- Reduce `recordsPerChunk` (try 5000 or lower)
- Check your server response time
- Ensure network connectivity

### HTTP 413 Errors

Reduce `recordsPerChunk` to send smaller payloads.

## Limitations

- iOS only (HealthKit is iOS-specific)
- Requires HealthKit authorization from user
- Background sync requires proper iOS background modes configuration
- Some health data types may have limited availability depending on device capabilities

## License

See LICENSE file for details.
