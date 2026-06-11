---
name: flask-review
description: Upload a video to Flask (flask.do) for human feedback and iterate on the review. Use when the user asks to share a video/render for review or feedback, mentions Flask, or when you have produced a video (Remotion, HyperFrames, ffmpeg, screen recording) that a human should approve before it ships. Covers the full review loop - upload, instant share link, waiting for feedback, reading recording transcripts, and uploading revisions as versions.
---

# Flask Review Loop

Flask is a video review platform. Humans leave feedback as timestamped comments and
voice/screen recordings; Flask AI extracts transcripts and summaries so you can read
every piece of feedback as text, anchored to a video timestamp.

All access goes through the Flask MCP server (`flask`), included with this plugin
(https://api.flask.do/api/mcp/mcp). If its tools are unavailable, the user needs to
authenticate once: tell them to run `/mcp`, pick `flask`, and complete the browser
sign-in.

## The loop

1. **Start the upload** - `upload_file_start(folder_id, file_size, title, content_type)`.
   Find a folder with `contents()` if you do not have one. The response includes the
   shareable link.
2. **Give the user the link IMMEDIATELY** - before uploading any bytes. They can open
   it right away and watch the upload and processing happen live. Never wait for
   "ready" to share the link.
3. **Upload the bytes** with the exact curl command from the tool's `next_step`
   (a presigned PUT - the Content-Type header is required).
4. **Finalize** with `upload_file_complete(asset_id)`. The video is watchable
   instantly via preview playback. Do not poll `asset_status`; there is nothing to
   wait for.
5. **Listen for feedback** - `wait_for_feedback(asset_id)`. It blocks up to ~45s and
   returns new feedback the moment it is left. Loop it, passing the returned
   `next_since` each time, while the user reviews.
6. **Read the feedback properly**:
   - `content` is the verbatim text of a comment; `timestamp` is seconds into the video.
   - Recording items carry `recording.transcript` (what the reviewer said, sliced to
     this segment) and `recording.ai_summary`. Treat the transcript as the feedback.
   - To see exactly what the reviewer was looking at, extract the frame at the
     feedback timestamp from your local render:
     `ffmpeg -ss <timestamp> -i <file> -frames:v 1 /tmp/frame.png` and read the image.
7. **Implement the changes**, re-render, and upload the revision as a NEW VERSION of
   the same asset: pass `version_of: "<asset_id>"` to `upload_file_start` AND
   `upload_file_complete`. Never create a separate asset for a revision.
8. **Share the `version_stack_url`** - it permanently shows the newest version, so the
   user keeps one link for the whole iteration.
9. Go back to step 5. Repeat until the user says they are done.

## Rules

- The share link exists from step 1. Surfacing it early is the whole point.
- Quote feedback verbatim when reporting it to the user; include each item's `url`.
- One asset per deliverable, versions for iterations, `version_stack_url` for sharing.
- Uploads cap at 5GB. Requires edit access on the folder and a paid/trialing Flask
  team - if the server returns a subscription or entitlement error, relay it to the
  user verbatim.
