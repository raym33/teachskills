# TeachSkills Architecture

This document explains the technical architecture of TeachSkills and how its components work together.

## System Overview

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│                              TEACHSKILLS                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐       │
│  │   SKILL FILES   │     │  REACT APP      │     │   EXTERNAL      │       │
│  │                 │     │                 │     │   SERVICES      │       │
│  │  • SKILL.md     │────▶│  • Components   │◀───▶│                 │       │
│  │  • PHYSICS.md   │     │  • 3D Scenes    │     │  • LM Studio    │       │
│  │  • CHEMISTRY.md │     │  • State        │     │  • Gemini API   │       │
│  │  • ...          │     │  • Styles       │     │  • Imagen 3     │       │
│  └─────────────────┘     └─────────────────┘     │  • Veo 3        │       │
│          │                       │               └─────────────────┘       │
│          │                       │                       │                  │
│          ▼                       ▼                       ▼                  │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │                        CLAUDE CODE                               │       │
│  │                                                                  │       │
│  │   Reads skills ──▶ Understands patterns ──▶ Generates code      │       │
│  │                                                                  │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Component Layers

### Layer 1: Skill Files (Instructions)

These Markdown files contain patterns and examples that Claude Code follows:

```text
.claude/skills/edu-webapp/
├── SKILL.md                    # Core patterns
├── MULTI-LLM-ORCHESTRATION.md  # Multi-model workflow
└── subjects/
    ├── PHYSICS.md              # Physics-specific patterns
    ├── CHEMISTRY.md            # Chemistry-specific patterns
    ├── MATHEMATICS.md          # Math-specific patterns
    ├── GAMIFICATION.md         # Game mechanics patterns
    ├── ADAPTIVE-TUTOR.md       # AI tutoring patterns
    └── VIRTUAL-LABS.md         # Lab simulation patterns
```

### Layer 2: React Components (Generated)

Claude Code generates these based on skill instructions:

```text
src/
├── components/
│   ├── layout/
│   │   ├── BookPage.tsx        # Page container
│   │   ├── Heading.tsx         # Semantic headings
│   │   └── Navigation.tsx      # Chapter navigation
│   │
│   ├── educational/
│   │   ├── EducationalBox.tsx  # Definition, example, warning boxes
│   │   ├── Term.tsx            # Highlighted terminology
│   │   └── Formula.tsx         # LaTeX formula display
│   │
│   ├── interactive/
│   │   ├── Quiz.tsx            # Basic quiz
│   │   ├── GamifiedQuiz.tsx    # Quiz with XP, combos
│   │   ├── DragDropExercise.tsx
│   │   ├── FillBlanksExercise.tsx
│   │   └── AIChatbot.tsx       # LM Studio integration
│   │
│   └── gamification/
│       ├── ProgressBar.tsx     # Level progress
│       ├── AchievementPanel.tsx
│       └── XPNotification.tsx
│
├── 3d/
│   ├── ScientificMethod3D.tsx  # 3D cycle visualization
│   ├── Laboratory3D.tsx        # Lab equipment
│   ├── AtomModel3D.tsx         # Bohr model
│   ├── MoleculeViewer3D.tsx    # 3D molecules
│   └── VirtualLabs/
│       ├── DensityLab.tsx
│       ├── MotionLab.tsx
│       └── CircuitLab.tsx
│
├── store/
│   ├── bookStore.ts            # Navigation state
│   ├── gamificationStore.ts    # XP, achievements
│   └── adaptiveTutorStore.ts   # Error tracking
│
└── styles/
    ├── global.css
    ├── components.css
    └── 3d.css
```

### Layer 3: State Management (Zustand)

Persistent state for gamification and learning analytics:

```typescript
// gamificationStore.ts
interface GamificationState {
  // Progress
  xp: number
  level: number
  streak: number

  // Achievements
  achievements: Achievement[]
  unlockedAchievements: string[]

  // Topic mastery
  topicProgress: Record<string, TopicProgress>

  // Actions
  addXP: (amount: number) => void
  unlockAchievement: (id: string) => void
  updateTopicProgress: (topicId: string, score: number) => void
}
```

