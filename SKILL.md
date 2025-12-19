# Skill: Educational Web App Creator (3D + WebGL)

Create premium interactive educational web applications with immersive 3D visualizations using Three.js and WebGL. Follow the "libro digital" pattern for Spanish curriculum education.

## When to Use

- Digital textbooks with 3D interactive content
- Science simulations (Physics, Chemistry, Biology)
- Virtual laboratories and experiments
- Math visualizations (geometry, calculus, algebra)
- Interactive learning platforms with gamification

---

## Tech Stack

| Layer | Technology | Purpose |
|-------|------------|---------|
| Framework | React 18+ TypeScript | Component architecture |
| 3D Engine | React Three Fiber | Declarative Three.js |
| 3D Helpers | @react-three/drei | Controls, effects, loaders |
| Physics | @react-three/rapier | Realistic simulations |
| Animation | Framer Motion | UI transitions |
| State | Zustand | Global state management |
| Shaders | GLSL | Custom visual effects |
| Post-processing | @react-three/postprocessing | Bloom, DOF, etc. |

### Installation

```bash
npm install three @react-three/fiber @react-three/drei @react-three/postprocessing
npm install framer-motion zustand
npm install @types/three --save-dev
```

---

## Project Structure

```
src/libro/
├── LibroApp.tsx
├── store/bookStore.ts
├── styles/global.css
├── components/
│   ├── layout/          # BookPage, BookLayout
│   ├── typography/      # Heading, Paragraph
│   ├── annotations/     # EducationalBox, Formula
│   ├── interactive/     # Quiz, DragAndDrop, Games
│   └── navigation/      # Sidebar, Navigation
├── 3d/
│   ├── core/            # Reusable 3D primitives
│   │   ├── GlassObject.tsx
│   │   ├── MetalObject.tsx
│   │   ├── LiquidSimulation.tsx
│   │   └── ParticleSystem.tsx
│   ├── scenes/          # Complete 3D scenes
│   │   ├── Laboratory3D.tsx
│   │   ├── SolarSystem3D.tsx
│   │   └── AtomicModel3D.tsx
│   ├── shaders/         # Custom GLSL shaders
│   │   ├── liquidShader.ts
│   │   ├── glowShader.ts
│   │   └── waveShader.ts
│   └── hooks/           # 3D-specific hooks
│       ├── useAnimation.ts
│       └── useInteraction.ts
└── pages/
    └── tema1/Tema1.tsx
```

---

## Three.js Best Practices for Education

### 1. Canvas Setup with Performance Optimization

```tsx
import { Canvas } from '@react-three/fiber'
import { Preload, AdaptiveDpr, AdaptiveEvents, BakeShadows } from '@react-three/drei'
import { EffectComposer, Bloom, SMAA } from '@react-three/postprocessing'

export const EducationalCanvas: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <Canvas
    camera={{ position: [0, 0, 5], fov: 50 }}
    dpr={[1, 2]}                    // Responsive DPR
    shadows                          // Enable shadows
    gl={{
      antialias: true,
      alpha: true,
      powerPreference: 'high-performance',
      stencil: false,
      depth: true,
    }}
    style={{ background: 'linear-gradient(180deg, #1a1a2e 0%, #16213e 100%)' }}
  >
    {/* Performance optimizations */}
    <AdaptiveDpr pixelated />
    <AdaptiveEvents />
    <BakeShadows />

    {/* Lighting setup */}
    <ambientLight intensity={0.4} />
    <directionalLight position={[10, 10, 5]} intensity={1} castShadow />
    <pointLight position={[-10, -10, -10]} intensity={0.3} color="#4FC3F7" />

    {/* Scene content */}
    <Suspense fallback={<LoadingSpinner3D />}>
      {children}
    </Suspense>

    {/* Post-processing for polish */}
    <EffectComposer>
      <Bloom luminanceThreshold={0.8} intensity={0.5} />
      <SMAA />
    </EffectComposer>

    <Preload all />
  </Canvas>
)
```

### 2. Realistic Glass/Crystal Materials (Lab Equipment)

