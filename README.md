# Ephemeral File Drop

Build a fullstack, no-signup file sharing service. Anyone can drop a file, get a short link, and share it. The link expires automatically, and an optional PIN can protect access. Use **TailwindCSS** for all styling — no other CSS frameworks, no custom CSS beyond the Tailwind entry point.

## Stack

- **Frontend**: Next.JS + TailwindCSS (port 3000)
- **Backend**: Flask + Sqlite

## How It Works

No accounts, no logins. A user selects a file (up to 10 MB), optionally sets a PIN and an expiry duration, and uploads it. The service stores the file server-side and returns a unique short link. Anyone with that link can visit the download page; if a PIN was set, they must enter it first. Files are automatically deleted once their expiry time passes.

## Pages

### `/` — Upload

The home page is the upload page. At its center is a **drag-and-drop zone**. Dragging a file onto it, or clicking it to open the OS file picker, selects a file for upload. Once a file is chosen, the zone displays the file's name and its size in human-readable format (e.g. `"3.2 MB"`).

Beneath the dropzone, offer three optional fields:

- **PIN** — a text input. If the user enters a value here, anyone visiting the download page must supply the same PIN before they can download the file. Leaving it blank means no PIN protection.
- **Expiry** — a `<select>` dropdown. The options and their values must be exactly:
  - `10s` → "10 seconds (testing)"
  - `10m` → "10 minutes"
  - `1h` → "1 hour" *(default selection)*
  - `24h` → "24 hours"
- **Upload button** — initiates the transfer. While the upload is in flight, show a progress bar reflecting actual upload progress and disable the button to prevent double-submission.

On **success**: hide the upload form and reveal a read-only element showing the full download URL, alongside a "Copy link" button that writes the URL to the clipboard.

On **failure** (file exceeds 10 MB, no file selected, network error, or any non-OK server response): keep the form visible and show an inline error message.

### `/d/[slug]` — Download

The download page is identified by a short random slug embedded in the URL path. This page has two distinct states:

**File unavailable**: If the slug is unknown or the file's expiry time has passed, show an appropriate message. Use a different visible element for "expired" versus "not found" — these are meaningfully distinct states.

**File available**: Display the original file name, its size in human-readable format, and a live countdown showing how much time remains before the link expires. The countdown must update every second.

If the file has a PIN, gate access behind a PIN entry form. The download button must not be shown until the correct PIN has been submitted. Show an error message when a wrong PIN is entered.

Once access is granted (no PIN required, or correct PIN entered), show a **Download** button. Clicking it delivers the file to the browser as a download with the original file name preserved.

## Technical Notes

- File content must be stored on disk or in object storage — not encoded into the database.
- Slugs must be randomly generated and at least 8 characters long.
- The backend enforces the 10 MB limit and returns a `400`-level error for oversized uploads.
- Expiry is enforced server-side: the download endpoint must check the expiry timestamp and refuse requests for files past their TTL.
- A Celery periodic task runs regularly to delete expired file records and their associated stored files.
- Uploaded files should be streamed to storage; avoid loading the entire file into memory at once.

## UI Requirements

Apply these `data-testid` attributes exactly as specified — the test harness depends on them.

### Upload page (`/`)

| Element | `data-testid` | Description |
|---------|---------------|-------------|
| Drag-and-drop zone | `"dropzone"` | The clickable and droppable area |
| Selected file name | `"selected-file-name"` | Appears after a file is chosen |
| Selected file size | `"selected-file-size"` | Human-readable, e.g. `"1.2 MB"` |
| PIN field | `"pin-input"` | Optional; empty means no PIN |
| Expiry dropdown | `"expiry-select"` | `<select>` with the four options above |
| Upload button | `"upload-btn"` | Disabled while uploading |
| Progress bar | `"upload-progress"` | **Visible throughout page load and during upload.** |
| Error message | `"upload-error"` | Shown when upload fails |
| Generated link | `"share-link"` | Read-only element with the full download URL |
| Copy button | `"copy-link-btn"` | Copies the URL to clipboard |

### Download page (`/d/[slug]`)

| Element | `data-testid` | Description |
|---------|---------------|-------------|
| File name | `"file-name"` | Original filename |
| File size | `"file-size"` | Human-readable size |
| Expiry countdown | `"expiry-countdown"` | Time remaining; updates every second |
| PIN field | `"pin-input"` | Shown only when the file has a PIN |
| PIN submit button | `"pin-submit"` | Submits the entered PIN |
| PIN error | `"pin-error"` | Shown when an incorrect PIN is entered |
| Download button | `"download-btn"` | Triggers the file download |
| Expired message | `"expired-message"` | Shown when the file's TTL has passed |
| Not-found message | `"not-found-message"` | Shown for unrecognized slugs |

## Tailwind Requirements

- Install via the official Vite plugin (`@tailwindcss/vite`).
- No other CSS framework (no Bootstrap, MUI, Chakra, etc.).
- A single `index.css` containing only `@import "tailwindcss"` is fine.
- All styling — layout, color, spacing, typography — must use Tailwind utility classes.
- The upload progress bar should use a visually distinct fill color (e.g. `bg-blue-500`) and its width must be proportional to the actual upload progress percentage.
