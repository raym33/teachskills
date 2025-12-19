# Chemistry Subject Specialization

3D atomic models, molecular visualization, reaction animations, periodic table, and chemical equation balancing.

## Chemistry-Specific Dependencies

```bash
npm install 3dmol                   # Molecular visualization (alternative)
npm install @react-three/drei       # 3D helpers
npm install katex react-katex       # Chemical equations in LaTeX
npm install mathjs                  # Stoichiometry calculations
npm install chroma-js               # Color interpolation for elements
```

---

## 1. Atomic Models (Bohr & Electron Cloud)

### Interactive Bohr Atom

```tsx
import { useRef, useMemo } from 'react'
import { useFrame } from '@react-three/fiber'
import * as THREE from 'three'

interface BohrAtomProps {
  element: {
    symbol: string
    atomicNumber: number
    electronConfiguration: number[]  // e.g., [2, 8, 1] for Na
  }
  showLabels?: boolean
  animationSpeed?: number
}

export const BohrAtom: React.FC<BohrAtomProps> = ({
  element,
  showLabels = true,
  animationSpeed = 1
}) => {
  const electronsRef = useRef<THREE.Group[]>([])

  // Orbital colors (K, L, M, N shells)
  const shellColors = ['#f44336', '#FF9800', '#4CAF50', '#2196F3', '#9C27B0']

  // Calculate electron positions for each shell
  const shells = useMemo(() => {
    return element.electronConfiguration.map((electronCount, shellIndex) => {
      const radius = (shellIndex + 1) * 1.5
      const electrons: { angle: number; speed: number }[] = []

      for (let i = 0; i < electronCount; i++) {
        electrons.push({
          angle: (i / electronCount) * Math.PI * 2,
          speed: (1 / (shellIndex + 1)) * animationSpeed * (Math.random() * 0.2 + 0.9)
        })
      }

      return { radius, electrons, color: shellColors[shellIndex] }
    })
  }, [element, animationSpeed])

  useFrame((_, delta) => {
    shells.forEach((shell, shellIndex) => {
      shell.electrons.forEach((electron, eIndex) => {
        electron.angle += delta * electron.speed
      })
    })
  })

  return (
    <group>
      {/* Nucleus */}
      <mesh>
        <sphereGeometry args={[0.5, 32, 32]} />
        <meshStandardMaterial
          color="#ff6b6b"
          emissive="#ff0000"
          emissiveIntensity={0.3}
        />
      </mesh>

      {/* Nucleus label */}
      {showLabels && (
        <Html position={[0, 0, 0.6]} center>
          <div style={{
            background: 'rgba(0,0,0,0.8)',
            color: 'white',
            padding: '4px 8px',
            borderRadius: '4px',
            fontSize: '14px',
            fontWeight: 'bold'
          }}>
            {element.symbol}
            <sup>{element.atomicNumber}</sup>
          </div>
        </Html>
      )}

      {/* Protons and Neutrons in nucleus */}
      <NucleusParticles
        protons={element.atomicNumber}
        neutrons={Math.round(element.atomicNumber * 1.1)} // Approximate
      />

      {/* Electron shells */}
      {shells.map((shell, shellIndex) => (
        <group key={shellIndex}>
          {/* Orbital ring */}
          <mesh rotation={[Math.PI / 2, 0, 0]}>
            <torusGeometry args={[shell.radius, 0.02, 8, 64]} />
            <meshBasicMaterial color={shell.color} transparent opacity={0.3} />
          </mesh>

          {/* Shell label */}
          {showLabels && (
            <Html position={[shell.radius + 0.3, 0, 0]} center>
              <span style={{ color: shell.color, fontSize: '12px' }}>
                {['K', 'L', 'M', 'N', 'O'][shellIndex]} ({shell.electrons.length}e⁻)
              </span>
            </Html>
          )}

          {/* Electrons */}
          {shell.electrons.map((electron, eIndex) => (
            <Electron
              key={eIndex}
              radius={shell.radius}
              angle={electron.angle}
              color={shell.color}
              speed={electron.speed}
            />
          ))}
        </group>
      ))}
    </group>
  )
}

// Individual electron with animation
const Electron: React.FC<{
  radius: number
  angle: number
  color: string
  speed: number
}> = ({ radius, angle, color, speed }) => {
  const meshRef = useRef<THREE.Mesh>(null)

  useFrame((state) => {
    if (meshRef.current) {
      const t = state.clock.elapsedTime * speed + angle
      meshRef.current.position.x = Math.cos(t) * radius
      meshRef.current.position.z = Math.sin(t) * radius
    }
  })

  return (
    <mesh ref={meshRef}>
      <sphereGeometry args={[0.15, 16, 16]} />
      <meshStandardMaterial
        color={color}
        emissive={color}
        emissiveIntensity={0.5}
      />
      {/* Electron trail */}
      <Trail
        width={0.1}
        length={8}
        color={color}
        attenuation={(t) => t * t}
      />
    </mesh>
  )
}

// Nucleus with protons and neutrons
const NucleusParticles: React.FC<{ protons: number; neutrons: number }> = ({
  protons, neutrons
}) => {
  const particles = useMemo(() => {
    const result: { position: THREE.Vector3; type: 'proton' | 'neutron' }[] = []
    const total = Math.min(protons + neutrons, 20) // Limit for visualization

    for (let i = 0; i < total; i++) {
      const phi = Math.acos(1 - 2 * (i + 0.5) / total)
      const theta = Math.PI * (1 + Math.sqrt(5)) * i
      const r = 0.3

      result.push({
        position: new THREE.Vector3(
          r * Math.sin(phi) * Math.cos(theta),
          r * Math.sin(phi) * Math.sin(theta),
          r * Math.cos(phi)
        ),
        type: i < protons ? 'proton' : 'neutron'
      })
    }
    return result
  }, [protons, neutrons])

  return (
    <group>
      {particles.map((p, i) => (
        <mesh key={i} position={p.position}>
          <sphereGeometry args={[0.12, 8, 8]} />
          <meshStandardMaterial
            color={p.type === 'proton' ? '#f44336' : '#9e9e9e'}
          />
        </mesh>
      ))}
    </group>
  )
}
```

