# Feature Specification: PhotoVault Core

**Feature Branch**: `001-photovault-core`
**Created**: 2026-03-12
**Status**: Draft
**Input**: User description: "PhotoVault Core - Auth, photo upload with S3 storage, anti-download protection, anti-screenshot protection, visibility controls, photographer portfolio, and storage dashboard"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Photographer Registration and Login (Priority: P1)

A photographer discovers PhotoVault and wants to create an account to start sharing their work. They register using their email and password or through a social login provider (Google, Apple). After registration, they can log in, access their dashboard, and manage their account. If they forget their password, they can reset it via email.

**Why this priority**: Without authentication, no other feature can function. This is the foundational gate for all platform interactions.

**Independent Test**: Can be fully tested by registering a new account, logging in, logging out, and resetting a password. Delivers immediate value: the photographer has a secure identity on the platform.

**Acceptance Scenarios**:

1. **Given** a visitor on the registration page, **When** they submit a valid email and password (min 8 chars, at least 1 uppercase, 1 number, 1 special character), **Then** an account is created and they are redirected to their dashboard.
2. **Given** a visitor on the registration page, **When** they choose "Sign in with Google", **Then** the OAuth flow completes and an account is created/linked.
3. **Given** a visitor on the registration page, **When** they choose "Sign in with Apple", **Then** the OAuth flow completes and an account is created/linked.
4. **Given** a registered photographer, **When** they submit valid credentials on the login page, **Then** they are authenticated and redirected to their dashboard.
5. **Given** a registered photographer, **When** they click "Forgot Password" and enter their email, **Then** they receive a password reset link via email that expires after 1 hour.
6. **Given** a logged-in photographer, **When** they click "Logout", **Then** their session is invalidated and they are redirected to the login page.
7. **Given** a visitor, **When** they attempt to register with an email already in use, **Then** they see an error message indicating the email is taken (without revealing if the account exists for security).
8. **Given** a user who has failed login 5 times in 1 minute, **When** they attempt a 6th login, **Then** their account is temporarily locked for 15 minutes and they see a message indicating the lockout.

---

### User Story 2 - Photo Upload with Protection (Priority: P1)

A photographer wants to upload their photos to the platform with confidence that they cannot be easily downloaded or captured via screenshot. They select one or more photos, configure visibility (public, private, or link-only), and upload them. The system stores the original in a secure location and generates optimized versions for viewing. When a visitor views the photo, anti-download and anti-screenshot protections are active.

**Why this priority**: This is the core value proposition of the platform — secure photo sharing. Without upload and protection, the product has no reason to exist.

**Independent Test**: Can be fully tested by uploading a photo, verifying it appears in the dashboard, viewing it as a visitor, and confirming protections are active (right-click disabled, overlay present, optimized version served instead of original).

**Acceptance Scenarios**:

1. **Given** a logged-in photographer, **When** they select a photo (JPEG, PNG, or WebP, max 50MB) and click "Upload", **Then** the photo is stored securely, a thumbnail and an optimized viewing version are generated, and the photo appears in their dashboard.
2. **Given** a logged-in photographer, **When** they upload multiple photos at once (batch upload, max 20), **Then** all photos are processed and appear in their dashboard with a progress indicator.
3. **Given** a logged-in photographer, **When** they set a photo's visibility to "public", **Then** the photo appears in their public portfolio and in the platform feed.
4. **Given** a logged-in photographer, **When** they set a photo's visibility to "private", **Then** the photo is only visible in their own dashboard.
5. **Given** a logged-in photographer, **When** they set a photo's visibility to "link-only", **Then** the photo is accessible only via a unique shareable link.
6. **Given** any user viewing a protected photo in a browser, **When** they right-click on the image, **Then** the context menu is suppressed or does not offer "Save Image As".
7. **Given** any user viewing a protected photo, **When** they attempt to drag the image, **Then** the drag action is blocked.
8. **Given** any user viewing a protected photo, **When** they view the page source or inspect elements, **Then** the original image URL is not exposed — only a time-limited signed URL to an optimized (lower quality) version is used.
9. **Given** any user viewing a protected photo, **When** they attempt a screenshot, **Then** a visible overlay/watermark is rendered on top of the image via a client-side canvas layer, degrading the quality of any capture.
10. **Given** a photographer, **When** they upload a file that is not an image or exceeds the size limit, **Then** they see a clear error message and the upload is rejected.

---

### User Story 3 - Photographer Portfolio (Priority: P2)

