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
- `shot_scope`: `full_body`, `upper_body`, `lower_body`, `unknown`
- `manual_note`

For every existing model image that may be reused as a pants source, add a `source_qc` entry keyed by its filename:

- `manual_verified: true|false`
- `product_accurate: true|false`
- `waist_accessory: none|product_integrated|belt|chain|visible_underwear|sash|other`
- `footwear_type: boots|high_top|low_top|other`
- `pant_leg_placement: outside_shafts|tucked|stacked_on_shafts|not_applicable`

Missing `source_qc` is unverified, not implicitly valid. It may be used only as the input to a product-correction step until it passes visual QC.

Never rely on file size to decide the flat lay. Visually decide which flat lay is front and which is back.

For actionable missing-model-image tasks that request a `半身图`, determine the framing from the target product before planning:

- Upper-body products such as tops, hoodies, sweatshirts, jackets, and knitwear require `target_scope: upper_body` and an upper-body model image.
- Lower-body products such as pants, jeans, shorts, jorts, and skirts require `target_scope: lower_body` and a lower-body model image.
- Do not use a generic `half-body` prompt. The prompt must explicitly request the chosen upper-body or lower-body e-commerce view.
- Treat the framing as a QC invariant: reject and redo outputs whose scope does not match the target product.
- Apply this validation to every execution route: current runner plans, external JSON plans, hand-written plans, resumed manifests, and legacy helper scripts. Rebuild or patch a stale plan before submitting it.
- Record `product_length: short|full_length|not_applicable` for lower-body products. Do not infer shorts from an incomplete model crop when the flat lay shows full-length pants.
- This rule applies to an actionable missing model image. A spreadsheet item that only asks to crop/trim an existing image remains a skipped non-generation request.

## Reference Selection

- Front and side model generation or recolor: use `front_flat`.
- Back model generation or recolor: use `back_flat`.
- Front and side try-on/color prompts should use `fit_notes` from `front_flat`; back prompts should use `fit_notes` from `back_flat`.
- If only one flat lay exists, use it only when it truly shows the required side; otherwise pause and ask.
- For existing-model missing-angle tasks, first verify that the chosen source model already wears the accurate product. If not, correct the garment/color before angle or pose generation.
- For back view expansion, use exactly two references: Figure 1 is the verified product-accurate front model image and Figure 2 is the back flat lay. Do not include the front flat lay. If the image model is GPT, use `生成背视图，人物重心放在其中一条腿上。`; if the image model is Gemini, use `生成全身景别的背视图，人物重心放在其中一条腿上。`
- Do not defer garment correction until after angle generation. The product-accurate model result must be the source of downstream view and pose steps.

### Half-body and detail-preservation references

- If an accurate full-body source already exists and only the composition must become upper-body or lower-body, prefer deterministic cropping/reframing. Do not ask a generative model to redraw unchanged clothing, shoes, or product details without need.
- If a generated lower-body front view is necessary, use the accurate full-body front model image and the front flat lay. Do not use a previous half-body output alone.
- If a generated lower-body back view is necessary, use the accurate full-body back model image and the back flat lay. When no accurate back model exists, first produce it from the verified product-accurate front model plus the back flat lay only.
- For full-length pants, explicitly preserve full length. Reject any result that becomes shorts, jorts, or cropped pants.
- Preserve the source footwear. When boots or high-top shoes are present, require pant legs to fall naturally outside the shafts; do not tuck the pants into the shafts or stack the hems on top of them.
- During pants try-on, visually record `waist_accessory: belt|chain|visible_underwear|sash|none`. Remove any original waist accessory that is not part of the target product. Do not let `keep all other clothing unchanged` override this removal.
- For back views, map reference roles as Figure 1 model and Figure 2 back flat. State that Figure 2 is authoritative for back pockets, seams, graphics, labels, hardware, and trims. Do not attach the front flat lay.
- Before treating an existing full-body model image as accurate, visually record `source_qc.manual_verified`, `source_qc.product_accurate`, `source_qc.waist_accessory`, `source_qc.footwear_type`, and `source_qc.pant_leg_placement` for that source. File naming, API success, or an older manifest is not proof of accuracy.
- For pants, any non-product belt, waist chain, sash, exposed underwear waistband, or other waist accessory fails source QC. Full-length pants tucked into or stacked on boot/high-top shafts also fail source QC. Schedule a new product try-on before any angle, pose, identity, or crop step that depends on that source.
- For detail-rich or asymmetric products, create a product-accurate full back before producing the back half-body composition. Do not rotate a generated half-front directly into a back half-body result.

