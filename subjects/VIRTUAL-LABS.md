# Laboratorios Virtuales Interactivos

Simulaciones de experimentos científicos donde el estudiante puede manipular variables, observar resultados y aprender mediante la experimentación activa.

---

## Arquitectura de Laboratorios

```
┌─────────────────────────────────────────────────────────────┐
│                    LABORATORIO VIRTUAL                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌──────────────┐     │
│  │   ESCENA 3D   │   │  CONTROLES   │   │   DATOS      │     │
│  │  Three.js +   │   │  Sliders,    │   │  Gráficas,   │     │
│  │    Rapier     │◄──│  Botones     │──►│  Tablas      │     │
│  └──────────────┘   └──────────────┘   └──────────────┘     │
│         │                  │                  │              │
│         └──────────────────┼──────────────────┘              │
│                            ▼                                 │
│                   ┌──────────────┐                          │
│                   │   GUÍA IA    │                          │
│                   │  Sugerencias │                          │
│                   │  y feedback  │                          │
│                   └──────────────┘                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Store de Laboratorio (Zustand)

```tsx
import { create } from 'zustand'

interface LabVariable {
  id: string
  name: string
  symbol: string
  unit: string
  value: number
  min: number
  max: number
  step: number
  isControlled: boolean // El estudiante puede modificarla
  isDependent: boolean  // Se calcula a partir de otras
}

interface DataPoint {
  timestamp: number
  variables: Record<string, number>
}

interface LabExperiment {
  id: string
  name: string
  objective: string
  hypothesis: string | null
  startTime: Date | null
  endTime: Date | null
  dataPoints: DataPoint[]
  notes: string[]
  completed: boolean
  score: number | null
}

interface VirtualLabState {
  // Configuración del laboratorio
  labId: string
  labName: string
  variables: LabVariable[]

  // Estado del experimento
  isRunning: boolean
  isPaused: boolean
  simulationSpeed: number
  elapsedTime: number

  // Datos recolectados
  currentExperiment: LabExperiment | null
  experimentHistory: LabExperiment[]

  // Guía del profesor IA
  aiHints: string[]
  showAIAssistant: boolean

  // Acciones
  setVariable: (id: string, value: number) => void
  startExperiment: (hypothesis?: string) => void
  pauseExperiment: () => void
  resumeExperiment: () => void
  stopExperiment: () => void
  recordDataPoint: () => void
  addNote: (note: string) => void
  setSimulationSpeed: (speed: number) => void
  requestAIHint: () => void
  resetLab: () => void
}

export const useVirtualLabStore = create<VirtualLabState>((set, get) => ({
  labId: '',
  labName: '',
  variables: [],
  isRunning: false,
  isPaused: false,
  simulationSpeed: 1,
  elapsedTime: 0,
  currentExperiment: null,
  experimentHistory: [],
  aiHints: [],
  showAIAssistant: true,

  setVariable: (id, value) => {
    set((state) => ({
      variables: state.variables.map((v) =>
        v.id === id && v.isControlled ? { ...v, value } : v
      ),
    }))
  },

  startExperiment: (hypothesis) => {
    set((state) => ({
      isRunning: true,
      isPaused: false,
      elapsedTime: 0,
      currentExperiment: {
        id: `exp-${Date.now()}`,
        name: `Experimento ${state.experimentHistory.length + 1}`,
        objective: state.labName,
        hypothesis: hypothesis || null,
        startTime: new Date(),
        endTime: null,
        dataPoints: [],
        notes: [],
        completed: false,
        score: null,
      },
    }))
  },

  pauseExperiment: () => set({ isPaused: true }),

  resumeExperiment: () => set({ isPaused: false }),

  stopExperiment: () => {
    const { currentExperiment, experimentHistory } = get()
    if (currentExperiment) {
      const completed: LabExperiment = {
        ...currentExperiment,
        endTime: new Date(),
        completed: true,
      }
      set({
        isRunning: false,
        isPaused: false,
        currentExperiment: null,
        experimentHistory: [...experimentHistory, completed],
      })
    }
  },

  recordDataPoint: () => {
    const { variables, currentExperiment, elapsedTime } = get()
    if (!currentExperiment) return

    const point: DataPoint = {
      timestamp: elapsedTime,
      variables: Object.fromEntries(variables.map((v) => [v.id, v.value])),
    }

    set({
      currentExperiment: {
        ...currentExperiment,
        dataPoints: [...currentExperiment.dataPoints, point],
      },
    })
  },

  addNote: (note) => {
    const { currentExperiment } = get()
    if (!currentExperiment) return

    set({
      currentExperiment: {
        ...currentExperiment,
        notes: [...currentExperiment.notes, `[${get().elapsedTime.toFixed(1)}s] ${note}`],
      },
    })
  },

  setSimulationSpeed: (speed) => set({ simulationSpeed: speed }),

  requestAIHint: () => {
    // Se implementa con la integración del chatbot
  },

  resetLab: () => {
    set((state) => ({
      isRunning: false,
      isPaused: false,
      elapsedTime: 0,
      currentExperiment: null,
      variables: state.variables.map((v) => ({
        ...v,
        value: (v.max + v.min) / 2,
      })),
    }))
  },
}))
```

---

## Laboratorio 1: Densidad de Líquidos

```tsx
import React, { useRef, useState, useEffect, useMemo } from 'react'
import { Canvas, useFrame } from '@react-three/fiber'
import { Physics, RigidBody, CuboidCollider } from '@react-three/rapier'
import { OrbitControls, Html, Environment } from '@react-three/drei'
import * as THREE from 'three'
import { useVirtualLabStore } from '../store/virtualLabStore'

// Materiales de líquidos con diferentes densidades
const LIQUIDS = [
  { id: 'honey', name: 'Miel', density: 1.42, color: '#DAA520', viscosity: 0.9 },
  { id: 'water', name: 'Agua', density: 1.0, color: '#4FC3F7', viscosity: 0.1 },
  { id: 'oil', name: 'Aceite', density: 0.92, color: '#FFD54F', viscosity: 0.3 },
  { id: 'alcohol', name: 'Alcohol', density: 0.79, color: '#E1BEE7', viscosity: 0.15 },
]

// Objetos para soltar
const OBJECTS = [
  { id: 'cork', name: 'Corcho', density: 0.24, color: '#8D6E63', size: 0.3 },
  { id: 'wood', name: 'Madera', density: 0.6, color: '#A1887F', size: 0.35 },
  { id: 'plastic', name: 'Plástico', density: 0.95, color: '#FF7043', size: 0.3 },
  { id: 'rubber', name: 'Goma', density: 1.1, color: '#424242', size: 0.28 },
  { id: 'metal', name: 'Metal', density: 7.8, color: '#78909C', size: 0.25 },
]

// Recipiente de vidrio
const GlassContainer: React.FC<{ height: number }> = ({ height }) => {
  return (
    <group>
      {/* Paredes del recipiente */}
      <mesh position={[0, height / 2, 0]}>
        <cylinderGeometry args={[1.2, 1.2, height, 32, 1, true]} />
        <meshPhysicalMaterial
          color="#ffffff"
          transparent
          opacity={0.15}
          roughness={0}
          transmission={0.95}
          thickness={0.1}
          ior={1.5}
          side={THREE.DoubleSide}
        />
      </mesh>
      {/* Base */}
      <mesh position={[0, 0.05, 0]} rotation={[-Math.PI / 2, 0, 0]}>
        <circleGeometry args={[1.2, 32]} />
        <meshPhysicalMaterial
          color="#ffffff"
          transparent
          opacity={0.2}
          roughness={0}
        />
      </mesh>
    </group>
  )
}

