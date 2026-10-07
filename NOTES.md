# Patch Exercise Notes

## Approach

I first ran the application in its original state and reproduced issues through the UI and API before making changes. I prioritized bugs that affected correctness, reliability, performance, or API robustness. I avoided unrelated feature work and did not change the mobile layout because the responsive pagination remained usable at a 375px viewport.

AI-assisted tools were used during investigation and implementation, but each change was manually reviewed and verified through runtime testing.

---

## 1. Search returned archived tasks

### Issue
Searching by a task description could return archived tasks even though archived tasks are intended to be excluded from normal search results.

### How I found it
I ran the application and searched for `deprecated`. The archived task `Legacy API cleanup` (ID 21) was returned.

I then traced the request to the native SQL query in `TaskRepository`.

### Root cause
The SQL boolean expression relied on `AND`/`OR` precedence:

    archived = FALSE
    AND title LIKE :term
    OR description LIKE :term
    AND status ...

This allowed description matches to bypass the archived condition.

### Change
Grouped the title/description search conditions explicitly:

    archived = FALSE
    AND (
        LOWER(title) LIKE :term
        OR LOWER(description) LIKE :term
    )
    AND (:status IS NULL OR status = :status)

The same logical fix was applied to the SQL reference files.

### Why
This keeps archived records out of normal search results and ensures the optional status filter applies consistently to both title and description matches.

### Verification
Searching for `deprecated` after the fix returned no archived task.

---

## 2. Artificial request delay

### Issue
Every task search request introduced an unnecessary server-side delay.

### How I found it
I measured the `/api/tasks?page=1&pageSize=10` request in the browser Network tab. The blank-search request consistently took about 1 second.

### Root cause
The controller calculated a query "complexity" value and called `Thread.sleep(...)` before executing the database query.

### Change
Removed the artificial sleep and the unused complexity calculation/logging.

### Why
The API should not deliberately block request threads for simulated work. Removing the delay improves response time and avoids unnecessary thread occupation under concurrent traffic.

### Verification
After the fix, the same request completed in tens of milliseconds (for example ~36 ms and ~92 ms in local testing), compared with roughly 1 second before the fix.

---

## 3. API failure left the UI stuck on "Loading tasks..."

### Issue
When the task API failed, the frontend could remain in the loading state instead of displaying the error.

### How I found it
I stopped the backend and reloaded the frontend. The API request failed and the UI remained on `Loading tasks...`.

### Root cause
The error handler set the error state but never reset the loading state.

### Change
Updated `useTasks.js` to clear stale errors at the beginning of a request and use `finally()` to always reset loading:

    setLoading(true);
    setError(null);

    ...
    .catch((err) => {
      setError(err.message);
    })
    .finally(() => {
      setLoading(false);
    });

### Why
Loading state must be cleared on both successful and failed requests.

### Verification
With the backend unavailable, the UI now exits the loading state and displays the request error.

---

## 4. Search/filter did not reset pagination

### Issue
Changing the search or status filter while viewing a later page could request a page that does not exist for the new result set.

### How I found it
I reproduced the issue by searching for `mobile` while on a later page. The UI showed `No tasks found` even though matching tasks existed.

### Root cause
Changing `query` or `status` did not reset the current page.

### Change
Added query/status change handlers that update the filter and reset the page to 1.

### Why
A new search/filter represents a new result set, so pagination should restart from the first page.

### Verification
After the fix, searching for `mobile` automatically returned to page 1 and displayed the matching tasks.

---

## 5. Invalid API parameters caused 500 errors

### Issue
Invalid pagination values and unknown status values could cause server errors.

### How I found it
Direct API testing showed:

- `page=0` produced HTTP 500.
- `pageSize=1000` was accepted as an invalidly large request.
- An invalid status such as `INVALID` could throw an exception.

### Root cause
The controller did not validate pagination inputs and called `TaskStatus.valueOf(...)` without handling invalid values.

### Change
Added controller validation:

- `page >= 1`
- `1 <= pageSize <= 100`
- invalid status values return a controlled `400 Bad Request`

### Why
Invalid client input should result in a predictable validation response rather than an unexpected server error.

### Verification
Confirmed that invalid pagination and status requests now return validation errors with HTTP 400.

---

## Additional verification

- Normal API request returned task data successfully.
- Search for `mobile` returned the expected matching tasks.
- Search for `deprecated` no longer returned the archived task.
- Valid status filtering worked.
- Responsive testing at 375px width showed pagination remained usable, so no unnecessary CSS changes were made.

## Scope decisions

I intentionally did not implement unrelated feature requests found in the sample task data. The exercise was treated as a bug-fixing and improvement task, so changes were limited to reproducible issues with clear user or API impact.