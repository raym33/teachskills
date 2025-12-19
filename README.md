# TeachSkills

A comprehensive Claude Code skill for building interactive educational web applications with 3D visualizations, gamification, AI tutoring, and multi-LLM content generation.

## Overview

TeachSkills provides detailed instructions for creating modern educational content that rivals commercial e-learning platforms. It leverages cutting-edge web technologies and AI to create engaging, personalized learning experiences.

### Core Technologies

- **React Three Fiber** - 3D visualizations and physics simulations
- **Rapier Physics** - Realistic physics for virtual labs
- **Framer Motion** - Smooth animations and transitions
- **Zustand** - Lightweight state management
- **KaTeX** - Beautiful mathematical notation
- **Local AI (LM Studio)** - Privacy-first AI tutoring
- **Gemini + Imagen 3 + Veo 3** - Multi-LLM content generation

## Repository Structure

```text
.
├── README.md                   # This file
├── SKILL.md                    # Core skill: 3D components, layouts, AI chatbot
├── MULTI-LLM-ORCHESTRATION.md  # Multi-LLM workflow documentation
│
└── subjects/
    ├── PHYSICS.md              # Physics simulations and formulas
    ├── CHEMISTRY.md            # Atomic models, molecules, reactions
    ├── MATHEMATICS.md          # Geometry, functions, equations
    ├── GAMIFICATION.md         # XP system, quizzes, achievements
    ├── ADAPTIVE-TUTOR.md       # AI error analysis and feedback
    └── VIRTUAL-LABS.md         # Interactive laboratory simulations
```

## Features

### 1. Core Components (SKILL.md)

The main skill file provides patterns for:

- **Educational Book Layout** - Pages, chapters, navigation
- **3D Canvas Setup** - Optimized React Three Fiber configuration
- **Glass/Crystal Materials** - Physical materials for scientific equipment
- **Particle Systems** - Background effects and visualizations
- **Animated Shaders** - GLSL for liquid effects, electron clouds
- **Local AI Chatbot** - LM Studio integration for student Q&A

### 2. Subject-Specific Instructions

Each subject file contains specialized components and patterns:

| Subject | Key Components |
| ------- | -------------- |
| **Physics** | Projectile motion, pendulums, electric fields, wave superposition, LaTeX formulas |
| **Chemistry** | Bohr atom model, electron probability clouds, 3D periodic table, molecule builder, equation balancer |
| **Mathematics** | Interactive geometry (Mafs), function plotter with derivatives, step-by-step equation solver |

### 3. Gamification System (GAMIFICATION.md)

Complete game mechanics for educational engagement:

- **XP & Leveling** - Logarithmic progression curve
- **Streak Tracking** - Daily engagement rewards
- **Achievements** - 5 categories with unlock notifications
- **5 Exercise Types**:
  - Multiple choice quizzes with timer
  - Drag-and-drop categorization
  - Fill-in-the-blanks
  - Sequence ordering
  - Matching pairs
- **Combo System** - Multipliers for consecutive correct answers
- **Visual Feedback** - Confetti, animations, sound effects

### 4. Adaptive AI Tutor (ADAPTIVE-TUTOR.md)

Intelligent tutoring that learns from student mistakes:

- **Error Classification** - Categorizes mistakes (conceptual, calculation, formula, reading, careless)
- **Pattern Detection** - Identifies recurring error types
- **Personalized Explanations** - Adapts to learning style and level
- **Spaced Repetition** - Recommends review based on forgetting curve
- **Diagnostic Dashboard** - Visual progress and weak areas

### 5. Virtual Laboratories (VIRTUAL-LABS.md)

Interactive science experiments with real physics:

| Lab | Concepts | Interactions |
| --- | -------- | ------------ |
| **Density Lab** | Buoyancy, liquid layers | Select liquids, drop objects, observe floating/sinking |
| **Motion Lab** | Kinematics, v-t graphs | Adjust velocity, acceleration, friction; see live graphs |
| **Circuit Lab** | Ohm's Law, V=IR | Connect components, adjust values, see LED brightness |