```tsx
import { MeshPhysicalMaterial } from 'three'
import { useRef } from 'react'

// Glass material for beakers, test tubes, etc.
export const GlassObject: React.FC<{
  geometry: JSX.Element
  color?: string
  opacity?: number
}> = ({ geometry, color = '#ffffff', opacity = 0.3 }) => (
  <mesh>
    {geometry}
    <meshPhysicalMaterial
      color={color}
      transparent
      opacity={opacity}
      roughness={0}
      metalness={0}
      transmission={0.95}        // Glass transparency
      thickness={0.5}            // Refraction depth
      ior={1.5}                  // Index of refraction (glass = 1.5)
      envMapIntensity={1}
      clearcoat={1}
      clearcoatRoughness={0}
    />
  </mesh>
)

// Example: Erlenmeyer Flask
export const ErlenmeyerFlask: React.FC<{ liquidColor?: string; liquidLevel?: number }> = ({
  liquidColor = '#4CAF50',
  liquidLevel = 0.5
}) => (
  <group>
    {/* Glass body */}
    <mesh>
      <coneGeometry args={[0.4, 0.8, 32, 1, true]} />
      <meshPhysicalMaterial
        transparent opacity={0.3} transmission={0.9}
        roughness={0} thickness={0.3} ior={1.5}
      />
    </mesh>

    {/* Neck */}
    <mesh position={[0, 0.5, 0]}>
      <cylinderGeometry args={[0.12, 0.15, 0.3, 32, 1, true]} />
      <meshPhysicalMaterial transparent opacity={0.3} transmission={0.9} />
    </mesh>

    {/* Liquid with animated surface */}
    <AnimatedLiquid color={liquidColor} level={liquidLevel} />
  </group>
)
```

### 3. Animated Liquids with Shaders

```tsx
// Custom liquid shader for realistic fluid appearance
const liquidVertexShader = `
  varying vec2 vUv;
  varying vec3 vNormal;
  uniform float uTime;
  uniform float uWaveIntensity;

  void main() {
    vUv = uv;
    vNormal = normal;

    vec3 pos = position;
    // Wave animation on surface
    pos.y += sin(pos.x * 10.0 + uTime * 2.0) * uWaveIntensity;
    pos.y += cos(pos.z * 8.0 + uTime * 1.5) * uWaveIntensity;

    gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
  }
`

const liquidFragmentShader = `
  varying vec2 vUv;
  varying vec3 vNormal;
  uniform vec3 uColor;
  uniform float uOpacity;

  void main() {
    // Fresnel effect for edge glow
    float fresnel = pow(1.0 - dot(vNormal, vec3(0.0, 0.0, 1.0)), 2.0);
    vec3 finalColor = mix(uColor, vec3(1.0), fresnel * 0.3);

    gl_FragColor = vec4(finalColor, uOpacity);
  }
`

export const AnimatedLiquid: React.FC<{ color: string; level: number }> = ({ color, level }) => {
  const materialRef = useRef<THREE.ShaderMaterial>(null)

  useFrame(({ clock }) => {
    if (materialRef.current) {
      materialRef.current.uniforms.uTime.value = clock.elapsedTime
    }
  })

  return (
    <mesh position={[0, -0.2 + level * 0.4, 0]}>
      <cylinderGeometry args={[0.28, 0.35, level * 0.6, 32]} />
      <shaderMaterial
        ref={materialRef}
        vertexShader={liquidVertexShader}
        fragmentShader={liquidFragmentShader}
        transparent
        uniforms={{
          uTime: { value: 0 },
          uColor: { value: new THREE.Color(color) },
          uOpacity: { value: 0.7 },
          uWaveIntensity: { value: 0.01 },
        }}
      />
    </mesh>
  )
}
```

### 4. Interactive 3D Objects with Hover/Click States

