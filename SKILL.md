---
name: capcut-highlight-draft
description: Use CapCut Highlight Draft when the user wants a meeting recording turned into a high-signal short-video highlight package and visible CapCut/剪映 project. This workflow starts from a Tencent Meeting or Feishu/Lark meeting recording link, prioritizes standalone high-value moments over meeting structure, creates a Markdown editing plan with highlight scoring, cuts the selected clips, and installs the generated CapCut/剪映 draft directly under the app draft root such as /Users/katerina/Movies/JianyingPro/User Data/Projects/com.lveditor.draft.
---

# CapCut Highlight Draft

## Purpose

CapCut Highlight Draft defines the local workflow for turning meeting recordings into high-signal short-video highlight clips and a CapCut/剪映 project. It is not just a CLI wrapper. The expected output is a working folder that contains source material, an editing plan Markdown file, cut clips, optional compilation videos, and a final CapCut/剪映 draft folder installed directly under the app's draft root so it appears in the CapCut/剪映 app.

This is a highlight-first skill. It should not default to a balanced meeting summary or agenda-preserving recap. Prioritize clips that can stand alone as useful, surprising, persuasive, or shareable short-video material.

Use this skill when the user says things like:

- "我给你一个腾讯会议录制链接，帮我剪视频集锦"
- "这是飞书会议录制，帮我挑片段并生成剪映工程"
- "把这段会议做成几个切片，放到指定文件夹"
- "生成剪映工程文件放到某个位置"
- "按高光片段优先剪，不要做会议精简版"
- "帮我从这个录屏里挑能单独发的短视频素材"

## Local Tools

Known local tools on this machine:

```bash
/Users/katerina/.local/bin/cutCLI --version
/Users/katerina/.nvm/versions/node/v22.21.1/bin/capcut --help
ffmpeg -version
ffprobe -version
```

Observed versions when this skill was created:

- `cutCLI`: `1.3.10`
- `capcut-cli`: npm package `capcut-cli@0.2.2`, command `capcut`

Use `ffmpeg`/`ffprobe` for deterministic media operations. Use `capcut-cli` or `cutCLI` for inspecting or assisting with CapCut/剪映 drafts when they fit the task. If the CLI cannot produce the needed draft structure, generate or repair `draft_info.json` directly from a known-good template.

## Workflow

### 1. Ingest The Recording Link

The first user input is usually a recording link:

- Tencent Meeting recording link
- Feishu/Lark meeting recording link
- Existing local source video path

Create a working folder for the job before downloading or cutting. Use the user-specified folder if given; otherwise create a clear subfolder under the current video collection path.

Recommended folder shape:

```text
<workdir>/
├── source/
├── clips/
├── draft-build/
├── checks/
├── edit-plan.md
├── concat.txt
└── notes.md
```

For Feishu/Lark meeting recordings, prefer the available Lark meeting/minutes skills or CLI flow to fetch recording metadata, transcript, summary, and media where possible. For Tencent Meeting links, use the browser or available authenticated download path if needed. If the link requires user login or manual permission, say exactly what is blocked and continue with any local material already available.

Record the source in `notes.md`:

```markdown
# Source

- Link:
- Local file:
- Retrieved at:
- Duration:
- Transcript:
- Notes:
```

### 2. Create The Markdown Editing Plan

Before cutting, write a Markdown plan in the folder. This is required.

File name:

```text
edit-plan.md
```

The plan should be based on the original media, transcript, title/context, and any user instructions. It should not be a vague outline; it should name the intended clips and include time ranges.

Highlight-first selection rules:

- Do not preserve meeting structure by default. Opening remarks, agenda setup, speaker transitions, and closing thanks are usually excluded unless they contain a strong hook.
- Prefer moments with a clear pain point, strong opinion, useful workflow, demo payoff, concrete before/after result, memorable quote, number, objection, or operational lesson.
- Each selected clip should make sense when viewed alone, without needing the full meeting context.
- Prefer 20-60 seconds per clip. Allow up to 90 seconds only when the demo payoff needs continuity. Avoid long clips that merely summarize context.
- Favor clips with visual evidence or on-screen change: product demo, workflow execution, output appearing, before/after comparison, data movement, or concrete UI operation.
- Avoid generic claims, long setup, repeated thanks, "大家好", "我开始投屏", "时间有限", "会后发资料", and other low-signal connective tissue.
- If transcript confidence is low, use transcript to find candidate regions, then inspect frames/audio around those regions before finalizing.

Candidate scoring:

Score candidate clips out of 10 before cutting. Keep candidates scoring `7+`; keep `6` only if it fills an important topic gap.