// Capa de líquido
const LiquidLayer: React.FC<{
  liquid: typeof LIQUIDS[0]
  yPosition: number
  height: number
}> = ({ liquid, yPosition, height }) => {
  const meshRef = useRef<THREE.Mesh>(null)

  // Animación suave de ondas
  useFrame((state) => {
    if (meshRef.current) {
      const positions = meshRef.current.geometry.attributes.position
      const time = state.clock.elapsedTime

      for (let i = 0; i < positions.count; i++) {
        const x = positions.getX(i)
        const z = positions.getZ(i)
        const distance = Math.sqrt(x * x + z * z)

        // Solo animar la superficie superior
        if (positions.getY(i) > height * 0.4) {
          const wave = Math.sin(distance * 3 - time * 2) * 0.02 * (1 - liquid.viscosity)
          positions.setY(i, height / 2 + wave)
        }
      }
      positions.needsUpdate = true
    }
  })

  return (
    <mesh ref={meshRef} position={[0, yPosition + height / 2, 0]}>
      <cylinderGeometry args={[1.15, 1.15, height, 32, 8]} />
      <meshPhysicalMaterial
        color={liquid.color}
        transparent
        opacity={0.7}
        roughness={0.1}
        transmission={0.3}
      />
    </mesh>
  )
}

// Objeto flotante/hundible con física
const FloatingObject: React.FC<{
  object: typeof OBJECTS[0]
  liquidLayers: { liquid: typeof LIQUIDS[0]; yStart: number; yEnd: number }[]
  position: [number, number, number]
}> = ({ object, liquidLayers, position }) => {
  const rigidBodyRef = useRef<any>(null)
  const [currentY, setCurrentY] = useState(position[1])
  const [isHovered, setIsHovered] = useState(false)

  useFrame(() => {
    if (!rigidBodyRef.current) return

    const pos = rigidBodyRef.current.translation()
    setCurrentY(pos.y)

    // Calcular fuerza de flotación según la densidad del líquido
    let buoyancyForce = 0
    let inLiquid = false

    for (const layer of liquidLayers) {
      if (pos.y >= layer.yStart && pos.y <= layer.yEnd) {
        inLiquid = true
        // Fuerza = (densidad_liquido - densidad_objeto) * g * volumen
        const submersion = Math.min(1, (layer.yEnd - pos.y) / object.size)
        const volume = Math.pow(object.size, 3)
        buoyancyForce = (layer.liquid.density - object.density) * 9.81 * volume * submersion

        // Arrastre viscoso
        const velocity = rigidBodyRef.current.linvel()
        const drag = -velocity.y * layer.liquid.viscosity * 5
        buoyancyForce += drag
      }
    }

    if (inLiquid) {
      rigidBodyRef.current.applyImpulse({ x: 0, y: buoyancyForce * 0.01, z: 0 }, true)
    }
  })

  // Determinar estado de flotación
  const getFloatStatus = () => {
    for (const layer of liquidLayers) {
      if (currentY >= layer.yStart && currentY <= layer.yEnd) {
        if (object.density < layer.liquid.density) return 'Flotando'
        if (object.density > layer.liquid.density) return 'Hundiéndose'
        return 'Equilibrio'
      }
    }
    return 'Cayendo'
  }

  return (
    <RigidBody
      ref={rigidBodyRef}
      position={position}
      colliders="cuboid"
      linearDamping={0.5}
      angularDamping={0.5}
    >
      <mesh
        onPointerOver={() => setIsHovered(true)}
        onPointerOut={() => setIsHovered(false)}
      >
        <boxGeometry args={[object.size, object.size, object.size]} />
        <meshStandardMaterial
          color={object.color}
          metalness={object.id === 'metal' ? 0.8 : 0.1}
          roughness={object.id === 'metal' ? 0.2 : 0.6}
        />
      </mesh>

      {isHovered && (
        <Html position={[0, object.size, 0]} center>
          <div className="object-tooltip">
            <strong>{object.name}</strong>
            <div>ρ = {object.density} g/cm³</div>
            <div className={`status ${getFloatStatus().toLowerCase()}`}>
              {getFloatStatus()}
            </div>
          </div>
        </Html>
      )}
    </RigidBody>
  )
}

// Panel de control del laboratorio
const LabControls: React.FC<{
  selectedLiquids: string[]
  onToggleLiquid: (id: string) => void
  onDropObject: (id: string) => void
  droppedObjects: string[]
}> = ({ selectedLiquids, onToggleLiquid, onDropObject, droppedObjects }) => {
  const { isRunning, startExperiment, stopExperiment, recordDataPoint } = useVirtualLabStore()

  return (
    <div className="lab-controls">
      <div className="control-section">
        <h4>🧪 Líquidos</h4>
        <p className="hint">Selecciona líquidos para crear capas (se ordenan por densidad)</p>
        <div className="liquid-buttons">
          {LIQUIDS.map((liquid) => (
            <button
              key={liquid.id}
              className={`liquid-btn ${selectedLiquids.includes(liquid.id) ? 'selected' : ''}`}
              onClick={() => onToggleLiquid(liquid.id)}
              style={{
                backgroundColor: selectedLiquids.includes(liquid.id)
                  ? liquid.color
                  : 'transparent',
                borderColor: liquid.color,
              }}
            >
              <span className="liquid-name">{liquid.name}</span>
              <span className="liquid-density">ρ = {liquid.density}</span>
            </button>
          ))}
        </div>
      </div>

      <div className="control-section">
        <h4>📦 Objetos</h4>
        <p className="hint">Suelta objetos para ver si flotan o se hunden</p>
        <div className="object-buttons">
          {OBJECTS.map((obj) => (
            <button
              key={obj.id}
              className={`object-btn ${droppedObjects.includes(obj.id) ? 'dropped' : ''}`}
              onClick={() => onDropObject(obj.id)}
              disabled={droppedObjects.includes(obj.id)}
              style={{ backgroundColor: obj.color }}
            >
              <span className="object-name">{obj.name}</span>
              <span className="object-density">ρ = {obj.density}</span>
            </button>
          ))}
        </div>
      </div>

      <div className="control-section">
        <h4>📊 Experimento</h4>
        <div className="experiment-controls">
          {!isRunning ? (
            <button className="start-btn" onClick={() => startExperiment()}>
              ▶ Iniciar experimento
            </button>
          ) : (
            <>
              <button className="record-btn" onClick={recordDataPoint}>
                📝 Registrar datos
              </button>
              <button className="stop-btn" onClick={stopExperiment}>
                ⏹ Finalizar
              </button>
            </>
          )}
        </div>
      </div>
    </div>
  )
}