```tsx
import { Float, Html } from '@react-three/drei'
import { useSpring, animated } from '@react-spring/three'

export const InteractiveObject: React.FC<{
  position: [number, number, number]
  color: string
  label: string
  description: string
  onClick?: () => void
}> = ({ position, color, label, description, onClick }) => {
  const [hovered, setHovered] = useState(false)
  const [active, setActive] = useState(false)

  // Smooth animations with react-spring
  const { scale, emissiveIntensity } = useSpring({
    scale: active ? 1.3 : hovered ? 1.1 : 1,
    emissiveIntensity: active ? 0.5 : hovered ? 0.3 : 0.1,
    config: { mass: 1, tension: 280, friction: 60 }
  })

  return (
    <Float speed={2} rotationIntensity={0.5} floatIntensity={0.5}>
      <animated.mesh
        position={position}
        scale={scale}
        onClick={(e) => {
          e.stopPropagation()
          setActive(!active)
          onClick?.()
        }}
        onPointerOver={(e) => {
          e.stopPropagation()
          setHovered(true)
          document.body.style.cursor = 'pointer'
        }}
        onPointerOut={() => {
          setHovered(false)
          document.body.style.cursor = 'auto'
        }}
      >
        <dodecahedronGeometry args={[0.5, 0]} />
        <animated.meshStandardMaterial
          color={color}
          emissive={color}
          emissiveIntensity={emissiveIntensity}
          metalness={0.3}
          roughness={0.4}
        />
      </animated.mesh>

      {/* HTML label overlay */}
      {(hovered || active) && (
        <Html position={[position[0], position[1] - 0.8, position[2]]} center>
          <div className="tooltip-3d">
            <strong>{label}</strong>
            {active && <p>{description}</p>}
          </div>
        </Html>
      )}
    </Float>
  )
}
```

### 5. Particle Systems for Visual Effects

```tsx
import { useMemo, useRef } from 'react'
import { Points, PointMaterial } from '@react-three/drei'
import * as THREE from 'three'

export const BackgroundParticles: React.FC<{
  count?: number
  color?: string
  size?: number
}> = ({ count = 500, color = '#4CAF50', size = 0.02 }) => {
  const ref = useRef<THREE.Points>(null)

  // Generate random positions
  const positions = useMemo(() => {
    const pos = new Float32Array(count * 3)
    for (let i = 0; i < count; i++) {
      pos[i * 3] = (Math.random() - 0.5) * 20      // x
      pos[i * 3 + 1] = (Math.random() - 0.5) * 20  // y
      pos[i * 3 + 2] = (Math.random() - 0.5) * 10  // z
    }
    return pos
  }, [count])

  useFrame((state, delta) => {
    if (ref.current) {
      ref.current.rotation.y += delta * 0.02
      ref.current.rotation.x += delta * 0.01
    }
  })

  return (
    <Points ref={ref} positions={positions} stride={3} frustumCulled={false}>
      <PointMaterial
        transparent
        color={color}
        size={size}
        sizeAttenuation
        depthWrite={false}
        opacity={0.6}
      />
    </Points>
  )
}
```

### 6. Connection Lines Between Objects

```tsx
import { Line, QuadraticBezierLine } from '@react-three/drei'

// Straight lines connecting nodes
export const ConnectionLines: React.FC<{
  points: [number, number, number][]
  closed?: boolean
}> = ({ points, closed = false }) => {
  const allPoints = closed ? [...points, points[0]] : points

  return (
    <Line
      points={allPoints}
      color="#666"
      lineWidth={2}
      transparent
      opacity={0.5}
      dashed
      dashScale={5}
      dashSize={0.5}
      gapSize={0.2}
    />
  )
}

// Curved bezier lines for flow diagrams
export const FlowLine: React.FC<{
  start: [number, number, number]
  end: [number, number, number]
  color?: string
}> = ({ start, end, color = '#4CAF50' }) => {
  const mid: [number, number, number] = [
    (start[0] + end[0]) / 2,
    (start[1] + end[1]) / 2 + 1,  // Curve upward
    (start[2] + end[2]) / 2,
  ]

  return (
    <QuadraticBezierLine
      start={start}
      mid={mid}
      end={end}
      color={color}
      lineWidth={3}
    />
  )
}
```

### 7. Camera Controls for Education

```tsx
import { OrbitControls, PresentationControls } from '@react-three/drei'

// For explorable 3D scenes (laboratory, solar system)
export const ExploratoryControls: React.FC = () => (
  <OrbitControls
    enablePan={false}           // Disable panning for focused learning
    enableZoom={true}
    minDistance={3}             // Prevent getting too close
    maxDistance={15}            // Prevent getting too far
    minPolarAngle={Math.PI / 6} // Limit vertical rotation
    maxPolarAngle={Math.PI / 2}
    autoRotate                  // Gentle auto-rotation
    autoRotateSpeed={0.5}
    dampingFactor={0.05}
    enableDamping
  />
)

// For showcasing single objects (instruments, molecules)
export const ShowcaseControls: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <PresentationControls
    global
    rotation={[0.1, 0.2, 0]}
    polar={[-Math.PI / 4, Math.PI / 4]}
    azimuth={[-Math.PI / 4, Math.PI / 4]}
    config={{ mass: 2, tension: 400 }}
    snap={{ mass: 4, tension: 400 }}
  >
    {children}
  </PresentationControls>
)
```

