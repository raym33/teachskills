# TeachSkills API Reference

Complete reference for all components, stores, and utilities.

## Table of Contents

- [Layout Components](#layout-components)
- [Educational Components](#educational-components)
- [Interactive Components](#interactive-components)
- [3D Components](#3d-components)
- [Gamification Components](#gamification-components)
- [State Stores](#state-stores)
- [Utilities](#utilities)

---

## Layout Components

### BookPage

Container for a lesson page with chapter context.

```tsx
interface BookPageProps {
  pageNumber: number
  chapter: string
  children: React.ReactNode
}
```

**Usage:**

```tsx
<BookPage pageNumber={12} chapter="Properties of Matter">
  <Heading level={2}>Density</Heading>
  <p>Content here...</p>
</BookPage>
```

### Heading

Semantic heading with consistent styling.

```tsx
interface HeadingProps {
  level: 1 | 2 | 3 | 4
  children: React.ReactNode
  id?: string
}
```

**Usage:**

```tsx
<Heading level={2} id="density-definition">
  What is Density?
</Heading>
```

---

## Educational Components

### EducationalBox

Highlighted box for definitions, examples, warnings, etc.

```tsx
interface EducationalBoxProps {
  type: 'definition' | 'example' | 'warning' | 'note' | 'experiment'
  title?: string
  children: React.ReactNode
}
```

**Usage:**

```tsx
<EducationalBox type="definition" title="Density">
  <p>Mass per unit volume of a substance.</p>
</EducationalBox>

<EducationalBox type="warning">
  <p>Handle chemicals with care!</p>
</EducationalBox>
```

### Term

Highlighted terminology with optional tooltip.

```tsx
interface TermProps {
  children: string
  definition?: string
}
```

**Usage:**

```tsx
<p>
  The <Term definition="Mass per unit volume">density</Term> determines buoyancy.
</p>
```

### Formula

LaTeX formula display using KaTeX.

```tsx
interface FormulaProps {
  math: string
  display?: boolean  // Block or inline
}
```

**Usage:**

```tsx
<Formula math="\rho = \frac{m}{V}" display />
<p>Where <Formula math="\rho" /> is density.</p>
```

---

## Interactive Components

### Quiz

Basic quiz component.

```tsx
interface Question {
  id: string
  text: string
  options: string[]
  correctIndex: number
  explanation: string
}

interface QuizProps {
  questions: Question[]
  exerciseId: string
  onComplete?: (score: number, total: number) => void
}
```

**Usage:**

```tsx
const questions = [
  {
    id: 'q1',
    text: 'What is the density of water?',
    options: ['0.5 g/cm³', '1.0 g/cm³', '1.5 g/cm³', '2.0 g/cm³'],
    correctIndex: 1,
    explanation: 'Pure water has a density of 1.0 g/cm³ at 4°C.',
  },
]

<Quiz questions={questions} exerciseId="density-quiz" />
```

### GamifiedQuiz

Enhanced quiz with XP, combos, timer, and achievements.

```tsx
interface GamifiedQuestion extends Question {
  difficulty: 'easy' | 'medium' | 'hard'
  points: number
  hint?: string
  latex?: string
  image?: string
}

interface GamifiedQuizProps {
  questions: GamifiedQuestion[]
  topicId: string
  title: string
  timeLimit?: number        // Seconds per question (0 = no limit)
  shuffleQuestions?: boolean
  shuffleOptions?: boolean
  showHints?: boolean
  allowRetry?: boolean
  onComplete?: (score: number, total: number) => void
}
```

**Usage:**

```tsx
<GamifiedQuiz
  questions={advancedQuestions}
  topicId="density"
  title="Density Challenge"
  timeLimit={30}
  showHints={true}
  allowRetry={true}
/>
```

### DragDropExercise

Categorization exercise with drag and drop.

```tsx
interface DragDropExerciseProps {
  instruction: string
  items: { id: string; content: string }[]
  targets: { id: string; label: string; correctItemId: string }[]
  onComplete: (correct: number, total: number) => void
}
```

**Usage:**

```tsx
<DragDropExercise
  instruction="Drag each object to whether it floats or sinks"
  items={[
    { id: 'cork', content: 'Cork' },
    { id: 'iron', content: 'Iron nail' },
  ]}
  targets={[
    { id: 'floats', label: 'Floats', correctItemId: 'cork' },
    { id: 'sinks', label: 'Sinks', correctItemId: 'iron' },
  ]}
  onComplete={(c, t) => console.log(`${c}/${t} correct`)}
/>
```

### FillBlanksExercise

Fill-in-the-blank text exercise.

```tsx
interface FillBlanksExerciseProps {
  text: string  // Use {{blank:id:answer}} for blanks
  hints?: Record<string, string>
  caseSensitive?: boolean
  onComplete: (correct: number, total: number) => void
}
```

**Usage:**

```tsx
<FillBlanksExercise
  text="Density equals {{blank:m:mass}} divided by {{blank:v:volume}}."
  hints={{ m: 'Measured in kg', v: 'Measured in m³' }}
  caseSensitive={false}
  onComplete={(c, t) => console.log(`${c}/${t}`)}
/>
```

### AIChatbot

Local AI tutor chatbot.

```tsx
interface AIChatbotProps {
  position?: 'floating' | 'inline'
  endpoint?: string              // Default: http://localhost:1234/v1/chat/completions
  systemPrompt?: string
  title?: string
  modelName?: string
  suggestedQuestions?: string[]
  defaultCollapsed?: boolean
  placeholder?: string
}
```

**Usage:**

```tsx
<AIChatbot
  position="floating"
  title="Physics Tutor"
  systemPrompt="You are a helpful physics teacher..."
  suggestedQuestions={[
    "Why do things float?",
    "How do I calculate density?",
  ]}
/>
```

---

## 3D Components

### Canvas Setup

Optimized React Three Fiber canvas.

```tsx
import { Canvas } from '@react-three/fiber'
import { OrbitControls, Environment } from '@react-three/drei'

<Canvas
  camera={{ position: [0, 0, 5], fov: 50 }}
  dpr={[1, 2]}
  gl={{ antialias: true, powerPreference: 'high-performance' }}
>
  <ambientLight intensity={0.5} />
  <pointLight position={[10, 10, 10]} />

  <Suspense fallback={null}>
    <YourScene />
  </Suspense>

  <OrbitControls enablePan={false} />
  <Environment preset="studio" />
</Canvas>
```

### Physics Setup

Rapier physics integration.

```tsx
import { Physics, RigidBody } from '@react-three/rapier'

<Physics gravity={[0, -9.81, 0]}>
  <RigidBody type="dynamic" position={[0, 5, 0]}>
    <mesh>
      <sphereGeometry args={[0.5]} />
      <meshStandardMaterial color="red" />
    </mesh>
  </RigidBody>

  <RigidBody type="fixed">
    <mesh position={[0, -1, 0]}>
      <boxGeometry args={[10, 0.5, 10]} />
      <meshStandardMaterial color="gray" />
    </mesh>
  </RigidBody>
</Physics>
```

### Glass Material

Physical glass/crystal material.

```tsx
<meshPhysicalMaterial
  color="#ffffff"
  transparent
  opacity={0.3}
  roughness={0}
  metalness={0}
  transmission={0.95}
  thickness={0.5}
  ior={1.5}
/>
```

### Float Animation

Gentle floating animation.

```tsx
import { Float } from '@react-three/drei'

<Float
  speed={2}
  rotationIntensity={0.5}
  floatIntensity={0.5}
>
  <mesh>...</mesh>
</Float>
```

---

## Gamification Components

### ProgressBar

Level progress indicator.

```tsx
interface ProgressBarProps {
  showLevel?: boolean
  showXP?: boolean
  compact?: boolean
}
```

**Usage:**

```tsx
<ProgressBar showLevel showXP />
```

### AchievementPanel

Display all achievements with unlock status.

```tsx
interface AchievementPanelProps {
  showLocked?: boolean
  filterCategory?: 'mastery' | 'streak' | 'explorer' | 'speed' | 'perfect'
}
```

**Usage:**

```tsx
<AchievementPanel showLocked />
```

### XPNotification

Toast notification for XP gains.

```tsx
interface XPNotificationProps {
  amount: number
  reason?: string
  duration?: number  // ms
}
```

**Usage:**

```tsx
// Typically triggered automatically by the store
showXPNotification({ amount: 50, reason: 'Perfect score!' })
```

---

## State Stores

### useGamificationStore

Zustand store for gamification state.

```tsx
interface GamificationState {
  // State
  xp: number
  level: number
  streak: number
  lastActivityDate: string | null
  achievements: Achievement[]
  unlockedAchievements: string[]
  topicProgress: Record<string, TopicProgress>

  // Actions
  addXP: (amount: number, reason?: string) => void
  incrementStreak: () => void
  resetStreak: () => void
  unlockAchievement: (id: string) => void
  updateTopicProgress: (topicId: string, score: number, total: number) => void
}
```

**Usage:**

```tsx
const { xp, level, addXP } = useGamificationStore()

// Award XP
addXP(100, 'Completed quiz')

// Get level
console.log(`Level ${level}`)
```

### useAdaptiveTutorStore

Zustand store for adaptive learning.

```tsx
interface AdaptiveTutorState {
  // State
  errors: ErrorRecord[]
  conceptMastery: Record<string, ConceptMastery>
  learningProfile: LearningProfile

  // Actions
  recordError: (error: Omit<ErrorRecord, 'id' | 'timestamp'>) => void
  recordCorrect: (questionId: string, topicId: string, concept: string) => void
  markUnderstood: (errorId: string, understood: boolean) => void
  getWeakConcepts: () => ConceptMastery[]
  getRecommendedReview: () => ConceptMastery[]
  analyzeErrorPattern: (concept: string) => ErrorType | null
}
```

**Usage:**

```tsx
const { recordError, getWeakConcepts } = useAdaptiveTutorStore()

// Record an error
recordError({
  questionId: 'q1',
  topicId: 'density',
  concept: 'density-calculation',
  questionText: '...',
  correctAnswer: '...',
  userAnswer: '...',
  errorType: 'calculation',
  timeSpentMs: 15000,
  hintsUsed: false,
  aiExplanationShown: false,
  understoodAfter: null,
})

// Get weak areas
const weakAreas = getWeakConcepts()
```

### useVirtualLabStore

Zustand store for lab experiments.

```tsx
interface VirtualLabState {
  // State
  labId: string
  variables: LabVariable[]
  isRunning: boolean
  isPaused: boolean
  simulationSpeed: number
  currentExperiment: LabExperiment | null
  experimentHistory: LabExperiment[]

  // Actions
  setVariable: (id: string, value: number) => void
  startExperiment: (hypothesis?: string) => void
  pauseExperiment: () => void
  stopExperiment: () => void
  recordDataPoint: () => void
  addNote: (note: string) => void
  resetLab: () => void
}
```

---

## Utilities

### classifyError

Classify student errors for adaptive feedback.

```tsx
async function classifyError(
  question: {
    text: string
    correctAnswer: string
    userAnswer: string
    concept: string
    hasFormula: boolean
    hasUnits: boolean
    timeSpentMs: number
    averageTimeMs: number
  },
  aiEndpoint?: string
): Promise<ErrorClassification>

interface ErrorClassification {
  type: ErrorType
  confidence: number
  explanation: string
  suggestedApproach: string
}

type ErrorType =
  | 'conceptual'
  | 'calculation'
  | 'unit_conversion'
  | 'formula'
  | 'reading'
  | 'careless'
  | 'partial'
  | 'unknown'
```

### generateExplanation

Generate personalized explanation for errors.

```tsx
async function generateExplanation(
  endpoint: string,
  request: ExplanationRequest
): Promise<AdaptiveExplanation>

interface ExplanationRequest {
  concept: string
  errorType: ErrorType
  question: string
  correctAnswer: string
  userAnswer: string
  studentLevel: 'beginner' | 'intermediate' | 'advanced'
  previousErrors: ErrorRecord[]
  prefersVisual: boolean
  prefersStepByStep: boolean
}

interface AdaptiveExplanation {
  mainExplanation: string
  steps?: string[]
  analogies?: string[]
  practiceQuestion?: {
    text: string
    answer: string
    hint: string
  }
  relatedConcepts?: string[]
}
```

### generateEducationalImage

Generate images with Imagen 3.

```tsx
async function generateEducationalImage(
  request: ImagenRequest
): Promise<string[]>

interface ImagenRequest {
  prompt: string
  negativePrompt?: string
  aspectRatio?: '1:1' | '16:9' | '9:16' | '4:3'
  style?: 'photorealistic' | 'digital-art' | 'sketch' | 'watercolor'
}
```

### generateEducationalVideo

Generate videos with Veo 3.

```tsx
async function generateEducationalVideo(
  request: VeoRequest
): Promise<string>

interface VeoRequest {
  prompt: string
  durationSeconds?: 5 | 8
  aspectRatio?: '16:9' | '9:16' | '1:1'
  fps?: 24 | 30
}
```

### askGemini

Call Gemini in headless mode.

```tsx
function askGemini(
  prompt: string,
  expectJson?: boolean
): GeminiResponse

interface GeminiResponse {
  success: boolean
  content: string
  parsed?: any
}
```

---

## CSS Variables

Standard design tokens used across components:

```css
:root {
  /* Colors */
  --color-primary: #4CAF50;
  --color-primary-dark: #388E3C;
  --color-secondary: #2196F3;
  --color-warning: #FF9800;
  --color-error: #f44336;
  --color-success: #4CAF50;

  /* Spacing */
  --spacing-xs: 0.25rem;
  --spacing-sm: 0.5rem;
  --spacing-md: 1rem;
  --spacing-lg: 1.5rem;
  --spacing-xl: 2rem;

  /* Border radius */
  --radius-sm: 4px;
  --radius-md: 8px;
  --radius-lg: 12px;

  /* Typography */
  --font-family: 'Inter', system-ui, sans-serif;
  --font-size-sm: 0.875rem;
  --font-size-md: 1rem;
  --font-size-lg: 1.25rem;

  /* Shadows */
  --shadow-sm: 0 1px 2px rgba(0,0,0,0.1);
  --shadow-md: 0 4px 6px rgba(0,0,0,0.1);
  --shadow-lg: 0 10px 20px rgba(0,0,0,0.15);
}
```

---

## TypeScript Types

Common types used throughout:

```tsx
// Achievement
interface Achievement {
  id: string
  name: string
  description: string
  icon: string
  category: 'mastery' | 'streak' | 'explorer' | 'speed' | 'perfect'
  unlockedAt?: Date
}

// Topic Progress
interface TopicProgress {
  completed: number
  total: number
  bestScore: number
  attempts: number
}

// Concept Mastery
interface ConceptMastery {
  conceptId: string
  name: string
  totalAttempts: number
  correctAttempts: number
  lastAttempt: Date
  masteryLevel: 'struggling' | 'learning' | 'practicing' | 'mastered'
  errorPatterns: ErrorType[]
  needsReview: boolean
  nextReviewDate: Date | null
}

// Lab Variable
interface LabVariable {
  id: string
  name: string
  symbol: string
  unit: string
  value: number
  min: number
  max: number
  step: number
  isControlled: boolean
  isDependent: boolean
}
```

---

For more examples, see the subject-specific files in the `subjects/` directory.
