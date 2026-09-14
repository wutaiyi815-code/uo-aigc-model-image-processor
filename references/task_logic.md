# Task Logic Reference

## Inputs

- Excel task sheet with `SKU` and `修改意见` columns.
- A target root folder where each direct child folder is one SKU folder.
- Optional `AIGC视觉识别.json`, corrected by Codex/user after visual inspection.
- Optional `AIGC_Eagle底图.json` for no-model tasks.

## Screening Every Modification Item

Split and inspect every numbered or line-separated item in `修改意见`. Do not stop after item 1. Keep the original order of actionable items and process their steps serially per SKU.

Skip these non-generation requests even when they appear beside actionable requests:

- 模特图顶部、底部留白缩短
- 补齐图片后，更改分辨率&尺寸
- 修改图片后，更改分辨率&尺寸
- 更改分辨率
- 更改尺寸
- 留白
- 平铺图加白底或导出 JPG
- 裁剪或修剪到只展示上半身、上身卫衣或其他局部
- 指定像素尺寸，例如 `1600*2400`

Process only model-image missing, angle expansion, pose/action changes, same-style recolor/change-outfit, same-person replacement, and the try-on steps directly required to complete those tasks.

Write the screening decision for every item to `AIGC任务筛选.json`, using `actionable_items` and `skipped_items` with a reason. This report is the audit trail that all items were reviewed.

If wording does not match a known type, do not use a generic edit prompt. Visually plan the task through `custom_steps`; generation must stop if that plan is absent or incomplete.

## Visual Classification

Classify all original image files, excluding `AIGC+*` outputs and logs.

Required fields per image:

- `kind`: `model`, `flat`, `detail`, `other`
- `view`: `front`, `side_3_4`, `back`, `unknown`
- `garment_type`: `top`, `hoodie`, `pants`, `skirt`, `dress`, `set`, `unknown`
- `fit_notes`: concise product fit/silhouette terms, such as `regular`, `fitted`, `boxy`, `oversized`, `baggy`, `jorts`, `loose`, or combinations like `oversized mesh jersey`
- `model_gender`: `male`, `female`, `unknown`
- `pose_notes`
- `manual_note`

Never rely on file size to decide the flat lay. Visually decide which flat lay is front and which is back.

## Reference Selection

- Front and side model generation or recolor: use `front_flat`.
- Back model generation or recolor: use `back_flat`.
- Front and side try-on/color prompts should use `fit_notes` from `front_flat`; back prompts should use `fit_notes` from `back_flat`.
- If only one flat lay exists, use it only when it truly shows the required side; otherwise pause and ask.
- For existing-model missing-angle tasks, first verify that the chosen source model already wears the accurate product. If not, correct the garment/color before angle or pose generation.
- For back view expansion, use three reference images together: the product-accurate front model image, front flat lay, and back flat lay. If the image model is GPT, use `生成背视图，人物重心放在其中一条腿上。`; if the image model is Gemini, use `生成全身景别的背视图，人物重心放在其中一条腿上。`
- Do not defer garment correction until after angle generation. The product-accurate model result must be the source of downstream view and pose steps.

## Task Splitting

Split every compound request into atomic steps, then order those steps by dependency rather than spreadsheet position. Apply garment replacement, product try-on, or recolor before identity/view/pose generation that depends on the edited model.

For unmatched tasks, make these decisions before generation:

1. Decide whether one direct edit is sufficient or whether the output of one edit must feed the next edit.
2. Choose references by visual function, not filename or file size: model image for pose/angle; model plus matching front/back flat lay for product edits; current image plus a detail reference for local product-detail correction; target-action image plus identity image for model replacement.
3. Record every atomic operation in `custom_steps`. Use `refs` for original reference filenames, `refs_from` for a prior step's `sets` key, and a short prompt containing only the requested visual change.
4. Mark bridge outputs as `intermediate: true` so they are stored in `过程文件`.

When both `refs_from` and `refs` are present, the generated prior output is Figure 1 and the listed original references follow as Figure 2 onward.

Before interpreting tasks like `缺失女模特正面、侧面、背面图（模特黑色同款换棕色）`, inspect the SKU folder classification:

- If the folder contains same-style model images in another color, treat it as a same-style model recolor/change outfit task. First recolor or replace the garment on a suitable existing view using the correct flat lay. Then expand missing angles from that product-accurate model result.
- If the folder contains no model images, treat the spreadsheet wording as unclear rather than assuming a same-style model exists. Use the no-model path: select a unique Eagle base, perform front product try-on, then expand the requested angles or poses from the accurate front result.
- Do not let parenthetical color-change text override the visual evidence in the folder.

Examples:

- Missing side + same-style recolor:
  1. Recolor or replace the garment on the existing front model using the front flat lay.
  2. Generate the side angle from the corrected front model.
- Missing back + same-style recolor:
  1. Recolor or replace the garment on the existing front model using the front flat lay.
  2. Generate the back view from the corrected front model plus front and back flat lays.
- No model images:
   1. Select unique Eagle base.
   2. Try-on front flat lay onto Eagle base.
   3. Expand to side from the front try-on.
   4. Expand to back from the front try-on using the front try-on, front flat lay, and back flat lay as references.
- Three model images into same person:
  1. Select one model image as base identity.
  2. Pair each other model image as Figure 1 with base identity as Figure 2.

## Output

- Generated images: `AIGC+<修改内容>+<序号>.<ext>` in the SKU folder.
- Process folder: create `过程文件/` inside every processed SKU folder. Store intermediate-only images there, such as back-angle base images that are immediately used for back flat-lay try-on.
- Per-SKU log: `AIGC处理日志_<SKU>.txt`.
- Batch summary: `AIGC批处理汇总.json`.
- Old results, when archiving: `旧AIGC生成结果_<timestamp>/SKU/...`.
- For same-style recolor/color-change tasks, after an SKU finishes, move the original un-recolored model reference images used as Figure 1 into `过程文件/` so the SKU root keeps the final AIGC outputs cleaner.
- Detect the original Figure 1 automatically for every `color` step, including custom steps. Move it only after every SKU step succeeds; never move flat lays or generated outputs.

## Eagle Base Selection Details

- Select only full-body model images.
- The base must still have a pure-color background or seamless wall.
- Do not reuse the same base in one batch; also treat similar same-model/same-background/same-outfit files as duplicates.
- For pants, prefer a cropped top or abdomen-revealing top to expose waistband details, unless doing so would force duplicate base styling.
- For tops that are not cropped, ensure the top hem can hang naturally rather than being tucked or hidden.
- Record every Eagle decision in `eagle_selection`: candidate path, search tags, full-body check, background type, gender match, styling compatibility, batch uniqueness, visual uniqueness signature, category preference note, and manual verification.
