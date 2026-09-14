---
name: uo-aigc-model-image-processor
description: Batch workflow for reading apparel AIGC modification tasks from Excel spreadsheets and processing SKU folders with ToAPIs image generation. Use when Codex needs to handle Chinese "修改意见" columns for model-image missing-angle generation, model pose edits, adding model angles, same-style color changes, same-person model replacement, product try-on from flat lays, Eagle MCP model-base selection, per-SKU logs, and 3:4 2K output naming.
---

# UO AIGC Model Image Processor

Use this skill for spreadsheet-driven apparel model-image AIGC batches. The core rule is: classify images first, manually verify references, then generate with short task-only prompts.

## Workflow

1. Read every numbered or line-separated item in each `修改意见` cell. Preserve item order and evaluate every item; do not stop after item 1.
2. Process only model-image missing, pose/action edits, angle additions, same-style recolor/change-outfit, same-person replacement, and directly related generation steps.
3. Skip and audit non-generation edits, including top/bottom whitespace, resolution or dimensions, adding a white background/exporting JPG, and cropping/trimming to the upper body or garment. A cell may contain both skipped and actionable items; process only its actionable items.
4. For every target SKU folder, run visual classification before generation:
   - image kind: `model`, `flat`, `detail`, `other`
   - view: `front`, `side_3_4`, `back`, `unknown`
   - garment type and visible product side
   - product fit/silhouette notes, such as `regular`, `fitted`, `boxy`, `oversized`, `baggy`, `jorts`, or short combinations like `oversized mesh jersey`
   - model gender, model angle, pose notes
5. Create a contact sheet, `AIGC任务筛选.json`, and an `AIGC视觉识别.json` file; the runner also writes ASCII alias `aigc_image_analysis.json`. Check the screening report, then visually inspect and correct one classification file before generation.
6. Select reference images from the corrected classification. Product accuracy is the first dependency:
   - front/side model edits use the front flat lay as product reference.
   - back model edits use the back flat lay as product reference.
   - recolor, garment replacement, and try-on must happen before pose expansion or generation of another view.
   - only a model image already wearing the accurate product may be used as the source for pose or ordinary angle expansion.
   - ordinary angle expansion uses that product-accurate model image only.
   - back-view expansion uses the product-accurate front model image, front flat lay, and back flat lay together.
   - garment try-on uses one model image plus the correct flat lay.
   - color change uses the just-generated/current model image plus the correct flat lay.
7. Read all actionable items in spreadsheet order, then reorder their atomic steps by dependency within each SKU: product correction first, identity normalization when requested, then missing views and pose edits. Do not combine angle generation with recolor or try-on in one prompt.
   - If an actionable item does not match a known task type, visually infer whether it can be completed in one generation or needs dependent steps.
   - Select the exact references for each atomic step and write concise task-specific prompts in `custom_steps` inside the verified classification JSON.
   - Do not generate an unmatched task until this plan is complete; never fall back to a vague generic prompt.
8. Generate images with ToAPIs using `size=3:4`, `resolution=2K`, 600-second timeout, and 3 retries.
9. For every processed SKU folder, create a `过程文件` folder before generation. Put intermediate step outputs and process artifacts there when a generated image is only a bridge to a later step.
10. Save final outputs in each SKU folder as `AIGC+修改内容+序号`.
11. Write `AIGC处理日志_<SKU>.txt` in each SKU folder with all actionable/skipped items, reference image paths, prompts, API request metadata, and responses/errors.

## Prompt Rules

Keep prompts short and task-specific. Do not include SKU, product type labels, background constraints, watermarks, API settings, or execution notes.

If the user explicitly invokes `$uo-image-prompt-reference`, read it only as a non-binding prompt and reference-selection aid. Continue to analyze the actual task and images, and make the final prompt and reference choices yourself. Never read or apply that skill when the user has not explicitly requested it.

Before sending a generation request, reject prompts containing SKU identifiers, file paths, image ratios, resolution or clarity parameters, model/API names, timeout/retry notes, or output-location instructions. Keep those values only in request metadata and logs.

Use these patterns:

- Angle to side: `Adjust the model to a three-quarter side view in a full-body shot, with both arms hanging naturally by the sides and the head facing directly forward. Maintain the original head-to-body ratio and framing while ensuring the overall pose is natural and harmonious.`
- Angle to back with GPT image models: `生成背视图，人物重心放在其中一条腿上。`
- Angle to back with Gemini image models: `生成全身景别的背视图，人物重心放在其中一条腿上。`
- Pose edit: `将图中人物动作更改为双手放下，自然垂于身体两侧，并调整为立正站姿。`
- Same person: `将图1的模特替换为图2的模特，并保持图1的动作不变。`
- Try-on: `Replace the garment worn by the person in Figure 1 with the garment from Figure 2, preserving the Figure 2 <fit_notes> fit, keeping the person's pose and all other clothing from Figure 1 unchanged.`
- Color change: `Change the color of the garment worn by the person in Figure 1 to match the garment in Figure 2, preserving the Figure 2 <fit_notes> fit, while keeping the silhouette, pattern placement, trims, and decorative details consistent with Figure 2.`