### 8. Measurement Tools with Real-time Feedback

```tsx
export const InteractiveScale: React.FC = () => {
  const [weight, setWeight] = useState(50)
  const needleRef = useRef<THREE.Group>(null)

  useFrame(() => {
    if (needleRef.current) {
      const targetAngle = (weight / 100) * Math.PI * 0.4 - Math.PI * 0.2
      needleRef.current.rotation.z = THREE.MathUtils.lerp(
        needleRef.current.rotation.z,
        targetAngle,
        0.05  // Smooth interpolation
      )
    }
  })

  return (
    <>
      {/* 3D Scale geometry */}
      <group>
        {/* Base, dial, needle... */}
        <group ref={needleRef} position={[0, 0.5, 0.12]}>
          <mesh>
            <boxGeometry args={[0.02, 0.3, 0.01]} />
            <meshBasicMaterial color="#f44336" />
          </mesh>
        </group>
      </group>

      {/* HTML overlay for precise reading */}
      <Html position={[0, -0.8, 0]} center>
        <div className="measurement-display">
          <span className="value">{weight}</span>
          <span className="unit">g</span>
        </div>
      </Html>
    </>
  )
}
```

### 9. Physics Simulations with Rapier

```tsx
import { Physics, RigidBody, CuboidCollider } from '@react-three/rapier'

export const FallingObjectsExperiment: React.FC = () => {
  const [objects, setObjects] = useState<Array<{ id: number; mass: number }>>([])

  const dropObject = (mass: number) => {
    setObjects(prev => [...prev, { id: Date.now(), mass }])
  }

  return (
    <Physics gravity={[0, -9.81, 0]}>
      {/* Ground plane */}
      <RigidBody type="fixed">
        <mesh position={[0, -2, 0]} rotation={[-Math.PI / 2, 0, 0]}>
          <planeGeometry args={[10, 10]} />
          <meshStandardMaterial color="#333" />
        </mesh>
        <CuboidCollider args={[5, 0.1, 5]} position={[0, -2, 0]} />
      </RigidBody>

      {/* Falling objects */}
      {objects.map(obj => (
        <RigidBody key={obj.id} mass={obj.mass} position={[0, 5, 0]}>
          <mesh>
            <sphereGeometry args={[0.2 + obj.mass * 0.01]} />
            <meshStandardMaterial color={obj.mass > 50 ? '#f44336' : '#4CAF50'} />
          </mesh>
        </RigidBody>
      ))}
    </Physics>
  )
}
```

---

## Complete Scene Example: Virtual Laboratory

