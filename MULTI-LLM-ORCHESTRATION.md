# Multi-LLM Orchestration for Educational Content

Orquestar múltiples LLMs especializados para crear contenido educativo de alta calidad: Claude para código, Gemini para diseño y generación de medios.

---

## Filosofía: Cada LLM en lo que mejor hace

```
┌─────────────────────────────────────────────────────────────────┐
│                    FLUJO DE CREACIÓN                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  USUARIO ──► CLAUDE CODE (Orquestador)                          │
│                    │                                             │
│         ┌─────────┼─────────┐                                   │
│         ▼         ▼         ▼                                   │
│    ┌─────────┐ ┌─────────┐ ┌─────────┐                         │
│    │ GEMINI  │ │ IMAGEN  │ │  VEO 3  │                         │
│    │ Design  │ │  3      │ │ Video   │                         │
│    └────┬────┘ └────┬────┘ └────┬────┘                         │
│         │           │           │                               │
│         └─────────┬─────────────┘                               │
│                   ▼                                             │
│            CLAUDE CODE                                          │
│         (Integra en React)                                      │
│                   │                                             │
│                   ▼                                             │
│         LIBRO EDUCATIVO COMPLETO                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

| Tarea | LLM | Razón |
|-------|-----|-------|
| Diseño UI/UX, layouts | Gemini | Mejor razonamiento espacial |
| Generación de imágenes | Imagen 3 (Nano/Pro) | Calidad fotorrealista |
| Videos educativos | Veo 3 | Videos de hasta 8 segundos |
| Código React/Three.js | Claude | Mejor estructura de código |
| Fórmulas LaTeX | Claude | Precisión matemática |
| Lógica de estado | Claude | Razonamiento complejo |
| Preguntas de quiz creativas | Gemini | Creatividad en contenido |

---

## Configuración

### Variables de entorno

```bash
# En .env o exportar en terminal
export GEMINI_API_KEY="tu-api-key-de-google"
export GOOGLE_AI_STUDIO_KEY="tu-api-key"  # Para Imagen 3 y Veo 3
```

### Instrucciones en CLAUDE.md

```markdown
# Multi-LLM para contenido educativo

## Gemini CLI (headless mode)
Para tareas de diseño visual, usa Gemini CLI:

```bash
gemini --headless --prompt "tu prompt"
```

## Generación de imágenes con Imagen 3
Para ilustraciones educativas:

```bash
curl -X POST "https://generativelanguage.googleapis.com/v1beta/models/imagen-3.0-generate-002:predict" \
  -H "Authorization: Bearer $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [{"prompt": "Educational illustration of..."}],
    "parameters": {"sampleCount": 1, "aspectRatio": "16:9"}
  }'
```

## Generación de videos con Veo 3
Para animaciones educativas cortas:

```bash
curl -X POST "https://generativelanguage.googleapis.com/v1beta/models/veo-3.0-generate-preview:predict" \
  -H "Authorization: Bearer $GEMINI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [{"prompt": "Educational animation showing..."}],
    "parameters": {"durationSeconds": 5, "aspectRatio": "16:9"}
  }'
```
```

---

## Gemini Headless Mode

### Instalación

```bash
# Instalar Gemini CLI
npm install -g @anthropic-ai/gemini-cli

# O con pip
pip install gemini-cli
```

### Uso desde Claude Code

```bash
# Diseño de un componente 3D
gemini --headless --prompt "Design a 3D layout for an interactive periodic table.
Describe:
1. Element card design (size, colors, typography)
2. 3D arrangement (camera angle, spacing)
3. Hover/click interactions
4. Color coding by element type
5. Responsive considerations

Output as JSON with specific measurements and hex colors."

# Generación de contenido de quiz
gemini --headless --prompt "Create 5 multiple choice questions about photosynthesis for 8th grade students.
Include:
- Question text
- 4 options (1 correct, 3 plausible distractors)
- Explanation of correct answer
- Difficulty level (easy/medium/hard)
- Related concept tags

