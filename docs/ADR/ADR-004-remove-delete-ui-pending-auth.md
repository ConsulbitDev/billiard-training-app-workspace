# ADR-004: Remove delete-shot UI pending authorization

## 📌 Context

`billiard-training-fe`#8 added a delete action to the shot list and shot detail pages, backed by
a real (soft-delete) `DELETE /api/shots/{id}` endpoint. There is currently no authorization on
the backend at all — anyone using the app can delete any shot. Real authentication/authorization
is explicitly V2 scope, not yet started (see ADR-003).

## 💡 Decision

Remove the delete UI (button/icon) from both the shot list and shot detail pages until some form
of authorization exists. The backend `DELETE /api/shots/{id}` endpoint itself is **not** changed
or disabled — it remains reachable by direct API call, since re-enabling the UI later should be a
matter of pointing a button back at an endpoint that already works, not rebuilding it.

This is explicitly a UX safety net, not a security fix: removing the button prevents *accidental*
deletion by someone using the app normally, but does not prevent a deliberate direct API call
(curl, browser devtools, etc.) — there is no authorization boundary on the backend to close that
gap. Real protection requires the V2 authorization work.

## 🔄 Consequences

**Positive:**
- Removes the easiest, most casual path to accidentally deleting real data before any access
  control exists.

**Accepted gap:**
- The backend endpoint remains unrestricted. This ADR does not close that gap, only the
  UI-level affordance for it.

**Follow-up needed:**
- When V2 authorization work begins, reinstate the delete UI gated behind whatever permission
  model is introduced, rather than simply restoring it unconditionally.