```tsx
import React, { Suspense, useState } from 'react'
import { Canvas } from '@react-three/fiber'
import { OrbitControls, Environment, ContactShadows, Html } from '@react-three/drei'
import { EffectComposer, Bloom } from '@react-three/postprocessing'

export const Laboratory3D: React.FC = () => {
  const [selectedEquipment, setSelectedEquipment] = useState<string | null>(null)
  const [bunsenOn, setBunsenOn] = useState(false)

  return (
    <div className="lab-container">
      <Canvas shadows camera={{ position: [0, 2, 4], fov: 45 }}>
        {/* Environment */}
        <color attach="background" args={['#1a1a2e']} />
        <fog attach="fog" args={['#1a1a2e', 5, 20]} />
        <Environment preset="city" />

        {/* Lighting */}
        <ambientLight intensity={0.3} />
        <spotLight position={[5, 10, 5]} intensity={1} castShadow />

        <Suspense fallback={<LoadingIndicator />}>
          {/* Lab table */}
          <mesh position={[0, -0.5, 0]} receiveShadow>
            <boxGeometry args={[4, 0.1, 2]} />
            <meshStandardMaterial color="#2d2d2d" roughness={0.8} />
          </mesh>

          {/* Equipment */}
          <ErlenmeyerFlask
            position={[-1.2, 0, 0.3]}
            onClick={() => setSelectedEquipment('flask')}
          />
          <BunsenBurner
            position={[-0.5, -0.45, 0]}
            isOn={bunsenOn}
            onClick={() => setBunsenOn(!bunsenOn)}
          />
          <TestTubeRack position={[0.5, -0.35, 0.2]} />
          <Microscope position={[1.3, -0.45, 0]} />

          {/* Shadows */}
          <ContactShadows
            position={[0, -0.49, 0]}
            opacity={0.5}
            scale={10}
            blur={2}
          />
        </Suspense>

        {/* Post-processing */}
        <EffectComposer>
          <Bloom luminanceThreshold={0.9} intensity={0.3} />
        </EffectComposer>

        {/* Controls */}
        <OrbitControls
          enablePan={false}
          minDistance={2}
          maxDistance={8}
          maxPolarAngle={Math.PI / 2}
        />
      </Canvas>

      {/* Info panel */}
      {selectedEquipment && (
        <EquipmentInfoPanel equipment={selectedEquipment} />
      )}
    </div>
  )
}
```

---

## Performance Guidelines

| Technique | When to Use |
|-----------|-------------|
| `<Suspense>` | Always wrap 3D content |
| `<AdaptiveDpr>` | Mobile/low-end devices |
| `<BakeShadows>` | Static scenes |
| `frustumCulled` | Large particle systems |
| `instancedMesh` | Repeated objects (>100) |
| `useGLTF.preload()` | Heavy 3D models |
| Geometry reuse | Same shape multiple times |
| LOD (Level of Detail) | Complex scenes |

---

## Accessibility

1. **Keyboard navigation**: Implement focus states for 3D objects
2. **Screen readers**: Use `<Html>` overlays with ARIA labels
3. **Reduced motion**: Respect `prefers-reduced-motion`
4. **Color contrast**: Ensure labels are readable against 3D backgrounds
5. **Touch support**: Test on mobile devices

```tsx
// Respect reduced motion preference
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches

<Float
  speed={prefersReducedMotion ? 0 : 2}
  floatIntensity={prefersReducedMotion ? 0 : 0.5}
>
  {children}
</Float>
```

---

## Local AI Chatbot (LM Studio / R CLI)

Integrate a local AI tutor that runs entirely on the student's machine for privacy and offline use.

### Prerequisites