| Dimension | Points | What qualifies |
|---|---:|---|
| Hook / pain point | 0-2 | Clear problem, tension, misconception, or surprising claim |
| Practical value | 0-2 | Viewer can reuse the workflow, command, pattern, or decision rule |
| Specificity | 0-2 | Contains concrete example, number, tool, scenario, or output |
| Demo / visual payoff | 0-2 | Screen changes, result appears, before/after is visible |
| Standalone shareability | 0-2 | Clip works with its own title and does not need surrounding context |

Recommended structure:

```markdown
# Editing Plan

## Source

- Recording:
- Local media:
- Duration:
- Transcript/material basis:

## Clip Plan

| Clip | Short-Video Title | Start | End | Duration | Score | Highlight Reason | Output |
|---|---|---:|---:|---:|---:|---|---|
| 1 |  | 00:00:00 | 00:00:00 | 0s | 0/10 |  | clips/clip1_name.mp4 |

## Rejected Candidates

| Candidate | Start | End | Score | Why rejected |
|---|---:|---:|---:|---|
|  | 00:00:00 | 00:00:00 | 0/10 |  |

## Style Notes

- Aspect ratio:
- Keep/remove:
- Captions:
- Intro/outro:
- Audio:

## Execution Notes

- ffmpeg command pattern:
- Draft output path:
- Validation checklist:
```

If the transcript is missing, inspect the video directly enough to propose useful ranges. If confidence is low, mark uncertain sections clearly in the plan and keep the clips reviewable.

### 3. Cut The Selected Clips

Use `ffprobe` to confirm source duration and media properties:

```bash
ffprobe -v error -show_entries format=duration -of default=nk=1:nw=1 "$source"
ffprobe -v error -select_streams v:0 -show_entries stream=width,height,r_frame_rate -of default=nw=1 "$source"
```

Cut each planned segment into `clips/`.

Fast copy when cuts can be approximate:

```bash
ffmpeg -y -ss "$start" -to "$end" -i "$source" -c copy "$out"
```

Accurate cuts when frame/audio precision matters:

```bash
ffmpeg -y -ss "$start" -to "$end" -i "$source" \
  -c:v libx264 -preset veryfast -crf 18 \
  -c:a aac -b:a 160k \
  "$out"
```

After each batch:

- Run `ffprobe` on outputs and compare durations with `edit-plan.md`.
- Put screenshots or frame checks in `checks/` when visual verification matters.
- If making a full compilation, create `concat.txt` and concatenate clips:

```bash
ffmpeg -y -f concat -safe 0 -i "$workdir/concat.txt" -c copy "$workdir/highlights_no_watermark.mp4"
```

If streams differ, re-encode the concat output.

### 4. Generate Opening Freeze Title Card, Captions, And Cover

Every created CapCut/剪映 draft should start with a 1-second opening title card before the first selected clip. The card should freeze the first frame of the video and layer an editable main title above it. Do not add a blue title frame/panel, do not use a full-screen `#0023ff` background, and do not bake the title text into the MP4.

Default title card style:

- Duration: `1s`
- Canvas: `1920x1080` unless the project uses another explicit ratio
- Background video layer: the first frame of the first selected clip, frozen for `1s`; this should contain no blue panel/frame and no title text
- Main title text layer: a separate CapCut/剪映 text segment centered on the frame; this must remain editable in the app
- Main title font size: fixed `18`
- Main title font color: white
- Main title content: the video's main title, not the clip title
- Use the first frame of this title card as `draft_cover.jpg`

Stable ffmpeg pattern for the freeze background:

```bash
printf '%s' '<main title>' > "$workdir/checks/title_text.txt"

ffmpeg -y -i "$workdir/clips/clip01_first_clip.mp4" -frames:v 1 "$workdir/checks/opening_freeze_frame.jpg"

# Freeze-only background. Do not draw the blue panel or title text into this MP4.
ffmpeg -y -loop 1 -i "$workdir/checks/opening_freeze_frame.jpg" -t 1 -r 25 \
  -vf "scale=1920:1080:force_original_aspect_ratio=increase,crop=1920:1080" \
  -c:v libx264 -pix_fmt yuv420p clips/clip00_title_card.mp4

# The draft-box cover may be a rendered composite, but the timeline must keep title text editable.
ffmpeg -y -loop 1 -i "$workdir/checks/opening_freeze_frame.jpg" -t 1 -r 25 \
  -vf "scale=1920:1080:force_original_aspect_ratio=increase,crop=1920:1080,drawtext=fontfile='/System/Library/Fonts/Supplemental/Arial Unicode.ttf':textfile='$workdir/checks/title_text.txt':fontcolor=white:fontsize=18:x=(w-text_w)/2:y=(h-text_h)/2" \
  -frames:v 1 checks/title_cover.jpg
```

When adding the title card to a draft:

- Copy `clip00_title_card.mp4` into the draft's `assets/video/`.
- Shift all original video segments by `+1s`.
- Add the freeze-only title card video segment at `0s` for `1s`.
- Do not add a blue panel/frame overlay.
- Add the main title as a separate text segment at `0s` for `1s`, centered with fixed font size `18`; the user must be able to edit this text in the app.
- Ensure the title card segment is also the first item in the video track `segments[]`; some Jianying builds may display segments by array order after opening the draft.
- Increase the draft duration by `1s`, or recompute duration from the max segment end.
- Copy `checks/title_cover.jpg` to `$draft_dir/draft_cover.jpg`.
- Update `draft_info.json.static_cover_image_path`, `draft_meta_info.json.draft_cover`, and the root index `draft_cover` to point at the installed cover.

Every created draft should also include a text/subtitle track.

Default subtitle style:

- Font color: `#0023ff`
- Font size: around `8` for 1920x1080. `12` may still look too large in Jianying; `34` is much too large and can cover the video.
- Position: bottom-center. On this machine, `y=-0.78` places text near the bottom; positive `y` may push text toward the top.
- Width: keep `line_max_width` around `0.74` and force wrapping for long subtitles.
- One subtitle segment per planned clip, aligned to that clip's start and duration after accounting for the 1-second title card

Useful `capcut-cli` pattern:

```bash
capcut shift-all "$draft_dir" +1s --track video
capcut add-video "$draft_dir" "$draft_dir/assets/video/clip00_title_card.mp4" 0s 1s
capcut add-text "$draft_dir" 0s 1s "<main title>" --font-size 18 --color '#FFFFFF' --align 1 --x 0 --y 0 --track-name title-text
capcut add-text "$draft_dir" 1s 29s "<subtitle>" --font-size 8 --color '#0023ff' --align 1 --x 0 --y -0.78
```

After inserting title card and subtitles, validate:

- The first video segment label/path is `clip00_title_card.mp4`, starts at `0`, and lasts `1000000` microseconds.
- `clip00_title_card.mp4` is freeze-only; it must not contain a blue title panel/frame or the main title text.
- There is no title-panel/blue-frame overlay track by default.
- The main title is a separate editable text material from `0` to `1000000` microseconds with font size `18`.
- Video track has `planned_clip_count + 1` segments.
- Text track has at least `planned_clip_count` segments.
- `materials.texts[].text_color` is `#0023ff`.
- `materials.texts[].font_size` and the size inside each text `content.styles[]` are both around `8`.
- Draft duration equals the maximum end time across video and text segments.

### 5. Generate And Install The CapCut/剪映 Draft Project

The final deliverable must be a CapCut/剪映 draft folder placed directly inside the app's draft root and registered in the root draft index. Do not leave the final draft only inside the working folder; the app will not show it there. Also do not stop after copying the draft folder into the root: JianyingPro relies on `root_meta_info.json` to show drafts in the草稿箱.

On this machine, prefer this JianyingPro draft root:

```text
/Users/katerina/Movies/JianyingPro/User Data/Projects/com.lveditor.draft
```

Fallback CapCut root, if present on another machine:

```text
/Users/katerina/Movies/CapCut/User Data/Projects/com.lveditor.draft
```

The final draft path should look like:

```text
/Users/katerina/Movies/JianyingPro/User Data/Projects/com.lveditor.draft/<project-name>_剪映工程
```

Use `<workdir>/draft-build/<project-name>_剪映工程` only as a staging directory while generating or repairing files. After validation, copy or move the complete draft folder into the draft root.

Check the target root before installing:

```bash
jy_root="/Users/katerina/Movies/JianyingPro/User Data/Projects/com.lveditor.draft"
capcut_root="/Users/katerina/Movies/CapCut/User Data/Projects/com.lveditor.draft"
if [ -d "$jy_root" ]; then
  draft_root="$jy_root"
elif [ -d "$capcut_root" ]; then
  draft_root="$capcut_root"
else
  echo "No CapCut/Jianying draft root found"
  exit 1
fi
```

Preferred approaches:

1. Use `capcut-cli` or `cutCLI` when the command can create the intended draft reliably.
2. Otherwise, copy a minimal known-good CapCut draft template and generate/repair the project JSON.

Useful inspection commands:

```bash
capcut info "$draft_dir"
capcut tracks "$draft_dir"
capcut segments "$draft_dir" --track video
cutCLI draft info "$draft_dir"
```

When directly generating/repairing `draft_info.json`, follow these patterns:

- Backup before any edit.
- Set canvas width/height based on source or requested output, commonly `1920x1080`.
- Set `duration` to the sum of segment durations in microseconds.
- Add one `materials.videos[]` entry per clip, with `path`, `media_path`, `material_name`, `duration`, `width`, `height`, and `has_audio`.
- Add one video track with contiguous `segments[]`.
- Each segment should use:
  - `target_timerange.start`: cumulative timeline start in microseconds
  - `target_timerange.duration`: clip duration in microseconds
  - `source_timerange.start`: `0`
  - `source_timerange.duration`: clip duration in microseconds
