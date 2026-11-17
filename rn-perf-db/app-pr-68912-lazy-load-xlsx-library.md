# Lazy load XLSX library
- **Repository:** Expensify/App
- **PR Link:** <https://github.com/Expensify/App/pull/68912>
- **Merged Date:** 2025-09-01
- **Category:** Bundle Size (+ Startup)

## Problem
`ImportSpreadsheet` imported the whole `xlsx` library at the top of the file, so it was part of the main bundle and evaluated at startup although it is only needed when a user imports a spreadsheet. The PR reports no measurements.

## Solution
Replace the static `import * as XLSX from 'xlsx'` with a dynamic `import('xlsx')` inside the file-read handler, so the library loads only when a spreadsheet is picked. The loading state is set before the import starts, and a new error message is shown if the library fails to load. The `xlsx` dependency was also moved from a vendored devDependency to a regular npm dependency (`^0.18.5`).

## Code Pattern (Before vs. After)
```tsx
// Before
import * as XLSX from 'xlsx';
// ...
const readWorkbook = () => {
    return fetch(fileURI)
        .then((data) => data.arrayBuffer())
        .then((arrayBuffer) => XLSX.read(new Uint8Array(arrayBuffer), {type: 'buffer'}));
};
readWorkbook().then((workbook) => { /* ... */ });

// After
setIsReadingFile(true);

import('xlsx')
    .then((XLSX) => {
        const readWorkbook = () => {
            return fetch(fileURI)
                .then((data) => data.arrayBuffer())
                .then((arrayBuffer) => XLSX.read(new Uint8Array(arrayBuffer), {type: 'buffer'}));
        };
        readWorkbook().then((workbook) => { /* ... */ });
    })
    .catch((error) => {
        console.error('Failed to load XLSX library:', error);
        setUploadFileError(true, 'spreadsheet.importFailedTitle', 'spreadsheet.importSpreadsheetLibraryError');
        setIsReadingFile(false);
    });
```
