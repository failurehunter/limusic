Implement endless radio continuation in the limusic desktop player.
The baseline treats a radio as ordinary autoplay: when the queue runs low it re-requests the seed or playlist and appends the answer. That repeats the same first page and stops where the first panel ends. The feature makes a radio follow YouTube’s own panel continuation token, while keeping autoplay behavior for finite queues. Implement it in the Rust/Tauri backend (src-tauri/src/state.rs), the Innertube crate (crates/innertube/src/{endpoints,models/metadata,transport}.rs), and the Svelte UI (ui/src/lib/{api.ts,player.svelte.ts,components/QueueList.svelte}).
Innertube parsing
- Extend NextResult with panel: bool (serde default false). It means “the answer contained a radio panel”, not “the answer contained a token”.
- Replace the old rule “take the first continuation anywhere in the response” with a panel-scoped rule:
- Find the first playlistPanelRenderer or playlistPanelContinuation.
- Inside it, inspect continuations arrays and take the first entry with nextRadioContinuationData.continuation, falling back to nextContinuationData.continuation.
- Return Some(Some(token)) when the panel has a token, Some(None) when the panel exists without one, and None when there is no panel.
- Set NextResult.continuation to that token and NextResult.panel to whether a panel was found.
- This prevents comments, Related, and other /next responses from supplying a token that belongs to something other than the radio.
- A panel with no continuation means the radio ended. No panel means “not an answer about a radio”, which must not be treated as the end.
- Add Error::is_refusal() for HTTP 401/403, and use it wherever refusal is checked. Do not classify SessionExpired as a refusal.
- Add InnerTube::radio_continuation(metadata_client, token, anchor_video):
- POST /youtubei/v1/next with context and continuation only. Do not send a videoId or playlistId.
- Use the non-healing POST path (post_no_heal). A background radio window must not wake the session healer or sign the user out. A refusal returns as an ordinary error.
- Parse the answer and apply the same filtering as next through a shared finish_next helper: remove hidden music videos except the anchor, and drop blocked songs except the anchor. The anchor is the video id passed by the caller.
- Keep next behavior unchanged apart from sharing that filtering and passing its requested video id as the anchor.
Queue model
Add these fields to QueueState:
- radio: bool as before.
- radio_seed: Option<String> as before: the playlist/mix id this radio continues from.
- radio_cont: Option<(String, Option<String>)>:
- None: no page of this radio has been seen in this process. A restored queue starts here.
- Some((seed, Some(token))): the panel’s token for the next window.
- Some((seed, None)): the last page fetched for that seed was the radio’s end.
- radio_fails: u8: consecutive failed windows for the current token.
Define is_radio(q) = q.radio && q.radio_seed.is_some(). A raw radio flag without a pinned seed is not a radio for top-up, UI, or add-placement purposes.
Add top_up: Mutex<()> to AppState. Hold it across the network fetch so only one top-up runs at a time. Background callers use try_lock and skip if a top-up is already in flight. Skip and the track-end fallback use a bounded wait for the lock, 20 seconds (TOP_UP_WAIT), so a hung request cannot block playback forever.
radio_cont is not persisted. radio_fails is not persisted. radio and radio_seed keep their existing persistence.
Installing a radio
play_song
- If the track is local or autoplay is off, do not hydrate a radio. The queue stays a single song.
- Otherwise create the one-track queue, clear radio, radio_cont, and radio_fails, and set hydrating = Some(gen).
- Fetch the radio page directly with next(anchor, Some(RDAMVM<video_id>)). Do not use the bare page here; the direct radio page is what carries the panel.
- After the fetch, re-lock the queue, recheck the generation, and clear hydrating under the same lock. This prevents the track-start autoplay trigger from requesting the same radio in parallel.
- If the hydrated page has more than one item:
- pin radio_seed = RDAMVM<video_id>,
- set radio = true, reset radio_fails = 0,
- set radio_cont = radio_state(next_window(panel, items.len(), continuation), seed).
- If the page has only the seed song, leave the queue unpinned. It is a dead radio, not a radio that ended; autoplay may keep re-seeding from the last track.
- Deduplicate hydrated rows against the existing queue, preserving tracks the user queued manually and adopting album/twin metadata from the radio rows where available.
- If shuffle is on, re-snapshot the queue and shuffle the upcoming tracks.
- If the fetch fails, clear hydrating and continue with the plain queue. Prime the lookahead afterwards.
- Once hydrated, the queue is a radio for the rest of its life, even if autoplay is switched off later.
start_radio
- Resolve the radio source as before:
- song → videoId, seed RDAMVM<id>.
- playlist → seed from radio_seed_for (VL stripped, RDAMPL prefix unless the id already starts with RD).
- album → the album’s audio playlist id, then radio_seed_for.
- artist → the artist page’s radio_playlist_id, or a song radio on the top song.
- Reject local songs, local albums/artists, ON_REPEAT_ID, and local playlists.
- Fetch the first page with fetch_radio, which must now return (items, actual_seed, NextWindow):
- First next(video_id, Some(playlist_id)).
- If it has more than one item, use it.
- Otherwise ask the bare page, follow automixPreviewVideoId/automix_playlist_id when present, and try that mix.
- A bare page with more than one item is returned with NextWindow::Unknown, because its token belongs to no radio.
- Otherwise return “YouTube has no radio for this”.
- Reorder the first page as before: start at the seed song when present, and freshen tracks after the seed; artist/album/playlist pages are freshened whole.
- If the radio seed is already the playing track and the player is not idle, splice the radio behind the current track. Do not rebuffer.
- Otherwise install the queue through a play_tracks_as variant that accepts a radio descriptor. play_tracks for chosen playlists passes no descriptor.
- Install radio, radio_seed, radio_cont, and radio_fails under the queue lock together with the queue itself, before awaiting the first stream resolve. Do not set radio = true after start_current.
- radio_install(Some(said), seed) returns the flag and token. Some(NextWindow::Unknown) means “radio, but no token yet”, so the first top-up asks from the seed.
- Return whether the first track started. If it did not, start_radio still emits and persists the installed queue so the panel and saved state do not remain on the old one.
- A started radio is endless regardless of the autoplay setting. The autoplay setting only gates the initial play_song hydration, not a radio that already exists.
Splicing
- splice_radio_into keeps the history, the current track, and any unplayed manual adds. It truncates everything after the current track, re-adds manual adds, then appends the radio page with deduplication.
- It must:
- set radio_seed,
- clear the old radio_cont before installing the new one,
- increment queue.epoch so a playlist walk for the replaced queue stops,
- set source_id = None,
- set radio = true,
- reset radio_fails = 0.
- splice_radio_state computes the new token from NextWindow and the seed, calls splice_radio_into, then installs the token. An Unknown first page leaves no token, so the first top-up asks from the seed.
- If shuffle is sticky and on, re-snapshot the spliced order and shuffle the upcoming tracks. Manual adds stay pinned.
- Clear any primed lookahead that belonged to the old queue, emit, persist, prime the lookahead, and broadcast to Listen Together.
Planning a top-up
Add a pure plan_top_up(q, gen, autoplay_on) returning (anchor_video, seed, Option<token>, TopUp) or None.
Return None when:
- the queue is a guest,
- the caller’s generation no longer matches,
- q.hydrating == Some(gen) (the queue’s own play_song radio is already being fetched),
- repeat mode forbids top-up,
- the queue is not a radio and autoplay is off,
- the queue is a radio whose last page was its end (radio_exhausted),
- the tail is not near,
- there is no last track,
- the queue ends on a local file and has no pinned radio seed.
Repeat rules:
- Radio: Off and One allow top-up; All blocks it.
- Autoplay: only Off allows top-up. One and All block it.
Tail thresholds:
- Radio: top up when fewer than 5 tracks remain after the playing track (len - (current + 1) < 5).
- Autoplay: top up on the last two tracks counting the current one (len - current <= 2).
Anchor and seed:
- Anchor is the last queue item’s video id.
- Seed is q.radio_seed when pinned, otherwise RDAMVM<last_video>.
- A radio uses its radio_token(q) only when the token’s seed still equals q.radio_seed. A token from another seed is not this radio’s token; the plan asks from the seed instead.
Fetching and applying a window
extend_queue_radio uses try_lock and returns 0 if another top-up is in flight. extend_queue_radio_wait waits up to 20 seconds for the lock, then runs.
run_top_up rechecks guest status and generation after acquiring the lock, builds the plan, and calls follow_top_up.
follow_top_up:
- Token present: call radio_continuation(client, token, Some(anchor)).
- No token: call next(client, Some(anchor), Some(seed)).
- On error: warn, call apply_failed_window, return 0.
- On success: for radio, compute NextWindow from the page; for autoplay, pass no NextWindow.
- Call absorb_radio with the page, generation, seed, top-up kind, NextWindow, and the token the request used.
- For radio, if the first window added 0 tracks, follow up to 3 further windows immediately:
- stop if a window added something,
- stop if the generation changed,
- stop if no token remains,
- stop if the fetch fails,
- stop if absorb_radio reports the window is not ours.
This matters at the tail, where no other request may arrive before playback ends.
- For autoplay, preserve the existing blocked-songs escalation: if a window adds nothing and the user has blocked anyone, retry once through fetch_radio on the anchor.
Window classification
next_window(panel, rows, continuation):
- No panel → Unknown, even if a token somehow exists.
- Panel with a token → Token(token).
- Panel without a token and rows > 1 → Ended.
- Panel without a token and rows <= 1 → Unknown. A single seed row and nothing else is a dead radio, not the end of a sequence.
rows is the number of items in the parsed, filtered page, after hidden-video and blocked-song filtering.
radio_state(said, seed):
- Token(t) → Some((seed, Some(t))).
- Ended → Some((seed, None)).
- Unknown → None.
radio_install(radio, seed):
- radio = None → (false, None) for a chosen playlist/album.
- radio = Some(said) → (true, radio_state(said, seed)), or (true, None) if there is no seed.
Applying what the page said
apply_radio_said(q, seed, said, capped):
- Token: if not capped, set radio_cont = Some((seed, Some(token))).
- Ended: if not capped, set radio_cont = Some((seed, None)).
- Unknown while q.radio_cont.is_some(): clear radio_cont but keep radio = true. The next top-up asks from the seed.
- Unknown while q.radio_cont is already None: this was a seed request and it produced no radio page either. Set radio = false, clear radio_cont, and report that the queue is no longer a radio so the caller emits the queue and the UI restores the Autoplay switch.
capped means the window had more new tracks than the batch cap. Do not advance the token or the end state in that case; the tracks that did not fit are reachable only by asking for the same window again. Keep the old token.
Merging
merge_radio(items, fresh, existing, cap, autoplay):
- Skip ids already in the queue or appended in this call. Duplicates do not consume the cap.
- If a new track does not fit, set left = true and continue scanning.
- Insert only appended rows.
- Mark appended rows autoplay only for autoplay. Radio rows are untagged.
- Return (added, left).
- Batch sizes: AUTOPLAY = 20, RADIO = 50.
absorb_radio:
- Recheck generation; if it moved, return None.
- If this is autoplay and autoplay was switched off while fetching, return Some(0) without appending. A radio is not affected by that setting.
- Freshen the page against recent listening before applying the cap.
- Re-lock the queue.
- Reject the whole window when it no longer belongs to this queue and sequence:
- Autoplay window: accept only when is_radio(q) is false.
- Radio window: accept only when is_radio(q) is true and q.radio_seed == seed.
- If the request used a token, the queue’s current radio_token(q) must equal that token.
- If the request used the seed, radio_token(q) must be None and the radio must not be exhausted.
Reject rather than appending a stale or foreign window. Do not update its token either.
- For a radio window that lands, reset radio_fails = 0, even if it added zero tracks.
- Deduplicate against the queue as it is now, not a pre-fetch snapshot.
- Merge with the correct batch and tag.
- Apply NextWindow through apply_radio_said using left as capped.
- If rows were appended and shuffle is on, append the same rows to shuffle_orig.
- Trim played rows only when something was appended. Keep KEEP_PLAYED = 200. Adjust current and lookahead_loaded by the number dropped, and filter shuffle_orig down to ids still present so un-shuffle cannot resurrect trimmed tracks.
- If the queue was demoted from radio to autoplay, emit the queue immediately.
- If something was appended: log the correct kind, emit either the appended delta (no trim) or the full queue (trim shifted indices), persist, broadcast to Listen Together, and prime the lookahead.
Tail playback and Skip
skip_repeat(q):
- For a non-radio queue under Repeat One, return Repeat All. A manual Skip escapes the repeat-one loop and wraps.
- For a radio under Repeat One, keep Repeat One. Its tail is topped up instead of wrapped.
next_in_queue at the tail:
- Wait for any in-flight top-up with extend_queue_radio_wait(gen).
- If it added tracks or the queue already has a next track, play current + 1.
- Else, if the queue is under Repeat One, play index 0. This preserves the old Skip behavior for a radio that has genuinely ended and cannot continue.
- Otherwise do nothing; the queue is over.
on_track_ended fallback when there is no next index:
- Wait for the in-flight top-up before deciding the queue is over.
- If the generation changed while waiting, return.
- Bump the generation, then if the wait added tracks or has_next() is true, seek to the next track and start it.
- Otherwise pause and update the OS media widget, Discord, and Listen Together as before.
autoplay_changed:
- On: trigger a top-up for a near tail.
- Off: drop upcoming rows tagged autoplay. Radio rows are untagged, so the radio tail survives.
- Also trigger a top-up after shuffle toggling when the tail is near.
Failures and token recovery
apply_failed_window(q, top_up, seed, from_token, refused):
- Ignore autoplay failures. They never spend a radio token’s tries.
- Ignore a failure unless window_may_land says the failed window still belongs to the current queue and sequence.
- Increment radio_fails.
- A refusal (401/403) drops the token immediately: reset radio_fails = 0, set radio_cont = None.
- Otherwise after RADIO_FAILS = 3 consecutive failures: reset radio_fails = 0, set radio_cont = None.
- Dropping the token means the next top-up asks from the seed again. The queue remains a radio.
- A successful radio window resets the count.
Do not let a failure from a radio spliced in behind the playing track, or from another seed/sequence, drop the current radio’s token or spend its tries.
Persistence, restore, and previous context
- Do not persist radio_cont or radio_fails.
- On restore, read radio and radio_seed as before, set radio_cont = None, and let the first top-up ask from the seed and the last track. That first window then teaches the queue its token or its end.
- After the existing saved-fingerprint check, if the saved queue is a radio (radio && radio_seed.is_some()), clear the autoplay tag on every restored row and on every row in shuffle_orig. Older builds topped radios up through autoplay and saved that tag; leaving it would draw an Autoplay divider and let switching autoplay off cut the radio. Do this after the fingerprint check because the fingerprint hashes that tag.
- A non-radio queue keeps its tags. A raw radio flag without a seed also keeps them.
- PrevContext must carry radio_cont when a queue is set aside, and restore it with the rest of the queue. Reset radio_fails = 0 on restore because the failures belonged to the replaced queue.
- When a guest rebuilds the queue from the host, clear radio_seed, radio_cont, and radio_fails. Guests never top up.
Shuffle and manual adds
- Radio top-ups must survive shuffle. Append new rows to shuffle_orig when shuffle is on.
- When trimming played rows, remove those ids from shuffle_orig too.
- Splicing a radio under sticky shuffle re-snapshots the new order and shuffles upcoming rows; manual adds stay pinned behind the current track.
- enqueue_at uses is_radio(q), not the raw flag. For a radio, a manual add goes after the current track and after any existing manual queued/queued_end rows, before the generated tail. For a non-radio queue it goes at the end. This now also applies to a queue that became a radio through play_song hydration.
UI
- Add optional radio?: boolean to QueueState and QueueIndex in ui/src/lib/api.ts.
- Include radio: q.radio when mapping queue state in ui/src/lib/player.svelte.ts.
- Emit radio: is_radio(&q) in the full queue state, the queue-index delta, and the queue snapshot. When a queue is demoted to autoplay, emit again so the panel learns immediately.
- In QueueList.svelte, hide the Autoplay switch when playback.queue.radio is true, and hide the “queue ends” line under the same condition. Keep the existing canEdit, left.count, view.autoplay.length, prefs.autoplay, and repeat checks.
- A radio has no Autoplay switch and no “queue ends” line because YouTube generates all of it and keeps it going.
Acceptance criteria
- parse_next takes a token from the radio panel and ignores a comments/Related continuation.
- A panel without a continuation and no panel at all produce different results: ended versus unknown.
- radio_continuation sends one /next request with the token, no video/playlist id, and returns the next panel token.
- A hidden or blocked track is filtered from a radio window, except the anchor track passed to the call.
- play_song with autoplay enabled pins the radio and its token when the hydrated page has more than one track; with autoplay off it leaves a single-song queue.
- A dead radio page (seed only) does not get pinned and does not mark a sequence ended.
- start_radio installs the radio flag and token atomically with the queue; if the first track fails to start, the installed queue is still emitted and persisted.
- Starting a radio on the already-playing seed splices without interrupting playback and preserves history, current track, and manual adds.
- A radio top-up uses the token when present and the seed otherwise.
- A radio tops up with autoplay off, with fewer than five tracks left, taking up to 50 untagged tracks.
- Autoplay still waits for the last two tracks, takes 20, and tags its rows.
- A radio under Repeat One tops up; a radio under Repeat All does not. Autoplay does not top up under any repeat mode.
- A cap-cut window does not advance the token or mark the radio ended; the same window is requested again.
- A panel with more than one row and no continuation marks the radio ended; a panel with one row and no continuation stays unknown.
- A window that adds zero tracks still advances the token, and the radio follows up to three more windows at the tail.
- A stale token, a token from another seed, a radio window against a non-radio queue, and an autoplay window against a radio are all rejected whole.
- A refusal drops the token at once; three ordinary failures drop it; a hiccup does not.
- A failure from a foreign sequence does not spend the current radio’s tries.
- An unknown window while following a token clears the token but keeps the radio; an unknown window asked from the seed demotes the queue to autoplay and emits the queue.
- Skip at the tail waits for an in-flight top-up and uses what it brought; a genuinely ended radio under Repeat One restarts at index 0.
- The track-end fallback waits for an in-flight top-up before pausing.
- A restored radio has its old autoplay tags stripped from both the queue and the shuffle snapshot, and starts with no token.
- A kept previous context restores its radio token and resets the failure count.
- Shuffle on/off does not duplicate, lose, or resurrect radio rows.
- A manual add in a radio goes in front of the generated tail.
- The UI hides the Autoplay switch and the “queue ends” line for a radio, and shows them again when the queue is demoted to autoplay.
Coverage boundary: this prompt covers the runtime behavior of the seven source/UI files in the feature diff. The branch’s separate manual-build GitHub workflow is not part of this behavior.
Known edge: a window requested from the seed with no token cannot be distinguished from a later same-seed sequence until a token exists, so a re-splice in that narrow window can still accept a stale page.