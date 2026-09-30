# Security Specification: NFC Wi-Fi Connect

## 1. Data Invariants
1. **Public Non-Enumerability**: Anonymous or public users must NEVER be able to list, query, or enumerate records in any collection. Public access is strictly restricted to single document `get` by exact, unguessable `publicId` on `/publicWifiPages/{publicId}`.
2. **Permanent Public ID (Immutability)**: Once a `publicId` is assigned to a client, it can NEVER be modified, safeguarding the physical NFC tag URLs already distributed to establishments.
3. **Immutable Creation Timestamps**: `createdAt` cannot be altered after document creation.
4. **Server-Enforced Timestamps**: Client mutations must strictly bind `createdAt` and `updatedAt` to `request.time`.
5. **Administrative Privilege Isolation**: Only verified Google accounts with authorized administrative emails or UID in `/admins/{uid}` can read, create, update, or delete clients and publish public pages.
6. **Strict Schema Constraints**: Strings are length-bounded, hex colors must conform to regex patterns, and unknown/shadow keys are rejected.

## 2. The "Dirty Dozen" Threat Payloads (Must Return PERMISSION_DENIED)
1. **Payload 1 (Anonymous List Public Pages)**: `GET /publicWifiPages` without authentication -> Rejected: collection listing disabled.
2. **Payload 2 (Anonymous List Clients)**: `GET /clients` -> Rejected: unauthenticated read.
3. **Payload 3 (Unverified Email Admin Spoof)**: `CREATE /clients` with header `email: caioperatone.beiral@gmail.com` but `email_verified: false` -> Rejected: email must be verified.
4. **Payload 4 (Non-Admin User List)**: `GET /clients` with authenticated non-whitelisted email -> Rejected: non-admin blocked.
5. **Payload 5 (Public ID Modification)**: `UPDATE /clients/{id}` changing `publicId: "new_hacked_id"` -> Rejected: immutable publicId invariant violated.
6. **Payload 6 (Creation Time Spoofing)**: `CREATE /clients/{id}` with `createdAt: 1999-01-01` -> Rejected: must equal `request.time`.
7. **Payload 7 (Shadow Fields Injection)**: `CREATE /clients/{id}` with `{ isSuperAdmin: true, ... }` -> Rejected: strict key schema check.
8. **Payload 8 (Oversized Hex Color Injection)**: `UPDATE /clients/{id}` with `backgroundColor: "#" + "f".repeat(2000)` -> Rejected: regex/length limit failed.
9. **Payload 9 (Oversized SSID/Password Poisoning)**: `CREATE /clients/{id}` with 10MB string -> Rejected: length limits.
10. **Payload 10 (Path Traversal / Malicious ID Injection)**: Document ID containing illegal chars or oversized strings -> Rejected: `isValidId()` failure.
11. **Payload 11 (Unauthorized Public Page Write)**: Anonymous or regular user `SET /publicWifiPages/{publicId}` -> Rejected: only admin can publish/sync.
12. **Payload 12 (Self-Assigned Admin Escalation)**: Regular user `SET /admins/{myUid}` -> Rejected: only existing admin can manage admins.