// Componente principal del laboratorio de densidad
export const DensityLab: React.FC = () => {
  const [selectedLiquids, setSelectedLiquids] = useState<string[]>(['water'])
  const [droppedObjects, setDroppedObjects] = useState<string[]>([])
  const [objectPositions, setObjectPositions] = useState<
    { id: string; position: [number, number, number] }[]
  >([])

  // Calcular capas de líquidos ordenadas por densidad
  const liquidLayers = useMemo(() => {
    const selected = LIQUIDS.filter((l) => selectedLiquids.includes(l.id))
      .sort((a, b) => b.density - a.density) // Más denso abajo

    const layerHeight = 3 / Math.max(selected.length, 1)
    let currentY = 0

    return selected.map((liquid) => {
      const layer = {
        liquid,
        yStart: currentY,
        yEnd: currentY + layerHeight,
      }
      currentY += layerHeight
      return layer
    })
  }, [selectedLiquids])

  const toggleLiquid = (id: string) => {
    setSelectedLiquids((prev) =>
      prev.includes(id) ? prev.filter((l) => l !== id) : [...prev, id]
    )
  }

  const dropObject = (id: string) => {
    if (droppedObjects.includes(id)) return

    setDroppedObjects((prev) => [...prev, id])
    setObjectPositions((prev) => [
      ...prev,
      {
        id,
        position: [
          (Math.random() - 0.5) * 1.5,
          5, // Altura inicial
          (Math.random() - 0.5) * 1.5,
        ],
      },
    ])
  }

  const resetLab = () => {
    setDroppedObjects([])
    setObjectPositions([])
  }

  return (
    <div className="virtual-lab density-lab">
      <div className="lab-header">
        <h2>🧪 Laboratorio de Densidad</h2>
        <p>
          Experimenta con diferentes líquidos y objetos para entender
          por qué algunas cosas flotan y otras se hunden.
        </p>
      </div>

      <div className="lab-content">
        {/* Escena 3D */}
        <div className="lab-canvas">
          <Canvas camera={{ position: [4, 3, 4], fov: 50 }}>
            <ambientLight intensity={0.5} />
            <pointLight position={[10, 10, 10]} intensity={1} />
            <spotLight position={[0, 10, 0]} intensity={0.5} />

            <Physics gravity={[0, -9.81, 0]}>
              {/* Recipiente */}
              <GlassContainer height={4} />

              {/* Suelo invisible */}
              <RigidBody type="fixed" position={[0, 0, 0]}>
                <CuboidCollider args={[1.5, 0.1, 1.5]} />
              </RigidBody>

              {/* Capas de líquidos */}
              {liquidLayers.map((layer, i) => (
                <LiquidLayer
                  key={layer.liquid.id}
                  liquid={layer.liquid}
                  yPosition={layer.yStart}
                  height={layer.yEnd - layer.yStart}
                />
              ))}

              {/* Objetos */}
              {objectPositions.map(({ id, position }) => {
                const obj = OBJECTS.find((o) => o.id === id)!
                return (
                  <FloatingObject
                    key={id}
                    object={obj}
                    liquidLayers={liquidLayers}
                    position={position}
                  />
                )
              })}
            </Physics>

            <OrbitControls
              enablePan={false}
              minDistance={3}
              maxDistance={10}
              minPolarAngle={Math.PI / 6}
              maxPolarAngle={Math.PI / 2.2}
            />
            <Environment preset="studio" />
          </Canvas>

          {/* Leyenda de densidades */}
          <div className="density-legend">
            <h5>Escala de densidad (g/cm³)</h5>
            <div className="legend-items">
              {liquidLayers.map((layer) => (
                <div
                  key={layer.liquid.id}
                  className="legend-item"
                  style={{ backgroundColor: layer.liquid.color }}
                >
                  {layer.liquid.name}: {layer.liquid.density}
                </div>
              ))}
            </div>
          </div>
        </div>

        {/* Controles */}
        <LabControls
          selectedLiquids={selectedLiquids}
          onToggleLiquid={toggleLiquid}
          onDropObject={dropObject}
          droppedObjects={droppedObjects}
        />
      </div>

      {/* Panel de datos */}
      <DataPanel
        liquidLayers={liquidLayers}
        droppedObjects={droppedObjects.map((id) => OBJECTS.find((o) => o.id === id)!)}
      />

      {/* Botón de reinicio */}
      <button className="reset-btn" onClick={resetLab}>
        🔄 Reiniciar laboratorio
      </button>

      {/* Guía del experimento */}
      <ExperimentGuide concept="densidad" />
    </div>
  )
}

// Panel de datos recolectados
const DataPanel: React.FC<{
  liquidLayers: { liquid: typeof LIQUIDS[0]; yStart: number; yEnd: number }[]
  droppedObjects: typeof OBJECTS
}> = ({ liquidLayers, droppedObjects }) => {
  return (
    <div className="data-panel">
      <h4>📊 Datos del experimento</h4>

      <table className="data-table">
        <thead>
          <tr>
            <th>Objeto</th>
            <th>Densidad (g/cm³)</th>
            <th>Predicción</th>
            <th>Resultado</th>
          </tr>
        </thead>
        <tbody>
          {droppedObjects.map((obj) => {
            // Determinar en qué capa quedará
            let prediction = 'Se hunde'
            for (const layer of liquidLayers) {
              if (obj.density < layer.liquid.density) {
                prediction = `Flota sobre ${layer.liquid.name}`
                break
              }
            }

            return (
              <tr key={obj.id}>
                <td>
                  <span
                    className="color-dot"
                    style={{ backgroundColor: obj.color }}
                  />
                  {obj.name}
                </td>
                <td>{obj.density}</td>
                <td>{prediction}</td>
                <td>
                  <input type="text" placeholder="Observa y anota..." />
                </td>
              </tr>
            )
          })}
        </tbody>
      </table>

      {droppedObjects.length === 0 && (
        <p className="empty-message">
          Suelta objetos para ver los datos aquí
        </p>
      )}
    </div>
  )
}

// Guía del experimento con asistente IA
const ExperimentGuide: React.FC<{ concept: string }> = ({ concept }) => {
  const [showGuide, setShowGuide] = useState(true)
  const [currentStep, setCurrentStep] = useState(0)

  const steps = [
    {
      title: 'Objetivo',
      content: 'Entender por qué algunos objetos flotan y otros se hunden en diferentes líquidos.',
      action: 'Lee el objetivo y pulsa "Siguiente"',
    },
    {
      title: 'Hipótesis',
      content: '¿Qué crees que determina si un objeto flota o se hunde?',
      action: 'Escribe tu hipótesis antes de experimentar',
      input: true,
    },
    {
      title: 'Experimentación',
      content: 'Selecciona diferentes líquidos y suelta objetos para observar qué ocurre.',
      action: 'Prueba combinaciones y observa los resultados',
    },
    {
      title: 'Análisis',
      content: 'Compara las densidades de los objetos con las de los líquidos. ¿Ves algún patrón?',
      action: 'Anota tus observaciones en la tabla',
    },
    {
      title: 'Conclusión',
      content: 'Un objeto flota cuando su densidad es MENOR que la del líquido.',
      action: '¿Tu hipótesis era correcta?',
      formula: '\\text{Si } \\rho_{objeto} < \\rho_{líquido} \\Rightarrow \\text{FLOTA}',
    },
  ]

  if (!showGuide) {
    return (
      <button className="show-guide-btn" onClick={() => setShowGuide(true)}>
        📖 Mostrar guía
      </button>
    )
  }

  return (
    <div className="experiment-guide">
      <div className="guide-header">
        <h4>📖 Guía del experimento</h4>
        <button onClick={() => setShowGuide(false)}>✕</button>
      </div>

      <div className="guide-steps">
        {steps.map((step, index) => (
          <div
            key={index}
            className={`guide-step ${
              index === currentStep
                ? 'current'
                : index < currentStep
                ? 'completed'
                : ''
            }`}
          >
            <div className="step-number">{index + 1}</div>
            <div className="step-content">
              <h5>{step.title}</h5>
              {index === currentStep && (
                <>
                  <p>{step.content}</p>
                  {step.input && (
                    <textarea
                      placeholder="Escribe tu hipótesis aquí..."
                      rows={2}
                    />
                  )}
                  {step.formula && (
                    <div className="formula-box">
                      <BlockMath math={step.formula} />
                    </div>
                  )}
                  <p className="action-hint">👉 {step.action}</p>
                </>
              )}
            </div>
          </div>
        ))}
      </div>

      <div className="guide-navigation">
        <button
          disabled={currentStep === 0}
          onClick={() => setCurrentStep((s) => s - 1)}
        >
          ← Anterior
        </button>
        <button
          disabled={currentStep === steps.length - 1}
          onClick={() => setCurrentStep((s) => s + 1)}
        >
          Siguiente →
        </button>
      </div>
    </div>
  )
}
```

---

## Laboratorio 2: Movimiento y Velocidad

```tsx
import React, { useRef, useState, useEffect } from 'react'
import { Canvas, useFrame } from '@react-three/fiber'
import { Physics, RigidBody } from '@react-three/rapier'
import { Line, Html } from '@react-three/drei'
import * as THREE from 'three'