### Electron Cloud (Probability Distribution)

```tsx
// Shader for electron probability cloud
const electronCloudShader = {
  vertexShader: `
    varying vec3 vPosition;
    void main() {
      vPosition = position;
      gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
    }
  `,
  fragmentShader: `
    uniform float uTime;
    uniform int uOrbitalType; // 0=s, 1=p, 2=d
    uniform vec3 uColor;

    varying vec3 vPosition;

    // Simplified orbital probability functions
    float sOrbital(vec3 p) {
      float r = length(p);
      return exp(-r * 2.0);
    }

    float pOrbital(vec3 p, int axis) {
      float r = length(p);
      float angular;
      if (axis == 0) angular = p.x;
      else if (axis == 1) angular = p.y;
      else angular = p.z;
      return angular * angular * exp(-r);
    }

    float dOrbital(vec3 p) {
      float r = length(p);
      float xy = p.x * p.y;
      return xy * xy * exp(-r * 0.5);
    }

    void main() {
      float density;

      if (uOrbitalType == 0) {
        density = sOrbital(vPosition);
      } else if (uOrbitalType == 1) {
        density = pOrbital(vPosition, 0) + pOrbital(vPosition, 1) + pOrbital(vPosition, 2);
      } else {
        density = dOrbital(vPosition);
      }

      // Animate
      density *= 0.8 + 0.2 * sin(uTime * 2.0 + length(vPosition) * 3.0);

      // Threshold for visibility
      if (density < 0.1) discard;

      gl_FragColor = vec4(uColor, density * 0.5);
    }
  `
}

export const ElectronCloud: React.FC<{
  orbitalType: 's' | 'p' | 'd'
  color?: string
}> = ({ orbitalType, color = '#4FC3F7' }) => {
  const materialRef = useRef<THREE.ShaderMaterial>(null)

  const orbitalIndex = { s: 0, p: 1, d: 2 }[orbitalType]

  useFrame(({ clock }) => {
    if (materialRef.current) {
      materialRef.current.uniforms.uTime.value = clock.elapsedTime
    }
  })

  return (
    <points>
      <bufferGeometry>
        {/* Generate point cloud */}
        <bufferAttribute
          attach="attributes-position"
          count={10000}
          array={useMemo(() => {
            const positions = new Float32Array(10000 * 3)
            for (let i = 0; i < 10000; i++) {
              const r = Math.random() * 3
              const theta = Math.random() * Math.PI * 2
              const phi = Math.acos(2 * Math.random() - 1)
              positions[i * 3] = r * Math.sin(phi) * Math.cos(theta)
              positions[i * 3 + 1] = r * Math.sin(phi) * Math.sin(theta)
              positions[i * 3 + 2] = r * Math.cos(phi)
            }
            return positions
          }, [])}
          itemSize={3}
        />
      </bufferGeometry>
      <shaderMaterial
        ref={materialRef}
        vertexShader={electronCloudShader.vertexShader}
        fragmentShader={electronCloudShader.fragmentShader}
        transparent
        depthWrite={false}
        uniforms={{
          uTime: { value: 0 },
          uOrbitalType: { value: orbitalIndex },
          uColor: { value: new THREE.Color(color) },
        }}
      />
    </points>
  )
}
```

