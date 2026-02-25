# Error codes

## Page creation (`StartUpPageCreateResult`)

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Invalid container configuration |
| 2 | Oversize – data too large for BLE transfer |
| 3 | Out of memory on glasses |

## Page rebuild

Returns `boolean`. The SDK has `APP_REQUEST_REBUILD_PAGE_SUCCESS` and `APP_REQUEST_REBUILD_PAGE_FAILD` (sic) error codes internally.

## Text upgrade

Returns `boolean`. Internal codes: `APP_REQUEST_UPGRADE_TEXT_DATA_SUCCESS` / `APP_REQUEST_UPGRADE_TEXT_DATA_FAILED`.

## Image update (`ImageRawDataUpdateResult`)

| Code | Meaning |
|---|---|
| success | OK |
| imageException | Image processing error |
| imageSizeInvalid | Image dimensions don't match container or are out of range |
| imageToGray4Failed | Greyscale conversion failed |
| sendFailed | BLE send failed |

## Shutdown

Returns `boolean`. Internal codes: `APP_REQUEST_UPGRADE_SHUTDOWN_SUCCESS` / `APP_REQUEST_UPGRADE_SHUTDOWN_FAILED`.

## SDK JSON compatibility

The SDK handles multiple key naming conventions from the host:
- camelCase: `containerID`
- PascalCase: `ContainerID`
- Proto-style: `Container_ID`

All are accepted via `pickLoose()` which normalises keys by removing underscores and lowercasing. You only need to use camelCase in your code – the SDK handles the rest.

Event data is also accepted in multiple shapes:
- `data: { type: 'listEvent', jsonData: {...} }`
- `data: { type: 'list_event', data: {...} }`
- `data: [ 'list_event', {...} ]` (array format)
- `data: { type: 'audioEvent', jsonData: { audioPcm: [...] } }`
