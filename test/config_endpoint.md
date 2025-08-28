## POST /api/v1/config — Endpoint Summary

### Overview
- **Purpose**: Return a bootstrap configuration payload for the signed-in user, including current user/account context, user list (filtered/paginated), memberships, environment flags, and AI feature toggles.
- **Business logic**: Filters users by `user_ids`, `membership_ids`, or `searchText`, applies optional pagination (`limit` + `offset`), computes `total_users` (filtered or full count), and adds account- and environment-driven flags.

### Request Details
- **Method/URL**: `POST /api/v1/config`
- **Headers**:
  - `Authorization: Bearer <jwt>` (when JWT flow enabled) or Devise session cookie
  - `Content-Type: application/json`
- **Query parameters (optional)**:
  - `user_ids`: array<string|integer> — restrict users to specific ids
  - `membership_ids`: array<string|integer> — restrict users by membership ownership
  - `searchText`: string — case-insensitive substring on name/email
  - `limit`: integer — page size (must be paired with `offset`)
  - `offset`: integer — page offset (must be paired with `limit`)
- **Body**: none required

#### Example Request
```http
POST /api/v1/config?searchText=ann&limit=10&offset=0
Authorization: Bearer <token>
Content-Type: application/json

{}
```

### Response Details
- **Status codes**:
  - `200 OK` — payload returned
  - `401 Unauthorized` — auth failed (access denied, bad/missing JWT)
  - `400 Bad Request` — invalid arguments/associations
  - `404 Not Found` — record not found
  - `500 Internal Server Error` — unhandled exception

#### Success Payload (representative)
```json
{
  "current_user": { "id": 123, "uid": "u-9f3a1c", "name": "Anna Smith", "email": "anna.smith@example.com" },
  "current_account": { "id": 456, "organization_id": 789, "name": "Acme Corp" },
  "users": [
    { "id": 123, "name": "Anna Smith", "email": "anna.smith@example.com" },
    { "id": 124, "name": "Ben Turner", "email": "ben.turner@example.com" }
  ],
  "current_memberships": [
    { "id": 12, "user_id": 123, "account_id": 456, "admin": true, "agent_manager": false }
  ],
  "is_gov_cloud": false,
  "environment": "production",
  "total_users": 257,
  "ai_supported": true,
  "ai_smart_script_enabled": false,
  "disable_robot_users_to_create_robots": false
}
```

#### Error Payload
```json
{
  "errors": [
    { "status": 401, "detail": "Access denied. Error ID: 72d8c5a4-..." }
  ]
}
```

### Business Logic & Processing Flow
```ruby
authenticate_request!  # JWT or Devise; host header + IP whitelist
users = users()        # from ConfigConcern; account-scoped + with_grc_subscription
memberships = current_memberships()
total = total_users()
flags = set_ai_flags(current_account, Rails.env)  # ai_supported, ai_smart_script_enabled
render ConfigSerializer.new(current_user, current_account, users, memberships, gov_cloud?, total)
```

#### Key Rules/Validations
- Users/memberships constrained to current account and `Membership.with_grc_subscription` where applicable.
- `searchText` applies ILIKE on user name/email.
- Pagination applies only when both `limit` and `offset` are provided.
- `total_users` equals filtered count when searching; otherwise full subscribed user count.

### Sequence Diagram

![Alt text](img2.png?raw=true "Title")

### Dependencies & Integrations
- Controllers/Concerns: `Api::V1::ConfigController`, `ConfigConcern`, `CurrentAccountConcern`, `LocaleConcern`, `ErrorHandlerConcern`, `MetricLogger`, `GrcShared::HostHeaderValidator`, `GrcShared::TermsOfServiceAcceptance`, `SauronClient::ActivityLogging::ControllerHook`.
- AuthZ/AuthN: `CanCanCan`, `Devise/ACL`, `JwtHighbondRuby::ControllerHelpers`.
- Serializers: `ConfigSerializer`, `CurrentUserSerializer`, `AccountSerializer`, `UserSerializer`, `MembershipSerializer`; `AiSupportConcern` for flags.
- Models/Scopes: `User`, `Account`, `Membership.with_grc_subscription`.
- Config/Flags: `configatron.is_gov_cloud`, `$flipper`, `Rails.env`, `ENV['SMART_SCRIPT_EAP_ORGS']`.
- Observability: `Airbrake`, `GrcInsight::Concerns::SegmentIoMonitoring`.

### Related References
- Code: `app/controllers/api/v1/config_controller.rb`, `app/controllers/concerns/config_concern.rb`, `app/serializers/config_serializer.rb`.
- Routes: `config/routes.rb` → `post :config, to: 'config#show'`.
- Tests: `spec/requests/api/v1/config_controller_spec.rb`, `spec/requests/api/public/v1/config_controller_spec.rb`.

### Examples & Edge Cases
- Examples:
  - Basic: `POST /api/v1/config`
  - Search + paginate: `POST /api/v1/config?searchText=ann&limit=10&offset=0`
  - Filter by user_ids: `POST /api/v1/config?user_ids[]=123&user_ids[]=124`
  - Filter by membership_ids: `POST /api/v1/config?membership_ids[]=42`
- Edge cases:
  - Only one of `limit`/`offset` supplied → no pagination applied.
  - Users without GRC subscription excluded from `users` and `current_memberships`.
  - In production, `ai_smart_script_enabled` depends on `ENV['SMART_SCRIPT_EAP_ORGS']` including the account’s `organization_id`.
  - Non-allowlisted IP may be redirected to accounts portal (policy enforcement).


