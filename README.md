# 🧩 Comfyroll Studio

Co-authored by Suzie1 and RockOfFire

Current version: 1.76

# Wiki

Please use our new wiki for info on the custom nodes, and lots of examples of their use.

https://github.com/Suzie1/ComfyUI_Comfyroll_CustomNodes/wiki

# Installation

1. cd custom_nodes
2. git clone https://github.com/Suzie1/ComfyUI_Comfyroll_CustomNodes.git
3. Restart ComfyUI

You can also install the nodes using the following methods:
* install using [ComfyUI Manager](https://github.com/ltdrdata/ComfyUI-Manager)
* download from [CivitAI](https://civitai.com/models/87609/comfyroll-custom-nodes-for-comfyui)

# Patch Notes

https://github.com/Suzie1/ComfyUI_Comfyroll_CustomNodes/blob/main/Patch_Notes.md

# Usage: Custom Text Utility Nodes

## 🔤 CR DateTime

Outputs a formatted datetime string using WAS-style `[time(fmt)]` tokens, where `fmt` is any [`strftime`](https://docs.python.org/3/library/time.html#time.strftime) format string. Multiple tokens and surrounding literal text are supported. The node re-evaluates on every workflow run.

Input:
- `format` — default `[time(%Y-%m-%d %H:%M:%S)]`.

Examples:
- `[time(%Y-%m-%d)]` → `2026-06-23`
- `[time(%Y-%m-%d %H:%M:%S)]` → `2026-06-23 14:30:05`
- `run_[time(%Y%m%d_%H%M%S)]_v1` → `run_20260623_143005_v1`

Common pairing: feed into `file_name` of **CR Save Text To File**, or into a **CR Text Concatenate** to tag prompts with the time they were generated.

### Safe formats for filenames

Filenames must avoid `:`, `/`, `\`, and spaces (Windows rejects `:`; Unix treats `/` as a path separator; spaces are ugly in shells). Pick a format with only digits, hyphens, and underscores:

| Format | Example | Notes |
| --- | --- | --- |
| `[time(%Y-%m-%d)]` | `2026-06-23` | Date only, universally safe. |
| `[time(%Y%m%d_%H%M%S)]` | `20260623_143005` | **Recommended** for dated filenames — lexicographically sortable. |
| `[time(%Y%m%dT%H%M%S)]` | `20260623T143005` | ISO-ish without separators. |
| `[time(%Y%m%dT%H%M%S%z)]` | `20260623T143005+0700` | Include UTC offset when timezone matters. |

Avoid these in filenames:
- `%Y-%m-%dT%H:%M:%S` — true ISO 8601 contains `:`, invalid on Windows. Use it for the `created_at` YAML field instead, where `:` is fine.
- `%c`, `%x`, `%X` — locale-dependent, may emit `/` or spaces.

### ISO 8601 formats

ISO 8601 is the international standard for representing dates and times. Use these in YAML/JSON metadata fields and API payloads — **not** in filenames (the `:` separator is invalid on Windows).

| Format | Example | Use case |
| --- | --- | --- |
| `[time(%Y-%m-%d)]` | `2026-06-23` | Date only. |
| `[time(%H:%M:%S)]` | `14:30:05` | Time only. |
| `[time(%Y-%m-%dT%H:%M:%S)]` | `2026-06-23T14:30:05` | **Recommended** local datetime — what to feed into `created_at` of CR Yaml Frontmatter. |
| `[time(%Y-%m-%dT%H:%M:%S%z)]` | `2026-06-23T14:30:05+0700` | Datetime with UTC offset. Use when the consumer needs an unambiguous timezone. |
| `[time(%Y-%m-%dT%H:%M:%SZ)]` | `2026-06-23T14:30:05Z` | "Zulu" / UTC datetime. Only correct if your system clock is UTC. |
| `[time(%G-W%V-%u)]` | `2026-W26-2` | ISO week date (year-week-weekday). |
| `[time(%Y-%j)]` | `2026-174` | Ordinal date (year-dayOfYear). |

Notes:
- The `T` between date and time is literal — `strftime` passes it through unchanged.
- `%z` emits `+HHMM` (e.g. `+0700`); strict ISO 8601 wants `+HH:MM`. Most parsers accept both.
- Python's `time.strftime` does not have a dedicated millisecond code. If you need sub-second precision, use a different tool — `[time(%Y-%m-%dT%H:%M:%S)]` is the most precise format this node supports.

## 🔤 CR Save Text To File

Saves a multiline string to a file on disk.

Inputs:
- `multiline_text` — text to write.
- `output_file_path` — directory to write into. Supports `[time(fmt)]` tokens (any `strftime` format), e.g. `/data/logs/[time(%Y-%m-%d)]`. Directory is created if missing. Works on Linux, macOS, and Windows.
- `file_name` — base file name (no extension). Also supports `[time(fmt)]` tokens, e.g. `run_[time(%H%M%S)]`.
- `file_extension` — `txt`, `csv`, or `md`.

Behavior: if the target file already exists, an `_1`, `_2`, … suffix is appended. `csv` writes one line per row via `csv.writer`; `txt`/`md` write the text as-is.

Example: `output_file_path = /workspace/out/[time(%Y-%m-%d)]`, `file_name = prompt`, `file_extension = md` produces `/workspace/out/2026-06-22/prompt.md`.

## 🔤 CR Text Hash

Hashes up to four input strings and returns the hex digest.

Inputs:
- `algorithm` — `sha256` (default), `sha1`, `sha512`, or `md5`.
- `length` — number of hex characters to keep (`0` = full digest).
- `text1`..`text4` *(optional, forceInput)* — strings to hash. Empty inputs are skipped.
- `separator` *(optional)* — joined between non-empty inputs before hashing.

Example: `text1 = "cat"`, `text2 = "dog"`, `separator = "|"`, `algorithm = sha256`, `length = 12` → hash of `"cat|dog"` truncated to 12 chars. Useful for building deterministic filenames or cache keys from prompt fragments — pair with **CR Save Text To File** by feeding the hash into `file_name`.

## 🔤 CR Yaml Frontmatter

Builds a Markdown/YAML document with a frontmatter header and a body. Useful for writing prompt logs together with metadata.

Inputs:
- `body` *(required, multiline)* — content placed after the closing `---`.
- `categories` *(optional)* — comma-separated; each item is emitted as a double-quoted entry in a flow-style list.
- `models` *(optional)* — comma-separated; each item is emitted as a double-quoted entry in a flow-style list.
- `loras` *(optional, multiline)* — one `name:weight` per line. Lines starting with `#` are skipped; lines without `:` default to weight `1.0`. `name` is double-quoted.
- `negative` *(optional, multiline)* — internal newlines are collapsed to single spaces, trimmed, then emitted as a double-quoted string.
- `created_at` *(optional)* — an ISO 8601 datetime string. Emitted **unquoted** so YAML parsers read it as a native timestamp. Designed to be wired from **CR DateTime** with format `[time(%Y-%m-%dT%H:%M:%S)]`.

Any empty optional field is omitted from the output.

Example inputs:
- `categories`: `hands, girl`
- `models`: `Anima Base 1.0`
- `loras`:
  ```
  gpt-style:0.8
  ```
- `negative`: `blurry, lowres, deformed`
- `created_at`: `2026-06-23T14:30:05` *(typically wired from CR DateTime)*
- `body`: `cinematic portrait, 1girl, head tilt, neutral expression, slightly parted lips, looking at the man, soft blush`

Output:
```yaml
---
categories: ["hands", "girl"]
models: ["Anima Base 1.0"]
loras:
  - { name: "gpt-style", weight: 0.8 }
negative: "blurry, lowres, deformed"
created_at: 2026-06-23T14:30:05
---
cinematic portrait, 1girl, head tilt, neutral expression, slightly parted lips, looking at the man, soft blush
```

Typical pipeline: **CR DateTime** (`[time(%Y-%m-%dT%H:%M:%S)]`) → `created_at` of **CR Yaml Frontmatter** → **CR Text Hash** (hash the body for a filename) → **CR Save Text To File** with `file_extension = md`.

# List of Custom Nodes
  
## Core Nodes
__📦 Essential Nodes__
* CR Image Output (changed 18/12/2023)
* CR Latent Batch Size
* CR Prompt Text
* CR Combine Prompt
* CR Seed
* CR Conditioning Mixer
* CR Select Model (new 24/1/2024)
* CR VAE Decode (new 24/1/2024)

__🔳 Aspect Ratio__
* CR Aspect Ratio
* CR SDXL Aspect Ratio
* CR SD1.5 Aspect Ratio
* CR Aspect Ratio Banners (new 18/12/2023)
* CR Aspect Ratio Social Media (new 15/1/2024)
* CR Aspect Ratio For Print (new 18/1/2024)

__📜 List Nodes__
* CR Text List (new 19/12/2023)
* CR Prompt List (new 1/1/2024)
* CR Float Range List (new 25/12/2023)
* CR Integer Range List (new 25/12/2023)
* CR Load Text List (new 27/12/2023)
* CR Binary To List (new 29/12/2023)
* CR Text List To String (updated 30/12/2023)
* CR Text Cycler (new 4/1/2024)
* CR Value Cycler (new 4/1/2024)

__📜 List IO__
* CR Load Image List (new 23/12/2023)
* CR Load Image List Plus (new 23/12/2023)
* CR Load GIF As List (new 6/1/2024)
* CR Font File List (new 18/12/2023)

__📜 List Utils__
* CR Batch Images From List (new 29/12/2023)    
* CR Intertwine_Lists (new 29/12/2023)
* CR Repeater (new 15/1/2024)   
* CR XY Product (new 2/1/2024)
* CR Text List To String (updated 30/12/2023)

__🌟 SDXL__
* CR SDXL Prompt Mix Presets
* CR SDXL Style Text
* CR SDXL Base Prompt Encoder

__💊 LoRA__
* CR Load LoRA
* CR LoRA Stack
* CR Apply LoRA Stack
* CR Random LoRA Stack (new 18/12/2023)
* CR Random Weight LoRA (new 18/12/2023)

__🕹️ ControlNet__
* CR Apply ControlNet
* CR Multi-ControlNet Stack
* CR Apply Multi-ControlNet Stack

__🚌 Bus__
* CR Data Bus In (new 12/1/2024)
* CR Data Bus Out (new 12/1/2024)
* CR 8 Channel In (new 12/1/2024)
* CR 8 Channel Out (new 12/1/2024)

__✈️ Module__
* CR Module Pipe Loader
* CR Module Input
* CR Module Output

__🛩️ Pipe__
* CR Image Pipe In
* CR Image Pipe Edit
* CR Image Pipe Out
* CR Pipe Switch

__⛏️ Model Merge__
* CR Model Stack
* CR Apply Model Merge

__🔍 Upscale__
* CR Multi Upscale Stack
* CR Upscale Image
* CR Apply Multi Upscale

__📉 XY Grid__
* CR XY List
* CR XY Interpolate   
* CR XY Index
* CR XY From Folder
* CR XY Save Grid Image
* CR Image Output

## 👾 Graphics Nodes

__👓 Graphics - Filter__
* CR Color Tint
* CR Halftone Filter
* CR Vignette Filter (new 21/12/2023)

__🌈 Graphics - Pattern__
* CR Halftone Grid
* CR Color Bars
* CR Style Bars   
* CR Checker Pattern
* CR Polygons
* CR Color Gradient
* CR Radial Gradiant
* CR Starburst Lines
* CR Starburst Colors
* CR Simple Binary Pattern
* CR Binary Pattern

__🟡 Graphics - Pattern__
* CR Draw Shape (new 24/12/2023)
* CR Draw Pie" (new 25/12/2023)  
* CR Random Shape Pattern" (new 25/12/2023)   

__🔤 Graphics - Text__
* CR Overlay Text
* CR Draw Text
* CR Mask Text
* CR Composite Text
* CR Select Font

__👽 Graphics - Template__
* CR Simple Meme Template
* CR Simple Banner
* CR Comic Panel Templates
* CR Simple Banner (new 18/12/2023)
* CR Simple Image Compare (new 18/12/2023)
* CR Thumbnail Preview (new 26/12/2023)
* CR Seamless Checker (new 18/1/2023)

__🌁 Graphics - Layout__
* CR Image Panel
* CR Page Layout
* CR Image Grid Panel
* CR Image Border
* CR Feathered Border (new 21/12/2023)
* CR Color Panel
* CR Simple Text Panel
* CR Half Drop Panel (new 23/1/2024)
* CR Diamond Panel (new 24/1/2024)
* CR Overlay Transparent Image
* CR Select ISO Size (new 18/1/2023)

## 🎥 Animation

__📋 Schedules__
* CR Simple Schedule
* CR Central Schedule
* CR Combine Schedules
* CR Output Schedule To File
* CR Load Schedule From File
* CR Schedule Input Switch

__📑 Schedulers__
* CR Simple Value Scheduler
* CR Simple Text Scheduler
* CR Value Scheduler
* CR Text Scheduler
* CR Load Scheduled Models
* CR Load Scheduled LoRAs
* CR Prompt Scheduler
* CR Simple Prompt Scheduler

__📝 Prompt__
* CR Keyframe List
* CR Load Prompt Style
* CR Encode Scheduled Prompts

__🔢 Interpolation__
* CR Gradient Float
* CR Gradient Integer
* CR Increment Float
* CR Increment Integer
* CR Interpolate Latents

__🛠️ Utils__
* CR Debatch Frames
* CR Current Frame

__⌨️ IO__
* CR Load Animation Frames
* CR Load Flow Frames
* CR Output Flow Frames

## 🛠️ Utility Nodes

__🔢 Utils Index__
* CR Index
* CR Index Increment
* CR Index Multiply
* CR Index Reset
* CR Trigger

__🔧 Utils Conversion__    
* CR String To Number (changed 18/12/2023)
* CR String To Combo    
* CR Float To String
* CR Float To Integer
* CR Integer To String  
* CR String To Boolean (new 17/1/2024)  

__🔀 Utils Logic__
* CR Image Input Switch
* CR Image Input Switch (4 way)
* CR Latent Input Switch
* CR Conditioning Input Switch
* CR Clip Input Switch
* CR Model Input Switch
* CR ControlNet Input Switch
* CR VAE Input Switch
* CR Text Input Switch
* CR Text Input Switch (4 way)
* CR Switch Model and CLIP

__🔂 Utils Process__
* CR Img2Img Process Switch
* CR Hires Fix Process Switch
* CR Batch Process Switch

__🎲 Utils Random__
* CR Random Hex Color
* CR Random RGB
* CR Random Multiline Values (updated 28/12/2023)
* CR Random Multiline Colors (new 28/12/2023)
* CR Random RGB Gradient
* CR Random Panel Code (new 26/12/2023)

__🔤 Utils Text__
* CR Text (new 3/1/2024)
* CR Multiline Text (new 24/12/2023)
* CR Split String
* CR Text Concatenate (new 2/1/2024)
* CR Text Replace (new 8/1/2024)
* CR Text Blacklist (new 13/1/2024)
* CR Text Length (new 10/1/2024)    
* CR Text Operation (new 10/1/2024)
* CR Save Text To File (new 27/12/2023)

__⚙️ Conditional__
* CR Set Value On Boolean (new 29/12/2023)
* CR Set Value On Binary (new 3/1/2024)
* CR Set Value On String (new 9/1/2024)
* CR Set Switch From String (new 17/1/2024)

__⚙️ Utils Other__
* CR Value
* CR Integer Multiple
* CR Clamp Value (new 29/12/2023)
* CR Math Operation (new 31/12/2023)  
* CR Get Parameter From Prompt (new 5/1/2024)
* CR Select Resize Method (new 16/1/2024)

## Legacy
__💀 Legacy Nodes__
* CR Seed to Int
* CR Aspect Ratio SDXL, replaced by CR SDXL Aspect Ratio
* CR Image Size, replaced by CR Aspect Ratio
* CR SDXL Prompt Mixer, replaced by CR SDXL Prompt Mix Presets

# Comfyroll Workflow Templates

The nodes were originally made for use in the Comfyroll Template Workflows.

[Comfyroll Template Workflows](https://civitai.com/models/59806/comfyroll-template-workflows)

[Comfyroll Pro Templates](https://civitai.com/models/85619/comfyroll-pro-template)

[Comfyroll SDXL Workflow Templates](https://civitai.com/models/118005/comfyroll-sdxl-workflow-templates)

[SDXL Workflow for ComfyUI with Multi-ControlNet](https://civitai.com/models/129858/sdxl-workflow-for-comfyui-with-multi-controlnet)

[SDXL and SD1.5 Model Merge Templates for ComfyUI](https://civitai.com/models/123125/sdxl-and-sd15-model-merge-templates-for-comfyui)

# Credits

comfyanonymous/[ComfyUI](https://github.com/comfyanonymous/ComfyUI) - A powerful and modular stable diffusion GUI.

WASasquatch/[was-node-suite-comfyui](https://github.com/WASasquatch/was-node-suite-comfyui) - A powerful custom node extensions of ComfyUI.

TinyTerra/[ComfyUI_tinyterraNodes](https://github.com/TinyTerra/ComfyUI_tinyterraNodes) - A selection of nodes for Stable Diffusion ComfyUI

hnmr293/[ComfyUI-nodes-hnmr](https://github.com/hnmr293/ComfyUI-nodes-hnmr) - ComfyUI custom nodes - merge, grid (aka xyz-plot) and others

SeargeDP/[SeargeSDXL](https://github.com/SeargeDP) - ComfyUI custom nodes - Prompt nodes and Conditioning nodes

LucianoCirino/[efficiency-nodes-comfyui](https://github.com/LucianoCirino/efficiency-nodes-comfyui) - A collection of ComfyUI custom nodes.

SLAPaper/[ComfyUI-Image-Selector](https://github.com/SLAPaper/ComfyUI-Image-Selector) - Select one or some of images from a batch

pythongosssss/[ComfyUI-Custom-Scripts](https://github.com/pythongosssss/ComfyUI-Custom-Scripts) - Enhancements & experiments for ComfyUI, mostly focusing on UI features

bash-j/[mikey_nodes](https://github.com/bash-j/mikey_nodes) - comfy nodes from mikey

ltdrdata/[ComfyUI-Impact-Pack](https://github.com/ltdrdata/ComfyUI-Impact-Pack) - 