---

## 2. Interactive Periodic Table

```tsx
import chroma from 'chroma-js'

interface Element {
  number: number
  symbol: string
  name: string
  nameES: string
  mass: number
  category: string
  electronConfig: string
  electronegativity?: number
  group: number
  period: number
}

const categoryColors: Record<string, string> = {
  'alkali-metal': '#ff6b6b',
  'alkaline-earth': '#ffa94d',
  'transition-metal': '#ffd43b',
  'post-transition-metal': '#69db7c',
  'metalloid': '#38d9a9',
  'nonmetal': '#4dabf7',
  'halogen': '#748ffc',
  'noble-gas': '#da77f2',
  'lanthanide': '#f783ac',
  'actinide': '#ff8787',
}

export const PeriodicTable3D: React.FC<{
  onElementClick?: (element: Element) => void
  highlightCategory?: string
  colorBy?: 'category' | 'electronegativity' | 'period'
}> = ({ onElementClick, highlightCategory, colorBy = 'category' }) => {
  const [hoveredElement, setHoveredElement] = useState<Element | null>(null)
  const [selectedElement, setSelectedElement] = useState<Element | null>(null)

  // Color scale for electronegativity
  const enScale = chroma.scale(['#4CAF50', '#ffeb3b', '#f44336']).domain([0.7, 4.0])

  const getColor = (element: Element) => {
    if (colorBy === 'electronegativity' && element.electronegativity) {
      return enScale(element.electronegativity).hex()
    }
    if (colorBy === 'period') {
      return `hsl(${element.period * 40}, 70%, 50%)`
    }
    return categoryColors[element.category] || '#666'
  }

  return (
    <group>
      {elements.map((element) => {
        const isHighlighted = !highlightCategory || element.category === highlightCategory
        const isHovered = hoveredElement?.number === element.number
        const isSelected = selectedElement?.number === element.number

        return (
          <group
            key={element.number}
            position={[
              (element.group - 9) * 1.2,
              -(element.period - 4) * 1.2,
              0
            ]}
          >
            <mesh
              onClick={() => {
                setSelectedElement(element)
                onElementClick?.(element)
              }}
              onPointerOver={() => setHoveredElement(element)}
              onPointerOut={() => setHoveredElement(null)}
            >
              <boxGeometry args={[1, 1, isSelected ? 0.5 : 0.2]} />
              <meshStandardMaterial
                color={getColor(element)}
                opacity={isHighlighted ? 1 : 0.3}
                transparent
                emissive={getColor(element)}
                emissiveIntensity={isHovered ? 0.5 : isSelected ? 0.3 : 0}
              />
            </mesh>

            {/* Element symbol */}
            <Html position={[0, 0, 0.15]} center>
              <div style={{
                width: '50px',
                textAlign: 'center',
                pointerEvents: 'none',
              }}>
                <div style={{ fontSize: '8px', color: '#666' }}>
                  {element.number}
                </div>
                <div style={{
                  fontSize: '16px',
                  fontWeight: 'bold',
                  color: isHighlighted ? '#000' : '#999'
                }}>
                  {element.symbol}
                </div>
                {isHovered && (
                  <div style={{ fontSize: '6px', color: '#666' }}>
                    {element.mass.toFixed(2)}
                  </div>
                )}
              </div>
            </Html>
          </group>
        )
      })}

      {/* Element detail panel */}
      {selectedElement && (
        <Html position={[12, 0, 0]}>
          <ElementDetailCard
            element={selectedElement}
            onClose={() => setSelectedElement(null)}
          />
        </Html>
      )}

      {/* Legend */}
      <Html position={[-12, -4, 0]}>
        <div className="periodic-legend">
          {Object.entries(categoryColors).map(([cat, color]) => (
            <div key={cat} style={{ display: 'flex', alignItems: 'center', gap: 4 }}>
              <div style={{ width: 12, height: 12, background: color, borderRadius: 2 }} />
              <span style={{ fontSize: 10 }}>{cat.replace('-', ' ')}</span>
            </div>
          ))}
        </div>
      </Html>
    </group>
  )
}

// Element detail card with electron configuration
const ElementDetailCard: React.FC<{
  element: Element
  onClose: () => void
}> = ({ element, onClose }) => (
  <div style={{
    background: 'white',
    borderRadius: 12,
    padding: 16,
    width: 250,
    boxShadow: '0 10px 40px rgba(0,0,0,0.2)'
  }}>
    <button onClick={onClose} style={{ float: 'right' }}>✕</button>

    <div style={{ textAlign: 'center', marginBottom: 16 }}>
      <div style={{ fontSize: 48, fontWeight: 'bold' }}>{element.symbol}</div>
      <div style={{ fontSize: 18 }}>{element.nameES}</div>
      <div style={{ color: '#666' }}>({element.name})</div>
    </div>

    <table style={{ width: '100%', fontSize: 12 }}>
      <tbody>
        <tr><td>Número atómico</td><td><strong>{element.number}</strong></td></tr>
        <tr><td>Masa atómica</td><td>{element.mass.toFixed(4)} u</td></tr>
        <tr><td>Config. electrónica</td><td><code>{element.electronConfig}</code></td></tr>
        {element.electronegativity && (
          <tr><td>Electronegatividad</td><td>{element.electronegativity}</td></tr>
        )}
        <tr><td>Grupo</td><td>{element.group}</td></tr>
        <tr><td>Periodo</td><td>{element.period}</td></tr>
      </tbody>
    </table>

    {/* Mini Bohr model */}
    <div style={{ height: 150, marginTop: 16 }}>
      <Canvas camera={{ position: [0, 0, 8] }}>
        <ambientLight intensity={0.5} />
        <pointLight position={[10, 10, 10]} />
        <BohrAtom element={element} showLabels={false} />
        <OrbitControls enableZoom={false} autoRotate />
      </Canvas>
    </div>
  </div>
)
```

