# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TuriX-CUA is an AI-powered desktop automation agent for macOS. It uses Vision-Language Models (VLMs) to understand screen content and execute desktop actions through macOS Accessibility APIs and Quartz events.

## Common Commands

```bash
# Activate the existing virtual environment
source .venv/bin/activate

# Run the agent
python examples/main.py

# Run with custom config
python examples/main.py -c path/to/config.json

# Run tests
python -m pytest
python -m pytest tests/test_actions.py  # specific test file
python -m pytest --cov=src              # with coverage
```

## Quick Start

1. 修改 ./examples/config.json 的字段 agent.task 以描述需求。
2. 激活环境: `source .venv/bin/activate`
3. 执行: `python examples/main.py`
4. Agent 会自动截屏 → 分析界面 → 执行鼠标/键盘操作 → 循环直到任务完成，然后 agent 会退出。
5. 如果要执行下一个任务，就重复如上步骤。

## Configuration

Edit `examples/config.json` to configure:
- `llm`: Main VLM settings (provider, api_key, base_url, model_name)
- `planner_llm`: Planner model settings (used for task decomposition)
- `agent.task`: The task instruction to execute
- `agent.use_turix`: Use TuriX-optimized prompts (default: true)
- `agent.use_ui`: Enable UI tree extraction (default: false, uses screenshot-only mode)
- `agent.max_steps`: Maximum execution steps

Supported LLM providers: `turix`, `google_flash`, `google_pro`, `gpt`, `anthropic`

## Architecture

### Core Components

**Agent Layer** (`src/agent/`)
- `service.py`: Main `Agent` class - orchestrates the step-by-step execution loop. Each step: captures screenshot → builds UI tree (optional) → sends to LLM → parses response → executes actions
- `planner_service.py`: `Planner` class - pre-processes user task into step-by-step plan before execution
- `prompts.py`: System prompts (`SystemPrompt`, `SystemPrompt_turix`, `PlannerPrompt`) that instruct the LLM on output format and action usage
- `message_manager/service.py`: Manages conversation history and context window for LLM interactions
- `output_schemas.py`: JSON schemas for structured LLM output (OpenAI response_format)
- `structured_llm.py`: Pydantic models for parsing LLM responses (`AgentStepOutput`, `PlannerOutput`)

**Controller Layer** (`src/controller/`)
- `service.py`: `Controller` class - registers and executes actions. Default actions include: `done`, `input_text`, `open_app`, `run_apple_script`, `Hotkey`, `multi_Hotkey`, `Click`, `RightSingle`, `Drag`, `scroll_up`, `scroll_down`, `move_mouse`, `record_info`, `wait`
- `registry/service.py`: `Registry` class - decorator-based action registration system. Use `@controller.action(description, param_model)` to register custom actions
- `views.py`: Pydantic models for action parameters (e.g., `InputTextAction`, `LeftClickPixel`)

**Mac Integration** (`src/mac/`)
- `tree.py`: `MacUITreeBuilder` - builds UI element tree via macOS Accessibility APIs (AXUIElement), captures/annotates screenshots
- `actions.py`: Low-level macOS actions using Quartz/CoreGraphics for invisible mouse events and keyboard input
- `element.py`: `MacElementNode` - represents a UI element in the accessibility tree

### Data Flow

1. User provides task in `config.json`
2. `Planner` (if enabled) decomposes task into steps
3. `Agent.run()` executes step loop:
   - Capture screenshot via `MacUITreeBuilder.capture_screenshot()`
   - Optionally build UI tree via `MacUITreeBuilder.build_tree(pid)`
   - `MessageManager` formats state + history into LLM messages
   - LLM returns structured JSON with `current_state` and `action` list
   - `Controller.multi_act()` executes actions sequentially
   - Results stored in memory for next step

### Key Design Patterns

- **Structured Output**: Uses `llm.bind(response_format=...)` for OpenAI or `llm.with_structured_output()` for Anthropic/Google to ensure valid JSON responses
- **Action Registry**: Decorator pattern for registering actions with Pydantic parameter validation
- **Normalized Coordinates**: All screen positions are normalized to 0-1000 range for cross-resolution compatibility

## macOS Permissions Required

- **Accessibility**: System Settings → Privacy & Security → Accessibility → Add Terminal/IDE
- **Screen Recording**: System Settings → Privacy & Security → Screen Recording
- **Safari Automation**: Safari → Settings → Advanced → Enable "Show features for web developers", then Develop menu → Enable remote automation

## Code Style

- Follow PEP 8 with 88 character line limit (Black formatter)
- Use type hints for all function parameters and return values
- Use Google-style docstrings
- Async functions for all I/O operations
- Pydantic models for data validation

## Branch Information

- `main`: macOS support
- `windows`: Windows-specific implementation
