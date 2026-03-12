---
name: meeting-video-analyzer
description: Automatically processes meeting video recordings and transcripts to generate comprehensive meeting summaries, action items, key decisions, and visual step-by-step guides. Use this skill whenever the user uploads a meeting recording (.mp4, .mov, .webm, .mkv) and/or a transcript file (.txt, .vtt, .srt, .md, .docx), or mentions wanting to analyze a meeting, extract action items from a recording, create a meeting summary, generate a knowledge transfer guide, or process meeting notes. Also trigger when the user says things like "summarize this meeting", "what happened in this call", "extract action items", "create a guide from this recording", or "process my meeting video".
---

# Meeting Video Analyzer

Automatically turns meeting recordings and transcripts into structured, actionable deliverables — summaries, action items, decisions, and visual guides.

## Inputs

The user provides one or both of:
- **Video file**: `.mp4`, `.mov`, `.webm`, `.mkv`
- **Transcript file**: `.txt`, `.vtt`, `.srt`, `.md`, `.docx`

## Pipeline

### Step 1: Validate Inputs

Check what the user provided:
- If **video only** (no transcript): Extract audio and attempt transcription (see Step 2a), then extract frames (Step 2b)
- If **transcript only** (no video): Skip frame extraction, go straight to analysis (Step 3)
- If **both video + transcript**: Extract frames from video (Step 2b), then analyze together (Step 3)

Tell the user what you found and what you're going to do.

### Step 2a: Audio Extraction & Transcription (only if no transcript provided)

If the user only provided a video with no transcript:

```bash
# Check if whisper is available, install if needed
which whisper || pip install openai-whisper --break-system-packages

# Extract audio from video
ffmpeg -i "<video_file>" -vn -acodec pcm_s16le -ar 16000 -ac 1 /home/claude/meeting_audio.wav

# Transcribe with whisper
whisper /home/claude/meeting_audio.wav --model base --output_format txt --output_dir /home/claude/
```

If whisper is unavailable or fails, inform the user:
> "I couldn't transcribe the audio automatically. Please upload a transcript file (you can export one from Zoom, Teams, Google Meet, or use a service like Otter.ai) and I'll process everything together."

### Step 2b: Extract Key Frames from Video

Extract frames at regular intervals for visual context:

```bash
# Create frames directory
mkdir -p /home/claude/meeting_frames

# Extract 1 frame every 5 seconds (good balance of coverage vs volume)
ffmpeg -i "<video_file>" -vf "fps=1/5" /home/claude/meeting_frames/frame_%04d.png

# Count frames extracted
FRAME_COUNT=$(ls /home/claude/meeting_frames/*.png | wc -l)
echo "Extracted $FRAME_COUNT frames"
```

If there are more than 200 frames (long meeting), reduce to 1 frame every 10 seconds:
```bash
rm -rf /home/claude/meeting_frames/*
ffmpeg -i "<video_file>" -vf "fps=1/10" /home/claude/meeting_frames/frame_%04d.png
```

### Step 3: Analyze Transcript

Read the transcript and identify:

1. **Participants**: Who spoke (from speaker labels if available, or infer from context)
2. **Topics discussed**: Major topic shifts and themes
3. **Key decisions**: Any decisions made or agreed upon
4. **Action items**: Tasks assigned, with owner and deadline if mentioned
5. **Questions raised**: Open questions that weren't resolved
6. **Screen shares / demos**: If frames show screen content, match to transcript timestamps

### Step 4: Match Frames to Transcript (if video was provided)

Review the extracted frames and correlate them with transcript content:
- Identify frames showing **screen shares, slides, diagrams, or demos**
- Skip frames that are just webcam feeds with no useful visual content
- For each useful frame, note the approximate timestamp and what was being discussed
- Select the **10-20 most informative frames** that add context beyond the transcript

### Step 5: Generate Outputs

Create a single comprehensive markdown document with these sections:

```markdown
# Meeting Summary: [Meeting Title / Topic]
**Date**: [extracted or inferred from context]
**Duration**: [from video length or transcript timestamps]
**Participants**: [list]

## TL;DR
[2-3 sentence executive summary of the meeting]

## Key Decisions
- [Decision 1]
- [Decision 2]

## Action Items
| Owner | Action | Deadline | Priority |
|-------|--------|----------|----------|
| [name] | [task] | [date if mentioned] | [High/Med/Low] |

## Discussion Summary
### [Topic 1]
[Summary of discussion, key points, any disagreements or concerns raised]

### [Topic 2]
[Summary...]

## Visual Guide (from screen shares/demos)
[If frames were extracted and useful screen content was found, include
a step-by-step walkthrough with frame references]

### Step: [Description]
![Frame](frame_XXXX.png)
[Explanation of what's shown and what was discussed at this point]

## Open Questions / Follow-ups
- [Unresolved question 1]
- [Unresolved question 2]

## Raw Notes
[Optional: detailed chronological notes for reference]
```

### Step 6: Deliver Results

1. Save the summary document as a `.md` file in the output directory
2. If useful frames were identified, copy them alongside the summary
3. Also generate a **standalone action items list** as a separate file for easy copy-paste into Jira/Linear/etc.
4. Present the files to the user

## Output Files

- `meeting-summary-[date-or-title].md` — Full meeting summary with all sections
- `action-items-[date-or-title].md` — Standalone action items table
- `visual-guide/` — Directory with selected key frames (only if video provided and useful visuals found)

## Edge Cases

- **Very long meetings (>2 hours)**: Break the summary into major segments/agenda items rather than one long narrative
- **No clear speaker labels in transcript**: Note this limitation and summarize by topic rather than by speaker
- **Poor video quality / only webcam feeds**: Skip the visual guide section entirely and note that no useful screen content was found
- **Multiple languages**: Identify the primary language and note any code-switching
- **Partial transcript**: Work with what's available and flag gaps

## Tips for Best Results

When talking to the user, mention these tips if relevant:
- **Zoom** exports transcripts via Recording settings or from the cloud recording page
- **Google Meet** transcripts are saved to Google Drive automatically if transcription was enabled
- **Microsoft Teams** transcripts are available in the meeting chat or recording details
- **Otter.ai**, **tl;dv**, **Fireflies.ai** can retroactively transcribe recordings
- For best visual guides, meetings with screen shares produce the most useful frame extractions