interface MotionLabProps {
  onDataUpdate?: (data: MotionData) => void
}

interface MotionData {
  time: number
  position: number
  velocity: number
  acceleration: number
}

// Carrito móvil con física
const Cart: React.FC<{
  initialVelocity: number
  acceleration: number
  friction: number
  onUpdate: (data: MotionData) => void
}> = ({ initialVelocity, acceleration, friction, onUpdate }) => {
  const rigidBodyRef = useRef<any>(null)
  const startTime = useRef(Date.now())
  const [trail, setTrail] = useState<THREE.Vector3[]>([])

  useFrame(() => {
    if (!rigidBodyRef.current) return

    const pos = rigidBodyRef.current.translation()
    const vel = rigidBodyRef.current.linvel()
    const elapsed = (Date.now() - startTime.current) / 1000

    // Aplicar aceleración constante
    rigidBodyRef.current.applyImpulse(
      { x: acceleration * 0.01, y: 0, z: 0 },
      true
    )

    // Aplicar fricción
    rigidBodyRef.current.applyImpulse(
      { x: -vel.x * friction * 0.01, y: 0, z: 0 },
      true
    )

    // Actualizar rastro
    setTrail((prev) => {
      const newTrail = [...prev, new THREE.Vector3(pos.x, 0.1, pos.z)]
      return newTrail.slice(-50) // Mantener últimos 50 puntos
    })

    // Enviar datos
    onUpdate({
      time: elapsed,
      position: pos.x,
      velocity: vel.x,
      acceleration: acceleration - vel.x * friction,
    })
  })

  return (
    <>
      <RigidBody
        ref={rigidBodyRef}
        position={[-5, 0.25, 0]}
        linearVelocity={[initialVelocity, 0, 0]}
        linearDamping={0}
      >
        {/* Cuerpo del carrito */}
        <mesh>
          <boxGeometry args={[0.8, 0.3, 0.5]} />
          <meshStandardMaterial color="#2196F3" />
        </mesh>
        {/* Ruedas */}
        {[[-0.3, -0.15, 0.3], [-0.3, -0.15, -0.3], [0.3, -0.15, 0.3], [0.3, -0.15, -0.3]].map(
          (pos, i) => (
            <mesh key={i} position={pos as [number, number, number]}>
              <cylinderGeometry args={[0.1, 0.1, 0.08, 16]} />
              <meshStandardMaterial color="#333" />
            </mesh>
          )
        )}
      </RigidBody>

      {/* Rastro del movimiento */}
      {trail.length > 1 && (
        <Line
          points={trail}
          color="#FF9800"
          lineWidth={3}
          transparent
          opacity={0.7}
        />
      )}
    </>
  )
}

// Pista con marcadores de distancia
const Track: React.FC = () => {
  const markers = Array.from({ length: 21 }, (_, i) => i - 10)

  return (
    <group>
      {/* Superficie de la pista */}
      <mesh position={[0, 0, 0]} rotation={[-Math.PI / 2, 0, 0]}>
        <planeGeometry args={[25, 2]} />
        <meshStandardMaterial color="#424242" />
      </mesh>

      {/* Marcadores de distancia */}
      {markers.map((m) => (
        <group key={m} position={[m, 0.01, 0]}>
          <mesh rotation={[-Math.PI / 2, 0, 0]}>
            <planeGeometry args={[0.05, 1.5]} />
            <meshBasicMaterial color={m === 0 ? '#4CAF50' : '#fff'} />
          </mesh>
          <Html position={[0, 0, 1]} center>
            <span className="track-marker">{m}m</span>
          </Html>
        </group>
      ))}
    </group>
  )
}

// Componente principal del laboratorio de movimiento
export const MotionLab: React.FC<MotionLabProps> = ({ onDataUpdate }) => {
  const [initialVelocity, setInitialVelocity] = useState(2)
  const [acceleration, setAcceleration] = useState(0)
  const [friction, setFriction] = useState(0.1)
  const [isRunning, setIsRunning] = useState(false)
  const [dataHistory, setDataHistory] = useState<MotionData[]>([])
  const [currentData, setCurrentData] = useState<MotionData>({
    time: 0,
    position: -5,
    velocity: 0,
    acceleration: 0,
  })

  const handleDataUpdate = (data: MotionData) => {
    setCurrentData(data)
    setDataHistory((prev) => [...prev, data])
    onDataUpdate?.(data)
  }

  const startExperiment = () => {
    setDataHistory([])
    setIsRunning(true)
  }

  const stopExperiment = () => {
    setIsRunning(false)
  }

  return (
    <div className="virtual-lab motion-lab">
      <div className="lab-header">
        <h2>🚗 Laboratorio de Movimiento</h2>
        <p>
          Experimenta con velocidad inicial, aceleración y fricción para
          entender las leyes del movimiento.
        </p>
      </div>

      <div className="lab-content">
        {/* Escena 3D */}
        <div className="lab-canvas">
          <Canvas camera={{ position: [0, 5, 8], fov: 60 }}>
            <ambientLight intensity={0.6} />
            <directionalLight position={[5, 10, 5]} intensity={0.8} />

            <Physics gravity={[0, -9.81, 0]}>
              <Track />

              {isRunning && (
                <Cart
                  initialVelocity={initialVelocity}
                  acceleration={acceleration}
                  friction={friction}
                  onUpdate={handleDataUpdate}
                />
              )}

              {/* Suelo invisible */}
              <RigidBody type="fixed" position={[0, -0.1, 0]}>
                <mesh>
                  <boxGeometry args={[30, 0.2, 5]} />
                  <meshStandardMaterial visible={false} />
                </mesh>
              </RigidBody>
            </Physics>
          </Canvas>

          {/* Datos en tiempo real */}
          <div className="realtime-data">
            <div className="data-item">
              <span className="label">⏱️ Tiempo</span>
              <span className="value">{currentData.time.toFixed(2)} s</span>
            </div>
            <div className="data-item">
              <span className="label">📍 Posición</span>
              <span className="value">{currentData.position.toFixed(2)} m</span>
            </div>
            <div className="data-item">
              <span className="label">🏃 Velocidad</span>
              <span className="value">{currentData.velocity.toFixed(2)} m/s</span>
            </div>
            <div className="data-item">
              <span className="label">🚀 Aceleración</span>
              <span className="value">{currentData.acceleration.toFixed(2)} m/s²</span>
            </div>
          </div>
        </div>

        {/* Controles */}
        <div className="lab-controls">
          <div className="control-group">
            <label>
              Velocidad inicial: {initialVelocity.toFixed(1)} m/s
            </label>
            <input
              type="range"
              min="0"
              max="10"
              step="0.5"
              value={initialVelocity}
              onChange={(e) => setInitialVelocity(parseFloat(e.target.value))}
              disabled={isRunning}
            />
          </div>

          <div className="control-group">
            <label>
              Aceleración: {acceleration.toFixed(1)} m/s²
            </label>
            <input
              type="range"
              min="-5"
              max="5"
              step="0.5"
              value={acceleration}
              onChange={(e) => setAcceleration(parseFloat(e.target.value))}
              disabled={isRunning}
            />
          </div>

          <div className="control-group">
            <label>
              Fricción: {friction.toFixed(2)}
            </label>
            <input
              type="range"
              min="0"
              max="1"
              step="0.05"
              value={friction}
              onChange={(e) => setFriction(parseFloat(e.target.value))}
              disabled={isRunning}
            />
          </div>

          <div className="control-buttons">
            {!isRunning ? (
              <button className="start-btn" onClick={startExperiment}>
                ▶️ Iniciar
              </button>
            ) : (
              <button className="stop-btn" onClick={stopExperiment}>
                ⏹️ Detener
              </button>
            )}
          </div>
        </div>
      </div>

      {/* Gráficas */}
      <MotionGraphs data={dataHistory} />

      {/* Fórmulas de referencia */}
      <div className="formulas-reference">
        <h4>📐 Fórmulas del movimiento</h4>
        <div className="formula-grid">
          <div className="formula-card">
            <BlockMath math="v = v_0 + a \cdot t" />
            <span>Velocidad</span>
          </div>
          <div className="formula-card">
            <BlockMath math="x = x_0 + v_0 t + \frac{1}{2} a t^2" />
            <span>Posición</span>
          </div>
          <div className="formula-card">
            <BlockMath math="v^2 = v_0^2 + 2a(x - x_0)" />
            <span>Sin tiempo</span>
          </div>
        </div>
      </div>
    </div>
  )
}

