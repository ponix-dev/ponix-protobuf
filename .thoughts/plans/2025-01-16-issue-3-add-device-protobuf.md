# Add Device Protobuf Message and Service Definition Implementation Plan

## Overview

Replace the existing IoT package with a simplified EndDevice protobuf definition for device management in the `end_device.v1` package. This will provide the protobuf schema needed for PostgreSQL device storage implementation in ponix-rs.

## Current State Analysis

The ponix-protobuf repository currently has:
- An `iot/v1/` package with complex EndDevice definitions including hardware types (LoRaWAN, HTTP), status enums, and multiple service definitions
- No external package dependencies on `iot.v1` (verified via grep)
- The existing EndDevice is overly complex for current needs with hardware-specific configurations

### Key Discoveries:
- No cross-package imports of `iot/v1` from envelope or organization packages ([grep results](grep-search))
- Self-contained iot package can be safely removed
- Repository follows pattern: `{domain}/v1/{domain}.proto` for file organization
- All messages use `buf/validate` for field validation
- Services follow `{Entity}Service` naming pattern

## Desired End State

A clean, simple EndDevice protobuf definition with:
- Core fields: device_id (server-generated), organization_id, name, timestamps
- CRUD service operations: CreateEndDevice, GetEndDevice, ListEndDevices
- Proper validation rules using buf/validate
- No status, metadata, or hardware-specific fields (deferred to future work)

### Verification Criteria:
- EndDevice message compiles and passes buf lint
- Service definition follows repository patterns
- No breaking changes to envelope or organization packages
- buf build succeeds

## What We're NOT Doing

