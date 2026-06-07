# Kudos System Specification

## Functional Requirements

### User Stories

1. As an authenticated employee, I can select another colleague from a list.
2. As an authenticated employee, I can write a short kudos message of appreciation (max 500 characters).
3. As an authenticated employee, I can submit kudos to a colleague and receive confirmation.
4. As an authenticated employee, I can view a public feed of recently submitted kudos on the dashboard.
5. As an administrator, I can hide or delete inappropriate kudos messages.
6. As an administrator, I can see why a kudos message was moderated and who moderated it.
7. As a user, I can only see kudos that are visible in the public feed.
8. As a user, I receive validation feedback for empty, too-long, or duplicate kudos content.
9. As a user, I cannot send kudos to myself.
10. As a user, I can use the kudos feature on desktop and mobile screens.

### Acceptance Criteria

- The user must be authenticated before accessing the kudos creation form or feed.
- The colleague selection control must display all eligible colleagues except the current user.
- Kudos messages must be limited to 500 characters and must not be empty.
- The public feed must show only kudos with `is_visible = true`.
- Administrators must have controls to hide or delete kudos.
- Moderated kudos must record `moderated_by`, `moderated_at`, and `moderation_reason` when hidden or deleted.
- The UI must display friendly validation and error messages.
- The system must prevent duplicate kudos submissions and self-kudos.
- The feed should be paginated or limited to the most recent items to support performance.
- The system must sanitize message content to prevent XSS and other injection attacks.

## Technical Design

### Database Schema

#### users

- `id` (UUID or integer, primary key)
- `name` (string, required)
- `email` (string, required, unique)
- `role` (enum: `user`, `admin`, required)
- `created_at` (timestamp, default current timestamp)
- `updated_at` (timestamp, default current timestamp)

#### kudos

- `id` (UUID or integer, primary key)
- `sender_id` (foreign key → `users.id`, required)
- `receiver_id` (foreign key → `users.id`, required)
- `message` (string/text, required, max 500 characters)
- `created_at` (timestamp, default current timestamp)
- `is_visible` (boolean, required, default true)
- `moderated_by` (foreign key → `users.id`, nullable)
- `moderated_at` (timestamp, nullable)
- `moderation_reason` (string, nullable, max 250 characters)
- `is_deleted` (boolean, required, default false)

#### Relationships

- `users` has many sent kudos (`sender_id`).
- `users` has many received kudos (`receiver_id`).
- `users` has many moderated kudos (`moderated_by`).

### API Endpoints

#### Authentication

- `GET /api/auth/me`
  - purpose: returns current user profile and role
  - response: `{ id, name, email, role }`

#### Users

- `GET /api/users`
  - purpose: return colleague list for selection
  - query: optional search term
  - response: array of `{ id, name, email }`

#### Kudos Feed

- `GET /api/kudos?limit=20&page=1`
  - purpose: return recent visible kudos for dashboard feed
  - response: array of kudos objects with sender and receiver metadata

#### Create Kudos

- `POST /api/kudos`
  - body: `{ receiver_id, message }`
  - validations: authenticated, `receiver_id` exists and is not current user, `message` non-empty and <= 500 chars, duplicate check
  - response: created kudos object

#### Moderation

- `PATCH /api/kudos/:id/hide`
  - body: `{ reason }`
  - permissions: admin only
  - effect: sets `is_visible` false, `moderated_by`, `moderated_at`, and `moderation_reason`
  - response: updated kudos object

- `PATCH /api/kudos/:id/delete`
  - body: `{ reason }`
  - permissions: admin only
  - effect: sets `is_deleted` true, `is_visible` false, moderation fields updated
  - response: updated kudos object

#### Optional Admin List

- `GET /api/admin/kudos?status=visible|hidden|deleted&limit=20&page=1`
  - purpose: return kudos items for moderation review
  - response: paginated kudos with moderation metadata

### Frontend Components

#### `KudosForm`