Output as JSON array."
```

### Wrapper para integración

```typescript
// utils/geminiHeadless.ts
import { execSync } from 'child_process'

interface GeminiResponse {
  success: boolean
  content: string
  parsed?: any
}

export const askGemini = (prompt: string, expectJson = false): GeminiResponse => {
  try {
    const result = execSync(
      `gemini --headless --prompt "${prompt.replace(/"/g, '\\"')}"`,
      { encoding: 'utf-8', maxBuffer: 1024 * 1024 * 10 }
    )

    return {
      success: true,
      content: result,
      parsed: expectJson ? JSON.parse(result) : undefined,
    }
  } catch (error) {
    return {
      success: false,
      content: error.message,
    }
  }
}

// Uso
const design = askGemini(`
  Design a 3D molecule viewer for H2O.
  Include atom positions, bond angles, and colors.
  Output as JSON.
`, true)

if (design.success && design.parsed) {
  // Claude usa el diseño para generar código React
  console.log(design.parsed)
}
```

---

## Imagen 3 - Generación de Ilustraciones

### API Setup

```typescript
// utils/imagen3.ts
const IMAGEN_API = 'https://generativelanguage.googleapis.com/v1beta/models/imagen-3.0-generate-002:predict'

interface ImagenRequest {
  prompt: string
  negativePrompt?: string
  aspectRatio?: '1:1' | '16:9' | '9:16' | '4:3' | '3:4'
  style?: 'photorealistic' | 'digital-art' | 'sketch' | 'watercolor'
}

interface ImagenResponse {
  images: { base64: string; mimeType: string }[]
}

export const generateEducationalImage = async (
  request: ImagenRequest
): Promise<string[]> => {
  const response = await fetch(IMAGEN_API, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.GEMINI_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      instances: [{
        prompt: buildEducationalPrompt(request.prompt, request.style),
      }],
      parameters: {
        sampleCount: 1,
        aspectRatio: request.aspectRatio || '16:9',
        negativePrompt: request.negativePrompt || 'text, watermarks, logos, blurry, low quality',
        safetyFilterLevel: 'block_some',
        personGeneration: 'dont_allow', // Para contenido educativo seguro
      },
    }),
  })

  const data: ImagenResponse = await response.json()
  return data.images.map(img => `data:${img.mimeType};base64,${img.base64}`)
}

// Prompt engineering para contenido educativo
const buildEducationalPrompt = (basePrompt: string, style?: string): string => {
  const styleGuide = {
    'photorealistic': 'Photorealistic, high detail, professional photography, educational textbook quality',
    'digital-art': 'Clean digital illustration, flat design, educational infographic style, vibrant colors',
    'sketch': 'Hand-drawn sketch style, pencil on paper, scientific illustration, labeled diagram',
    'watercolor': 'Soft watercolor painting, artistic, nature illustration style',
  }

  return `${basePrompt}. ${styleGuide[style || 'digital-art']}. Safe for educational use, no text overlays.`
}
```

### Prompts Educativos Optimizados

```typescript
// Ejemplos de prompts para cada asignatura

const PHYSICS_PROMPTS = {
  projectileMotion: `
    A clean diagram showing projectile motion trajectory.
    A ball launched at 45 degrees, showing parabolic path with dotted lines.
    Velocity vectors at different points (initial, peak, final).
    Grid background for scale reference.
    Scientific illustration style, educational textbook quality.
  `,

  waveInterference: `
    Two water waves meeting and creating interference pattern.
    Bird's eye view of circular wave fronts.
    Constructive interference shown in bright areas.
    Destructive interference in calm areas.
    Blue and white color scheme, high contrast.
  `,

  electricField: `
    Electric field lines around a positive and negative charge.
    Field lines flowing from positive to negative.
    Color gradient showing field strength.
    Clean scientific diagram style.
    Dark background with glowing lines.
  `,
}