```typescript
// adaptiveTutorStore.ts
interface AdaptiveTutorState {
  // Error history
  errors: ErrorRecord[]

  // Concept mastery
  conceptMastery: Record<string, ConceptMastery>

  // Learning profile
  learningProfile: LearningProfile

  // Actions
  recordError: (error: ErrorRecord) => void
  analyzeErrorPattern: (concept: string) => ErrorType
  getWeakConcepts: () => ConceptMastery[]
}
```

### Layer 4: 3D Rendering (React Three Fiber)

Optimized 3D canvas setup:

```tsx
<Canvas
  camera={{ position: [0, 0, 5], fov: 50 }}
  dpr={[1, 2]}
  performance={{ min: 0.5 }}
  gl={{
    antialias: true,
    powerPreference: 'high-performance',
  }}
>
  <Suspense fallback={<Loader />}>
    <Scene />
  </Suspense>

  <OrbitControls
    enablePan={false}
    minDistance={3}
    maxDistance={10}
  />
</Canvas>
```

Physics simulation with Rapier:

```tsx
<Physics gravity={[0, -9.81, 0]}>
  <RigidBody type="dynamic">
    <mesh>
      <sphereGeometry />
      <meshStandardMaterial />
    </mesh>
  </RigidBody>
</Physics>
```

### Layer 5: External Services

**LM Studio (Local AI):**

```text
┌─────────────────────────────────────────┐
│              LM Studio                   │
│                                          │
│  localhost:1234                          │
│    └── /v1/chat/completions             │
│    └── /v1/models                       │
│                                          │
│  Models:                                 │
│    • Mistral-7B-Instruct                │
│    • Llama-3-8B-Instruct                │
│    • Phi-3-mini                          │
└─────────────────────────────────────────┘
```

**Google AI APIs:**

```text
┌─────────────────────────────────────────┐
│           Google AI Studio               │
│                                          │
│  Gemini API                              │
│    └── gemini-2.0-flash (design)        │
│                                          │
│  Imagen 3 API                            │
│    └── imagen-3.0-generate-002          │
│    └── imagen-3.0-fast-generate-001     │
│                                          │
│  Veo 3 API                               │
│    └── veo-3.0-generate-preview         │
└─────────────────────────────────────────┘
```

## Data Flow

### Quiz Completion Flow

```text
User answers question
        │
        ▼
┌───────────────────┐
│ GamifiedQuiz      │
│ Component         │
└────────┬──────────┘
         │
         ├──────────────────────────────────┐
         ▼                                  ▼
┌───────────────────┐           ┌───────────────────┐
│ gamificationStore │           │ adaptiveTutorStore│
│                   │           │                   │
│ • addXP()         │           │ • recordError()   │
│ • incrementStreak │           │ • classifyError() │
│ • checkAchievement│           │ • updateMastery() │
└────────┬──────────┘           └────────┬──────────┘
         │                               │
         ▼                               ▼
┌───────────────────┐           ┌───────────────────┐
│ XP Notification   │           │ Adaptive Feedback │
│ Achievement Toast │           │ Personalized      │
└───────────────────┘           │ Explanation       │
                                └───────────────────┘
```

### AI Chatbot Flow

```text
User asks question
        │
        ▼
┌───────────────────┐
│ AIChatbot         │
│ Component         │
└────────┬──────────┘
         │
         ▼
┌───────────────────────────────────────┐
│ Build message with:                    │
│ • System prompt (subject-specific)    │
│ • Conversation history                │
│ • Current context (lesson topic)      │
└────────┬──────────────────────────────┘
         │
         ▼
┌───────────────────────────────────────┐
│ POST localhost:1234/v1/chat/completions│
│                                        │
│ {                                      │
│   "model": "local-model",             │
│   "messages": [...],                  │
│   "stream": true                      │
│ }                                      │
└────────┬──────────────────────────────┘
         │
         ▼
┌───────────────────┐
│ Stream response   │
│ to UI             │
└───────────────────┘
```

### Multi-LLM Generation Flow