## Task Splitting

Split every compound request into atomic steps, then order those steps by dependency rather than spreadsheet position. Apply garment replacement, product try-on, or recolor before identity/view/pose generation that depends on the edited model.

For unmatched tasks, make these decisions before generation:

1. Decide whether one direct edit is sufficient or whether the output of one edit must feed the next edit.
2. Choose references by visual function, not filename or file size: model image for pose/angle; model plus matching front/back flat lay for product edits; current image plus a detail reference for local product-detail correction; target-action image plus identity image for model replacement.
3. Record every atomic operation in `custom_steps`. Use `refs` for original reference filenames, `refs_from` for a prior step's `sets` key, and a short prompt containing only the requested visual change.
   For every step whose output name contains `半身`, also set `target_scope` to `upper_body` or `lower_body`; planning must fail when it is absent or inconsistent with the prompt.
   Treat both `name` and `output_name` as output names for this validation. For lower-body generation, also record the accurate full-body source and matching flat-lay reference; an archived or prior half-body image alone is insufficient.
   Reject a plan that reuses a failed pants source without an earlier try-on correction. For back-angle steps, reject plans that include `front_flat`; the only product flat reference is `back_flat`.
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
  2. Visually verify the corrected front model, then generate the back view from it plus the back flat lay only.
- No model images:
   1. Select unique Eagle base.
   2. Try-on front flat lay onto Eagle base.
   3. Expand to side from the front try-on.
   4. Visually verify the front try-on, then expand to back using the front try-on plus the back flat lay only.
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

## Generation Failure Fallback

- Interpret the retry policy as one initial generation attempt plus two retries, for three total attempts.
- Retry a complete generation only when the provider explicitly returns a terminal failure such as `failed`, `error`, `cancelled`, or a confirmed safety-review rejection. For an ambiguous submit, polling, timeout, or connection failure, recover using the existing provider task ID before considering another paid submission.
- When the primary model is exhausted, use the entry written to `过程文件/AIGC生图回退请求.json`. Call Codex's built-in image generation tool first, using exactly the recorded prompt and reference-image order.
- Save a successful built-in result to the recorded `output_base`, update the request status to `completed_builtin_imagegen`, append the result to `AIGC处理日志_<SKU>.txt`, and use that image for any dependent steps.
- If the built-in tool fails or is blocked, call ToAPI `gemini-3.1-flash-image-official` with the same prompt and ordered references, `size=3:4`, `resolution=2K`, 600-second timeout, and one initial attempt plus two retries. Update the request status to `completed_gemini_fallback` when successful.
- If Gemini also fails, set the request status to `failed_all_fallbacks` and log the built-in error, Gemini response/error, task ID when available, and the last confirmed state. Never omit the failed task from the final batch summary.

## Eagle Base Selection Details

- Select only full-body model images.
- The base must still have a pure-color background or seamless wall.
- Do not reuse the same base in one batch; also treat similar same-model/same-background/same-outfit files as duplicates.
- For pants, prefer a cropped top or abdomen-revealing top to expose waistband details, unless doing so would force duplicate base styling.
- For tops that are not cropped, ensure the top hem can hang naturally rather than being tucked or hidden.
- For pants candidates, record visible waist accessories and footwear type during visual review. Prefer a compatible base without a belt, waist chain, exposed waistband, boots, or high-tops when that avoids unnecessary product-detail conflicts; otherwise add the required removal or pant-leg placement instruction to the try-on prompt.
- Record every Eagle decision in `eagle_selection`: candidate path, search tags, full-body check, background type, gender match, styling compatibility, batch uniqueness, visual uniqueness signature, category preference note, and manual verification.