const CHEMISTRY_PROMPTS = {
  atomicStructure: `
    Cutaway view of an atom showing nucleus with protons and neutrons.
    Electron shells with orbiting electrons at different energy levels.
    Soft glow effect around electrons.
    Scientific illustration, not cartoonish.
    Purple and blue color scheme.
  `,

  waterMolecule: `
    H2O molecule in 3D showing molecular geometry.
    Oxygen atom (red) bonded to two hydrogen atoms (white).
    Bond angle of 104.5 degrees visible.
    Electron density clouds around atoms.
    Clean white background.
  `,

  periodicTrends: `
    Periodic table heat map showing atomic radius trend.
    Larger atoms in warm colors (red, orange).
    Smaller atoms in cool colors (blue, purple).
    Clean infographic style with gradient.
  `,
}

const BIOLOGY_PROMPTS = {
  cellStructure: `
    Cross-section of an animal cell showing all organelles.
    Nucleus, mitochondria, endoplasmic reticulum, golgi apparatus visible.
    Soft pastel colors, each organelle distinct.
    Scientific illustration style, textbook quality.
  `,

  photosynthesis: `
    Diagram of a chloroplast during photosynthesis.
    Light rays entering, CO2 and H2O arrows going in.
    O2 and glucose arrows coming out.
    Green color scheme with energy flow visualization.
  `,

  dnaHelix: `
    DNA double helix structure with base pairs visible.
    Adenine-Thymine and Guanine-Cytosine pairs color coded.
    Sugar-phosphate backbone clearly visible.
    Soft blue lighting, 3D rendered look.
  `,
}

const MATH_PROMPTS = {
  pythagoreanTheorem: `
    Right triangle with squares on each side.
    a² + b² = c² visualization.
    Clean geometric diagram, primary colors.
    Grid paper background.
  `,

  functionGraph: `
    Coordinate plane with parabola y = x².
    Clean axes with numbers marked.
    Curve in blue, grid in light gray.
    Mathematical graph style.
  `,

  fractalPattern: `
    Mandelbrot set fractal visualization.
    Vibrant colors showing iteration depth.
    Mathematical beauty, infinite detail zoom.
    High resolution, psychedelic but educational.
  `,
}
```

### Componente React para Imágenes Generadas

```tsx
// components/GeneratedImage.tsx
import React, { useState, useEffect } from 'react'
import { generateEducationalImage } from '../utils/imagen3'

interface GeneratedImageProps {
  prompt: string
  aspectRatio?: '1:1' | '16:9' | '9:16'
  style?: 'photorealistic' | 'digital-art' | 'sketch'
  alt: string
  fallbackSrc?: string
  caption?: string
  onGenerated?: (src: string) => void
}

export const GeneratedImage: React.FC<GeneratedImageProps> = ({
  prompt,
  aspectRatio = '16:9',
  style = 'digital-art',
  alt,
  fallbackSrc,
  caption,
  onGenerated,
}) => {
  const [imageSrc, setImageSrc] = useState<string | null>(null)
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const generateImage = async () => {
      try {
        setIsLoading(true)
        const images = await generateEducationalImage({
          prompt,
          aspectRatio,
          style,
        })

        if (images.length > 0) {
          setImageSrc(images[0])
          onGenerated?.(images[0])
        }
      } catch (err) {
        setError('Error generating image')
        console.error(err)
      } finally {
        setIsLoading(false)
      }
    }

    generateImage()
  }, [prompt, aspectRatio, style])

  if (isLoading) {
    return (
      <div className="generated-image-placeholder">
        <div className="loading-spinner" />
        <p>Generando ilustración...</p>
      </div>
    )
  }

  if (error && fallbackSrc) {
    return <img src={fallbackSrc} alt={alt} className="generated-image" />
  }

  return (
    <figure className="generated-image-container">
      <img
        src={imageSrc || fallbackSrc}
        alt={alt}
        className="generated-image"
        loading="lazy"
      />
      {caption && <figcaption>{caption}</figcaption>}
      <span className="ai-badge">🤖 Generado con IA</span>
    </figure>
  )
}
```

---

## Veo 3 - Videos Educativos

### API Setup

```typescript
// utils/veo3.ts
const VEO_API = 'https://generativelanguage.googleapis.com/v1beta/models/veo-3.0-generate-preview:predict'

