# Physics Subject Specialization

Advanced 3D simulations, vector fields, motion physics, and interactive experiments for Physics education.

## Physics-Specific Dependencies

```bash
npm install @react-three/rapier    # Physics engine
npm install mafs                    # Math visualizations
npm install katex react-katex       # LaTeX equations
npm install d3                      # Data visualization
npm install mathjs                  # Math calculations
```

---

## 1. Motion & Kinematics Simulations

### Projectile Motion with Real Physics

```tsx
import { Physics, RigidBody, vec3 } from '@react-three/rapier'
import { useRef, useState } from 'react'
import { Line } from '@react-three/drei'

interface ProjectileProps {
  initialVelocity: number  // m/s
  angle: number            // degrees
  gravity?: number         // m/s²
}

export const ProjectileMotion: React.FC<ProjectileProps> = ({
  initialVelocity,
  angle,
  gravity = 9.81
}) => {
  const [trajectory, setTrajectory] = useState<[number, number, number][]>([])
  const [isLaunched, setIsLaunched] = useState(false)
  const ballRef = useRef<RapierRigidBody>(null)

  const launch = () => {
    if (!ballRef.current) return

    const angleRad = (angle * Math.PI) / 180
    const vx = initialVelocity * Math.cos(angleRad)
    const vy = initialVelocity * Math.sin(angleRad)

    ballRef.current.setLinvel(vec3({ x: vx, y: vy, z: 0 }), true)
    setIsLaunched(true)
    setTrajectory([])
  }

  // Record trajectory
  useFrame(() => {
    if (ballRef.current && isLaunched) {
      const pos = ballRef.current.translation()
      setTrajectory(prev => [...prev, [pos.x, pos.y, pos.z]])
    }
  })

  // Theoretical trajectory (parabola)
  const theoreticalPath = useMemo(() => {
    const points: [number, number, number][] = []
    const angleRad = (angle * Math.PI) / 180
    const vx = initialVelocity * Math.cos(angleRad)
    const vy = initialVelocity * Math.sin(angleRad)
    const tMax = (2 * vy) / gravity

    for (let t = 0; t <= tMax; t += 0.05) {
      const x = vx * t
      const y = vy * t - 0.5 * gravity * t * t
      if (y >= 0) points.push([x, y, 0])
    }
    return points
  }, [initialVelocity, angle, gravity])

  return (
    <Physics gravity={[0, -gravity, 0]}>
      {/* Ground */}
      <RigidBody type="fixed" position={[0, -0.5, 0]}>
        <mesh>
          <boxGeometry args={[50, 1, 10]} />
          <meshStandardMaterial color="#2d5a27" />
        </mesh>
      </RigidBody>

      {/* Projectile */}
      <RigidBody ref={ballRef} position={[0, 1, 0]} colliders="ball">
        <mesh>
          <sphereGeometry args={[0.3]} />
          <meshStandardMaterial color="#f44336" />
        </mesh>
      </RigidBody>

      {/* Actual trajectory trail */}
      {trajectory.length > 1 && (
        <Line points={trajectory} color="#f44336" lineWidth={3} />
      )}

      {/* Theoretical parabola (dashed) */}
      <Line
        points={theoreticalPath}
        color="#4CAF50"
        lineWidth={2}
        dashed
        dashSize={0.3}
        gapSize={0.1}
      />

      {/* Velocity vector */}
      <Arrow
        origin={[0, 1, 0]}
        direction={[
          Math.cos((angle * Math.PI) / 180),
          Math.sin((angle * Math.PI) / 180),
          0
        ]}
        length={initialVelocity / 10}
        color="#2196F3"
      />

      {/* Launch button overlay */}
      <Html position={[0, 3, 0]}>
        <button onClick={launch}>🚀 Lanzar</button>
      </Html>
    </Physics>
  )
}
```

### Pendulum with Energy Visualization