Adjust garment words only when visually necessary, such as hoodie, pants, skirt, top, or dress.

## Task Logic

Read `references/task_logic.md` before implementing or modifying task planning logic.

Important interpretation rule: for requests such as `缺失女模特正面、侧面、背面图（模特黑色同款换棕色）`, inspect the SKU folder first. If same-style model images exist, recolor or replace the garment on a suitable existing model image first, then expand missing angles from that product-accurate result. If no model image exists, use the Eagle base -> front product try-on -> requested angle/pose expansion path.

Use `scripts/uo_aigc_batch.py` as the reusable runner. Typical flow:

```powershell
python "<SKILL_DIR>\scripts\uo_aigc_batch.py" `
  --root "\\server\path\AIGC修改" `
  --excel "新建 Microsoft Excel 工作表.xlsx" `
  --mode classify
```

After correcting `AIGC视觉识别.json` or `aigc_image_analysis.json` and, when needed, adding Eagle base paths to the `eagle_base` fields, run:

```powershell
python "<SKILL_DIR>\scripts\uo_aigc_batch.py" `
  --root "\\server\path\AIGC修改" `
  --excel "新建 Microsoft Excel 工作表.xlsx" `
  --mode plan `
  --image-model "gpt-image-2-vip"
```

Review `AIGC任务计划.json` and each SKU log. Generate only after the task screening, visual classification, reference assignment, and step ordering are correct:

```powershell
$env:TOAPIS_API_KEY="..."
python "<SKILL_DIR>\scripts\uo_aigc_batch.py" `
  --root "\\server\path\AIGC修改" `
  --excel "新建 Microsoft Excel 工作表.xlsx" `
  --mode generate `
  --image-model "gpt-image-2-vip" `
  --archive-old
```

The user can invoke the skill conversationally, for example: `使用 uo-aigc-model-image-processor 处理 <根目录>，生图模型使用 gpt-image-2-vip。` If the root path, model, or API credential required for generation is missing, request only the missing value; never guess a path or expose the credential in logs.

## Eagle Base Selection

When a SKU has no model image, use Eagle MCP before generation:

- Prefer the configured `eagle-mcp` server. Its local endpoints are `http://127.0.0.1:41596/mcp` and `http://127.0.0.1:41596/sse`; on Windows, the usual STDIO proxy is `%APPDATA%/Eagle/Plugins/mcp-server/modules/mcp-proxy.js`, invoked with Node.js. Resolve `%APPDATA%` for the current user rather than hard-coding a profile path.
- Retrieve candidates through Eagle search, create a candidate contact sheet when needed, and visually inspect the candidates yourself before assigning one. Do not let filename or tags replace visual judgment.

- Search tags such as `白底`, `白底图`, `男`, `女`.
- The selected base must be pure-color background or seamless wall.
- The selected base must be a full-body image.
- Do not reuse the same base in one batch.
- Treat bases as repeated if the model/background/outfit styling is essentially the same, even if the file is different.
- Manually judge whether remaining styling items are compatible with the product.
- If the product is pants, prefer bases with cropped tops or tops that reveal the abdomen so the waistband can be shown, but uniqueness across the batch remains the highest priority.
- If the product is a top and the product is not cropped, choose or generate styling where the top hem hangs naturally.
- Write selected base paths into `AIGC_Eagle底图.json`.
- In `AIGC视觉识别.json`, complete `eagle_selection` with the selected path, search tags, full-body/background/gender/styling checks, a batch uniqueness signature, and `manual_verified=true`.
- Stop the batch SKU before generation if any required Eagle check is incomplete or if the path/visual signature duplicates another no-model SKU in the batch.

The background constraint belongs only to Eagle selection, not to generation prompts.

## Unmatched Task Planning

For an unmatched task, decide the minimum reliable sequence after visual classification:

- Use one step when the requested edit can be applied directly to the selected source image without first creating a new angle, identity, pose, or product state.
- Use two or more steps when a generated result must become a later reference. When garment accuracy and angle/pose work are both requested, garment replacement/recolor must be the earlier step.
- Use model-only references for pose or ordinary angle edits; model plus the matching flat/detail image for garment, color, silhouette, or detail edits; target-action model plus identity reference for person replacement.
- Keep each prompt limited to its atomic edit. Put dependencies in `refs_from`, original image filenames in `refs`, and the generated output key in `sets`.
- If the optional prompt reference has no matching entry, independently choose the references and write a concise prompt. For same-person replacement, use the target-action image as Figure 1 and the identity base as Figure 2; preserve Figure 1's action.