- Keep helper material references consistent when cloning an existing draft.

Backup pattern:

```bash
stamp="$(date +%Y%m%d%H%M%S)"
mkdir -p "$draft_dir/.codex_backup_$stamp"
cp "$draft_dir/draft_info.json" "$draft_dir/.codex_backup_$stamp/draft_info.json"
cp "$draft_dir/draft_meta_info.json" "$draft_dir/.codex_backup_$stamp/draft_meta_info.json" 2>/dev/null || true
```

Install pattern:

```bash
final_draft="$draft_root/$(basename "$draft_dir")"
test ! -e "$final_draft" || mv "$final_draft" "$final_draft.backup.$(date +%Y%m%d%H%M%S)"
cp -R "$draft_dir" "$final_draft"
```

After copying, update the root draft index:

```text
$draft_root/root_meta_info.json
```

The index must contain one `all_draft_store[]` entry for the new draft. Back up this file before editing.

Required root index fields for a visible draft:

- `draft_fold_path`: final draft folder path under `com.lveditor.draft`
- `draft_json_file`: `$draft_fold_path/draft_info.json`
- `draft_cover`: `$draft_fold_path/draft_cover.jpg`
- `draft_name`: project name shown in草稿箱
- `draft_id`: draft id, preferably uppercase in root index
- `draft_root_path`: the `com.lveditor.draft` root
- `draft_is_invisible`: `false`
- `streaming_edit_draft_ready`: `true`
- `tm_draft_create` and `tm_draft_modified`: microsecond-level timestamps, not millisecond timestamps
- `tm_duration`: draft duration in microseconds

Set `root_meta_info.json.draft_ids` to the current number of `all_draft_store` entries. Existing files on this machine show `draft_ids` is a count, not an array.

Also use microsecond-level timestamps in the draft files where local examples do so:

- `draft_info.json.create_time`
- `draft_info.json.update_time`
- `draft_meta_info.json.tm_draft_create`
- `draft_meta_info.json.tm_draft_modified`

If the app is already open, fully quit and reopen CapCut/剪映 after updating `root_meta_info.json`; the草稿箱 may not reload the root index while running.

Validation:

```bash
node - <<'NODE' "$draft_dir/draft_info.json"
const fs = require('fs');
const p = process.argv[2];
const j = JSON.parse(fs.readFileSync(p, 'utf8'));
const videos = j.materials?.videos || [];
const videoById = Object.fromEntries(videos.map(v => [v.id, v]));
const videoTrack = (j.tracks || []).find(t => t.type === 'video');
const segs = videoTrack?.segments || [];
console.log({name:j.name, duration:j.duration, canvas:j.canvas_config, videos:videos.length, segments:segs.length});
let end = 0;
for (const s of segs) {
  if (s.target_timerange.start !== end) console.log('gap_or_overlap', {expected:end, actual:s.target_timerange.start, id:s.id});
  end = s.target_timerange.start + s.target_timerange.duration;
}
const first = segs[0] && videoById[segs[0].material_id];
if (!first || !String(first.path || first.material_name || '').includes('clip00_title_card')) console.log('missing_initial_title_card');
if (end !== j.duration) console.log('duration_mismatch', {timeline:end, draft:j.duration});
NODE
```

Then inspect with `capcut info` and open the draft only after JSON validation passes.

## Optional Breath-Cut / 剪气口 Step

If the user asks for a tighter version, run silence detection on the compilation:

```bash
ffmpeg -i "$input" -af silencedetect=noise=-38dB:d=0.35 -f null - 2> "$workdir/silence_detect_38db_035.log"
```

Use rules like:

- Silence `< 0.65s`: keep all.
- Silence `0.65s-1.2s`: keep `0.35s`.
- Silence `1.2s-5s`: keep `0.45s`.
- Silence `5s-30s`: keep `0.80s`.
- Silence `>=30s`: keep `1.20s`.

Store the resulting plan as `剪气口_裁剪清单.json`, rebuild the tighter video, and optionally generate a separate one-video CapCut draft for that output.

## Guardrails

- Always create `edit-plan.md` before cutting.
- Do not silently invent clip choices when the source cannot be accessed; state the access issue and use available material.
- Do not edit a CapCut/剪映 draft while the app has it open.
- Never edit `draft_info.json` without a timestamped backup.
- Prefer `ffprobe` durations over guessed durations.
- Keep all generated outputs inside the working folder unless the user specifies another destination.
- Clearly distinguish source material, planned cuts, actual cuts, and final draft output in the final response.