interface VeoRequest {
  prompt: string
  durationSeconds?: 5 | 8  // Veo 3 soporta hasta 8 segundos
  aspectRatio?: '16:9' | '9:16' | '1:1'
  fps?: 24 | 30
}

interface VeoResponse {
  videos: {
    base64: string
    mimeType: string
    durationSeconds: number
  }[]
}

export const generateEducationalVideo = async (
  request: VeoRequest
): Promise<string> => {
  const response = await fetch(VEO_API, {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.GEMINI_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      instances: [{
        prompt: buildVideoPrompt(request.prompt),
      }],
      parameters: {
        durationSeconds: request.durationSeconds || 5,
        aspectRatio: request.aspectRatio || '16:9',
        fps: request.fps || 24,
        safetyFilterLevel: 'block_some',
      },
    }),
  })

  const data: VeoResponse = await response.json()

  if (data.videos.length > 0) {
    return `data:${data.videos[0].mimeType};base64,${data.videos[0].base64}`
  }

  throw new Error('No video generated')
}

const buildVideoPrompt = (basePrompt: string): string => {
  return `${basePrompt}.
    Educational animation style, smooth motion, clear visualization.
    No text overlays, suitable for all ages.
    Scientific accuracy, professional quality.
    Smooth camera movement, good lighting.`
}
```

### Prompts para Videos Educativos

```typescript
const VIDEO_PROMPTS = {
  // FÍSICA
  physics: {
    pendulum: `
      Simple pendulum swinging back and forth.
      Smooth sinusoidal motion, showing period of oscillation.
      Soft lighting, dark background, glowing pendulum bob.
      Camera static, side view.
    `,

    projectile: `
      Ball being thrown in parabolic arc.
      Slow motion, showing trajectory clearly.
      Dotted trail following the path.
      Outdoor setting, blue sky background.
    `,

    waves: `
      Water wave propagating across surface.
      Bird's eye view, concentric circles expanding.
      Blue water, white wave crests.
      Smooth, mesmerizing motion.
    `,

    magneticField: `
      Iron filings aligning around a bar magnet.
      Time lapse showing field lines forming.
      Dark background, particles glowing slightly.
      Camera slowly zooming out.
    `,
  },

  // QUÍMICA
  chemistry: {
    electronOrbit: `
      Electron orbiting atomic nucleus.
      Smooth circular motion, glowing trail.
      Nucleus pulsing slightly, space-like background.
      Quantum probability cloud fading in and out.
    `,

    chemicalReaction: `
      Two molecules approaching and bonding.
      Slow motion collision, energy release as light.
      Atoms rearranging into new molecule.
      Space-like dark background.
    `,

    stateChange: `
      Ice cube melting into water.
      Close-up showing molecular motion increasing.
      Steam rising as it heats further.
      Side lighting showing phase transitions.
    `,

    crystalGrowth: `
      Crystal structure growing from solution.
      Time lapse, geometric patterns forming.
      Translucent crystal catching light.
      Macro photography style.
    `,
  },

  // BIOLOGÍA
  biology: {
    cellDivision: `
      Cell undergoing mitosis.
      Chromosomes aligning, separating, cell pinching.
      Soft colors, microscope view aesthetic.
      Educational animation style.
    `,

    heartbeat: `
      Heart pumping blood in cross-section view.
      Rhythmic contraction, blood flow visible.
      Red and blue for oxygenated/deoxygenated.
      Medical illustration style.
    `,

    photosynthesis: `
      Sunlight hitting a leaf, zooming into chloroplast.
      Light energy converting to chemical energy.
      Green glow, molecular visualization.
      Nature documentary style.
    `,

    neuronFiring: `
      Electrical signal traveling down neuron axon.
      Glowing pulse moving along nerve fiber.
      Synaptic transmission at the end.
      Dark background, bioluminescent effect.
    `,
  },

  // MATEMÁTICAS
  math: {
    fractal: `
      Mandelbrot set zoom, infinite detail emerging.
      Psychedelic colors, mathematical beauty.
      Smooth continuous zoom, never ending.
      Mesmerizing patterns.
    `,

    geometricTransform: `
      Shape rotating and transforming in 3D space.
      Cube morphing into sphere, then back.
      Clean geometric style, grid background.
      Smooth continuous motion.
    `,

    graphPlotting: `
      Function being drawn on coordinate plane.
      Sine wave appearing from left to right.
      Glowing line, dark background with grid.
      Mathematical visualization style.
    `,
  },
}
```

### Componente de Video Generado

```tsx
// components/GeneratedVideo.tsx
import React, { useState, useEffect, useRef } from 'react'
import { generateEducationalVideo } from '../utils/veo3'

