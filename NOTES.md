# Patch Exercise Notes

## Approach
I reproduced each issue before changing the code and retested the same scenario after the fix. AI-assisted tools were used as allowed, but I reviewed and verified the changes manually.

### 1. Archived tasks appeared in search
**File:** `backend/src/main/java/com/internal/tasktracker/TaskRepository.java`
**Layer:** Backend / SQL
**Found:** Searching `deprecated` returned archived task ID 21.
**Root cause:** `AND`/`OR` precedence allowed a description match to bypass `archived = FALSE`.
**Fix:** Grouped title/description conditions and kept `archived = FALSE` outside the `OR`. Updated the SQL reference files too.
**Verified:** `deprecated` returned no archived task.

### 2. Artificial API delay
**File:** `backend/src/main/java/com/internal/tasktracker/TaskController.java`
**Layer:** Backend / Java
**Found:** `/api/tasks` took about 1 second in Network timing.
**Root cause:** `Thread.sleep()` simulated query complexity.
**Fix:** Removed the artificial delay and unused complexity calculation.
**Verified:** Local requests dropped to about 36–92 ms.

### 3. Loading state stuck after API failure
**File:** `frontend/src/hooks/useTasks.js`
**Layer:** Frontend / React
**Found:** Stopping the backend left `Loading tasks...` visible.
**Root cause:** Loading was cleared only on success.
**Fix:** Reset loading in `finally()` and clear stale errors when a request starts.
**Verified:** Failed requests now show the error state.

### 4. Search/filter did not reset pagination
**File:** `frontend/src/App.jsx`
**Layer:** Frontend / React
**Found:** Changing filters on a later page could show no results incorrectly.
**Fix:** Search and status changes now reset page to 1.
**Verified:** `mobile` search returned the expected results.

### 5. Invalid API parameters caused server errors
**File:** `backend/src/main/java/com/internal/tasktracker/TaskController.java`
**Layer:** Backend / API
**Found:** Tested `page=0`, `pageSize=1000`, and `status=INVALID`.
**Fix:** Added validation: page >= 1, pageSize 1–100, and invalid status returns 400.
**Verified:** Invalid inputs are handled as `400 Bad Request`.

### Scope
I also tested the UI at 375px width. Pagination remained usable, so I avoided an unnecessary CSS change.