- Adding status fields (e.g., ACTIVE, INACTIVE)
- Adding metadata or extensibility fields (google.protobuf.Struct)
- Implementing pagination for ListEndDevices
- Adding device authentication/authorization messages
- Implementing the service in Rust (that's ponix-rs issue #16)
- Migrating existing data (no data exists yet)

## Implementation Approach

Two-phase approach: first clean up the old iot package, then create the new simplified end_device package. This ensures a clean slate and avoids any confusion between old and new structures.

---

## Phase 1: Remove Existing IoT Package

### Overview
Clean removal of the legacy iot package to make way for the simplified EndDevice definition.

### Changes Required:

#### 1. Delete IoT Package Directory
**Directory**: `iot/`
**Changes**: Remove entire directory and all contents

```bash
rm -rf iot/
```

This removes:
- `iot/v1/end_device.proto` - Legacy EndDevice with hardware types
- `iot/v1/end_device_data.proto` - Historical data querying
- `iot/v1/flow_sensor.proto` - Device-specific data type
- `iot/v1/lorawan.proto` - LoRaWAN hardware types
- `iot/v1/ingestion.proto` - Data ingestion service

### Success Criteria:

#### Automated Verification:
- [x] Directory removed: `ls iot/ 2>&1 | grep "No such file"`
- [x] Other packages still build: `buf build`
- [x] No import errors: `buf lint`

#### Manual Verification:
- [ ] Confirm no references to iot package remain in codebase
- [ ] envelope and organization packages unaffected

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to Phase 2.

---

## Phase 2: Create EndDevice Protobuf Definition

### Overview
Create new simplified EndDevice message and service definition following repository patterns.

### Changes Required:

#### 1. Create Directory Structure
**Directory**: `end_device/v1/`
**Changes**: Create package directory

```bash
mkdir -p end_device/v1
```

#### 2. Create EndDevice Protobuf File
**File**: `end_device/v1/end_device.proto`
**Changes**: Define complete EndDevice message, service, and supporting types

```protobuf
syntax = "proto3";

package end_device.v1;

import "buf/validate/validate.proto";
import "google/protobuf/timestamp.proto";

// EndDevice represents a device registered in the system.
// Devices are scoped to organizations for multi-tenancy.
message EndDevice {
  // Server-generated unique identifier (UUID format)
  string device_id = 1 [(buf.validate.field).required = true];

  // Organization that owns this device
  string organization_id = 2 [(buf.validate.field).required = true];

  // Human-readable name for the device
  string name = 3 [(buf.validate.field).required = true];

  // When the device was created
  google.protobuf.Timestamp created_at = 4 [(buf.validate.field).required = true];

  // When the device was last updated
  google.protobuf.Timestamp updated_at = 5 [(buf.validate.field).required = true];
}

// Request to create a new device
// Device ID is generated server-side
message CreateEndDeviceRequest {
  // Organization that will own this device
  string organization_id = 1 [(buf.validate.field).required = true];

  // Human-readable name for the device
  string name = 2 [(buf.validate.field).required = true];
}

// Response containing the newly created device
message CreateEndDeviceResponse {
  // The created device with server-generated ID and timestamps
  EndDevice end_device = 1 [(buf.validate.field).required = true];
}

// Request to get a device by ID
message GetEndDeviceRequest {
  // Unique identifier of the device to retrieve
  string device_id = 1 [(buf.validate.field).required = true];
}

// Response containing the requested device
message GetEndDeviceResponse {
  // The requested device
  EndDevice end_device = 1 [(buf.validate.field).required = true];
}

// Request to list all devices for an organization
message ListEndDevicesRequest {
  // Organization ID to filter devices by
  string organization_id = 1 [(buf.validate.field).required = true];
}

// Response containing devices for the organization
message ListEndDevicesResponse {
  // List of devices belonging to the organization
  // No pagination for initial implementation
  repeated EndDevice end_devices = 1;
}

// Service for managing device lifecycle
service EndDeviceService {
  // Register a new device
  rpc CreateEndDevice(CreateEndDeviceRequest) returns (CreateEndDeviceResponse);

  // Retrieve a device by its ID
  rpc GetEndDevice(GetEndDeviceRequest) returns (GetEndDeviceResponse);

  // List all devices for an organization
  rpc ListEndDevices(ListEndDevicesRequest) returns (ListEndDevicesResponse);
}
```

**Design Decisions**:
- Device ID is server-generated (not in CreateEndDeviceRequest)
- All timestamps are server-managed (created_at, updated_at)
- No pagination for ListEndDevices (keep it simple for now)
- No status field (deferred to future work)
- No metadata extensibility (deferred to future work)
- Organization-scoped resources for multi-tenancy

### Success Criteria:

#### Automated Verification:
- [x] File created: `ls end_device/v1/end_device.proto`
- [x] Protobuf compiles: `buf build`
- [x] Linting passes: `buf lint`
- [x] Follows STANDARD lint rules (enforced by buf.yaml)
- [x] No validation errors: `buf build` succeeds with validation imports

#### Manual Verification:
- [ ] Message fields match issue requirements (device_id, organization_id, name, timestamps)
- [ ] Service methods match requirements (Create, Get, List)
- [ ] Validation rules are appropriate for each field
- [ ] Comments are clear and helpful

**Implementation Note**: After completing this phase and all automated verification passes, pause here for manual confirmation before proceeding to final verification.

---

## Phase 3: Final Verification and Breaking Change Documentation

### Overview
Verify the complete implementation and document breaking changes for consumers.

### Changes Required:

#### 1. Run Full Build and Lint
**Commands**: Comprehensive verification
**Changes**: Ensure everything builds and passes checks

```bash
# Full build
buf build

# Lint check
buf lint

# Breaking change detection (expected to show iot.v1 removal)
buf breaking --against '.git#branch=main'
```

### Success Criteria:

#### Automated Verification:
- [x] Build succeeds: `buf build` exits 0
- [x] Lint passes: `buf lint` exits 0
- [x] Breaking changes documented (removal of iot.v1 package)

#### Manual Verification:
- [ ] Review breaking changes output - should only show iot.v1 removal
- [ ] Confirm end_device.v1 package is properly structured
- [ ] envelope.v1 and organization.v1 packages unaffected

**Implementation Note**: After completing this phase and all automated verification passes, the implementation is complete and ready for review.

---

## Testing Strategy

### Unit Tests:
Not applicable - protobuf definitions don't have unit tests. Validation is done via buf tooling.

### Integration Tests:
Integration testing will happen in ponix-rs issue #16 when the PostgreSQL storage implementation is created.

### Manual Testing Steps:
1. Clone the repository and run `buf build`
2. Verify the generated code can be imported in a test project
3. Check that the BSR publish process works (when ready to publish)

## Performance Considerations

Protobuf schema changes have minimal performance impact. The simplified schema (compared to the old iot package) will:
- Reduce message size (fewer fields)
- Simplify serialization/deserialization
- Reduce code generation output size

## Migration Notes

### For Consumers:
Breaking changes in this update:
- **REMOVED**: `iot.v1` package entirely
  - `iot.v1.EndDevice` - Use `end_device.v1.EndDevice` instead
  - `iot.v1.EndDeviceService` - Use `end_device.v1.EndDeviceService` instead
  - `iot.v1.DataIngestionService` - No replacement (out of scope)
  - `iot.v1.EndDeviceDataService` - No replacement (out of scope)
  - `iot.v1.LoRaWANService` - No replacement (out of scope)

### New Package:
- **ADDED**: `end_device.v1` package
  - `end_device.v1.EndDevice` - Simplified device entity
  - `end_device.v1.EndDeviceService` - Basic CRUD operations

### Field Mapping (old iot.v1.EndDevice → new end_device.v1.EndDevice):
- `id` → `device_id` (naming change for clarity)
- `name` → `name` (unchanged)
- ~~`status`~~ → Removed (not needed yet)
- ~~`hardware_type`~~ → Removed (not needed yet)
- ~~`description`~~ → Removed (not needed yet)
- ~~`hardware_config`~~ → Removed (not needed yet)
- `organization_id` → Not in old schema, added for multi-tenancy
- `created_at` → Not in old schema, added for audit trail
- `updated_at` → Not in old schema, added for audit trail

## References

- Original ticket: [ponix-dev/ponix-protobuf#3](https://github.com/ponix-dev/ponix-protobuf/issues/3)
- Related implementation: [ponix-dev/ponix-rs#16](https://github.com/ponix-dev/ponix-rs/issues/16) (PostgreSQL device storage)
- Repository patterns: Existing `organization/v1/organization.proto` structure
- Buf documentation: [https://buf.build/docs](https://buf.build/docs)
