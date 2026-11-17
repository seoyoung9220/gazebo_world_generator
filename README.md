# Gazebo World Generator (personal fork — Jazzy/Harmonic port)
![License](https://img.shields.io/badge/License-MIT-blue.svg)
![ROS Version](https://img.shields.io/badge/ROS-Jazzy-orange.svg)
![Gazebo Version](https://img.shields.io/badge/Gazebo-Harmonic-purple.svg)
![Python Version](https://img.shields.io/badge/Python-3.8+-green.svg)

**Build Complex Gazebo Worlds with a Single Sentence.**

Gazebo World Generator transforms your ideas into fully-functional Gazebo simulations. Leveraging the power of Large Language Models, this ROS2 package intelligently interprets plain English descriptions to automatically generate collision-free layouts, place furniture, and even produce 2D navigation maps.

Console Interface             |  Generated World
:-------------------------:|:-------------------------:
![console](https://github.com/robotics-upo/gazebo_world_generator/blob/master/media/gazebo_world_gen_console.gif) | *"A 12x10 m warehouse with 8 pallets and 10 shelves connected to a small office with a desk and a chair"* ![world](https://github.com/robotics-upo/gazebo_world_generator/blob/master/media/gazebo_world_gen_world.gif)

> [!NOTE]
> Please be mindful that this is a work in progress, and while we strive for accuracy, the generated worlds may require some manual adjustments to meet specific needs.

## About This Fork

This is a personal fork of [robotics-upo/gazebo_world_generator](https://github.com/robotics-upo/gazebo_world_generator), kept here as a working record of the changes I made to run it on my own setup — **not** intended to be merged upstream.

**Why I forked it:** the upstream project targets ROS2 Humble + Gazebo Classic (gazebo11), but my workspace runs ROS2 Jazzy + Gazebo Harmonic (gz-sim8). Gazebo Classic reached end-of-life, so I needed a version that works against the current `gz-sim` stack instead of patching around a dead dependency.

**What I changed, and why:**

- **Ported Classic SDF idioms to Harmonic.** Harmonic ships no bundled `model://ground_plane`, so `ground_plane` is now emitted as an inline `<plane>` geometry instead of an `<include>`. Likewise the `Gazebo/Grey` script material on walls (a Classic-only Ogre material name) is replaced with explicit `ambient`/`diffuse`/`specular` values, since Harmonic's renderer doesn't resolve Classic material scripts.
- **Moved all model/cache paths from `~/.gazebo/...` to `~/.gz/...`.** Harmonic's model cache and Fuel downloads live under `~/.gz/models` and `~/.gz/fuel/fuel.gazebosim.org`, not the Classic `~/.gazebo/models` path, so every path in `generator_config.yaml` and the resolver code needed updating. `~/.gazebo/models` is kept as a secondary search path for backward compatibility with any models I'd already cached there.
- **Switched the default LLM backend to a hosted Groq model** (`llama-3.3-70b-versatile` via `https://api.groq.com/openai/v1`) instead of a local LM Studio server, since that's what I actually run day to day.
- **Added a tokenizer fallback** in `placement/engine.py` and `placement/semantic_grouping.py`. The tokenizer there is only used for approximate token counting, but `AutoTokenizer.from_pretrained(DEFAULT_MODEL, ...)` fails outright for provider-internal model names like `llama-3.3-70b-versatile` that don't exist as a HuggingFace repo. It now falls back to `Qwen/Qwen2.5-0.5B-Instruct` for counting purposes only — it never affects which model actually generates the world.
- **Fixed a case-sensitivity bug in model symlinking** (`models/resolver.py`). The mesh-URI-vs-directory-name comparison used to ignore case (`mesh_model_name.lower() != dir_name.lower()`), so on a case-sensitive Linux filesystem a model whose mesh `<uri>` only differed from its directory name by case never got its compatibility symlink created, and failed to load.
- **Fixed freshly-downloaded models reporting as "not found."** After downloading a new online model, the code returned a `model://` URI but never updated `_model_name_to_dir_map`, which is what `get_directory_for_model()` actually checks — so the model the generator had just downloaded would immediately fail the next lookup.
- **Improved map-generation error messages.** `generate_map.sh` writes most of its diagnostics to stdout via `echo`, not stderr, so on failure the raised `RuntimeError` was often empty. It now falls back to the tail of stdout when stderr is blank.

If you're on ROS2 Humble + Gazebo Classic, use the original upstream repo instead — this fork isn't meant to be a general-purpose replacement, just my own environment's working copy.

## Table of Contents

- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Usage Examples](#usage-examples-non-interactive)
- [Configuration](#configuration)
- [Output Files](#output-files)
- [Command Line Options](#command-line-options)
- [How It Works](#how-it-works)
- [Troubleshooting](#troubleshooting)

## Features

- **Natural language input** – Describe your world in plain English
- **Automatic placement** – Smart furniture grouping (offices) or systematic grid layouts (warehouses)
- **Navigation ready** – Generates 2D maps for robot navigation
- **Interactive refinement** – Modify existing worlds with simple commands
- **Flexible dimensions** – Specify exact room sizes

## Prerequisites

- ROS2 Jazzy
- Gazebo Harmonic (gz-sim8)
- Python 3.8 or newer
- OpenAI-compatible LLM server (e.g., Groq, LM Studio, Ollama, OpenAI API)

## Installation

### 1. Clone the repository

```bash
cd ~/ros2_ws/src
git clone https://github.com/<your-fork>/gazebo_world_generator.git
git clone https://github.com/<your-fork>/gazebo_ros2_2Dmap_plugin.git   # harmonic-port branch
```

### 2. Install dependencies

```bash
# Python packages
cd gazebo_world_generator
pip install -r requirements.txt --break-system-packages

# ROS2 / Gazebo packages (Jazzy + Gazebo Harmonic)
sudo apt-get install libgz-sim8-dev ros-jazzy-ros-gz-sim ros-jazzy-ros-gz-bridge

# Build the occupancy map plugin (from the harmonic-port branch)
cd ~/ros2_ws
colcon build --packages-select gazebo_ros2_2dmap_plugin gazebo_world_generator
source install/setup.bash
```

### 3. Configure your LLM server

Edit the configuration file `config/generator_config.yaml`:

```bash
nano config/generator_config.yaml
```

Set your server details:

```yaml
llm:
  server_url: "http://localhost:1234/v1"
  model_name: "your-model-name"
```

### 4. Build the package

```bash
cd ~/ros2_ws
colcon build --packages-select gazebo_world_generator
source install/setup.bash
```

## Quick Start

Run the generator:

```bash
ros2 run gazebo_world_generator generate_world
```

This opens an interactive menu:

1. Create a new world from description
2. Refine an existing world
3. Test your LLM server connection
4. Exit

## Usage Examples (non-interactive)

### Basic world creation

```bash
ros2 run gazebo_world_generator generate_world \
    --description "Office with 4 desks and chairs"
```

### With room dimensions

Specify exact sizes using these formats: `"12m x 10m office"`, `"warehouse (20x15m)"`, or `"office of size 8x6"`

```bash
ros2 run gazebo_world_generator generate_world \
    --description "12m x 10m warehouse with 8 storage racks"
```

### Warehouse environments

For warehouses with storage racks and pallets, the system automatically uses optimized grid-based placement:

```bash
ros2 run gazebo_world_generator generate_world \
    --description "Warehouse with 10 storage racks and 5 pallets"
```

**Note:** Always specify exact quantities for objects. The system will generate only what you explicitly request (e.g., "5 pallets" creates exactly 5 pallets, not more).

### All available options

```bash
ros2 run gazebo_world_generator generate_world \
    --description "your world description" \
    --llm-server http://localhost:1234/v1 \
    --model your-model-name \
    --output custom-filename.sdf \
    --debug
```

If no output filename is specified, a timestamped name is generated automatically.

### Refinement examples

Use the interactive menu (option 2) to modify existing worlds:

- "Add 3 more desks with chairs"
- "Remove all bookshelves"
- "Move desks closer to the entrance"

## Configuration

Edit `config/generator_config.yaml` to configure your LLM server and other settings:

```yaml
llm:
  server_url: "http://localhost:1234/v1"  # Your LLM server endpoint
  model_name: "your-model-name"            # Your model name

output:
  base_directory: "generated_worlds"       # Output directory
  auto_generate_map: true                  # Generate navigation maps

placement:
  min_object_distance: 0.5                 # Object spacing (meters)
  corridor_width: 1.2                      # Traffic flow space (meters)
```

The generator requires an **OpenAI-compatible LLM server endpoint**. Compatible servers include LM Studio, Ollama, OpenAI API, or any custom OpenAI-compatible API.

### Custom configuration location

For user-specific settings that persist across package updates:

```bash
mkdir -p ~/.config/gazebo_world_generator
cp config/generator_config.yaml ~/.config/gazebo_world_generator/
nano ~/.config/gazebo_world_generator/generator_config.yaml
```

Config priority: package directory → user home (`~/.config/`) → system (`/etc/`)

## Output Files

Generated worlds are saved in `generated_worlds/` with these files:

```plaintext
generated_worlds/
├── worlds/
│   └── world_YYYYMMDD_HHMMSS.sdf    # World file (SDF format)
├── logs/
│   └── world_YYYYMMDD_HHMMSS.log    # Generation log
└── occupancy_maps/
    ├── world_YYYYMMDD_HHMMSS.yaml   # Map metadata
    └── world_YYYYMMDD_HHMMSS.pgm    # 2D navigation map
```

**File format glossary:**

- SDF (Simulation Description Format) – Gazebo's world file format
- PGM (Portable Gray Map) – 2D occupancy grid for robot navigation
- YAML – Map configuration and metadata

## Command Line Options

Override config settings from the command line:

- `--description` – World description (required for non-interactive mode)
- `--llm-server` – Override LLM server URL
- `--model` – Override model name
- `--output` – Custom output filename
- `--debug` – Enable verbose logging

## How It Works

The generator uses an optimized multi-phase pipeline to transform your description into a complete simulation world:

1. **Dimension Extraction** – Parses explicit measurements (e.g., "12m x 10m")
2. **LLM Parsing** – Converts natural language to structured JSON with exact object counts
3. **Object Expansion** – Expands compressed object counts into individual instances
4. **Model Resolution** – Maps object names to Gazebo models with intelligent caching
5. **Layout Strategy** – Chooses between semantic grouping (offices) or grid placement (warehouses)
6. **Room Layout** – Optimizes room positions and connections
7. **Collision-Free Placement** – Positions objects with automatic collision avoidance
8. **World Generation** – Creates the SDF file with physics and lighting
9. **Map Generation** – Produces 2D navigation maps

## Troubleshooting

### LLM server connection issues

Test your connection:

```bash
# Built-in test
ros2 run gazebo_world_generator generate_world
# Select option 3: "Test LLM connection"

# Manual test
curl http://localhost:1234/v1/models
```

### No compatible Gazebo models found

This error occurs when the generator cannot find the model files corresponding to your request.

First, ensure your Gazebo model paths are set up correctly. If you have custom models, add their location to the GZ_SIM_RESOURCE_PATH environment variable:

```bash
export GZ_SIM_RESOURCE_PATH=~/my_gazebo_models:$GZ_SIM_RESOURCE_PATH
```

**Important**: For the generator to successfully spawn a model, its files must exist in one of two places:

- Your local GAZEBO_MODEL_PATH.

- The provided online Gazebo model repositories.

The system is smart enough to also search for synonyms (e.g., searching for "couch" if you ask for a "sofa"), but it still needs to find the actual model files in one of those locations. If the files are missing, the process will fail.

### LLM context limit errors

If you see errors like `'max_tokens' is too large`, the system automatically handles this by:

- Estimating input token count from your description
- Adjusting output token limits to fit within model constraints

For very large worlds (20+ objects), the system will automatically reduce response size. If generation fails:

- Split into smaller rooms or use batch refinement operations
- Use a model with larger context

### Missing objects or incorrect counts

The generator creates **only** the objects you explicitly specify:

- ❌ "warehouse" → Empty room (no objects added automatically)
- ✅ "warehouse with 10 storage racks" → Exactly 10 racks, nothing more

Always specify exact quantities in your description.

### Object overlaps

The system includes automatic collision avoidance. If issues persist:

- Increase room size in your description
- Reduce furniture quantity
- Check `generated_worlds/logs/` for warnings

### Build failures

Clean rebuild:

```bash
cd ~/ros2_ws
rm -rf build/ install/ log/
rosdep install --from-paths src --ignore-src -r -y
colcon build --packages-select gazebo_world_generator
```

### Package not found

Source your workspace:

```bash
source ~/ros2_ws/install/setup.bash
ros2 pkg list | grep gazebo_world_generator
```

## License

MIT License - Copyright (c) 2025 Service Robotics Lab

See [LICENSE](LICENSE) file for full details.

Forked from [robotics-upo/gazebo_world_generator](https://github.com/robotics-upo/gazebo_world_generator). This fork is a personal working copy for ROS2 Jazzy + Gazebo Harmonic — see [About This Fork](#about-this-fork) above for what changed and why.