// Gráficas del movimiento
const MotionGraphs: React.FC<{ data: MotionData[] }> = ({ data }) => {
  const canvasRef = useRef<HTMLCanvasElement>(null)

  useEffect(() => {
    if (!canvasRef.current || data.length < 2) return

    const ctx = canvasRef.current.getContext('2d')!
    const width = canvasRef.current.width
    const height = canvasRef.current.height

    // Limpiar
    ctx.fillStyle = '#1a1a2e'
    ctx.fillRect(0, 0, width, height)

    // Configurar
    const margin = 40
    const graphWidth = (width - margin * 3) / 2
    const graphHeight = height - margin * 2

    // Gráfica posición-tiempo
    drawGraph(ctx, data, 'position', margin, margin, graphWidth, graphHeight, '#4CAF50', 'x (m)')

    // Gráfica velocidad-tiempo
    drawGraph(ctx, data, 'velocity', margin * 2 + graphWidth, margin, graphWidth, graphHeight, '#2196F3', 'v (m/s)')

  }, [data])

  const drawGraph = (
    ctx: CanvasRenderingContext2D,
    data: MotionData[],
    key: 'position' | 'velocity',
    x: number,
    y: number,
    width: number,
    height: number,
    color: string,
    label: string
  ) => {
    // Ejes
    ctx.strokeStyle = '#666'
    ctx.lineWidth = 1
    ctx.beginPath()
    ctx.moveTo(x, y)
    ctx.lineTo(x, y + height)
    ctx.lineTo(x + width, y + height)
    ctx.stroke()

    // Etiquetas
    ctx.fillStyle = '#fff'
    ctx.font = '12px Inter'
    ctx.fillText(label, x + 5, y + 15)
    ctx.fillText('t (s)', x + width - 30, y + height - 5)

    if (data.length < 2) return

    // Escala
    const maxTime = Math.max(...data.map(d => d.time), 1)
    const values = data.map(d => d[key])
    const maxVal = Math.max(...values.map(Math.abs), 1)

    // Dibujar línea
    ctx.strokeStyle = color
    ctx.lineWidth = 2
    ctx.beginPath()

    data.forEach((point, i) => {
      const px = x + (point.time / maxTime) * width
      const py = y + height / 2 - (point[key] / maxVal) * (height / 2) * 0.8

      if (i === 0) ctx.moveTo(px, py)
      else ctx.lineTo(px, py)
    })

    ctx.stroke()

    // Línea del cero
    ctx.strokeStyle = '#444'
    ctx.setLineDash([5, 5])
    ctx.beginPath()
    ctx.moveTo(x, y + height / 2)
    ctx.lineTo(x + width, y + height / 2)
    ctx.stroke()
    ctx.setLineDash([])
  }

  return (
    <div className="motion-graphs">
      <h4>📈 Gráficas en tiempo real</h4>
      <canvas ref={canvasRef} width={600} height={200} />
    </div>
  )
}
```

---

## Laboratorio 3: Circuitos Eléctricos

```tsx
import React, { useState, useCallback } from 'react'
import { Canvas } from '@react-three/fiber'
import { OrbitControls, Html } from '@react-three/drei'

interface CircuitComponent {
  id: string
  type: 'battery' | 'resistor' | 'led' | 'switch' | 'wire'
  value?: number // Voltaje o resistencia
  position: [number, number]
  rotation: number
  isOn?: boolean
}

interface CircuitConnection {
  from: string
  to: string
}

// Componente de batería 3D
const Battery3D: React.FC<{
  position: [number, number, number]
  voltage: number
}> = ({ position, voltage }) => {
  return (
    <group position={position}>
      <mesh>
        <cylinderGeometry args={[0.3, 0.3, 0.8, 16]} />
        <meshStandardMaterial color="#333" />
      </mesh>
      {/* Terminal positivo */}
      <mesh position={[0, 0.45, 0]}>
        <cylinderGeometry args={[0.15, 0.15, 0.1, 16]} />
        <meshStandardMaterial color="#c0392b" />
      </mesh>
      {/* Terminal negativo */}
      <mesh position={[0, -0.45, 0]}>
        <cylinderGeometry args={[0.2, 0.2, 0.05, 16]} />
        <meshStandardMaterial color="#2c3e50" />
      </mesh>
      <Html position={[0.5, 0, 0]}>
        <div className="component-label">{voltage}V</div>
      </Html>
    </group>
  )
}

// Componente de resistencia 3D
const Resistor3D: React.FC<{
  position: [number, number, number]
  resistance: number
  current: number
}> = ({ position, resistance, current }) => {
  // Calcular color basado en temperatura (más corriente = más caliente)
  const heat = Math.min(current / 2, 1)
  const color = `rgb(${139 + heat * 116}, ${90 - heat * 90}, ${43 - heat * 43})`

  return (
    <group position={position} rotation={[0, 0, Math.PI / 2]}>
      {/* Cuerpo */}
      <mesh>
        <cylinderGeometry args={[0.15, 0.15, 0.6, 16]} />
        <meshStandardMaterial color={color} />
      </mesh>
      {/* Bandas de color (simplificado) */}
      {[0.1, 0, -0.1].map((y, i) => (
        <mesh key={i} position={[0, y, 0]}>
          <torusGeometry args={[0.16, 0.02, 8, 16]} />
          <meshStandardMaterial color={i === 0 ? '#c0392b' : i === 1 ? '#9b59b6' : '#f39c12'} />
        </mesh>
      ))}
      {/* Terminales */}
      {[-0.4, 0.4].map((y) => (
        <mesh key={y} position={[0, y, 0]}>
          <cylinderGeometry args={[0.05, 0.05, 0.2, 8]} />
          <meshStandardMaterial color="#bdc3c7" metalness={0.8} />
        </mesh>
      ))}
      <Html position={[0, 0.5, 0]}>
        <div className="component-label">{resistance}Ω</div>
      </Html>
    </group>
  )
}