A photographer wants a public-facing portfolio page that showcases their best work. They can customize their profile (bio, profile picture, social links) and organize featured albums. Visitors can access the portfolio via a unique URL (username-based slug).

**Why this priority**: The portfolio is the photographer's public identity and the primary way they attract clients and followers. It's high-value but depends on auth and photo upload being functional first.

**Independent Test**: Can be fully tested by setting up a profile, adding a bio and profile picture, marking photos/albums as featured, and visiting the public portfolio URL as an unauthenticated visitor.

**Acceptance Scenarios**:

1. **Given** a logged-in photographer, **When** they navigate to "Edit Profile", **Then** they can set their display name, bio (max 500 characters), profile picture, and up to 5 social media links.
2. **Given** a logged-in photographer, **When** they choose a username/slug, **Then** the system validates it is unique, alphanumeric with hyphens (3-30 chars), and not a reserved word.
3. **Given** a logged-in photographer with a configured profile, **When** a visitor navigates to `/photographer/<username>`, **Then** they see the photographer's portfolio with profile info, featured photos, and public albums.
4. **Given** a visitor on a portfolio page, **When** they view the photos, **Then** all anti-download and anti-screenshot protections are active on every image.
5. **Given** a logged-in photographer, **When** they mark specific photos or albums as "featured", **Then** those appear prominently at the top of their portfolio page.
6. **Given** a visitor, **When** they navigate to a non-existent portfolio URL, **Then** they see a friendly 404 page suggesting to search for photographers.

---

### User Story 4 - Storage Dashboard and Usage Tracking (Priority: P2)

A photographer wants to understand how much storage they are using and how close they are to their plan's limit. They can view a dashboard showing total storage used, breakdown by album, and their current plan tier. When approaching their limit, they receive notifications.

**Why this priority**: The storage model is the monetization backbone. Photographers need visibility into their usage to understand costs and plan upgrades.

**Independent Test**: Can be fully tested by uploading several photos of known sizes, checking the dashboard reflects accurate totals, and verifying that a warning appears when usage exceeds 80% of the plan limit.

**Acceptance Scenarios**:

1. **Given** a logged-in photographer, **When** they navigate to "Storage" in their dashboard, **Then** they see total storage used (in MB/GB), storage limit of their current plan, a visual usage bar, and a breakdown by album.
2. **Given** a photographer whose storage usage reaches 80% of their plan, **When** they log in, **Then** they see a warning banner indicating they are approaching their limit.
3. **Given** a photographer whose storage usage reaches 95% of their plan, **When** they log in, **Then** they see an urgent warning and a prompt to upgrade their plan.
4. **Given** a photographer who has reached 100% of their storage, **When** they attempt to upload a new photo, **Then** the upload is blocked with a message to delete photos or upgrade their plan.
5. **Given** a photographer, **When** they delete a photo, **Then** the storage dashboard reflects the freed space within 5 minutes.
6. **Given** a photographer viewing storage breakdown, **When** they click on an album name, **Then** they see the individual photos in that album with their respective file sizes.

---

### User Story 5 - Photo Management (Priority: P2)

A photographer wants to manage their uploaded photos: view, edit metadata (title, description, tags), change visibility, and delete photos. They need a central dashboard to efficiently manage their entire library.

**Why this priority**: Managing existing photos is essential for a usable platform. Without edit/delete capabilities, photographers cannot maintain their content.

**Independent Test**: Can be fully tested by uploading photos, editing their metadata, changing visibility settings, and deleting a photo — then verifying the changes persist.

**Acceptance Scenarios**:

1. **Given** a logged-in photographer, **When** they navigate to their photo library, **Then** they see a paginated grid of all their photos with thumbnails, titles, and visibility indicators.
2. **Given** a logged-in photographer, **When** they click on a photo, **Then** they can edit its title (max 100 chars), description (max 1000 chars), and tags (max 20 tags).
3. **Given** a logged-in photographer, **When** they change a photo's visibility from "private" to "public", **Then** the photo immediately appears in their portfolio and the feed.
4. **Given** a logged-in photographer, **When** they click "Delete" on a photo, **Then** they are asked for confirmation and upon confirming, the photo is permanently removed from storage and the dashboard.
5. **Given** a logged-in photographer, **When** they use the search/filter in their library, **Then** they can filter by visibility status, date range, tags, or album.

---

### Edge Cases