```tsx
export const PendulumSimulation: React.FC<{
  length: number      // meters
  initialAngle: number // degrees
  damping?: number
}> = ({ length, initialAngle, damping = 0.01 }) => {
  const [angle, setAngle] = useState(initialAngle * Math.PI / 180)
  const [angularVelocity, setAngularVelocity] = useState(0)
  const [energyHistory, setEnergyHistory] = useState<{ke: number, pe: number}[]>([])

  const g = 9.81
  const mass = 1 // kg

  useFrame((_, delta) => {
    // Simple pendulum equation: θ'' = -(g/L)sin(θ) - damping * θ'
    const angularAcceleration = -(g / length) * Math.sin(angle) - damping * angularVelocity

    const newVelocity = angularVelocity + angularAcceleration * delta
    const newAngle = angle + newVelocity * delta

    setAngularVelocity(newVelocity)
    setAngle(newAngle)

    // Calculate energies
    const height = length * (1 - Math.cos(angle))
    const velocity = length * Math.abs(angularVelocity)
    const ke = 0.5 * mass * velocity * velocity
    const pe = mass * g * height

    setEnergyHistory(prev => [...prev.slice(-100), { ke, pe }])
  })

  const bobX = length * Math.sin(angle)
  const bobY = -length * Math.cos(angle)

  return (
    <group>
      {/* Pivot point */}
      <mesh position={[0, 0, 0]}>
        <sphereGeometry args={[0.1]} />
        <meshStandardMaterial color="#666" />
      </mesh>

      {/* String */}
      <Line
        points={[[0, 0, 0], [bobX, bobY, 0]]}
        color="#333"
        lineWidth={2}
      />

      {/* Bob with color based on velocity */}
      <mesh position={[bobX, bobY, 0]}>
        <sphereGeometry args={[0.3]} />
        <meshStandardMaterial
          color={`hsl(${Math.abs(angularVelocity) * 30}, 80%, 50%)`}
          emissive={`hsl(${Math.abs(angularVelocity) * 30}, 80%, 30%)`}
          emissiveIntensity={0.3}
        />
      </mesh>

      {/* Velocity vector */}
      <Arrow
        origin={[bobX, bobY, 0]}
        direction={[Math.cos(angle), Math.sin(angle), 0]}
        length={Math.abs(angularVelocity) * 0.5}
        color="#2196F3"
      />

      {/* Energy bar overlay */}
      <Html position={[2, 0, 0]}>
        <EnergyBar
          kineticEnergy={energyHistory[energyHistory.length - 1]?.ke || 0}
          potentialEnergy={energyHistory[energyHistory.length - 1]?.pe || 0}
        />
      </Html>
    </group>
  )
}

// Energy visualization component
const EnergyBar: React.FC<{ kineticEnergy: number; potentialEnergy: number }> = ({
  kineticEnergy, potentialEnergy
}) => {
  const total = kineticEnergy + potentialEnergy
  const kePercent = (kineticEnergy / total) * 100
  const pePercent = (potentialEnergy / total) * 100

  return (
    <div style={{ width: 150, background: '#1a1a2e', padding: 10, borderRadius: 8 }}>
      <div style={{ marginBottom: 8, color: 'white', fontSize: 12 }}>Energía</div>
      <div style={{ display: 'flex', height: 20, borderRadius: 4, overflow: 'hidden' }}>
        <div style={{ width: `${kePercent}%`, background: '#f44336' }} title="Cinética" />
        <div style={{ width: `${pePercent}%`, background: '#4CAF50' }} title="Potencial" />
      </div>
      <div style={{ display: 'flex', justifyContent: 'space-between', marginTop: 4, fontSize: 10, color: '#aaa' }}>
        <span>Ec: {kineticEnergy.toFixed(2)} J</span>
        <span>Ep: {potentialEnergy.toFixed(2)} J</span>
      </div>
    </div>
  )
}
```

---

## 2. Vector Fields & Forces

### Electric Field Visualization