// LED 3D
const LED3D: React.FC<{
  position: [number, number, number]
  isOn: boolean
  color: string
}> = ({ position, isOn, color }) => {
  return (
    <group position={position}>
      {/* Cúpula del LED */}
      <mesh>
        <sphereGeometry args={[0.2, 16, 16, 0, Math.PI * 2, 0, Math.PI / 2]} />
        <meshPhysicalMaterial
          color={color}
          emissive={isOn ? color : '#000'}
          emissiveIntensity={isOn ? 2 : 0}
          transparent
          opacity={0.9}
          transmission={0.3}
        />
      </mesh>
      {/* Base */}
      <mesh position={[0, -0.15, 0]}>
        <cylinderGeometry args={[0.18, 0.2, 0.1, 16]} />
        <meshStandardMaterial color="#7f8c8d" />
      </mesh>
      {/* Patas */}
      {[-0.08, 0.08].map((x) => (
        <mesh key={x} position={[x, -0.3, 0]}>
          <cylinderGeometry args={[0.02, 0.02, 0.2, 8]} />
          <meshStandardMaterial color="#bdc3c7" metalness={0.9} />
        </mesh>
      ))}
      {/* Luz */}
      {isOn && <pointLight color={color} intensity={1} distance={2} />}
    </group>
  )
}

// Interruptor 3D
const Switch3D: React.FC<{
  position: [number, number, number]
  isOn: boolean
  onToggle: () => void
}> = ({ position, isOn, onToggle }) => {
  return (
    <group position={position} onClick={onToggle}>
      {/* Base */}
      <mesh>
        <boxGeometry args={[0.4, 0.1, 0.3]} />
        <meshStandardMaterial color="#34495e" />
      </mesh>
      {/* Palanca */}
      <mesh
        position={[isOn ? 0.1 : -0.1, 0.1, 0]}
        rotation={[0, 0, isOn ? -0.3 : 0.3]}
      >
        <boxGeometry args={[0.3, 0.08, 0.15]} />
        <meshStandardMaterial color={isOn ? '#27ae60' : '#c0392b'} />
      </mesh>
      <Html position={[0, 0.3, 0]}>
        <div className="component-label">{isOn ? 'ON' : 'OFF'}</div>
      </Html>
    </group>
  )
}

// Línea de conexión (cable)
const Wire3D: React.FC<{
  start: [number, number, number]
  end: [number, number, number]
  current: number
}> = ({ start, end, current }) => {
  const hasFlow = current > 0

  return (
    <line>
      <bufferGeometry>
        <bufferAttribute
          attach="attributes-position"
          array={new Float32Array([...start, ...end])}
          count={2}
          itemSize={3}
        />
      </bufferGeometry>
      <lineBasicMaterial
        color={hasFlow ? '#f39c12' : '#7f8c8d'}
        linewidth={hasFlow ? 3 : 2}
      />
    </line>
  )
}

// Calculadora de circuitos (Ley de Ohm)
const calculateCircuit = (
  components: CircuitComponent[],
  connections: CircuitConnection[]
): { current: number; voltages: Record<string, number> } => {
  const battery = components.find(c => c.type === 'battery')
  const resistors = components.filter(c => c.type === 'resistor')
  const switches = components.filter(c => c.type === 'switch')

  // Si hay un interruptor abierto, no hay corriente
  if (switches.some(s => !s.isOn)) {
    return { current: 0, voltages: {} }
  }

  const totalVoltage = battery?.value || 0
  const totalResistance = resistors.reduce((sum, r) => sum + (r.value || 0), 0) || 1

  // Ley de Ohm: I = V / R
  const current = totalVoltage / totalResistance

  // Calcular caída de voltaje en cada resistencia
  const voltages: Record<string, number> = {}
  resistors.forEach(r => {
    voltages[r.id] = current * (r.value || 0)
  })

  return { current, voltages }
}