- What happens when a photographer uploads a photo with the same filename as an existing one? → The system assigns a unique identifier; filenames do not conflict.
- What happens when the storage service (S3) is temporarily unavailable? → The system queues the upload and retries automatically up to 3 times, showing the user a "processing" status. After 3 failures, the user is notified with an option to retry manually.
- What happens when a user's session token expires mid-upload? → The upload fails gracefully with a message to re-authenticate; no partial files are left in storage.
- What happens when a visitor shares a signed URL after it expires? → The URL returns a 403 with a message that the link has expired.
- What happens when a photographer tries to register with a disposable email? → The system accepts it (no disposable email blocking) but validates format and deliverability via confirmation email.
- What happens when a photo file is corrupted? → The system validates the file header (magic bytes) during upload and rejects files that do not match their declared MIME type.
- What happens when the browser does not support canvas (anti-screenshot)? → The system falls back to a CSS overlay with reduced protection, still serving the optimized (not original) version.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: System MUST allow photographers to register with email/password with validation (min 8 chars, 1 uppercase, 1 number, 1 special character).
- **FR-002**: System MUST support OAuth login via Google and Apple.
- **FR-003**: System MUST support password reset via email with time-limited tokens (1 hour expiry).
- **FR-004**: System MUST enforce rate limiting on login attempts (max 5 per minute per account, 15-minute lockout).
- **FR-005**: System MUST store uploaded photos in secure cloud storage with originals never publicly accessible.
- **FR-006**: System MUST generate thumbnails and optimized viewing versions automatically upon upload.
- **FR-007**: System MUST validate uploaded files (MIME type via magic bytes, max 50MB per file, supported formats: JPEG, PNG, WebP).
- **FR-008**: System MUST support batch upload of up to 20 photos simultaneously.
- **FR-009**: System MUST implement anti-download protections: context menu suppression, drag prevention, no direct image URLs exposed.
- **FR-010**: System MUST serve images via time-limited signed URLs (max 15 minutes expiry) to optimized versions only.
- **FR-011**: System MUST implement anti-screenshot protections: transparent canvas overlay with dynamic watermark.
- **FR-012**: System MUST support three visibility levels per photo: public, private, link-only.
- **FR-013**: System MUST provide a public portfolio page per photographer accessible via unique slug URL.
- **FR-014**: System MUST allow photographers to customize their profile: display name, bio, profile picture, social links.
- **FR-015**: System MUST track storage usage per photographer in real-time (within 5 minutes of changes).
- **FR-016**: System MUST display storage dashboard with total used, limit, visual bar, and per-album breakdown.
- **FR-017**: System MUST send warnings at 80% and 95% storage thresholds and block uploads at 100%.
- **FR-018**: System MUST allow photographers to edit photo metadata (title, description, tags) and change visibility.
- **FR-019**: System MUST allow photographers to permanently delete photos with confirmation, freeing storage.
- **FR-020**: System MUST log all authentication events and access to protected resources for security auditing.
- **FR-021**: System MUST invalidate sessions server-side on logout.

### Key Entities

- **Photographer**: A registered user who uploads and manages photos. Attributes: email, display name, username/slug, bio, profile picture, social links, storage plan, storage used.
- **Photo**: An image uploaded by a photographer. Attributes: title, description, tags, visibility (public/private/link-only), original file reference, optimized version reference, thumbnail reference, file size, upload date, photographer (owner).
- **Album**: A collection of photos organized by a photographer. Attributes: name, description, cover photo, visibility, featured status, photographer (owner).
- **Session**: An authenticated user session. Attributes: user reference, token, expiration, creation timestamp, IP address.
- **Storage Plan**: A tier defining the photographer's storage limits. Attributes: name, storage limit (GB), price, features included.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Photographers can complete registration in under 2 minutes (either email/password or OAuth).
- **SC-002**: Photo upload completes in under 10 seconds for a 10MB photo on a standard broadband connection (20 Mbps).
- **SC-003**: 100% of protected photos are served with anti-download measures active (no direct URL exposure, context menu blocked, drag blocked).
- **SC-004**: Optimized viewing versions are at least 60% smaller than originals while maintaining acceptable visual quality.
- **SC-005**: Storage dashboard reflects accurate usage (±1% of actual) within 5 minutes of any change.
- **SC-006**: Portfolio pages load in under 3 seconds on standard broadband connections.
- **SC-007**: System handles at least 500 concurrent photographers uploading simultaneously without degradation.
- **SC-008**: All authentication events are logged with no gaps (100% audit trail coverage).
- **SC-009**: Signed URLs expire correctly — 0% of expired URLs serve content.
- **SC-010**: 95% of photographers can configure their portfolio (bio, picture, slug) without assistance on their first attempt.