- Dropdown or autocomplete to select a colleague
- Textarea for kudos message
- Submit button
- Inline validation for required fields and max length
- Prevent self-selection
- Show success / error toast after submit

#### `KudosFeed`

- Displays a list of recent visible kudos
- Each card shows sender, receiver, message, and timestamp
- Supports pagination or "load more"
- Responsive layout for mobile and desktop

#### `AdminModerationPanel`

- Displays kudos items flagged for review or recent submissions
- Includes `Hide` and `Delete` actions for each item
- Shows moderation reason input before action
- Confirms each action

#### `Dashboard`

- Integrates `KudosForm` and `KudosFeed`
- Displays current user greeting and moderatable controls for admins

### Security Considerations

- Require authentication for all kudos and user endpoints.
- Enforce role-based authorization for moderation endpoints.
- Validate all input on the server and client.
- Sanitize kudos message text before rendering to prevent XSS.
- Protect against self-kudos and duplicate submissions.
- Use parameterized queries or ORM protections to prevent SQL injection.
- Limit output returned to the client to only needed fields.

### Performance Considerations

- Add database indexes on `kudos.created_at`, `kudos.receiver_id`, `kudos.sender_id`, and `kudos.is_visible`.
- Return paginated results for feed and admin lists.
- Cache public feed responses if traffic is high and update on new kudos.
- Load colleague list with incremental search if the user list grows large.

### Error Handling and Logging

- Standardize API error responses with status codes and messages.
- Return `400` for validation failures, `401` for authentication failures, and `403` for authorization failures.
- Log server-side failures with context, including request path and user ID.
- Log moderation actions with admin ID, kudos ID, and reason.
- Display user-friendly errors in the UI.

## Implementation Plan

### Phase 1: Specification and Setup

1. Create the `SPECIFICATION.md` file and finalize requirements.
2. Initialize a new project repository.
3. Choose the application stack (e.g. React + Node/Express + PostgreSQL or similar).
4. Add project dependencies and scaffolding.

### Phase 2: Data Model and Backend

1. Define the `users` and `kudos` tables in the database schema.
2. Implement authentication and user profile endpoints.
3. Implement kudos creation endpoint with validation.
4. Implement kudos feed endpoint with `is_visible` filtering.
5. Implement moderation endpoints for hide and delete actions.
6. Add moderation fields to the kudos model and database migrations.

### Phase 3: Frontend Implementation

1. Build `KudosForm` with colleague selection and message validation.
2. Build `KudosFeed` to display recent visible kudos.
3. Build `AdminModerationPanel` for admin hide/delete controls.
4. Add responsive layout and error handling to the dashboard.

### Phase 4: Validation, Moderation, and Edge Cases

1. Add duplicate-check logic for repeated kudos submissions.
2. Prevent self-kudos at the API and UI levels.
3. Enforce character limits and input sanitization.
4. Add admin-only visibility controls and moderation logging.

### Phase 5: Testing and Review

1. Write unit tests for backend services and API routes.
2. Write integration tests for the kudos creation flow.
3. Write frontend tests for form validation and feed rendering.
4. Review the full feature against acceptance criteria.

### Phase 6: Deployment and Repository Delivery

1. Add documentation and startup instructions to `README.md`.
2. Initialize Git, commit all files, and ensure the repo is clean.
3. Push the repository to a new public GitHub or GitLab repository.
4. Confirm the final `SPECIFICATION.md` is included in the repository.

## Notes on Moderation Design

- `is_visible` defaults to `true` so kudos appear immediately in the public feed.
- `moderated_by`, `moderated_at`, and `moderation_reason` provide an audit trail for admin actions.
- `is_deleted` is separate from `is_visible` to preserve soft-delete behavior and audit history.
- Admin hide/delete actions should be reversible by an admin if needed.

## Additional Considerations

- Consider adding a soft “flag as inappropriate” feature in a later iteration.
- Consider email notifications or activity summaries once the kudos feature is stable.
- Consider rate limiting to prevent spam and abuse.