interface GeneratedVideoProps {
  prompt: string
  duration?: 5 | 8
  aspectRatio?: '16:9' | '9:16'
  autoPlay?: boolean
  loop?: boolean
  controls?: boolean
  caption?: string
  onGenerated?: (src: string) => void
}

export const GeneratedVideo: React.FC<GeneratedVideoProps> = ({
  prompt,
  duration = 5,
  aspectRatio = '16:9',
  autoPlay = true,
  loop = true,
  controls = true,
  caption,
  onGenerated,
}) => {
  const [videoSrc, setVideoSrc] = useState<string | null>(null)
  const [isLoading, setIsLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)
  const [progress, setProgress] = useState(0)
  const videoRef = useRef<HTMLVideoElement>(null)

  useEffect(() => {
    const generateVideo = async () => {
      try {
        setIsLoading(true)
        setProgress(10)

        // Simular progreso mientras genera
        const progressInterval = setInterval(() => {
          setProgress(p => Math.min(p + 10, 90))
        }, 2000)

        const videoData = await generateEducationalVideo({
          prompt,
          durationSeconds: duration,
          aspectRatio,
        })

        clearInterval(progressInterval)
        setProgress(100)
        setVideoSrc(videoData)
        onGenerated?.(videoData)
      } catch (err) {
        setError('Error generating video')
        console.error(err)
      } finally {
        setIsLoading(false)
      }
    }

    generateVideo()
  }, [prompt, duration, aspectRatio])

  if (isLoading) {
    return (
      <div className="video-placeholder">
        <div className="loading-animation">
          <div className="film-reel" />
        </div>
        <p>Generando video educativo...</p>
        <div className="progress-bar">
          <div className="progress-fill" style={{ width: `${progress}%` }} />
        </div>
        <span className="progress-text">{progress}%</span>
      </div>
    )
  }

  if (error) {
    return (
      <div className="video-error">
        <p>❌ {error}</p>
        <button onClick={() => window.location.reload()}>Reintentar</button>
      </div>
    )
  }

  return (
    <figure className="generated-video-container">
      <video
        ref={videoRef}
        src={videoSrc || undefined}
        autoPlay={autoPlay}
        loop={loop}
        controls={controls}
        muted={autoPlay} // Necesario para autoplay
        playsInline
        className="generated-video"
      />
      {caption && <figcaption>{caption}</figcaption>}
      <span className="ai-badge">🎬 Video generado con Veo 3</span>
    </figure>
  )
}
```

---

## Flujo Completo: Gemini Diseña → Claude Implementa

### Script de Orquestación

```typescript
// scripts/generateLesson.ts
import { askGemini } from '../utils/geminiHeadless'
import { generateEducationalImage } from '../utils/imagen3'
import { generateEducationalVideo } from '../utils/veo3'
import { writeFileSync } from 'fs'

interface LessonRequest {
  topic: string
  subject: 'physics' | 'chemistry' | 'biology' | 'math'
  gradeLevel: number
  includeImages: boolean
  includeVideos: boolean
}