1. **LM Studio** (recommended): Download from [lmstudio.ai](https://lmstudio.ai)
2. **Or R CLI**: `pip install git+https://github.com/raym33/r.git`

### LM Studio Setup

```bash
# 1. Download and install LM Studio
# 2. Download a model (recommended: Mistral-7B-Instruct, Phi-3, or Llama-3-8B)
# 3. Start the local server (default: http://localhost:1234)
# 4. The API is OpenAI-compatible at /v1/chat/completions
```

### R CLI Setup (Alternative)

```yaml
# ~/.r-cli/config.yaml
llm:
  backend: lmstudio  # or "ollama"
  model: mistral-7b-instruct
  base_url: http://localhost:1234/v1
```

```bash
r serve --port 8765  # Start API server
```

### AIChatbot Component

```tsx
import { AIChatbot } from './components/interactive/AIChatbot'

// Floating chatbot (bottom-right corner)
<AIChatbot
  position="floating"
  title="Profesor de Física y Química"
  endpoint="http://localhost:1234/v1/chat/completions"
  modelName="Mistral 7B"
  systemPrompt={`Eres un profesor experto en Física y Química del currículo español.
    Explicas conceptos de forma clara y adaptada al nivel del estudiante.
    Usas ejemplos cotidianos y fórmulas cuando sea necesario.`}
  suggestedQuestions={[
    '¿Qué es la densidad?',
    '¿Cómo funciona el método científico?',
    'Explícame la notación científica',
  ]}
/>

// Inline chatbot (within page content)
<AIChatbot
  position="inline"
  title="Pregunta al profesor"
  defaultCollapsed={false}
/>
```

### Component Props

| Prop | Type | Default | Description |
|------|------|---------|-------------|
| `position` | `'floating' \| 'inline'` | `'floating'` | Floating button or inline in page |
| `endpoint` | `string` | `http://localhost:1234/v1/chat/completions` | LM Studio API URL |
| `systemPrompt` | `string` | Physics/Chemistry tutor | AI persona and instructions |
| `title` | `string` | `'Profesor IA'` | Header title |
| `modelName` | `string` | `'Local LLM'` | Shown in header |
| `suggestedQuestions` | `string[]` | Subject-specific | Quick question buttons |
| `defaultCollapsed` | `boolean` | `true` | Initial state (floating only) |
| `placeholder` | `string` | `'Escribe tu pregunta...'` | Input placeholder |

### Where to Place the Chatbot

**Option 1: Global floating button** (recommended)
```tsx
// In LibroApp.tsx - available on all pages
export const LibroApp: React.FC = () => (
  <div className="libro-app">
    {renderTema()}
    <AIChatbot position="floating" />
  </div>
)
```

**Option 2: Per-page inline**
```tsx
// At the end of each topic page
<BookPage pageNumber={35} chapter="Resumen">
  <Heading level={2}>¿Tienes dudas?</Heading>
  <AIChatbot position="inline" defaultCollapsed={false} />
</BookPage>
```

**Option 3: In sidebar/settings**
```tsx
// In SettingsPanel.tsx
<EducationalBox type="note" title="Asistente IA">
  <p>Activa el asistente local para resolver dudas.</p>
  <button onClick={() => setShowChatbot(true)}>Abrir chat</button>
</EducationalBox>
```

### Customizing for Different Subjects

```tsx
// Mathematics tutor
<AIChatbot
  systemPrompt={`Eres un profesor de Matemáticas. Cuando resuelvas problemas:
    1. Explica el planteamiento paso a paso
    2. Muestra todas las operaciones
    3. Verifica el resultado
    4. Sugiere ejercicios similares para practicar`}
  suggestedQuestions={[
    '¿Cómo resuelvo ecuaciones de segundo grado?',
    'Explícame las fracciones',
    '¿Qué es el teorema de Pitágoras?',
  ]}
/>

// Biology tutor
<AIChatbot
  systemPrompt={`Eres un profesor de Biología experto en el currículo español.
    Explicas procesos biológicos con claridad, usando diagramas cuando sea útil.`}
  suggestedQuestions={[
    '¿Cómo funciona la fotosíntesis?',
    'Explícame la mitosis',
    '¿Qué son los ecosistemas?',
  ]}
/>
```

### Recommended Local Models

| Model | Size | RAM Needed | Quality | Speed |
|-------|------|------------|---------|-------|
| Phi-3-mini | 3.8B | 8GB | Good | Fast |
| Mistral-7B-Instruct | 7B | 16GB | Excellent | Medium |
| Llama-3-8B-Instruct | 8B | 16GB | Excellent | Medium |
| Mixtral-8x7B | 47B | 32GB | Best | Slow |

### Connection Status Handling

The component automatically:
- Shows connection status (green/red indicator)
- Retries on connection failure
- Displays helpful error messages with troubleshooting

```tsx
// The chatbot checks /v1/models endpoint on mount
// If LM Studio isn't running, it shows:
"⚠️ No se pudo conectar con el modelo local.
Asegúrate de que LM Studio esté ejecutándose en http://localhost:1234"
```

---

## Integration with Educational Components

```tsx
// Combine 3D visualization with 2D educational UI
const FullLessonPage: React.FC = () => (
  <BookPage pageNumber={5} chapter="The Scientific Method">
    <Heading level={2}>Interactive Cycle</Heading>

    <Paragraph lead>
      Explore the scientific method by clicking each step in the 3D diagram.
    </Paragraph>

    {/* 3D Visualization */}
    <Suspense fallback={<Loading3D />}>
      <ScientificMethod3D onStepSelect={handleStepSelect} />
    </Suspense>

    {/* Educational content */}
    <EducationalBox type="definition">
      <p>The <Term>scientific method</Term> is a systematic approach...</p>
    </EducationalBox>

    {/* Assessment */}
    <Quiz questions={methodQuiz} exerciseId="method-quiz" />

    {/* AI Tutor for questions */}
    <AIChatbot position="floating" />
  </BookPage>
)
```