### 6. Multi-LLM Orchestration (MULTI-LLM-ORCHESTRATION.md)

Leverage multiple AI models for optimal content creation:

| Model | Best For | Example Use |
| ----- | -------- | ----------- |
| **Gemini (headless)** | UI/UX design, creative content | "Design a 3D periodic table layout" |
| **Imagen 3** | Educational illustrations | Diagrams, scientific visualizations |
| **Veo 3** | Short educational videos | 5-8 second animations of physics concepts |
| **Claude** | Code implementation | React components, state logic, LaTeX |

**Workflow:**
```text
User Request → Claude (orchestrator)
                    ├── Gemini: designs structure
                    ├── Imagen 3: generates illustrations
                    ├── Veo 3: creates animations
                    └── Claude: implements in React
                            ↓
               Complete Educational Lesson
```

## Installation

### 1. Add to Claude Code Project

```bash
# Clone this repository
git clone https://github.com/raym33/teachskills.git

# Copy to your Claude Code skills directory
mkdir -p .claude/skills/edu-webapp
cp -r teachskills/* .claude/skills/edu-webapp/
```

### 2. Install Dependencies

```bash
npm install @react-three/fiber @react-three/drei @react-three/rapier \
  three zustand framer-motion react-katex canvas-confetti \
  @dnd-kit/core @dnd-kit/sortable
```

### 3. Optional: Local AI Tutor

1. Download [LM Studio](https://lmstudio.ai)
2. Download a model (recommended: Mistral-7B-Instruct or Llama-3-8B)
3. Start the local server (default: `http://localhost:1234`)

### 4. Optional: Multi-LLM Generation

```bash
# Set up API keys
export GEMINI_API_KEY="your-google-api-key"

# Install Gemini CLI for headless mode
npm install -g @anthropic-ai/gemini-cli
```

## Usage Examples

When working with Claude Code, use prompts like:

```text
"Create a 3D visualization of the scientific method with 6 interactive steps"

"Build a chemistry quiz about the periodic table with gamification"

"Make an interactive density lab where students can drop objects in different liquids"

"Add a function plotter with derivative visualization and tangent lines"

"Generate illustrations for a lesson about photosynthesis"

"Create a motion simulation with adjustable velocity and real-time graphs"
```

Claude Code will automatically follow the skill instructions to generate appropriate components.

## How It Works

1. **Claude Code reads the skill files** when you start a conversation
2. **Based on your request**, it identifies which subject/feature applies
3. **It generates React components** following the documented patterns
4. **3D visualizations use Three.js** with optimized settings
5. **Gamification hooks into Zustand** for persistent progress
6. **AI tutor connects to LM Studio** for local, private tutoring

## Requirements

| Requirement | Minimum | Recommended |
| ----------- | ------- | ----------- |
| Node.js | 18+ | 20+ |
| React | 18+ | 18.2+ |
| Browser | WebGL 2.0 | Chrome/Firefox latest |
| RAM (for AI) | 8GB | 16GB+ |

## Documentation

| Document | Description |
| -------- | ----------- |
| [Getting Started](docs/GETTING-STARTED.md) | Step-by-step setup guide |
| [Architecture](docs/ARCHITECTURE.md) | System design and data flows |
| [API Reference](docs/API-REFERENCE.md) | Complete component API |
| [Examples Gallery](docs/EXAMPLES-GALLERY.md) | 50+ project ideas with prompts |
| [Printable Content](docs/PRINTABLE-CONTENT.md) | Print-ready books, manuals, and materials |

## Contributing

Contributions are welcome! Areas that could use expansion:

- Additional subjects (Biology, Geography, History)
- More virtual lab experiments
- Accessibility improvements
- Mobile-optimized 3D components
- Additional language support

## License

MIT

---

**Created with Claude Code** - The AI-powered coding assistant