export const generateLesson = async (request: LessonRequest) => {
  console.log(`📚 Generating lesson: ${request.topic}`)

  // 1. Gemini diseña la estructura y contenido
  console.log('🎨 Step 1: Gemini designing lesson structure...')
  const designResponse = await askGemini(`
    Design an interactive lesson about "${request.topic}" for grade ${request.gradeLevel}.
    Subject: ${request.subject}

    Include:
    1. Learning objectives (3-5 bullet points)
    2. Key concepts with simple explanations
    3. 3D visualization ideas (describe what to show)
    4. Image descriptions for illustrations needed
    5. Short video animation ideas (5-8 seconds each)
    6. Quiz questions (5 multiple choice)
    7. Interactive exercise ideas

    Output as detailed JSON.
  `, true)

  if (!designResponse.success) {
    throw new Error('Gemini design failed')
  }

  const lessonDesign = designResponse.parsed

  // 2. Generar imágenes con Imagen 3
  const images: string[] = []
  if (request.includeImages && lessonDesign.imageDescriptions) {
    console.log('🖼️ Step 2: Generating images with Imagen 3...')
    for (const desc of lessonDesign.imageDescriptions) {
      const img = await generateEducationalImage({
        prompt: desc,
        style: 'digital-art',
        aspectRatio: '16:9',
      })
      images.push(img[0])
    }
  }

  // 3. Generar videos con Veo 3
  const videos: string[] = []
  if (request.includeVideos && lessonDesign.videoIdeas) {
    console.log('🎬 Step 3: Generating videos with Veo 3...')
    for (const idea of lessonDesign.videoIdeas.slice(0, 2)) { // Limit to 2 videos
      const video = await generateEducationalVideo({
        prompt: idea,
        durationSeconds: 5,
      })
      videos.push(video)
    }
  }

  // 4. Compilar todo en un objeto de lección
  console.log('📦 Step 4: Compiling lesson package...')
  const lessonPackage = {
    ...lessonDesign,
    generatedImages: images,
    generatedVideos: videos,
    generatedAt: new Date().toISOString(),
  }

  // 5. Guardar para que Claude genere el código React
  writeFileSync(
    `./generated/lesson-${request.topic.replace(/\s+/g, '-')}.json`,
    JSON.stringify(lessonPackage, null, 2)
  )

  console.log('✅ Lesson package ready for Claude to implement!')
  return lessonPackage
}

// Uso
generateLesson({
  topic: 'Photosynthesis',
  subject: 'biology',
  gradeLevel: 8,
  includeImages: true,
  includeVideos: true,
})
```

### Prompt para Claude: Implementar el Diseño

```markdown
## Instrucciones para Claude Code

Cuando recibas un archivo JSON de lección generado por el flujo multi-LLM:

1. **Lee el archivo JSON** con el diseño de Gemini
2. **Genera componentes React** siguiendo el skill SKILL.md
3. **Integra las imágenes** base64 en componentes <img>
4. **Integra los videos** en componentes <video>
5. **Crea el quiz** con el sistema de gamificación
6. **Añade visualizaciones 3D** basadas en las descripciones

### Ejemplo de transformación:

```json
// Input: lesson-photosynthesis.json
{
  "topic": "Photosynthesis",
  "objectives": ["Understand light absorption", "..."],
  "visualization3D": "Chloroplast cross-section with thylakoids",
  "generatedImages": ["data:image/png;base64,..."],
  "generatedVideos": ["data:video/mp4;base64,..."],
  "quiz": [...]
}
```