// Componente principal del laboratorio de circuitos
export const CircuitLab: React.FC = () => {
  const [components, setComponents] = useState<CircuitComponent[]>([
    { id: 'battery1', type: 'battery', value: 9, position: [-2, 0], rotation: 0 },
    { id: 'switch1', type: 'switch', position: [0, 1], rotation: 0, isOn: false },
    { id: 'resistor1', type: 'resistor', value: 100, position: [2, 0], rotation: 0 },
    { id: 'led1', type: 'led', position: [0, -1], rotation: 0 },
  ])

  const [connections] = useState<CircuitConnection[]>([
    { from: 'battery1', to: 'switch1' },
    { from: 'switch1', to: 'resistor1' },
    { from: 'resistor1', to: 'led1' },
    { from: 'led1', to: 'battery1' },
  ])

  const circuitData = calculateCircuit(components, connections)

  const toggleSwitch = useCallback((id: string) => {
    setComponents(prev =>
      prev.map(c =>
        c.id === id ? { ...c, isOn: !c.isOn } : c
      )
    )
  }, [])

  const updateResistance = useCallback((id: string, value: number) => {
    setComponents(prev =>
      prev.map(c =>
        c.id === id ? { ...c, value } : c
      )
    )
  }, [])

  const updateVoltage = useCallback((id: string, value: number) => {
    setComponents(prev =>
      prev.map(c =>
        c.id === id ? { ...c, value } : c
      )
    )
  }, [])

  const battery = components.find(c => c.type === 'battery')
  const resistor = components.find(c => c.type === 'resistor')
  const switchComp = components.find(c => c.type === 'switch')

  return (
    <div className="virtual-lab circuit-lab">
      <div className="lab-header">
        <h2>⚡ Laboratorio de Circuitos</h2>
        <p>
          Construye circuitos y observa cómo la corriente fluye según la Ley de Ohm.
        </p>
      </div>

      <div className="lab-content">
        {/* Escena 3D */}
        <div className="lab-canvas">
          <Canvas camera={{ position: [0, 5, 5], fov: 50 }}>
            <ambientLight intensity={0.5} />
            <pointLight position={[10, 10, 10]} intensity={1} />

            <Battery3D
              position={[-2, 0, 0]}
              voltage={battery?.value || 9}
            />

            <Switch3D
              position={[0, 0, 1]}
              isOn={switchComp?.isOn || false}
              onToggle={() => toggleSwitch('switch1')}
            />

            <Resistor3D
              position={[2, 0, 0]}
              resistance={resistor?.value || 100}
              current={circuitData.current}
            />

            <LED3D
              position={[0, 0, -1]}
              isOn={circuitData.current > 0.01}
              color="#e74c3c"
            />

            {/* Cables */}
            <Wire3D start={[-1.5, 0, 0]} end={[-0.5, 0, 1]} current={circuitData.current} />
            <Wire3D start={[0.5, 0, 1]} end={[1.5, 0, 0]} current={circuitData.current} />
            <Wire3D start={[2.5, 0, 0]} end={[0.5, 0, -1]} current={circuitData.current} />
            <Wire3D start={[-0.5, 0, -1]} end={[-1.5, 0, 0]} current={circuitData.current} />

            <OrbitControls enablePan={false} />
          </Canvas>

          {/* Panel de mediciones */}
          <div className="measurements-panel">
            <h4>📊 Mediciones</h4>
            <div className="measurement">
              <span>Voltaje total:</span>
              <span className="value">{battery?.value || 0} V</span>
            </div>
            <div className="measurement">
              <span>Resistencia total:</span>
              <span className="value">{resistor?.value || 0} Ω</span>
            </div>
            <div className="measurement">
              <span>Corriente:</span>
              <span className="value">{(circuitData.current * 1000).toFixed(1)} mA</span>
            </div>
            <div className="measurement">
              <span>Potencia:</span>
              <span className="value">
                {((battery?.value || 0) * circuitData.current).toFixed(2)} W
              </span>
            </div>
          </div>
        </div>

        {/* Controles */}
        <div className="lab-controls">
          <div className="control-group">
            <label>⚡ Voltaje de la batería: {battery?.value} V</label>
            <input
              type="range"
              min="1"
              max="12"
              step="0.5"
              value={battery?.value || 9}
              onChange={(e) => updateVoltage('battery1', parseFloat(e.target.value))}
            />
          </div>

          <div className="control-group">
            <label>🔧 Resistencia: {resistor?.value} Ω</label>
            <input
              type="range"
              min="10"
              max="1000"
              step="10"
              value={resistor?.value || 100}
              onChange={(e) => updateResistance('resistor1', parseFloat(e.target.value))}
            />
          </div>

          <div className="control-group">
            <label>🔌 Interruptor</label>
            <button
              className={`switch-btn ${switchComp?.isOn ? 'on' : 'off'}`}
              onClick={() => toggleSwitch('switch1')}
            >
              {switchComp?.isOn ? '🟢 Encendido' : '🔴 Apagado'}
            </button>
          </div>
        </div>
      </div>

      {/* Fórmulas */}
      <div className="formulas-reference">
        <h4>📐 Ley de Ohm</h4>
        <div className="formula-grid">
          <div className="formula-card">
            <BlockMath math="V = I \cdot R" />
            <span>Voltaje = Corriente × Resistencia</span>
          </div>
          <div className="formula-card">
            <BlockMath math="I = \frac{V}{R}" />
            <span>Corriente = Voltaje / Resistencia</span>
          </div>
          <div className="formula-card">
            <BlockMath math="P = V \cdot I = I^2 R" />
            <span>Potencia eléctrica</span>
          </div>
        </div>
      </div>

      {/* Retos */}
      <CircuitChallenges
        current={circuitData.current}
        voltage={battery?.value || 0}
        resistance={resistor?.value || 0}
      />
    </div>
  )
}

// Retos del laboratorio
const CircuitChallenges: React.FC<{
  current: number
  voltage: number
  resistance: number
}> = ({ current, voltage, resistance }) => {
  const [completedChallenges, setCompletedChallenges] = useState<string[]>([])

  const challenges = [
    {
      id: 'challenge1',
      title: 'Enciende el LED',
      description: 'Haz que la corriente fluya por el circuito',
      check: () => current > 0.01,
    },
    {
      id: 'challenge2',
      title: 'Corriente de 50mA',
      description: 'Ajusta el circuito para obtener exactamente 50mA (±5mA)',
      check: () => Math.abs(current * 1000 - 50) < 5,
    },
    {
      id: 'challenge3',
      title: 'Máxima eficiencia',
      description: 'Consigue que el LED brille con menos de 20mA',
      check: () => current > 0.01 && current < 0.02,
    },
    {
      id: 'challenge4',
      title: 'Calcula primero',
      description: 'Con V=6V, ¿qué R necesitas para I=30mA? (R=200Ω)',
      check: () => voltage === 6 && Math.abs(resistance - 200) < 10 && Math.abs(current * 1000 - 30) < 3,
    },
  ]

  useEffect(() => {
    challenges.forEach(challenge => {
      if (challenge.check() && !completedChallenges.includes(challenge.id)) {
        setCompletedChallenges(prev => [...prev, challenge.id])
      }
    })
  }, [current, voltage, resistance])

  return (
    <div className="challenges-panel">
      <h4>🎯 Retos</h4>
      <div className="challenges-list">
        {challenges.map(challenge => (
          <div
            key={challenge.id}
            className={`challenge ${completedChallenges.includes(challenge.id) ? 'completed' : ''}`}
          >
            <span className="challenge-status">
              {completedChallenges.includes(challenge.id) ? '✅' : '⬜'}
            </span>
            <div className="challenge-content">
              <strong>{challenge.title}</strong>
              <p>{challenge.description}</p>
            </div>
          </div>
        ))}
      </div>
    </div>
  )
}
```

---

## Estilos CSS para Laboratorios

```css
/* Contenedor principal del laboratorio */
.virtual-lab {
  background: white;
  border-radius: var(--radius-lg);
  overflow: hidden;
  margin-bottom: var(--spacing-xl);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.lab-header {
  background: linear-gradient(135deg, var(--color-primary), var(--color-primary-dark));
  color: white;
  padding: var(--spacing-lg);
}

.lab-header h2 {
  margin: 0 0 var(--spacing-xs) 0;
}

.lab-header p {
  margin: 0;
  opacity: 0.9;
}

.lab-content {
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: var(--spacing-md);
  padding: var(--spacing-md);
}

@media (max-width: 768px) {
  .lab-content {
    grid-template-columns: 1fr;
  }
}

/* Canvas 3D */
.lab-canvas {
  position: relative;
  height: 400px;
  background: linear-gradient(180deg, #1a1a2e 0%, #16213e 100%);
  border-radius: var(--radius-md);
  overflow: hidden;
}

/* Controles del laboratorio */
.lab-controls {
  background: #f8f9fa;
  padding: var(--spacing-md);
  border-radius: var(--radius-md);
}

.control-section {
  margin-bottom: var(--spacing-md);
}

.control-section h4 {
  margin: 0 0 var(--spacing-xs) 0;
  font-size: 0.9rem;
}

.control-section .hint {
  font-size: 0.75rem;
  color: #666;
  margin-bottom: var(--spacing-sm);
}

/* Botones de líquidos */
.liquid-buttons {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: var(--spacing-xs);
}

.liquid-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: var(--spacing-sm);
  border: 2px solid;
  border-radius: var(--radius-md);
  background: transparent;
  cursor: pointer;
  transition: all 0.2s;
}

.liquid-btn.selected {
  color: white;
  transform: scale(1.02);
}

.liquid-name {
  font-weight: 600;
  font-size: 0.875rem;
}

.liquid-density {
  font-size: 0.7rem;
  opacity: 0.8;
}

/* Botones de objetos */
.object-buttons {
  display: flex;
  flex-wrap: wrap;
  gap: var(--spacing-xs);
}

.object-btn {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: var(--spacing-xs) var(--spacing-sm);
  border: none;
  border-radius: var(--radius-sm);
  color: white;
  cursor: pointer;
  transition: all 0.2s;
}

.object-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.object-btn:hover:not(:disabled) {
  transform: translateY(-2px);
}

/* Panel de datos */
.data-panel {
  padding: var(--spacing-md);
  background: #f8f9fa;
  margin: var(--spacing-md);
  border-radius: var(--radius-md);
}

.data-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.875rem;
}