```tsx
import * as THREE from 'three'

interface Charge {
  position: [number, number, number]
  value: number  // positive or negative
}

export const ElectricFieldVisualization: React.FC<{
  charges: Charge[]
  gridSize?: number
  gridResolution?: number
}> = ({ charges, gridSize = 10, gridResolution = 15 }) => {
  // Calculate field at each point
  const fieldVectors = useMemo(() => {
    const vectors: { position: THREE.Vector3; field: THREE.Vector3 }[] = []
    const k = 8.99e9 // Coulomb constant (scaled for visualization)

    for (let x = -gridSize/2; x <= gridSize/2; x += gridSize/gridResolution) {
      for (let y = -gridSize/2; y <= gridSize/2; y += gridSize/gridResolution) {
        const point = new THREE.Vector3(x, y, 0)
        const totalField = new THREE.Vector3(0, 0, 0)

        charges.forEach(charge => {
          const chargePos = new THREE.Vector3(...charge.position)
          const r = point.clone().sub(chargePos)
          const distance = r.length()

          if (distance > 0.5) { // Avoid singularity
            const magnitude = (k * Math.abs(charge.value)) / (distance * distance)
            const direction = r.normalize()
            if (charge.value < 0) direction.negate()
            totalField.add(direction.multiplyScalar(magnitude * 0.001))
          }
        })

        if (totalField.length() > 0.01) {
          vectors.push({ position: point, field: totalField })
        }
      }
    }
    return vectors
  }, [charges, gridSize, gridResolution])

  return (
    <group>
      {/* Charges */}
      {charges.map((charge, i) => (
        <mesh key={i} position={charge.position}>
          <sphereGeometry args={[0.3]} />
          <meshStandardMaterial
            color={charge.value > 0 ? '#f44336' : '#2196F3'}
            emissive={charge.value > 0 ? '#f44336' : '#2196F3'}
            emissiveIntensity={0.5}
          />
          <Html center>
            <span style={{ color: 'white', fontWeight: 'bold' }}>
              {charge.value > 0 ? '+' : '−'}
            </span>
          </Html>
        </mesh>
      ))}

      {/* Field arrows */}
      {fieldVectors.map((v, i) => (
        <Arrow
          key={i}
          origin={v.position.toArray() as [number, number, number]}
          direction={v.field.normalize().toArray() as [number, number, number]}
          length={Math.min(v.field.length() * 100, 0.8)}
          color={`hsl(${v.field.length() * 5000}, 80%, 50%)`}
          headLength={0.15}
          headWidth={0.1}
        />
      ))}

      {/* Field lines (streamlines) */}
      {charges.filter(c => c.value > 0).map((charge, i) => (
        <FieldLines key={i} startCharge={charge} allCharges={charges} />
      ))}
    </group>
  )
}

// Field lines emanating from positive charges
const FieldLines: React.FC<{
  startCharge: Charge
  allCharges: Charge[]
  numLines?: number
}> = ({ startCharge, allCharges, numLines = 8 }) => {
  const lines = useMemo(() => {
    const result: THREE.Vector3[][] = []

    for (let i = 0; i < numLines; i++) {
      const angle = (i / numLines) * Math.PI * 2
      const line: THREE.Vector3[] = []
      let pos = new THREE.Vector3(
        startCharge.position[0] + 0.4 * Math.cos(angle),
        startCharge.position[1] + 0.4 * Math.sin(angle),
        0
      )

      for (let step = 0; step < 200; step++) {
        line.push(pos.clone())

        // Calculate field at current position
        const field = new THREE.Vector3(0, 0, 0)
        allCharges.forEach(charge => {
          const chargePos = new THREE.Vector3(...charge.position)
          const r = pos.clone().sub(chargePos)
          const dist = r.length()
          if (dist > 0.3) {
            const mag = charge.value / (dist * dist)
            field.add(r.normalize().multiplyScalar(mag))
          }
        })

        if (field.length() < 0.001) break

        // Move along field direction
        pos = pos.clone().add(field.normalize().multiplyScalar(0.1))

        // Stop if we hit a negative charge
        const hitNegative = allCharges.some(c =>
          c.value < 0 && pos.distanceTo(new THREE.Vector3(...c.position)) < 0.4
        )
        if (hitNegative) break

        // Stop if out of bounds
        if (pos.length() > 10) break
      }

      if (line.length > 2) result.push(line)
    }
    return result
  }, [startCharge, allCharges, numLines])

  return (
    <>
      {lines.map((line, i) => (
        <Line
          key={i}
          points={line.map(v => v.toArray() as [number, number, number])}
          color="#ffeb3b"
          lineWidth={1.5}
          transparent
          opacity={0.6}
        />
      ))}
    </>
  )
}
```

---

## 3. Waves & Oscillations

### Interactive Wave Superposition