---

## 3. Molecule Builder & Visualization

```tsx
interface Atom {
  id: string
  element: string
  position: [number, number, number]
  color: string
}

interface Bond {
  atom1: string
  atom2: string
  order: 1 | 2 | 3  // single, double, triple
}

interface MoleculeProps {
  atoms: Atom[]
  bonds: Bond[]
  showLabels?: boolean
  animate?: boolean
}

// Common molecules database
export const molecules = {
  water: {
    formula: 'H₂O',
    atoms: [
      { id: 'O1', element: 'O', position: [0, 0, 0], color: '#f44336' },
      { id: 'H1', element: 'H', position: [-0.8, 0.6, 0], color: '#e0e0e0' },
      { id: 'H2', element: 'H', position: [0.8, 0.6, 0], color: '#e0e0e0' },
    ],
    bonds: [
      { atom1: 'O1', atom2: 'H1', order: 1 },
      { atom1: 'O1', atom2: 'H2', order: 1 },
    ]
  },
  co2: {
    formula: 'CO₂',
    atoms: [
      { id: 'C1', element: 'C', position: [0, 0, 0], color: '#333' },
      { id: 'O1', element: 'O', position: [-1.2, 0, 0], color: '#f44336' },
      { id: 'O2', element: 'O', position: [1.2, 0, 0], color: '#f44336' },
    ],
    bonds: [
      { atom1: 'C1', atom2: 'O1', order: 2 },
      { atom1: 'C1', atom2: 'O2', order: 2 },
    ]
  },
  methane: {
    formula: 'CH₄',
    atoms: [
      { id: 'C1', element: 'C', position: [0, 0, 0], color: '#333' },
      { id: 'H1', element: 'H', position: [0.6, 0.6, 0.6], color: '#e0e0e0' },
      { id: 'H2', element: 'H', position: [-0.6, -0.6, 0.6], color: '#e0e0e0' },
      { id: 'H3', element: 'H', position: [-0.6, 0.6, -0.6], color: '#e0e0e0' },
      { id: 'H4', element: 'H', position: [0.6, -0.6, -0.6], color: '#e0e0e0' },
    ],
    bonds: [
      { atom1: 'C1', atom2: 'H1', order: 1 },
      { atom1: 'C1', atom2: 'H2', order: 1 },
      { atom1: 'C1', atom2: 'H3', order: 1 },
      { atom1: 'C1', atom2: 'H4', order: 1 },
    ]
  },
  benzene: {
    formula: 'C₆H₆',
    // ... hexagonal ring structure
  }
}

export const Molecule3D: React.FC<MoleculeProps> = ({
  atoms,
  bonds,
  showLabels = true,
  animate = true
}) => {
  const groupRef = useRef<THREE.Group>(null)

  // Gentle rotation
  useFrame((_, delta) => {
    if (animate && groupRef.current) {
      groupRef.current.rotation.y += delta * 0.3
    }
  })

  // Element radii (van der Waals, scaled)
  const atomRadii: Record<string, number> = {
    H: 0.25, C: 0.35, N: 0.33, O: 0.32, S: 0.40, P: 0.38, Cl: 0.36
  }

  return (
    <group ref={groupRef}>
      {/* Atoms */}
      {atoms.map((atom) => (
        <group key={atom.id} position={atom.position}>
          <mesh>
            <sphereGeometry args={[atomRadii[atom.element] || 0.3, 32, 32]} />
            <meshPhysicalMaterial
              color={atom.color}
              metalness={0.1}
              roughness={0.3}
              clearcoat={0.5}
            />
          </mesh>

          {showLabels && (
            <Html center>
              <span style={{
                fontSize: '14px',
                fontWeight: 'bold',
                color: 'white',
                textShadow: '0 0 3px black'
              }}>
                {atom.element}
              </span>
            </Html>
          )}
        </group>
      ))}

      {/* Bonds */}
      {bonds.map((bond, i) => {
        const atom1 = atoms.find(a => a.id === bond.atom1)!
        const atom2 = atoms.find(a => a.id === bond.atom2)!

        return (
          <ChemicalBond
            key={i}
            start={atom1.position}
            end={atom2.position}
            order={bond.order}
          />
        )
      })}
    </group>
  )
}

// Chemical bond visualization
const ChemicalBond: React.FC<{
  start: [number, number, number]
  end: [number, number, number]
  order: 1 | 2 | 3
}> = ({ start, end, order }) => {
  const startVec = new THREE.Vector3(...start)
  const endVec = new THREE.Vector3(...end)
  const midpoint = startVec.clone().add(endVec).multiplyScalar(0.5)
  const direction = endVec.clone().sub(startVec)
  const length = direction.length()

  // Calculate rotation to align cylinder with bond direction
  const quaternion = new THREE.Quaternion()
  quaternion.setFromUnitVectors(
    new THREE.Vector3(0, 1, 0),
    direction.normalize()
  )

  const bondOffset = order === 1 ? 0 : order === 2 ? 0.08 : 0.12

  return (
    <group position={midpoint.toArray()} quaternion={quaternion}>
      {Array.from({ length: order }).map((_, i) => {
        const offset = (i - (order - 1) / 2) * bondOffset * 2
        return (
          <mesh key={i} position={[offset, 0, 0]}>
            <cylinderGeometry args={[0.05, 0.05, length * 0.8, 8]} />
            <meshStandardMaterial color="#666" />
          </mesh>
        )
      })}
    </group>
  )
}
```

