# Vellum - Frontend-Backend API Contract

## 1. Core Architecture & Assumptions

*   **Communication:** Frontend will communicate with the backend using Tauri's `invoke` system for actions, and listen to Tauri `events` for background progress.
*   **Database:** You've got nothing to do with it!
*   **Asset Serving:** The backend will provide safe `vellum://` or custom protocol URLs for the frontend to render images and HTML.
*   **PDF Handling:** Handled entirely by the frontend using `PDF.js`. I'll only provide the absolute file path and ensures the file exists.

## 2. Data Models (TypeScript Types)

```typescript
type BookFormat = 'EPUB' | 'CBZ' | 'CBR' | 'PDF';

interface Book {
  id: string; // UUID
  title: string;
  author: string | null;
  coverPath: string | null; // URL to load the cover image
  filePath: string; // Absolute path on the filesystem
  format: BookFormat;
  totalPages: number | null;
  addedAt: string; // Date string
}

interface ReadingProgress {
  bookId: string;
  percentage: number; // 0.0 to 100.0
  cfi: string | null; // For EPUBs (EPUB Canonical Fragment Identifier)
  pageNumber: number | null; // For PDFs and CBZ/CBR
  lastReadAt: string; // Date string
}

interface BookSession {
  bookId: string;
  format: BookFormat;
  // The base URL to the extracted temp directory for fetching assets
  // e.g., "vellum://cache/book_id/"
  baseAssetUrl: string; 
  // For EPUB: Path to the root OPF or index.html
  // For CBZ/CBR: Array of sorted image URLs
  // For PDF: Just the local file path
  entryData: any; 
}
```

## 3. Tauri Commands (Backend API)

Use `import { invoke } from '@tauri-apps/api/core';` to call these.

### Library Management

*   **`import_file`**
    *   *Payload:* `{ filePath: string }`
    *   *Returns:* `Promise<Book>`
    *   *Description:* Tells the backend to parse a file, extract its metadata/cover, add it to the SQLite database, and return the Book object.
*   **`get_library`**
    *   *Payload:* None
    *   *Returns:* `Promise<Book[]>`
    *   *Description:* Retrieves all books currently in the user's library.
*   **`delete_book`**
    *   *Payload:* `{ id: string, deleteFile: boolean }`
    *   *Returns:* `Promise<void>`
    *   *Description:* Removes the book from the database. Optionally deletes the original file from the OS if `deleteFile` is true.

### Reader Session

*   **`open_book`**
    *   *Payload:* `{ id: string }`
    *   *Returns:* `Promise<BookSession>`
    *   *Description:* Prepare a book for reading. For EPUB/CBZ/CBR, it extract the archive to a temp dir and return the URL needed to render it.
*   **`close_book`**
    *   *Payload:* `{ id: string }`
    *   *Returns:* `Promise<void>`
    *   *Description:* Cleans up any temporary extracted files to free disk space.

### Progress Tracking

*   **`update_progress`**
    *   *Payload:* `{ progress: ReadingProgress }`
    *   *Returns:* `Promise<void>`
    *   *Description:* Saves the user's current reading position to the db.
*   **`get_progress`**
    *   *Payload:* `{ id: string }`
    *   *Returns:* `Promise<ReadingProgress | null>`
    *   *Description:* Retrieve last saved reading position for the book.

## 4. Error Handling

For you errors will be thrown as Promise rejections. Please wrap every `invoke` call in `try/catch` blocks. :)
