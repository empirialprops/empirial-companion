# Move everything from Firebase to Lovable Cloud

## What gets moved
- **All data**: ~27 collections (firms, deals, challenges, reviews, blog posts, users, payouts, events, testimonials, FAQs, pricing, partners, awards, support tickets, community posts, referrals, spreads, tickers, loyalty rewards, site settings/stats, analytics, audit logs, admins/staff roles, platforms). Every document copied with its original ID; nothing deleted from Firebase.
- **Files**: everything in Firebase Storage (logos, images, uploads) copied to Cloud storage, and stored links rewritten to the new location.
- **User accounts**: every Firebase user recreated in Cloud with the same email, name, photo and a link to their old Firebase ID so profiles, reviews and roles stay attached.
- **Admin / staff roles**: moved into a protected roles table.

## Logins (important limitation)
Firebase does not let passwords be read out in a form Cloud accepts directly. To make sure nobody is locked out:
- On first sign-in after the move, the app checks the password against Firebase once, then saves it into Cloud silently. Users notice nothing.
- Anyone who never signs in can use "Forgot password".
- Google sign-in moves to Cloud's built-in Google login (same email = same account).
- Discord login keeps working through the existing Discord app, but now signs users into Cloud instead of Firebase.

## Steps
1. Create matching Cloud tables (one per collection, flexible fields kept as JSON so no data is dropped), file buckets, roles table and access rules.
2. Run a one-time copy script using the Firebase service account already stored: data, files, users. Produce a count report (Firebase vs Cloud per collection) to prove nothing is missing.
3. Swap the app's data layer (`src/lib/firebase/services.ts` and the stores/pages that call Firestore directly) to Cloud.
4. Swap sign-in/sign-up/sign-out, auth modal, admin login, profile and Discord callback to Cloud auth, with the silent password carry-over.
5. Test: browse pages, sign in as an existing user, admin edits, uploads.
6. Keep Firebase read-only as a backup until you confirm, then the Firebase code can be removed.

## Technical details
- Tables: `id text pk` (Firestore doc id), typed columns for fields used in queries/sorting, plus `data jsonb` holding the full original document. Sub-collections flattened with a `parent_id`.
- Users: created via admin API with `user_metadata.firebase_uid`; a `profiles` table keyed by auth id with `firebase_uid` unique; all `userId` references remapped.
- Lazy password migration: server function verifies via Firebase Identity Toolkit REST `signInWithPassword`, then sets the password in Cloud auth with the admin client.
- Migration script runs server-side with firebase-admin from the sandbox, idempotent (upserts) so it can be re-run for a final sync right before switchover.
