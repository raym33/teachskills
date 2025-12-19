# Getting Started with TeachSkills

This guide walks you through setting up TeachSkills and creating your first interactive educational content.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Installation](#installation)
3. [Quick Start](#quick-start)
4. [Your First Lesson](#your-first-lesson)
5. [Adding Gamification](#adding-gamification)
6. [Integrating AI Tutor](#integrating-ai-tutor)
7. [Multi-LLM Content Generation](#multi-llm-content-generation)
8. [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before you begin, ensure you have:

- **Node.js 18+** - [Download](https://nodejs.org/)
- **Claude Code** - [Installation guide](https://docs.anthropic.com/claude-code)
- **A React project** - Create one with `npx create-react-app my-edu-app --template typescript`

Optional but recommended:

- **LM Studio** - For local AI tutoring ([lmstudio.ai](https://lmstudio.ai))
- **Google AI API Key** - For Imagen 3 and Veo 3 content generation

---

## Installation

### Step 1: Clone TeachSkills

```bash
git clone https://github.com/raym33/teachskills.git
cd teachskills
```

### Step 2: Copy to Your Claude Code Project

```bash
# Create the skills directory in your project
mkdir -p /path/to/your/project/.claude/skills/edu-webapp

# Copy all skill files
cp -r * /path/to/your/project/.claude/skills/edu-webapp/
```

### Step 3: Install Dependencies

In your React project:

```bash
npm install @react-three/fiber @react-three/drei @react-three/rapier three
npm install zustand framer-motion react-katex katex
npm install canvas-confetti @dnd-kit/core @dnd-kit/sortable
```

### Step 4: Verify Installation

Your project structure should look like:

```text
your-project/
├── .claude/
│   └── skills/
│       └── edu-webapp/
│           ├── SKILL.md
│           ├── MULTI-LLM-ORCHESTRATION.md
│           └── subjects/
│               ├── PHYSICS.md
│               ├── CHEMISTRY.md
│               ├── MATHEMATICS.md
│               ├── GAMIFICATION.md
│               ├── ADAPTIVE-TUTOR.md
│               └── VIRTUAL-LABS.md
├── src/
├── package.json
└── ...
```

---

## Quick Start

Once installed, Claude Code will automatically use these skills when you make relevant requests.

### Example Prompts

**Create a 3D visualization:**

```text
Create an interactive 3D model of the solar system with orbiting planets
```

**Build a quiz:**

```text
Build a gamified quiz about chemical elements with XP and achievements
```

**Make a virtual lab:**

```text
Create a virtual lab for experimenting with pendulum motion
```

Claude Code reads the skill files and generates appropriate React components following the documented patterns.

---

## Your First Lesson

Let's create a simple physics lesson about density.

### Step 1: Ask Claude Code

```text
Create an interactive lesson page about density for 8th grade students.
Include:
- A definition box
- A 3D visualization of objects floating in water
- A simple quiz
```

### Step 2: Claude Code Generates

Claude will create components like:

```tsx
// src/lessons/DensityLesson.tsx
import React from 'react'
import { BookPage, Heading, EducationalBox } from '../components'
import { DensityVisualization3D } from '../3d/DensityVisualization3D'
import { Quiz } from '../components/Quiz'

const DensityLesson: React.FC = () => (
  <BookPage chapter="Properties of Matter" pageNumber={12}>
    <Heading level={2}>What is Density?</Heading>

    <EducationalBox type="definition">
      <p>
        <strong>Density</strong> is the amount of mass per unit volume.
        It determines whether an object floats or sinks in a fluid.
      </p>
    </EducationalBox>

    <DensityVisualization3D />

    <Quiz
      questions={densityQuiz}
      exerciseId="density-intro"
    />
  </BookPage>
)
```

### Step 3: Run and Test

```bash
npm start
```

Navigate to your lesson page and interact with the 3D visualization.

---

## Adding Gamification

Enhance engagement with XP, achievements, and progress tracking.

### Ask Claude Code

```text
Add gamification to the density lesson:
- Award XP for correct quiz answers
- Track streaks for daily study
- Add achievement badges
- Show a progress bar
```

### What Gets Added

Claude will integrate:

1. **Zustand store** for persistent state
2. **XP notifications** on correct answers
3. **Achievement unlocks** with animations
4. **Progress dashboard** component

```tsx
// Gamification automatically hooks into quizzes
<GamifiedQuiz
  questions={densityQuiz}
  topicId="density"
  title="Density Quiz"
  timeLimit={30}
  showHints={true}
/>

// Progress bar in header
<ProgressBar />

// Achievement notifications (automatic)
```

---

## Integrating AI Tutor

Add a local AI assistant for student questions.

### Setup LM Studio

1. Download from [lmstudio.ai](https://lmstudio.ai)
2. Install a model (recommended: **Mistral-7B-Instruct**)
3. Start the local server (runs on `http://localhost:1234`)

### Ask Claude Code

```text
Add an AI tutor chatbot to help students with density questions
```

### What Gets Added

```tsx
import { AIChatbot } from '../components/AIChatbot'

// In your lesson component
<AIChatbot
  position="floating"
  title="Density Tutor"
  systemPrompt={`You are a helpful physics tutor specializing in density.
    Explain concepts simply using everyday examples.
    When students make mistakes, guide them to the answer.`}
  suggestedQuestions={[
    "Why does ice float on water?",
    "How do I calculate density?",
    "What units is density measured in?",
  ]}
/>
```

The chatbot:

- Floats in the corner of the screen
- Connects to LM Studio automatically
- Shows connection status
- Provides suggested questions

---

## Multi-LLM Content Generation

Use Gemini for design and Imagen 3/Veo 3 for media generation.

### Setup

```bash
# Set your Google AI API key
export GEMINI_API_KEY="your-api-key-here"
```

### Ask Claude Code

```text
Generate a complete lesson about photosynthesis with:
- AI-generated illustrations of chloroplasts
- A short video animation of the process
- Interactive quiz designed by Gemini
```

### What Happens

1. **Claude orchestrates** the multi-LLM workflow
2. **Gemini designs** the lesson structure and quiz content
3. **Imagen 3 generates** scientific illustrations
4. **Veo 3 creates** a 5-second animation
5. **Claude implements** everything in React

```tsx
// Generated component with AI media
<figure>
  <img
    src={generatedChloroplastImage}
    alt="Chloroplast structure"
  />
  <figcaption>AI-generated illustration</figcaption>
</figure>

<video
  src={generatedPhotosynthesisVideo}
  autoPlay
  loop
  muted
/>
```

---

## Troubleshooting

### 3D Not Rendering

**Problem:** Canvas is blank or shows errors.

**Solution:** Ensure WebGL is enabled and you're using a modern browser.

```tsx
// Add error boundary
<Canvas
  onCreated={({ gl }) => {
    console.log('WebGL context created:', gl)
  }}
  fallback={<div>WebGL not supported</div>}
>
```

### AI Chatbot Not Connecting

**Problem:** "Cannot connect to model" error.

**Solutions:**

1. Ensure LM Studio is running
2. Check the server is on port 1234
3. Verify a model is loaded and active

```bash
# Test connection
curl http://localhost:1234/v1/models
```

### Gamification Not Persisting

**Problem:** Progress resets on page refresh.

**Solution:** Ensure Zustand persist middleware is configured:

```tsx
import { persist } from 'zustand/middleware'

export const useGamificationStore = create(
  persist(
    (set) => ({
      // ... state
    }),
    {
      name: 'gamification-storage',
    }
  )
)
```

### Physics Simulation Slow

**Problem:** Virtual lab runs at low FPS.

**Solutions:**

1. Reduce particle count
2. Lower physics timestep
3. Use simpler geometries

```tsx
<Physics
  gravity={[0, -9.81, 0]}
  timeStep={1/60}  // Reduce for performance
>
```

---

## Next Steps

- **Explore subjects:** Read `subjects/PHYSICS.md`, `CHEMISTRY.md`, `MATHEMATICS.md`
- **Add more labs:** See `subjects/VIRTUAL-LABS.md` for lab patterns
- **Customize AI tutor:** See `subjects/ADAPTIVE-TUTOR.md` for error analysis
- **Generate content:** See `MULTI-LLM-ORCHESTRATION.md` for media generation

---

## Getting Help

- **GitHub Issues:** [raym33/teachskills/issues](https://github.com/raym33/teachskills/issues)
- **Claude Code Docs:** [docs.anthropic.com/claude-code](https://docs.anthropic.com/claude-code)

---

**Happy Teaching!** 🎓
