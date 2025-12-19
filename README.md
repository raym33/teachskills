# TeachSkills

Claude Code skill for building interactive educational web applications with 3D visualizations, gamification, and AI tutoring.

## Overview

This skill provides comprehensive instructions for creating modern educational content using:

- **React Three Fiber** - 3D visualizations and simulations
- **Rapier Physics** - Realistic physics simulations
- **Framer Motion** - Smooth animations
- **Zustand** - State management
- **KaTeX** - Mathematical notation
- **Local AI (LM Studio)** - AI tutoring without cloud dependencies

## Structure

```
.
├── SKILL.md                    # Core skill with 3D techniques and components
└── subjects/
    ├── PHYSICS.md              # Physics simulations, vectors, waves
    ├── CHEMISTRY.md            # Atomic models, molecules, reactions
    ├── MATHEMATICS.md          # Geometry, functions, equations
    ├── GAMIFICATION.md         # Quizzes, XP, achievements, exercises
    ├── ADAPTIVE-TUTOR.md       # AI error analysis and personalized feedback
    └── VIRTUAL-LABS.md         # Interactive lab simulations
```

## Features

### Core Components (SKILL.md)
- Educational book layout with pages and chapters
- 3D Canvas with optimized performance
- Glass/crystal materials with physical properties
- Particle systems and animated shaders
- Local AI chatbot integration

### Subject-Specific Content

| Subject | Key Features |
|---------|-------------|
| **Physics** | Projectile motion, pendulums, electric fields, wave superposition |
| **Chemistry** | Bohr atom model, electron clouds, periodic table, molecule builder |
| **Mathematics** | Interactive geometry, function plotter, equation solver |

### Gamification System
- XP and leveling with logarithmic curve
- Streak tracking and achievements
- 5 exercise types: quiz, drag-drop, fill-blanks, ordering, matching
- Combo multipliers and speed bonuses
- Confetti celebrations

### Adaptive AI Tutor
- Error classification (conceptual, calculation, formula, etc.)
- Pattern detection across attempts
- Personalized explanations based on learning profile
- Spaced repetition recommendations
- Student diagnostic panel

### Virtual Laboratories
- **Density Lab** - Liquids, floating objects, real physics
- **Motion Lab** - Kinematics with live graphs
- **Circuit Lab** - Ohm's Law with interactive components

## Installation

1. Copy the skill files to your Claude Code project:
```bash
mkdir -p .claude/skills/edu-webapp
cp -r * .claude/skills/edu-webapp/
```

2. Install dependencies:
```bash
npm install @react-three/fiber @react-three/drei @react-three/rapier three zustand framer-motion react-katex canvas-confetti @dnd-kit/core @dnd-kit/sortable
```

3. For AI tutoring, install [LM Studio](https://lmstudio.ai) and start the local server.

## Usage

When creating educational content, Claude Code will automatically follow these instructions based on the subject matter.

Example prompts:
- "Create a 3D visualization of the scientific method"
- "Build a quiz about chemical reactions with gamification"
- "Make an interactive density lab experiment"
- "Add a math function plotter with derivative visualization"

## Requirements

- React 18+
- Node.js 18+
- For 3D: WebGL-capable browser
- For AI tutor: LM Studio running locally (optional)

## License

MIT