```tsx
// Output: Photosynthesis.tsx
import React from 'react'
import { BookPage, Heading, EducationalBox } from '../components'
import { GamifiedQuiz } from '../components/GamifiedQuiz'
import { ChloroplastViewer3D } from '../3d/ChloroplastViewer3D'

const PhotosynthesisLesson: React.FC = () => (
  <BookPage chapter="Biología" pageNumber={42}>
    <Heading level={2}>La Fotosíntesis</Heading>

    {/* Objetivos */}
    <EducationalBox type="objective">
      <ul>
        <li>Comprender la absorción de luz</li>
        {/* ... */}
      </ul>
    </EducationalBox>

    {/* Video generado */}
    <figure>
      <video src="data:video/mp4;base64,..." autoPlay loop muted />
      <figcaption>Animación del proceso de fotosíntesis</figcaption>
    </figure>

    {/* Visualización 3D */}
    <ChloroplastViewer3D />

    {/* Imagen generada */}
    <img src="data:image/png;base64,..." alt="Diagrama de cloroplasto" />

    {/* Quiz */}
    <GamifiedQuiz questions={photosynthesisQuiz} topicId="photosynthesis" />
  </BookPage>
)
```
```

---

## Caché y Optimización

### Almacenar Medios Generados

```typescript
// utils/mediaCache.ts
import { existsSync, readFileSync, writeFileSync, mkdirSync } from 'fs'
import { createHash } from 'crypto'

const CACHE_DIR = './generated-media-cache'

// Crear directorio de caché si no existe
if (!existsSync(CACHE_DIR)) {
  mkdirSync(CACHE_DIR, { recursive: true })
}

const getPromptHash = (prompt: string): string => {
  return createHash('md5').update(prompt).digest('hex').slice(0, 12)
}

export const getCachedImage = (prompt: string): string | null => {
  const hash = getPromptHash(prompt)
  const path = `${CACHE_DIR}/img-${hash}.txt`

  if (existsSync(path)) {
    return readFileSync(path, 'utf-8')
  }
  return null
}

export const cacheImage = (prompt: string, base64: string): void => {
  const hash = getPromptHash(prompt)
  const path = `${CACHE_DIR}/img-${hash}.txt`
  writeFileSync(path, base64)
}

export const getCachedVideo = (prompt: string): string | null => {
  const hash = getPromptHash(prompt)
  const path = `${CACHE_DIR}/vid-${hash}.txt`

  if (existsSync(path)) {
    return readFileSync(path, 'utf-8')
  }
  return null
}

export const cacheVideo = (prompt: string, base64: string): void => {
  const hash = getPromptHash(prompt)
  const path = `${CACHE_DIR}/vid-${hash}.txt`
  writeFileSync(path, base64)
}
```

### Generación Lazy con Placeholder

```tsx
// components/LazyGeneratedMedia.tsx
import React, { useState, useEffect, useRef } from 'react'
import { useInView } from 'react-intersection-observer'

interface LazyGeneratedMediaProps {
  type: 'image' | 'video'
  prompt: string
  placeholder?: React.ReactNode
}