```tsx
const waveShader = {
  vertexShader: `
    uniform float uTime;
    uniform float uAmplitude1;
    uniform float uFrequency1;
    uniform float uAmplitude2;
    uniform float uFrequency2;
    uniform float uPhase2;

    varying vec2 vUv;
    varying float vElevation;

    void main() {
      vUv = uv;

      vec3 pos = position;

      // Wave 1
      float wave1 = uAmplitude1 * sin(uFrequency1 * pos.x - uTime * 2.0);

      // Wave 2
      float wave2 = uAmplitude2 * sin(uFrequency2 * pos.x - uTime * 2.0 + uPhase2);

      // Superposition
      pos.z = wave1 + wave2;
      vElevation = pos.z;

      gl_Position = projectionMatrix * modelViewMatrix * vec4(pos, 1.0);
    }
  `,
  fragmentShader: `
    varying float vElevation;

    void main() {
      // Color based on elevation (blue for troughs, red for peaks)
      vec3 lowColor = vec3(0.1, 0.3, 0.8);
      vec3 highColor = vec3(0.8, 0.2, 0.1);

      float t = (vElevation + 1.0) / 2.0;
      vec3 color = mix(lowColor, highColor, t);

      gl_FragColor = vec4(color, 1.0);
    }
  `
}

export const WaveSuperposition: React.FC = () => {
  const [wave1, setWave1] = useState({ amplitude: 0.5, frequency: 2 })
  const [wave2, setWave2] = useState({ amplitude: 0.5, frequency: 3, phase: 0 })
  const materialRef = useRef<THREE.ShaderMaterial>(null)

  useFrame(({ clock }) => {
    if (materialRef.current) {
      materialRef.current.uniforms.uTime.value = clock.elapsedTime
    }
  })

  return (
    <>
      <mesh rotation={[-Math.PI / 2, 0, 0]}>
        <planeGeometry args={[10, 5, 200, 100]} />
        <shaderMaterial
          ref={materialRef}
          vertexShader={waveShader.vertexShader}
          fragmentShader={waveShader.fragmentShader}
          uniforms={{
            uTime: { value: 0 },
            uAmplitude1: { value: wave1.amplitude },
            uFrequency1: { value: wave1.frequency },
            uAmplitude2: { value: wave2.amplitude },
            uFrequency2: { value: wave2.frequency },
            uPhase2: { value: wave2.phase },
          }}
          side={THREE.DoubleSide}
        />
      </mesh>

      <Html position={[-6, 2, 0]}>
        <WaveControls
          wave1={wave1}
          wave2={wave2}
          onWave1Change={setWave1}
          onWave2Change={setWave2}
        />
      </Html>
    </>
  )
}
```

---

## 4. LaTeX Equations with KaTeX

### Physics Formula Component