---

## 4. Chemical Equation Balancer

```tsx
import { BlockMath } from 'react-katex'

interface ChemicalEquation {
  reactants: { formula: string; coefficient: number }[]
  products: { formula: string; coefficient: number }[]
}

export const EquationBalancer: React.FC = () => {
  const [equation, setEquation] = useState<ChemicalEquation>({
    reactants: [
      { formula: 'H_2', coefficient: 2 },
      { formula: 'O_2', coefficient: 1 },
    ],
    products: [
      { formula: 'H_2O', coefficient: 2 },
    ]
  })

  const [isBalanced, setIsBalanced] = useState(true)

  // Format equation for LaTeX
  const formatEquation = (eq: ChemicalEquation): string => {
    const reactantsStr = eq.reactants
      .map(r => `${r.coefficient > 1 ? r.coefficient : ''}${r.formula}`)
      .join(' + ')

    const productsStr = eq.products
      .map(p => `${p.coefficient > 1 ? p.coefficient : ''}${p.formula}`)
      .join(' + ')

    return `${reactantsStr} \\rightarrow ${productsStr}`
  }

  // Parse formula to count atoms
  const countAtoms = (formula: string, coefficient: number): Record<string, number> => {
    const counts: Record<string, number> = {}
    // Simplified parser - would need expansion for complex formulas
    const regex = /([A-Z][a-z]?)(\d*)/g
    let match

    while ((match = regex.exec(formula)) !== null) {
      const element = match[1]
      const count = parseInt(match[2] || '1')
      counts[element] = (counts[element] || 0) + count * coefficient
    }

    return counts
  }

  // Check if balanced
  const checkBalance = () => {
    const reactantCounts: Record<string, number> = {}
    const productCounts: Record<string, number> = {}

    equation.reactants.forEach(r => {
      const atoms = countAtoms(r.formula, r.coefficient)
      Object.entries(atoms).forEach(([el, count]) => {
        reactantCounts[el] = (reactantCounts[el] || 0) + count
      })
    })

    equation.products.forEach(p => {
      const atoms = countAtoms(p.formula, p.coefficient)
      Object.entries(atoms).forEach(([el, count]) => {
        productCounts[el] = (productCounts[el] || 0) + count
      })
    })

    const balanced = JSON.stringify(reactantCounts) === JSON.stringify(productCounts)
    setIsBalanced(balanced)
    return { reactantCounts, productCounts, balanced }
  }

  const { reactantCounts, productCounts, balanced } = checkBalance()

  return (
    <div className="equation-balancer">
      <div className="equation-display" style={{
        background: balanced ? '#e8f5e9' : '#ffebee',
        padding: 20,
        borderRadius: 12,
        textAlign: 'center'
      }}>
        <BlockMath math={formatEquation(equation)} />
        <div style={{ marginTop: 10 }}>
          {balanced ? '✅ Ecuación ajustada' : '❌ Ecuación no ajustada'}
        </div>
      </div>

      {/* Atom count comparison */}
      <div className="atom-counts" style={{ display: 'flex', gap: 20, marginTop: 20 }}>
        <div style={{ flex: 1 }}>
          <h4>Reactivos</h4>
          {Object.entries(reactantCounts).map(([el, count]) => (
            <div key={el} style={{
              color: productCounts[el] === count ? '#4CAF50' : '#f44336'
            }}>
              {el}: {count}
            </div>
          ))}
        </div>
        <div style={{ flex: 1 }}>
          <h4>Productos</h4>
          {Object.entries(productCounts).map(([el, count]) => (
            <div key={el} style={{
              color: reactantCounts[el] === count ? '#4CAF50' : '#f44336'
            }}>
              {el}: {count}
            </div>
          ))}
        </div>
      </div>

      {/* Coefficient adjusters */}
      <div className="coefficients" style={{ marginTop: 20 }}>
        <h4>Ajustar coeficientes:</h4>
        <div style={{ display: 'flex', gap: 10, flexWrap: 'wrap' }}>
          {equation.reactants.map((r, i) => (
            <div key={`r-${i}`} className="coef-control">
              <label>{r.formula}</label>
              <input
                type="number"
                min={1}
                max={10}
                value={r.coefficient}
                onChange={(e) => {
                  const newReactants = [...equation.reactants]
                  newReactants[i].coefficient = parseInt(e.target.value) || 1
                  setEquation({ ...equation, reactants: newReactants })
                }}
              />
            </div>
          ))}
          <span style={{ alignSelf: 'center' }}>→</span>
          {equation.products.map((p, i) => (
            <div key={`p-${i}`} className="coef-control">
              <label>{p.formula}</label>
              <input
                type="number"
                min={1}
                max={10}
                value={p.coefficient}
                onChange={(e) => {
                  const newProducts = [...equation.products]
                  newProducts[i].coefficient = parseInt(e.target.value) || 1
                  setEquation({ ...equation, products: newProducts })
                }}
              />
            </div>
          ))}
        </div>
      </div>
    </div>
  )
}
```