export const LazyGeneratedMedia: React.FC<LazyGeneratedMediaProps> = ({
  type,
  prompt,
  placeholder,
}) => {
  const [media, setMedia] = useState<string | null>(null)
  const [isGenerating, setIsGenerating] = useState(false)
  const { ref, inView } = useInView({
    triggerOnce: true,
    threshold: 0.1,
  })

  useEffect(() => {
    if (inView && !media && !isGenerating) {
      generateMedia()
    }
  }, [inView])

  const generateMedia = async () => {
    setIsGenerating(true)

    // Primero intentar caché
    const cached = type === 'image'
      ? getCachedImage(prompt)
      : getCachedVideo(prompt)

    if (cached) {
      setMedia(cached)
      setIsGenerating(false)
      return
    }

    // Si no hay caché, generar
    try {
      const result = type === 'image'
        ? await generateEducationalImage({ prompt })
        : await generateEducationalVideo({ prompt })

      const mediaData = Array.isArray(result) ? result[0] : result

      // Guardar en caché
      if (type === 'image') {
        cacheImage(prompt, mediaData)
      } else {
        cacheVideo(prompt, mediaData)
      }

      setMedia(mediaData)
    } catch (error) {
      console.error('Generation failed:', error)
    } finally {
      setIsGenerating(false)
    }
  }

  return (
    <div ref={ref} className="lazy-media-container">
      {media ? (
        type === 'image' ? (
          <img src={media} alt="Generated illustration" />
        ) : (
          <video src={media} autoPlay loop muted playsInline />
        )
      ) : isGenerating ? (
        <div className="generating-placeholder">
          <div className="spinner" />
          <p>Generando {type === 'image' ? 'imagen' : 'video'}...</p>
        </div>
      ) : (
        placeholder || <div className="placeholder">📷</div>
      )}
    </div>
  )
}
```

---

## Estilos CSS

```css
/* Media generada */
.generated-image-container,
.generated-video-container {
  position: relative;
  margin: var(--spacing-lg) 0;
  border-radius: var(--radius-lg);
  overflow: hidden;
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.generated-image,
.generated-video {
  width: 100%;
  height: auto;
  display: block;
}

.ai-badge {
  position: absolute;
  bottom: 8px;
  right: 8px;
  background: rgba(0, 0, 0, 0.7);
  color: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 0.7rem;
}

figcaption {
  text-align: center;
  padding: var(--spacing-sm);
  background: #f5f5f5;
  font-size: 0.875rem;
  color: #666;
}

/* Placeholder de carga */
.generated-image-placeholder,
.video-placeholder {
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  min-height: 200px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  border-radius: var(--radius-lg);
  color: white;
}

.loading-spinner {
  width: 40px;
  height: 40px;
  border: 3px solid rgba(255, 255, 255, 0.3);
  border-top-color: white;
  border-radius: 50%;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  to { transform: rotate(360deg); }
}

.progress-bar {
  width: 200px;
  height: 6px;
  background: rgba(255, 255, 255, 0.3);
  border-radius: 3px;
  margin-top: var(--spacing-sm);
  overflow: hidden;
}

.progress-fill {
  height: 100%;
  background: white;
  border-radius: 3px;
  transition: width 0.3s ease;
}

/* Film reel animation for video */
.film-reel {
  width: 50px;
  height: 50px;
  border: 4px solid white;
  border-radius: 50%;
  position: relative;
  animation: spin 2s linear infinite;
}

.film-reel::before,
.film-reel::after {
  content: '';
  position: absolute;
  width: 8px;
  height: 8px;
  background: white;
  border-radius: 50%;
}

.film-reel::before {
  top: 5px;
  left: 50%;
  transform: translateX(-50%);
}

.film-reel::after {
  bottom: 5px;
  left: 50%;
  transform: translateX(-50%);
}

/* Lazy loading placeholder */
.lazy-media-container {
  min-height: 200px;
  background: #f0f0f0;
  border-radius: var(--radius-lg);
  display: flex;
  align-items: center;
  justify-content: center;
}

.generating-placeholder {
  text-align: center;
  color: #666;
}

.placeholder {
  font-size: 3rem;
  opacity: 0.3;
}
```

---

## Dependencias Adicionales

```json
{
  "dependencies": {
    "react-intersection-observer": "^9.5.0"
  },
  "devDependencies": {
    "@anthropic-ai/gemini-cli": "^1.0.0"
  }
}
```

---

## Resumen de APIs

| Servicio | Modelo | Uso | Límites |
|----------|--------|-----|---------|
| Gemini CLI | gemini-2.0-flash | Diseño, contenido | Gratis con cuenta Google |
| Imagen 3 | imagen-3.0-generate-002 | Ilustraciones | Pay-per-use |
| Imagen 3 Nano | imagen-3.0-fast-generate-001 | Drafts rápidos | Más barato, menor calidad |
| Veo 3 | veo-3.0-generate-preview | Videos 5-8s | Preview, límites de uso |

---

## Ejemplo de Uso Completo

```bash
# Terminal 1: Usuario pide crear una lección
claude "Crea una lección interactiva sobre el ciclo del agua para 6º de primaria"

# Claude internamente:
# 1. Llama a Gemini headless para diseño
# 2. Genera imágenes con Imagen 3 (nube, lluvia, evaporación)
# 3. Genera video con Veo 3 (agua evaporándose)
# 4. Implementa componentes React con Three.js
# 5. Crea quiz gamificado
# 6. Devuelve la lección completa
```