```text
User requests lesson
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│                        Claude Code                             │
│                      (Orchestrator)                            │
└───────────────────────────────────────────────────────────────┘
        │
        ├───────────────────┬───────────────────┬───────────────┐
        ▼                   ▼                   ▼               │
┌───────────────┐   ┌───────────────┐   ┌───────────────┐      │
│ Gemini        │   │ Imagen 3      │   │ Veo 3         │      │
│ (headless)    │   │               │   │               │      │
│               │   │ Generate      │   │ Generate      │      │
│ Design:       │   │ images for:   │   │ videos for:   │      │
│ • Layout      │   │ • Diagrams    │   │ • Animations  │      │
│ • Quiz Q's    │   │ • Concepts    │   │ • Processes   │      │
│ • Structure   │   │ • Scenes      │   │ • Demos       │      │
└───────┬───────┘   └───────┬───────┘   └───────┬───────┘      │
        │                   │                   │               │
        └───────────────────┴───────────────────┘               │
                            │                                   │
                            ▼                                   │
                    ┌───────────────┐                          │
                    │ lesson.json   │◀─────────────────────────┘
                    │               │
                    │ • design      │
                    │ • images[]    │
                    │ • videos[]    │
                    │ • quiz[]      │
                    └───────┬───────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Claude        │
                    │ Implements    │
                    │ in React      │
                    └───────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Complete      │
                    │ Lesson.tsx    │
                    └───────────────┘
```

## Performance Considerations

### 3D Optimization

```tsx
// Use instancing for repeated elements
<Instances>
  {particles.map((p, i) => (
    <Instance key={i} position={p.position} />
  ))}
</Instances>

// Level of detail
<Detailed distances={[0, 10, 25]}>
  <HighDetail />
  <MediumDetail />
  <LowDetail />
</Detailed>

// Suspense for loading
<Suspense fallback={<Loader />}>
  <HeavyComponent />
</Suspense>
```

### State Optimization

```tsx
// Selective subscriptions
const xp = useGamificationStore(state => state.xp)

// Memoized selectors
const weakConcepts = useAdaptiveTutorStore(
  useCallback(state => state.getWeakConcepts(), [])
)
```

### Media Caching

```typescript
// Cache generated images/videos
const getCachedMedia = (prompt: string): string | null => {
  const hash = md5(prompt).slice(0, 12)
  const cached = localStorage.getItem(`media-${hash}`)
  return cached
}

const cacheMedia = (prompt: string, data: string): void => {
  const hash = md5(prompt).slice(0, 12)
  localStorage.setItem(`media-${hash}`, data)
}
```

## Security Considerations

### Local AI

- All AI processing happens locally via LM Studio
- No student data sent to external servers
- Conversations stored only in browser localStorage

### API Keys

```bash
# Store in environment variables, never in code
export GEMINI_API_KEY="..."

# Use .env files (gitignored)
GEMINI_API_KEY=...
```

### Content Safety

```typescript
// Imagen 3 safety settings
{
  safetyFilterLevel: 'block_some',
  personGeneration: 'dont_allow',
}
```

## Extensibility

### Adding New Subjects

Create a new file in `subjects/`:

```markdown
# subjects/BIOLOGY.md

## Cell Structure Components

### Cell3D Component
...patterns and examples...

## Photosynthesis Simulation
...patterns and examples...
```

### Adding New Exercise Types

Extend `GAMIFICATION.md` with new patterns:

```markdown
### New Exercise Type: Labeling

```tsx
interface LabelingExerciseProps {
  image: string
  labels: { id: string; text: string; correctPosition: Point }[]
}
```
```

### Adding New Virtual Labs

Extend `VIRTUAL-LABS.md`:

```markdown
## Laboratory 4: Optics Lab

### Components
- Light source with adjustable wavelength
- Lenses and mirrors
- Ray tracing visualization
```

---

## Summary

TeachSkills is a layered architecture:

1. **Skill files** provide patterns and examples
2. **Claude Code** interprets and generates components
3. **React components** render the UI and 3D
4. **Zustand stores** manage persistent state
5. **External services** provide AI capabilities

This separation allows:

- Easy addition of new subjects
- Consistent code generation
- Flexible deployment options
- Privacy-first AI integration
