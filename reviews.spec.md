# Reviews and Moderation Spec

Status: draft, not yet validated. Feature 1 of 3 (reviews, shared itineraries, profiles). Background: `docs/ui-polish-design.md` and the social plan in `~/.claude/plans/brief-adding-social-features-soft-abelson.md`.

## Intent

Signed in visitors can leave one editable review (1 to 5 stars, optional text) on a place or a shop item, and see the average rating and the reviews of others. Moderation ships with it: report, author delete, admin delete, and automatic hiding after 3 distinct reports. Anonymous visitors see the site exactly as it is today. Out of scope: replies, reactions, bulk deletion of one's data, user bans, a restore UI, pagination, per minute rate limits, shared itineraries, profiles.

## Success Criteria

- SC-1 A signed in visitor can rate a place and see the updated average within one page interaction, with no reload.
- SC-2 An abusive review can be removed by the admin, and no non admin can remove someone else's review.
- SC-3 A signed out visitor's browser makes zero social requests and shows zero social UI.
- SC-4 Existing private sync (checklist, itinerary, saved recommendations) and AI search behave as before.

## Acceptance Criteria

Identity and authorization
- AC-1 (Ubiquitous) Every social action shall require a Google ID token that Google validates and whose audience equals the configured OAuth client id; otherwise the system shall answer 401.
- AC-2 (Unwanted) If the configured client id is missing, then the system shall reject every authenticated action with 401.
- AC-3 (Ubiquitous) Admin status shall be derived only from the verified token's user id being listed in the `ADMIN_SUBS` configuration, and never from client supplied data.
- AC-4 (Event) When a signed in user calls `social_me`, the system shall create the user record if absent, with a random public handle and a display name taken from the Google given name, and return the handle, display name and `isAdmin`.
- AC-5 (Event) When a user updates their display name to 1 to 30 characters after trimming, the system shall store it and show it on that user's reviews. If the trimmed name is empty or longer than 30 characters, then the system shall reject the request with 400.

Reviews
- AC-6 (Event) When a user submits a review for a valid place key with an integer rating from 1 to 5 and text of at most 1000 characters, the system shall create the review, or update the user's existing review for that place.
- AC-7 (Unwanted) If the place key does not match `^[ps]:.{1,200}$`, the rating is not an integer from 1 to 5, or the text exceeds 1000 characters, then the system shall reject the request with 400 and change nothing.
- AC-8 (Unwanted) If the user's earlier review of that place was deleted by an admin, then the system shall reject a new review of that place with 403.
- AC-9 (Event) When a user submits a review for a place where they deleted their own earlier review, the system shall restore that review with the new content.
- AC-10 (Event) When a signed in user lists reviews for a place key, the system shall return reviews that are neither deleted nor hidden, plus hidden reviews when the caller is their author or an admin (marked as hidden), newest first, with the caller's own review first, each with author display name, picture, handle, rating, text, timestamps, and whether it is the caller's and whether the caller reported it.
- AC-11 (Event) When a signed in user requests place stats, the system shall return the average rating and the count per place key, counting only reviews that are neither deleted nor hidden.

Moderation
- AC-12 (Event) When a signed in user other than the author reports a review with a reason of at most 300 characters, the system shall record the report once per reporter and review.
- AC-13 (Unwanted) If the reporter is the review's author, has reported that review before, or the review is deleted, then the system shall reject the report and change nothing.
- AC-13a (Event) When an author edits a hidden review, the system shall keep it hidden.
- AC-14 (Event) When a review reaches 3 distinct open reports, the system shall hide it from everyone except its author and admins, and show the author that it is pending review.
- AC-15 (Event) When the author or an admin deletes a review, the system shall soft delete it, recording who deleted it and when, and mark its open reports resolved.
- AC-16 (Unwanted) If a user who is neither the author nor an admin deletes a review, then the system shall reject the request with 403 and change nothing.
- AC-17 (Event) When an admin lists reports, the system shall return reviews with open reports, with the reports and reasons. If the caller is not an admin, then the system shall answer 403.
- AC-18 (Event) When an admin resolves a review's reports as deleted, the system shall soft delete the review; when as dismissed, the system shall clear the hidden state. In both cases the system shall mark the open reports resolved.
- AC-19 (Unwanted) If a user has created or edited reviews on 30 distinct places, or filed 20 reports, in the last 24 hours, counted from stored rows, then the system shall reject further review writes or reports respectively with 429. Repeated edits of the same review count once.

Client
- AC-20 (State) While the visitor is not signed in, the site shall make no social requests and render no social elements.
- AC-21 (State) While the visitor is signed in, each place card and shop item shall show its average rating and review count, or an invitation to be the first to rate, and open the review sheet when tapped.
- AC-22 (Ubiquitous) The site shall render all review text, names and reasons as escaped text, with no link detection.
- AC-23 (Unwanted) If a social request fails because the token expired, then the site shall tell the visitor to sign in again and shall not lose the text they typed.
- AC-24 (State) While the caller is an admin, My Area shall show the report queue with delete and dismiss actions; otherwise it shall not.

Coexistence
- AC-25 (Ubiquitous) Social actions shall not be blocked by AI budget, rate limit or bot check state.
- AC-26 (Ubiquitous) The existing actions (`save_user_data`, `load_user_data`, place search, ranking, itinerary summary, itinerary questions) shall behave as before.

## Constraints

- One worker endpoint, `POST {action, idToken, ...}`, JSON in and out, same CORS as today.
- New actions: `social_me`, `update_profile`, `get_place_stats`, `list_reviews`, `upsert_review`, `delete_review`, `report_review`, `admin_list_reports`, `admin_resolve_report`.
- Place key is `p:<title>` for DATA places and `s:<shop item name>` for shop items.
- Reviews live in Cloudflare D1. KV keeps the private per user blob unchanged.
- Soft delete only. Nothing is removed from storage by the application.
- No email address is stored.

## Assumptions

- The two cards for the Nakatanidou duplicate title share one set of reviews.
- A renamed title orphans its reviews; an alias map is deferred until it happens.
- About 1 hour token lifetime is handled by asking the visitor to sign in again, not by refreshing silently.
- Rating stats for all places fit in one response at this scale.

## Non goals

Replies, reactions, bulk "delete my data" (the owner removes a user by SQL on request), user bans, admin restore UI, pagination, per minute limits, review photos, shared itineraries, profiles, site visit analytics (separate small change).

## Edge Cases

- Author deletes own review, then posts again: covered by AC-9.
- Admin dismisses reports, then a new reporter reports again: hidden again at 3 distinct open reports; earlier reporters cannot report again (AC-13).
- Review text empty with a rating: allowed.
- Display name changed after reviews were posted: reviews show the current name.

## Decisions on former open questions

- Display names are not reportable in v1. An abusive name is fixed by the owner via SQL.
- The admin queue shows the report reasons only, not reporter names (owner had no preference).