.data-table th,
.data-table td {
  padding: var(--spacing-sm);
  text-align: left;
  border-bottom: 1px solid #e0e0e0;
}

.data-table th {
  background: #e8e8e8;
  font-weight: 600;
}

.color-dot {
  display: inline-block;
  width: 12px;
  height: 12px;
  border-radius: 50%;
  margin-right: var(--spacing-xs);
}

/* Guía del experimento */
.experiment-guide {
  position: absolute;
  top: var(--spacing-md);
  right: var(--spacing-md);
  width: 280px;
  background: white;
  border-radius: var(--radius-md);
  box-shadow: 0 4px 16px rgba(0, 0, 0, 0.2);
  z-index: 10;
}

.guide-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: var(--spacing-sm) var(--spacing-md);
  border-bottom: 1px solid #e0e0e0;
}

.guide-header h4 {
  margin: 0;
  font-size: 0.9rem;
}

.guide-header button {
  background: none;
  border: none;
  cursor: pointer;
  font-size: 1.2rem;
  color: #666;
}

.guide-steps {
  padding: var(--spacing-sm);
  max-height: 300px;
  overflow-y: auto;
}

.guide-step {
  display: flex;
  gap: var(--spacing-sm);
  padding: var(--spacing-xs);
  margin-bottom: var(--spacing-xs);
  border-radius: var(--radius-sm);
  opacity: 0.5;
}

.guide-step.current {
  opacity: 1;
  background: rgba(76, 175, 80, 0.1);
}

.guide-step.completed {
  opacity: 0.7;
}

.guide-step.completed .step-number {
  background: var(--color-primary);
  color: white;
}

.step-number {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #e0e0e0;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 0.75rem;
  font-weight: 600;
  flex-shrink: 0;
}

.step-content h5 {
  margin: 0 0 var(--spacing-xs) 0;
  font-size: 0.8rem;
}

.step-content p {
  margin: 0;
  font-size: 0.75rem;
  color: #666;
}

.action-hint {
  color: var(--color-primary);
  font-style: italic;
}

/* Datos en tiempo real */
.realtime-data {
  position: absolute;
  bottom: var(--spacing-md);
  left: var(--spacing-md);
  display: flex;
  gap: var(--spacing-md);
  background: rgba(0, 0, 0, 0.8);
  padding: var(--spacing-sm) var(--spacing-md);
  border-radius: var(--radius-md);
}

.data-item {
  display: flex;
  flex-direction: column;
  align-items: center;
}

.data-item .label {
  font-size: 0.7rem;
  color: #aaa;
}

.data-item .value {
  font-size: 1rem;
  font-weight: 600;
  color: white;
  font-family: monospace;
}

/* Leyenda de densidad */
.density-legend {
  position: absolute;
  top: var(--spacing-md);
  left: var(--spacing-md);
  background: rgba(255, 255, 255, 0.95);
  padding: var(--spacing-sm);
  border-radius: var(--radius-md);
}

.density-legend h5 {
  margin: 0 0 var(--spacing-xs) 0;
  font-size: 0.75rem;
}

.legend-item {
  padding: 2px 8px;
  border-radius: 4px;
  font-size: 0.7rem;
  margin-bottom: 2px;
  color: white;
  text-shadow: 0 1px 2px rgba(0, 0, 0, 0.3);
}

/* Gráficas */
.motion-graphs {
  padding: var(--spacing-md);
  background: #1a1a2e;
  margin: var(--spacing-md);
  border-radius: var(--radius-md);
}

.motion-graphs h4 {
  color: white;
  margin: 0 0 var(--spacing-sm) 0;
}

.motion-graphs canvas {
  width: 100%;
  border-radius: var(--radius-sm);
}

/* Fórmulas */
.formulas-reference {
  padding: var(--spacing-md);
  background: #f8f9fa;
  margin: var(--spacing-md);
  border-radius: var(--radius-md);
}

.formulas-reference h4 {
  margin: 0 0 var(--spacing-md) 0;
}

.formula-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: var(--spacing-md);
}

.formula-card {
  background: white;
  padding: var(--spacing-md);
  border-radius: var(--radius-md);
  text-align: center;
  border: 1px solid #e0e0e0;
}

.formula-card span {
  display: block;
  font-size: 0.75rem;
  color: #666;
  margin-top: var(--spacing-xs);
}

/* Retos */
.challenges-panel {
  padding: var(--spacing-md);
  margin: var(--spacing-md);
  background: linear-gradient(135deg, #fff8e1, #fff3e0);
  border-radius: var(--radius-md);
}

.challenges-panel h4 {
  margin: 0 0 var(--spacing-md) 0;
}

.challenge {
  display: flex;
  align-items: flex-start;
  gap: var(--spacing-sm);
  padding: var(--spacing-sm);
  background: white;
  border-radius: var(--radius-sm);
  margin-bottom: var(--spacing-xs);
  transition: all 0.3s;
}

.challenge.completed {
  background: rgba(76, 175, 80, 0.1);
  border: 1px solid #4CAF50;
}

.challenge-status {
  font-size: 1.2rem;
}

.challenge-content strong {
  display: block;
  font-size: 0.875rem;
}

.challenge-content p {
  margin: 0;
  font-size: 0.75rem;
  color: #666;
}

/* Tooltips de componentes */
.object-tooltip,
.component-label {
  background: white;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 11px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  white-space: nowrap;
}

.object-tooltip .status {
  font-weight: 600;
}

.object-tooltip .status.flotando { color: #4CAF50; }
.object-tooltip .status.hundiéndose { color: #f44336; }
.object-tooltip .status.equilibrio { color: #FF9800; }

/* Botón de reinicio */
.reset-btn {
  display: block;
  width: calc(100% - var(--spacing-md) * 2);
  margin: 0 var(--spacing-md) var(--spacing-md);
  padding: var(--spacing-sm);
  background: #f5f5f5;
  border: none;
  border-radius: var(--radius-md);
  cursor: pointer;
  font-size: 0.875rem;
  transition: background 0.2s;
}

.reset-btn:hover {
  background: #e0e0e0;
}
```

---

## Integración con Tutor IA

```tsx
// Añadir asistente IA contextual a los laboratorios
import { AIChatbot } from './AIChatbot'

const labSystemPrompt = `Eres un profesor de laboratorio virtual.
Guías a los estudiantes mientras experimentan.

Cuando el estudiante tenga dudas:
1. No des la respuesta directa
2. Haz preguntas que le guíen a descubrirla
3. Sugiere qué variable modificar
4. Relaciona con la teoría

Responde de forma breve y alentadora.`

// En cada laboratorio, añadir:
<AIChatbot
  position="floating"
  title="Asistente de Laboratorio"
  systemPrompt={labSystemPrompt}
  suggestedQuestions={[
    '¿Por qué el corcho flota?',
    '¿Cómo calculo la corriente?',
    '¿Qué pasa si aumento la velocidad?',
  ]}
/>
```

---

## Dependencias

```json
{
  "dependencies": {
    "@react-three/fiber": "^8.15.0",
    "@react-three/drei": "^9.88.0",
    "@react-three/rapier": "^1.2.0",
    "three": "^0.158.0",
    "zustand": "^4.5.0",
    "framer-motion": "^11.0.0",
    "react-katex": "^3.0.1"
  }
}
```