```tsx
import 'katex/dist/katex.min.css'
import { InlineMath, BlockMath } from 'react-katex'

// Common physics equations
export const PhysicsFormulas = {
  // Kinematics
  velocity: 'v = v_0 + at',
  position: 'x = x_0 + v_0 t + \\frac{1}{2}at^2',
  velocitySquared: 'v^2 = v_0^2 + 2a(x - x_0)',

  // Dynamics
  newtonSecond: 'F = ma',
  gravitationalForce: 'F = G\\frac{m_1 m_2}{r^2}',
  friction: 'f = \\mu N',

  // Energy
  kineticEnergy: 'E_c = \\frac{1}{2}mv^2',
  potentialEnergy: 'E_p = mgh',
  workEnergyTheorem: 'W = \\Delta E_c',

  // Waves
  waveEquation: 'y = A\\sin(kx - \\omega t + \\phi)',
  waveSpeed: 'v = \\lambda f',
  frequency: 'f = \\frac{1}{T}',

  // Electromagnetism
  coulombLaw: 'F = k\\frac{q_1 q_2}{r^2}',
  electricField: 'E = \\frac{F}{q} = k\\frac{Q}{r^2}',
  ohmsLaw: 'V = IR',

  // Thermodynamics
  idealGas: 'PV = nRT',
  heatTransfer: 'Q = mc\\Delta T',

  // Modern Physics
  einsteinMassEnergy: 'E = mc^2',
  photoelectricEffect: 'E = hf - W',
  deBroglie: '\\lambda = \\frac{h}{p}',
}

// Interactive formula with variable substitution
export const InteractiveFormula: React.FC<{
  formula: string
  variables: Record<string, { value: number; unit: string; min: number; max: number }>
  result: { formula: string; unit: string }
  calculate: (vars: Record<string, number>) => number
}> = ({ formula, variables, result, calculate }) => {
  const [values, setValues] = useState(
    Object.fromEntries(Object.entries(variables).map(([k, v]) => [k, v.value]))
  )

  const resultValue = calculate(values)

  // Build formula with substituted values
  let substitutedFormula = formula
  Object.entries(values).forEach(([key, val]) => {
    substitutedFormula = substitutedFormula.replace(
      new RegExp(key, 'g'),
      val.toString()
    )
  })

  return (
    <div className="interactive-formula">
      <div className="formula-display">
        <BlockMath math={formula} />
      </div>

      <div className="variables">
        {Object.entries(variables).map(([key, config]) => (
          <div key={key} className="variable-slider">
            <label>
              <InlineMath math={key} /> = {values[key]} {config.unit}
            </label>
            <input
              type="range"
              min={config.min}
              max={config.max}
              step={(config.max - config.min) / 100}
              value={values[key]}
              onChange={(e) => setValues({
                ...values,
                [key]: parseFloat(e.target.value)
              })}
            />
          </div>
        ))}
      </div>

      <div className="calculation">
        <BlockMath math={`${substitutedFormula} = ${resultValue.toFixed(2)} \\text{ ${result.unit}}`} />
      </div>
    </div>
  )
}

// Usage example
<InteractiveFormula
  formula="E_c = \\frac{1}{2}mv^2"
  variables={{
    m: { value: 10, unit: 'kg', min: 1, max: 100 },
    v: { value: 5, unit: 'm/s', min: 0, max: 30 },
  }}
  result={{ formula: 'E_c', unit: 'J' }}
  calculate={({ m, v }) => 0.5 * m * v * v}
/>
```

---

## 5. Unit Conversion with Dimensional Analysis

```tsx
const physicsUnits = {
  length: {
    m: 1,
    km: 1000,
    cm: 0.01,
    mm: 0.001,
    ft: 0.3048,
    inch: 0.0254,
  },
  mass: {
    kg: 1,
    g: 0.001,
    mg: 0.000001,
    lb: 0.453592,
  },
  time: {
    s: 1,
    ms: 0.001,
    min: 60,
    h: 3600,
  },
  velocity: {
    'm/s': 1,
    'km/h': 1/3.6,
    'mph': 0.44704,
  },
  acceleration: {
    'm/s²': 1,
    'g': 9.81,
  },
  force: {
    N: 1,
    kN: 1000,
    dyn: 0.00001,
    lbf: 4.44822,
  },
  energy: {
    J: 1,
    kJ: 1000,
    cal: 4.184,
    kcal: 4184,
    eV: 1.602e-19,
    kWh: 3.6e6,
  },
  power: {
    W: 1,
    kW: 1000,
    MW: 1e6,
    hp: 745.7,
  },
}

export const PhysicsUnitConverter: React.FC = () => {
  const [category, setCategory] = useState<keyof typeof physicsUnits>('length')
  const [fromUnit, setFromUnit] = useState('m')
  const [toUnit, setToUnit] = useState('km')
  const [value, setValue] = useState(1)

  const units = physicsUnits[category]
  const result = value * units[fromUnit] / units[toUnit]

  return (
    <div className="unit-converter">
      <select value={category} onChange={e => setCategory(e.target.value as any)}>
        {Object.keys(physicsUnits).map(cat => (
          <option key={cat} value={cat}>{cat}</option>
        ))}
      </select>

      <div className="conversion">
        <input
          type="number"
          value={value}
          onChange={e => setValue(parseFloat(e.target.value) || 0)}
        />
        <select value={fromUnit} onChange={e => setFromUnit(e.target.value)}>
          {Object.keys(units).map(u => <option key={u} value={u}>{u}</option>)}
        </select>

        <span>=</span>

        <span className="result">{result.toExponential(4)}</span>
        <select value={toUnit} onChange={e => setToUnit(e.target.value)}>
          {Object.keys(units).map(u => <option key={u} value={u}>{u}</option>)}
        </select>
      </div>
    </div>
  )
}
```

---

## 6. Interactive Experiments

### Galileo's Inclined Plane