---

## 5. Reaction Animation

```tsx
export const ReactionAnimation: React.FC<{
  reaction: 'combustion' | 'synthesis' | 'decomposition' | 'neutralization'
}> = ({ reaction }) => {
  const [phase, setPhase] = useState<'start' | 'reacting' | 'complete'>('start')

  const reactions = {
    combustion: {
      title: 'Combustión del Metano',
      equation: 'CH_4 + 2O_2 \\rightarrow CO_2 + 2H_2O',
      reactants: [molecules.methane, molecules.oxygen, molecules.oxygen],
      products: [molecules.co2, molecules.water, molecules.water],
    },
    // ... more reactions
  }

  const currentReaction = reactions[reaction]

  return (
    <div className="reaction-animation">
      <h3>{currentReaction.title}</h3>
      <BlockMath math={currentReaction.equation} />

      <div style={{ height: 400 }}>
        <Canvas camera={{ position: [0, 0, 10] }}>
          <ambientLight intensity={0.5} />
          <pointLight position={[10, 10, 10]} />

          <AnimatePresence>
            {phase === 'start' && (
              <group>
                {/* Show reactants separated */}
                <Molecule3D {...currentReaction.reactants[0]} />
              </group>
            )}

            {phase === 'reacting' && (
              <group>
                {/* Collision animation */}
                <ReactionCollision molecules={currentReaction.reactants} />
              </group>
            )}

            {phase === 'complete' && (
              <group>
                {/* Show products */}
                {currentReaction.products.map((mol, i) => (
                  <group key={i} position={[(i - 1) * 3, 0, 0]}>
                    <Molecule3D {...mol} />
                  </group>
                ))}
              </group>
            )}
          </AnimatePresence>

          <OrbitControls />
        </Canvas>
      </div>

      <div className="controls">
        <button onClick={() => setPhase('start')}>Inicio</button>
        <button onClick={() => setPhase('reacting')}>▶ Reaccionar</button>
        <button onClick={() => setPhase('complete')}>Productos</button>
      </div>
    </div>
  )
}
```

