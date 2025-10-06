# Unity-MetaXR-AI-Florence
Unity project integrating Microsoft Florence-2 (Vision-Language Model) via NVIDIA’s AI API, with an end-to-end controller and UI to run image understanding tasks in XR.

![florence-xr-Trim-Trim-ezgif com-optimize](https://github.com/user-attachments/assets/429c9837-574e-4857-8843-1727167f73c3)

## 🔎 Overview
- Florence-2 is a multi-task vision-language model by Microsoft that supports captioning, detection, OCR, segmentation, region descriptions, and more using a single, tag-driven prompt format.
- This project calls Florence-2 through NVIDIA’s hosted endpoint and parses the response to draw 2D bounding boxes or spawn 3D anchors in the scene.

## 📁 Key Paths
- Scene: `Assets/XR-AI-Florence2/Scenes/XR-AI-Florence2.unity`
- Controller: `Assets/XR-AI-Florence2/Scripts/Florence2Controller.cs`
- API Config asset class: `Assets/XR-AI-Florence2/Scripts/ApiConfig.cs`

## ✅ What’s Implemented
- Tasks enumerated in `Florence2Task`:
  - Caption, DetailedCaption, MoreDetailedCaption
  - ObjectDetection
  - DenseRegionCaption, RegionProposal
  - CaptionToPhraseGrounding, OpenVocabularyDetection
  - ReferringExpressionSegmentation, RegionToSegmentation
  - RegionToCategory, RegionToDescription
  - OCR, OCRWithRegion
- Visuals currently implemented for Object Detection: draws 2D UI boxes and/or places 3D labels per detection.
- Other tasks return text/entities; basic display is included in `resultText`, with room to extend visuals if desired.

## ⚙️ Requirements
- Unity 6 LTS recommended.
- Meta XR Core and MRUK packages. (Or All-In-One)
- NVIDIA API key with access to Florence-2 endpoint.

## ☁️ NVIDIA Endpoint
- URL used by the controller: `https://ai.api.nvidia.com/v1/vlm/microsoft/florence-2`
- Auth: Bearer token in `Authorization` header.
- Content-Type: `application/json`
- Accept: `application/zip` (response is a ZIP containing a `.response` JSON file and optionally `overlay.png`).

## ⚡ Setup: 5 Minutes
1) Get an NVIDIA API Key
   - Obtain a key from NVIDIA’s AI API portal and ensure access to the Florence-2 VLM endpoint. https://build.nvidia.com/

2) Create an API Config asset
   - Project window: Go to XR-AI-Florence/Data folder, right click, create → API → API Configuration.
   - Name it, `ApiConfig.asset`. (so it's properly ignored keeping your api key safe)
   - Paste your API key into the `apiKey` field.

3) Open the sample scene
   - `Assets/XR-AI-Florence2/Scenes/XR-AI-Florence2.unity`.

4) Assign the `Florence2Controller` field:
   - `Api Configuration`: assign the ScriptableObject you created.
   - Optional
     - `Anchor Mode`: BoundingBox2D, SpatialAnchor3D, or Both.
  
Other field descriptions that are already assigned:
   - `Source Texture` (`RawImage`): the image to analyze, it's by default assigned to a RawImage that is fed by the Passthrough Camera of the Quest 3.
   - `Task`: choose a task from the dropdown. For now, the only task that is fully implemented is Object Recognition.
   - `Region Of Interest`: used by region-based tasks. Coordinates are normalized (0–1) as a Rect (x, y, width, height).
   - UI
     - `Result Text` (`TMP_Text`): summary and counts.
     - `Result Image` (`RawImage`): where overlay or source is shown.
     - `Bounding Box Container` (`RectTransform`): parent for box UI.
     - `Bounding Box Prefab`: prefab containing a root `RectTransform` and a `TextMeshProUGUI` child for the label.
     - `Status Text` (`TMP_Text`): request status and errors.
     - `Loading Icon` (`GameObject`): optional spinner shown during requests.

5) Run a request / Build to device
   - In Play Mode, click the `SendRequest()` button shown in the Inspector (NaughtyAttributes adds the button to the component). In the editor only the Anchor Mode "Bounding Box 2D" will work.
   - Or call it via script if you have a reference: `controller.SendRequest();`
   - If you want to test the "Spatial Label 3D" anchor mode (the one shown in the video above), you must build the scene to your Quest 3 device.

## 🛠️ How It Works (Under the Hood)
1) Image encoding
   - `EncodeTextureToJPG(Texture)` converts the `sourceTexture.texture` into JPEG bytes and base64-embeds it in HTML `<img src="data:image/jpeg;base64,..." />`.

2) Prompt construction
   - `Florence2Task` maps to Florence-2 tags, e.g. `<OD>` for Object Detection, `<CAPTION>`, etc.
   - For text-conditional tasks (e.g., `<CAPTION_TO_PHRASE_GROUNDING>`, `<OPEN_VOCABULARY_DETECTION>`), your `Text Prompt` is appended after the tag.

3) Request/Response
   - HTTP POST to the NVIDIA endpoint with `Authorization: Bearer <apiKey>` and `Accept: application/zip`.
   - The ZIP contains `*.response` JSON and possibly `overlay.png`.
   - JSON is deserialized into `Florence2Response` → `Choices[0].Message.Entities` with `bboxes` and `labels` (when applicable).

4) Visuals
   - 2D: Converts Florence coordinates `[x1, y1, x2, y2]` to width/height and spawns the bounding box prefab under `BoundingBoxContainer`, scaled to `Result Image` size.
   - 3D: Projects box center to a world-space ray and uses `EnvironmentRaycastManager.Raycast` to place an anchor prefab at the hit point, labeled with the detection class.

## ⚠️ Limitations
- Because requests are network-bound, latency can cause pose drift relative to the original capture. If you move, the raycast from the detected 2D box center may no longer intersect the same real-world surface.
- Tips:
  - Prefer testing while stationary, or on a tripod/stand when possible.

## 🧩 Extending
- Segmentation: Use `overlay.png` (if returned) or the `Entities` segmentation data to render masks or outlines.
- OCR: Display `Message.Content`/entities in the UI, draw text regions.
- Region tasks: Use `regionOfInterest` in prompts and visualize per-task outputs.

## 🧯 Troubleshooting
- "API Key or Source Image is missing": Ensure the ApiConfig asset is assigned and `sourceTexture.texture` is valid.
- HTTP 4xx with error JSON in Console: Verify your key, model access, and request payload format.
- No boxes drawn:
  - Make sure `Result Image` has a texture with correct dimensions; scaling uses `resultImage.texture.width/height`.
  - Confirm `Bounding Box Container` and `Bounding Box Prefab` are assigned.
- 3D anchors not appearing: Ensure `EnvironmentRaycastManager` is in scene and `spatialAnchorPrefab` is set. Also confirm passthrough/camera utilities are available.

## 🔐 Security
- Do not commit your API key. Keep the `ApiConfig` asset out of version control or remove the key before committing. The Gitignore of the project will leave out /Assets/XR-AI-Florence2/Data/ApiConfig.asset

## 📚 References
- Microsoft Florence-2: https://huggingface.co/microsoft/Florence-2-large
- NVIDIA AI API (VLM Florence-2): https://build.nvidia.com/

## 🔎 Relevant Sources & Opportunities 

SensAI Kits GitHub - Main hub for all XR AI kits
SensAI Hackademy - Early access program for courses and toolkits
SensAI Hack - Upcoming hackathons where you can use the kits"

## 📄 License
MIT – Free to use, modify and learn from.