```tsx
export const InclinedPlaneExperiment: React.FC = () => {
  const [angle, setAngle] = useState(30)  // degrees
  const [mass, setMass] = useState(1)      // kg
  const [friction, setFriction] = useState(0.1)
  const [isRunning, setIsRunning] = useState(false)
  const ballRef = useRef<RapierRigidBody>(null)

  const g = 9.81
  const angleRad = (angle * Math.PI) / 180

  // Theoretical acceleration
  const acceleration = g * (Math.sin(angleRad) - friction * Math.cos(angleRad))

  return (
    <Physics gravity={[0, -g, 0]}>
      {/* Inclined plane */}
      <RigidBody type="fixed" rotation={[0, 0, -angleRad]}>
        <mesh position={[2, 1, 0]}>
          <boxGeometry args={[6, 0.2, 2]} />
          <meshStandardMaterial color="#8B4513" />
        </mesh>
      </RigidBody>

      {/* Rolling ball */}
      <RigidBody
        ref={ballRef}
        position={[0, 3, 0]}
        colliders="ball"
        linearDamping={friction * 2}
      >
        <mesh>
          <sphereGeometry args={[0.2]} />
          <meshStandardMaterial color="#f44336" metalness={0.5} />
        </mesh>
      </RigidBody>

      {/* Force vectors */}
      <ForceVectorDisplay
        position={[0, 3, 0]}
        mass={mass}
        angle={angle}
        g={g}
      />

      {/* Info overlay */}
      <Html position={[-3, 3, 0]}>
        <div className="experiment-controls">
          <h4>Plano Inclinado</h4>

          <label>
            Ángulo: {angle}°
            <input type="range" min={5} max={60} value={angle}
              onChange={e => setAngle(parseInt(e.target.value))} />
          </label>

          <label>
            Masa: {mass} kg
            <input type="range" min={0.5} max={10} step={0.5} value={mass}
              onChange={e => setMass(parseFloat(e.target.value))} />
          </label>

          <label>
            Coef. rozamiento: {friction}
            <input type="range" min={0} max={0.5} step={0.05} value={friction}
              onChange={e => setFriction(parseFloat(e.target.value))} />
          </label>

          <div className="calculations">
            <BlockMath math={`a = g(\\sin\\theta - \\mu\\cos\\theta)`} />
            <BlockMath math={`a = ${acceleration.toFixed(2)} \\text{ m/s}^2`} />
          </div>
        </div>
      </Html>
    </Physics>
  )
}
```

---

## Physics-Specific AI Chatbot Prompt

```tsx
const PHYSICS_TUTOR_PROMPT = `Eres un profesor experto en Física del currículo español (ESO y Bachillerato).

ESPECIALIDADES:
- Cinemática y dinámica
- Energía y trabajo
- Ondas y sonido
- Electricidad y magnetismo
- Óptica
- Física moderna

FORMATO DE RESPUESTAS:
1. Siempre usa notación LaTeX para fórmulas: $v = v_0 + at$
2. Incluye unidades SI en todos los cálculos
3. Resuelve problemas paso a paso mostrando:
   - Datos conocidos
   - Incógnitas
   - Fórmulas aplicables
   - Desarrollo matemático
   - Resultado con unidades
4. Usa analogías cotidianas para conceptos abstractos
5. Relaciona los conceptos con aplicaciones reales

EJEMPLO DE RESPUESTA A PROBLEMA:
"Para calcular la energía cinética de un coche de 1000 kg a 90 km/h:

**Datos:**
- m = 1000 kg
- v = 90 km/h = 25 m/s (convertimos a SI)

**Fórmula:**
$$E_c = \\frac{1}{2}mv^2$$

**Desarrollo:**
$$E_c = \\frac{1}{2} \\cdot 1000 \\cdot 25^2 = \\frac{1}{2} \\cdot 1000 \\cdot 625$$

**Resultado:**
$$E_c = 312.500 \\text{ J} = 312,5 \\text{ kJ}$$"
`

<AIChatbot
  systemPrompt={PHYSICS_TUTOR_PROMPT}
  title="Profesor de Física"
  suggestedQuestions={[
    '¿Cómo calculo la velocidad final en caída libre?',
    'Explica las leyes de Newton con ejemplos',
    '¿Qué es la energía potencial gravitatoria?',
    '¿Cómo funciona un circuito eléctrico?',
  ]}
/>
```