---

## 6. Chemistry AI Tutor Prompt

```tsx
const CHEMISTRY_TUTOR_PROMPT = `Eres un profesor experto en Química del currículo español (ESO y Bachillerato).

ESPECIALIDADES:
- Estructura atómica y tabla periódica
- Enlace químico (iónico, covalente, metálico)
- Formulación y nomenclatura (IUPAC)
- Reacciones químicas y estequiometría
- Química orgánica
- Equilibrio químico
- Ácidos, bases y pH

FORMATO DE RESPUESTAS:
1. Usa notación química correcta: H₂O, CO₂, Na⁺, Cl⁻
2. Para ecuaciones usa LaTeX: $2H_2 + O_2 \\rightarrow 2H_2O$
3. En estequiometría muestra:
   - Ecuación ajustada
   - Relación molar
   - Factor de conversión
   - Cálculo paso a paso
4. Explica el "porqué" de las reacciones (energía, electrones)
5. Relaciona con la vida cotidiana

EJEMPLO - Ajuste de ecuación:
"Para ajustar $Fe + O_2 \\rightarrow Fe_2O_3$:

1. **Contamos átomos:**
   - Reactivos: 1 Fe, 2 O
   - Productos: 2 Fe, 3 O

2. **Ajustamos Fe:** ponemos 2 Fe a la izquierda
   $2Fe + O_2 \\rightarrow Fe_2O_3$

3. **Ajustamos O:** necesitamos 3 O, ponemos 3/2 O₂
   $2Fe + \\frac{3}{2}O_2 \\rightarrow Fe_2O_3$

4. **Eliminamos fracciones:** multiplicamos todo por 2
   $$4Fe + 3O_2 \\rightarrow 2Fe_2O_3$$

✅ Ahora tenemos: 4 Fe y 6 O a cada lado"
`

<AIChatbot
  systemPrompt={CHEMISTRY_TUTOR_PROMPT}
  title="Profesor de Química"
  suggestedQuestions={[
    '¿Cómo ajusto una ecuación química?',
    'Explica la diferencia entre enlace iónico y covalente',
    '¿Qué es el número de oxidación?',
    '¿Cómo calculo la masa molar?',
  ]}
/>
```
