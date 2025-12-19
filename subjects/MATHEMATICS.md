# Mathematics Subject Specialization

Interactive geometry, function graphing, algebraic manipulation, calculus visualization, and step-by-step problem solving.

## Mathematics-Specific Dependencies

```bash
npm install mafs                    # React math visualization library
npm install katex react-katex       # LaTeX rendering
npm install mathjs                  # Symbolic math & calculations
npm install function-plot          # Function graphing (optional)
npm install @react-three/drei      # 3D geometry
npm install nerdamer                # Symbolic algebra (optional)
```

---

## 1. Interactive Geometry (2D & 3D)

### Euclidean Geometry Canvas

```tsx
import { Mafs, Coordinates, Point, Line, Circle, Polygon, useMovablePoint, Text } from 'mafs'
import 'mafs/core.css'

// Interactive triangle with measurements
export const InteractiveTriangle: React.FC = () => {
  // Movable vertices
  const A = useMovablePoint([0, 0])
  const B = useMovablePoint([4, 0])
  const C = useMovablePoint([2, 3])

  // Calculate measurements
  const sideAB = Math.sqrt((B.x - A.x) ** 2 + (B.y - A.y) ** 2)
  const sideBC = Math.sqrt((C.x - B.x) ** 2 + (C.y - B.y) ** 2)
  const sideCA = Math.sqrt((A.x - C.x) ** 2 + (A.y - C.y) ** 2)
  const perimeter = sideAB + sideBC + sideCA

  // Heron's formula for area
  const s = perimeter / 2
  const area = Math.sqrt(s * (s - sideAB) * (s - sideBC) * (s - sideCA))

  // Angles using dot product
  const angleA = calculateAngle(C.point, A.point, B.point)
  const angleB = calculateAngle(A.point, B.point, C.point)
  const angleC = calculateAngle(B.point, C.point, A.point)

  return (
    <div>
      <Mafs viewBox={{ x: [-2, 6], y: [-2, 5] }}>
        <Coordinates.Cartesian />

        {/* Triangle */}
        <Polygon
          points={[A.point, B.point, C.point]}
          color="#4CAF50"
          fillOpacity={0.2}
        />

        {/* Vertices */}
        {A.element}
        {B.element}
        {C.element}

        {/* Labels */}
        <Text x={A.x - 0.3} y={A.y - 0.3}>A</Text>
        <Text x={B.x + 0.3} y={B.y - 0.3}>B</Text>
        <Text x={C.x} y={C.y + 0.3}>C</Text>

        {/* Angle arcs */}
        <AngleArc center={A.point} angle={angleA} start={getAngleStart(C.point, A.point)} />
        <AngleArc center={B.point} angle={angleB} start={getAngleStart(A.point, B.point)} />
        <AngleArc center={C.point} angle={angleC} start={getAngleStart(B.point, C.point)} />

        {/* Side lengths */}
        <MidpointLabel p1={A.point} p2={B.point} text={sideAB.toFixed(2)} />
        <MidpointLabel p1={B.point} p2={C.point} text={sideBC.toFixed(2)} />
        <MidpointLabel p1={C.point} p2={A.point} text={sideCA.toFixed(2)} />
      </Mafs>

      {/* Measurements panel */}
      <div className="measurements">
        <h4>Medidas del triángulo</h4>
        <table>
          <tbody>
            <tr><td>Lado AB:</td><td>{sideAB.toFixed(2)} u</td></tr>
            <tr><td>Lado BC:</td><td>{sideBC.toFixed(2)} u</td></tr>
            <tr><td>Lado CA:</td><td>{sideCA.toFixed(2)} u</td></tr>
            <tr><td>Perímetro:</td><td>{perimeter.toFixed(2)} u</td></tr>
            <tr><td>Área:</td><td>{area.toFixed(2)} u²</td></tr>
            <tr><td>Ángulo A:</td><td>{(angleA * 180 / Math.PI).toFixed(1)}°</td></tr>
            <tr><td>Ángulo B:</td><td>{(angleB * 180 / Math.PI).toFixed(1)}°</td></tr>
            <tr><td>Ángulo C:</td><td>{(angleC * 180 / Math.PI).toFixed(1)}°</td></tr>
          </tbody>
        </table>

        {/* Triangle classification */}
        <div className="classification">
          <strong>Clasificación:</strong>
          <p>Por lados: {classifyBySides(sideAB, sideBC, sideCA)}</p>
          <p>Por ángulos: {classifyByAngles(angleA, angleB, angleC)}</p>
        </div>
      </div>
    </div>
  )
}

// Helper functions
const calculateAngle = (
  p1: [number, number],
  vertex: [number, number],
  p2: [number, number]
): number => {
  const v1 = [p1[0] - vertex[0], p1[1] - vertex[1]]
  const v2 = [p2[0] - vertex[0], p2[1] - vertex[1]]
  const dot = v1[0] * v2[0] + v1[1] * v2[1]
  const mag1 = Math.sqrt(v1[0] ** 2 + v1[1] ** 2)
  const mag2 = Math.sqrt(v2[0] ** 2 + v2[1] ** 2)
  return Math.acos(dot / (mag1 * mag2))
}

const classifyBySides = (a: number, b: number, c: number): string => {
  const tolerance = 0.01
  if (Math.abs(a - b) < tolerance && Math.abs(b - c) < tolerance) return 'Equilátero'
  if (Math.abs(a - b) < tolerance || Math.abs(b - c) < tolerance || Math.abs(a - c) < tolerance) return 'Isósceles'
  return 'Escaleno'
}

const classifyByAngles = (a: number, b: number, c: number): string => {
  const rightAngle = Math.PI / 2
  const tolerance = 0.01
  if ([a, b, c].some(angle => Math.abs(angle - rightAngle) < tolerance)) return 'Rectángulo'
  if ([a, b, c].every(angle => angle < rightAngle)) return 'Acutángulo'
  return 'Obtusángulo'
}
```

### Pythagorean Theorem Visualization

```tsx
export const PythagoreanTheorem: React.FC = () => {
  const [a, setA] = useState(3)
  const [b, setB] = useState(4)
  const c = Math.sqrt(a * a + b * b)

  return (
    <div className="pythagorean">
      <Mafs viewBox={{ x: [-1, 10], y: [-1, 10] }}>
        <Coordinates.Cartesian />

        {/* Right triangle */}
        <Polygon
          points={[[0, 0], [a, 0], [a, b]]}
          color="#2196F3"
          fillOpacity={0.3}
        />

        {/* Square on side a */}
        <Polygon
          points={[[0, 0], [0, -a], [a, -a], [a, 0]]}
          color="#f44336"
          fillOpacity={0.3}
        />
        <Text x={a/2} y={-a/2}>a² = {(a*a).toFixed(1)}</Text>

        {/* Square on side b */}
        <Polygon
          points={[[a, 0], [a + b, 0], [a + b, b], [a, b]]}
          color="#4CAF50"
          fillOpacity={0.3}
        />
        <Text x={a + b/2} y={b/2}>b² = {(b*b).toFixed(1)}</Text>

        {/* Square on hypotenuse (rotated) */}
        <HypotenuseSquare a={a} b={b} c={c} />

        {/* Right angle marker */}
        <Polygon
          points={[[a, 0], [a, 0.3], [a - 0.3, 0.3], [a - 0.3, 0]]}
          color="#666"
          fillOpacity={0.5}
        />
      </Mafs>

      {/* Controls */}
      <div className="controls">
        <label>
          a = {a}
          <input type="range" min={1} max={6} step={0.5} value={a}
            onChange={e => setA(parseFloat(e.target.value))} />
        </label>
        <label>
          b = {b}
          <input type="range" min={1} max={6} step={0.5} value={b}
            onChange={e => setB(parseFloat(e.target.value))} />
        </label>
      </div>

      {/* Formula */}
      <div className="formula">
        <BlockMath math={`a^2 + b^2 = c^2`} />
        <BlockMath math={`${a}^2 + ${b}^2 = ${c.toFixed(2)}^2`} />
        <BlockMath math={`${a*a} + ${b*b} = ${(c*c).toFixed(1)}`} />
      </div>
    </div>
  )
}
```

### 3D Polyhedra

```tsx
import { useRef } from 'react'
import { useFrame } from '@react-three/fiber'
import { Html } from '@react-three/drei'

// Platonic solids with formulas
const platonicSolids = {
  tetrahedron: {
    name: 'Tetraedro',
    faces: 4,
    vertices: 4,
    edges: 6,
    geometry: <tetrahedronGeometry args={[1, 0]} />,
    volumeFormula: 'V = \\frac{a^3\\sqrt{2}}{12}',
    areaFormula: 'A = a^2\\sqrt{3}',
  },
  cube: {
    name: 'Cubo',
    faces: 6,
    vertices: 8,
    edges: 12,
    geometry: <boxGeometry args={[1, 1, 1]} />,
    volumeFormula: 'V = a^3',
    areaFormula: 'A = 6a^2',
  },
  octahedron: {
    name: 'Octaedro',
    faces: 8,
    vertices: 6,
    edges: 12,
    geometry: <octahedronGeometry args={[1, 0]} />,
    volumeFormula: 'V = \\frac{a^3\\sqrt{2}}{3}',
    areaFormula: 'A = 2a^2\\sqrt{3}',
  },
  dodecahedron: {
    name: 'Dodecaedro',
    faces: 12,
    vertices: 20,
    edges: 30,
    geometry: <dodecahedronGeometry args={[1, 0]} />,
    volumeFormula: 'V = \\frac{a^3(15+7\\sqrt{5})}{4}',
    areaFormula: 'A = 3a^2\\sqrt{25+10\\sqrt{5}}',
  },
  icosahedron: {
    name: 'Icosaedro',
    faces: 20,
    vertices: 12,
    edges: 30,
    geometry: <icosahedronGeometry args={[1, 0]} />,
    volumeFormula: 'V = \\frac{5a^3(3+\\sqrt{5})}{12}',
    areaFormula: 'A = 5a^2\\sqrt{3}',
  },
}

export const PlatonicSolid3D: React.FC<{
  type: keyof typeof platonicSolids
  edgeLength?: number
  showWireframe?: boolean
  autoRotate?: boolean
}> = ({ type, edgeLength = 2, showWireframe = true, autoRotate = true }) => {
  const meshRef = useRef<THREE.Mesh>(null)
  const solid = platonicSolids[type]

  useFrame((_, delta) => {
    if (autoRotate && meshRef.current) {
      meshRef.current.rotation.y += delta * 0.5
      meshRef.current.rotation.x += delta * 0.2
    }
  })

  // Calculate volume and area
  const a = edgeLength
  const volume = calculateVolume(type, a)
  const area = calculateArea(type, a)

  return (
    <group>
      <mesh ref={meshRef} scale={edgeLength}>
        {solid.geometry}
        <meshStandardMaterial
          color="#4CAF50"
          transparent
          opacity={0.7}
          side={THREE.DoubleSide}
        />
      </mesh>

      {showWireframe && (
        <mesh scale={edgeLength}>
          {solid.geometry}
          <meshBasicMaterial wireframe color="#333" />
        </mesh>
      )}

      <Html position={[3, 0, 0]}>
        <div className="solid-info" style={{
          background: 'white',
          padding: 16,
          borderRadius: 8,
          boxShadow: '0 4px 20px rgba(0,0,0,0.15)',
          width: 200
        }}>
          <h3>{solid.name}</h3>
          <table>
            <tbody>
              <tr><td>Caras:</td><td>{solid.faces}</td></tr>
              <tr><td>Vértices:</td><td>{solid.vertices}</td></tr>
              <tr><td>Aristas:</td><td>{solid.edges}</td></tr>
            </tbody>
          </table>

          <div className="euler">
            <InlineMath math={`V - A + C = ${solid.vertices} - ${solid.edges} + ${solid.faces} = 2`} />
            <small>(Fórmula de Euler)</small>
          </div>

          <div className="formulas">
            <BlockMath math={solid.volumeFormula} />
            <p>V = {volume.toFixed(2)} u³</p>
            <BlockMath math={solid.areaFormula} />
            <p>A = {area.toFixed(2)} u²</p>
          </div>
        </div>
      </Html>
    </group>
  )
}
```

---

## 2. Function Graphing & Calculus

### Interactive Function Plotter

```tsx
import { Mafs, Coordinates, Plot, useMovablePoint, Line, Text } from 'mafs'
import * as math from 'mathjs'

export const FunctionPlotter: React.FC = () => {
  const [expression, setExpression] = useState('x^2')
  const [domain, setDomain] = useState<[number, number]>([-5, 5])
  const [showDerivative, setShowDerivative] = useState(false)
  const [showIntegral, setShowIntegral] = useState(false)
  const [showTangent, setShowTangent] = useState(false)

  // Tangent point
  const tangentPoint = useMovablePoint([1, 1])

  // Parse and evaluate function
  const f = (x: number): number => {
    try {
      return math.evaluate(expression, { x })
    } catch {
      return NaN
    }
  }

  // Numerical derivative
  const df = (x: number): number => {
    const h = 0.0001
    return (f(x + h) - f(x - h)) / (2 * h)
  }

  // Tangent line at point
  const tangentSlope = df(tangentPoint.x)
  const tangentY = f(tangentPoint.x)
  const tangentLine = (x: number) =>
    tangentY + tangentSlope * (x - tangentPoint.x)

  // Symbolic derivative (using mathjs)
  const derivativeExpr = useMemo(() => {
    try {
      const node = math.parse(expression)
      const derivative = math.derivative(node, 'x')
      return derivative.toString()
    } catch {
      return 'Error'
    }
  }, [expression])

  return (
    <div className="function-plotter">
      {/* Input */}
      <div className="input-section">
        <label>
          f(x) =
          <input
            type="text"
            value={expression}
            onChange={e => setExpression(e.target.value)}
            placeholder="x^2, sin(x), exp(x), etc."
          />
        </label>

        <div className="toggles">
          <label>
            <input type="checkbox" checked={showDerivative}
              onChange={e => setShowDerivative(e.target.checked)} />
            Mostrar f'(x)
          </label>
          <label>
            <input type="checkbox" checked={showTangent}
              onChange={e => setShowTangent(e.target.checked)} />
            Recta tangente
          </label>
          <label>
            <input type="checkbox" checked={showIntegral}
              onChange={e => setShowIntegral(e.target.checked)} />
            Área bajo curva
          </label>
        </div>
      </div>

      {/* Graph */}
      <Mafs viewBox={{ x: domain, y: [-5, 5] }}>
        <Coordinates.Cartesian />

        {/* Main function */}
        <Plot.OfX y={f} color="#2196F3" weight={3} />

        {/* Derivative */}
        {showDerivative && (
          <Plot.OfX y={df} color="#f44336" weight={2} style="dashed" />
        )}

        {/* Integral shading */}
        {showIntegral && (
          <Plot.Inequality
            y={{ "<=": f, ">=": () => 0 }}
            color="#4CAF50"
          />
        )}

        {/* Tangent line */}
        {showTangent && (
          <>
            {tangentPoint.element}
            <Plot.OfX y={tangentLine} color="#FF9800" />
            <Point x={tangentPoint.x} y={f(tangentPoint.x)} color="#FF9800" />
          </>
        )}
      </Mafs>

      {/* Analysis */}
      <div className="analysis">
        <h4>Análisis</h4>
        <div className="formulas">
          <div>
            <strong>Función:</strong>
            <BlockMath math={`f(x) = ${expression}`} />
          </div>
          {showDerivative && (
            <div>
              <strong>Derivada:</strong>
              <BlockMath math={`f'(x) = ${derivativeExpr}`} />
            </div>
          )}
          {showTangent && (
            <div>
              <strong>Recta tangente en x = {tangentPoint.x.toFixed(2)}:</strong>
              <BlockMath math={`y = ${tangentY.toFixed(2)} + ${tangentSlope.toFixed(2)}(x - ${tangentPoint.x.toFixed(2)})`} />
            </div>
          )}
        </div>
      </div>
    </div>
  )
}
```

### Limit Visualization

```tsx
export const LimitVisualization: React.FC<{
  expression: string
  point: number
}> = ({ expression, point }) => {
  const [epsilon, setEpsilon] = useState(0.5)

  const f = (x: number) => math.evaluate(expression, { x })

  // Calculate limit (numerical approach)
  const leftLimit = f(point - 0.0001)
  const rightLimit = f(point + 0.0001)
  const limitExists = Math.abs(leftLimit - rightLimit) < 0.001
  const limitValue = limitExists ? (leftLimit + rightLimit) / 2 : NaN

  return (
    <div className="limit-viz">
      <BlockMath math={`\\lim_{x \\to ${point}} ${expression}`} />

      <Mafs viewBox={{ x: [point - 3, point + 3], y: [-2, 5] }}>
        <Coordinates.Cartesian />

        {/* Function */}
        <Plot.OfX y={f} color="#2196F3" />

        {/* Vertical line at point */}
        <Line.Segment
          point1={[point, -10]}
          point2={[point, 10]}
          color="#666"
          style="dashed"
        />

        {/* Epsilon neighborhood */}
        <Plot.Inequality
          x={{ ">=": point - epsilon, "<=": point + epsilon }}
          y={{ ">=": limitValue - epsilon, "<=": limitValue + epsilon }}
          color="#4CAF50"
        />

        {/* Approaching arrows */}
        <Arrow from={[point - 1.5, f(point - 1.5)]} to={[point - 0.3, leftLimit]} />
        <Arrow from={[point + 1.5, f(point + 1.5)]} to={[point + 0.3, rightLimit]} />

        {/* Limit point (if exists) */}
        {limitExists && (
          <Point x={point} y={limitValue} color="#f44336" />
        )}
      </Mafs>

      <div className="controls">
        <label>
          ε = {epsilon.toFixed(2)}
          <input type="range" min={0.1} max={2} step={0.1} value={epsilon}
            onChange={e => setEpsilon(parseFloat(e.target.value))} />
        </label>
      </div>

      <div className="result">
        {limitExists ? (
          <BlockMath math={`\\lim_{x \\to ${point}} ${expression} = ${limitValue.toFixed(4)}`} />
        ) : (
          <p style={{ color: '#f44336' }}>
            El límite no existe (límites laterales diferentes)
          </p>
        )}
      </div>
    </div>
  )
}
```

---

## 3. Algebraic Step-by-Step Solver

```tsx
import * as math from 'mathjs'

interface SolutionStep {
  description: string
  expression: string
  latex: string
}

export const EquationSolver: React.FC = () => {
  const [equation, setEquation] = useState('2x + 5 = 13')
  const [steps, setSteps] = useState<SolutionStep[]>([])
  const [solution, setSolution] = useState<string | null>(null)

  const solve = () => {
    const newSteps: SolutionStep[] = []

    try {
      // Parse equation
      const [left, right] = equation.split('=').map(s => s.trim())

      newSteps.push({
        description: 'Ecuación original',
        expression: equation,
        latex: `${left} = ${right}`
      })

      // Move constants to right side
      // This is a simplified solver - would need expansion for full algebra

      // For 2x + 5 = 13
      // Step 1: Subtract 5 from both sides
      const constant = extractConstant(left)
      if (constant !== 0) {
        const newRight = `${right} - (${constant})`
        const newRightEval = math.evaluate(newRight)
        newSteps.push({
          description: `Restamos ${constant} en ambos lados`,
          expression: `${left} - ${constant} = ${right} - ${constant}`,
          latex: `${left.replace(`+ ${constant}`, '').replace(`- ${Math.abs(constant)}`, '')} = ${newRightEval}`
        })
      }

      // Step 2: Divide by coefficient
      const coefficient = extractCoefficient(left)
      if (coefficient !== 1) {
        const finalValue = math.evaluate(`${right} - ${constant}`) / coefficient
        newSteps.push({
          description: `Dividimos entre ${coefficient} en ambos lados`,
          expression: `x = ${finalValue}`,
          latex: `x = ${finalValue}`
        })
      }

      setSteps(newSteps)
      setSolution(newSteps[newSteps.length - 1]?.latex || null)

    } catch (error) {
      setSteps([{ description: 'Error al resolver', expression: '', latex: 'Error' }])
      setSolution(null)
    }
  }

  return (
    <div className="equation-solver">
      <div className="input">
        <input
          type="text"
          value={equation}
          onChange={e => setEquation(e.target.value)}
          placeholder="2x + 5 = 13"
        />
        <button onClick={solve}>Resolver paso a paso</button>
      </div>

      <div className="steps">
        {steps.map((step, i) => (
          <motion.div
            key={i}
            className="step"
            initial={{ opacity: 0, x: -20 }}
            animate={{ opacity: 1, x: 0 }}
            transition={{ delay: i * 0.3 }}
          >
            <div className="step-number">{i + 1}</div>
            <div className="step-content">
              <p className="description">{step.description}</p>
              <BlockMath math={step.latex} />
            </div>
          </motion.div>
        ))}
      </div>

      {solution && (
        <div className="solution">
          <h4>Solución:</h4>
          <BlockMath math={solution} />
        </div>
      )}
    </div>
  )
}
```

---

## 4. Statistical Visualization

```tsx
import { Mafs, Coordinates, Point, Line, Plot } from 'mafs'

interface DataPoint {
  x: number
  y: number
}

export const StatisticsVisualization: React.FC<{
  data: DataPoint[]
}> = ({ data }) => {
  // Calculate statistics
  const n = data.length
  const sumX = data.reduce((acc, p) => acc + p.x, 0)
  const sumY = data.reduce((acc, p) => acc + p.y, 0)
  const sumXY = data.reduce((acc, p) => acc + p.x * p.y, 0)
  const sumX2 = data.reduce((acc, p) => acc + p.x * p.x, 0)
  const sumY2 = data.reduce((acc, p) => acc + p.y * p.y, 0)

  const meanX = sumX / n
  const meanY = sumY / n

  // Standard deviations
  const stdX = Math.sqrt(data.reduce((acc, p) => acc + (p.x - meanX) ** 2, 0) / n)
  const stdY = Math.sqrt(data.reduce((acc, p) => acc + (p.y - meanY) ** 2, 0) / n)

  // Linear regression: y = mx + b
  const m = (n * sumXY - sumX * sumY) / (n * sumX2 - sumX * sumX)
  const b = (sumY - m * sumX) / n

  // Correlation coefficient
  const r = (n * sumXY - sumX * sumY) /
    Math.sqrt((n * sumX2 - sumX * sumX) * (n * sumY2 - sumY * sumY))

  const regressionLine = (x: number) => m * x + b

  return (
    <div className="statistics-viz">
      <Mafs viewBox={{ x: [-1, 12], y: [-1, 12] }}>
        <Coordinates.Cartesian />

        {/* Data points */}
        {data.map((point, i) => (
          <Point key={i} x={point.x} y={point.y} color="#2196F3" />
        ))}

        {/* Mean point */}
        <Point x={meanX} y={meanY} color="#f44336" />
        <Text x={meanX + 0.3} y={meanY + 0.3} color="#f44336">
          (x̄, ȳ)
        </Text>

        {/* Regression line */}
        <Plot.OfX y={regressionLine} color="#4CAF50" />

        {/* Residuals */}
        {data.map((point, i) => (
          <Line.Segment
            key={i}
            point1={[point.x, point.y]}
            point2={[point.x, regressionLine(point.x)]}
            color="#FF9800"
            opacity={0.5}
          />
        ))}
      </Mafs>

      <div className="stats-table">
        <h4>Estadísticas descriptivas</h4>
        <table>
          <tbody>
            <tr><td>n</td><td>{n}</td></tr>
            <tr><td>Media X (x̄)</td><td>{meanX.toFixed(2)}</td></tr>
            <tr><td>Media Y (ȳ)</td><td>{meanY.toFixed(2)}</td></tr>
            <tr><td>Desv. típica X (σx)</td><td>{stdX.toFixed(2)}</td></tr>
            <tr><td>Desv. típica Y (σy)</td><td>{stdY.toFixed(2)}</td></tr>
          </tbody>
        </table>

        <h4>Regresión lineal</h4>
        <BlockMath math={`y = ${m.toFixed(3)}x + ${b.toFixed(3)}`} />

        <h4>Correlación</h4>
        <BlockMath math={`r = ${r.toFixed(4)}`} />
        <p>
          {Math.abs(r) > 0.8 ? 'Correlación fuerte' :
           Math.abs(r) > 0.5 ? 'Correlación moderada' : 'Correlación débil'}
          {r > 0 ? ' positiva' : ' negativa'}
        </p>
      </div>
    </div>
  )
}
```

---

## 5. Math AI Tutor Prompt

```tsx
const MATH_TUTOR_PROMPT = `Eres un profesor experto en Matemáticas del currículo español (ESO y Bachillerato).

ESPECIALIDADES:
- Aritmética y álgebra
- Geometría plana y del espacio
- Funciones y análisis
- Estadística y probabilidad
- Trigonometría
- Cálculo (límites, derivadas, integrales)

FORMATO DE RESPUESTAS:
1. SIEMPRE usa LaTeX para fórmulas y ecuaciones:
   - Inline: $x^2 + 2x + 1$
   - Block: $$\\frac{-b \\pm \\sqrt{b^2-4ac}}{2a}$$

2. Para resolver problemas SIEMPRE sigue esta estructura:
   **Datos:**
   - Lista de lo que sabemos

   **Incógnita:**
   - Lo que buscamos

   **Planteamiento:**
   - Ecuación o fórmula a usar

   **Resolución:**
   - Paso 1: [explicación] → resultado
   - Paso 2: [explicación] → resultado
   - ...

   **Solución:**
   - Respuesta final con unidades

3. Usa dibujos ASCII cuando ayuden:
   \`\`\`
       C
      /|\\
     / | \\
    /  |  \\
   A---+---B
   \`\`\`

4. Relaciona con la vida real cuando sea posible

EJEMPLO - Ecuación de segundo grado:
"Para resolver $x^2 - 5x + 6 = 0$:

**Identificamos coeficientes:**
- a = 1, b = -5, c = 6

**Aplicamos la fórmula:**
$$x = \\frac{-b \\pm \\sqrt{b^2-4ac}}{2a}$$

**Sustituimos:**
$$x = \\frac{-(-5) \\pm \\sqrt{(-5)^2-4(1)(6)}}{2(1)}$$

$$x = \\frac{5 \\pm \\sqrt{25-24}}{2} = \\frac{5 \\pm 1}{2}$$

**Soluciones:**
$$x_1 = \\frac{5+1}{2} = 3$$
$$x_2 = \\frac{5-1}{2} = 2$$

✅ Las soluciones son **x = 2** y **x = 3**"
`

<AIChatbot
  systemPrompt={MATH_TUTOR_PROMPT}
  title="Profesor de Matemáticas"
  suggestedQuestions={[
    '¿Cómo resuelvo una ecuación de segundo grado?',
    'Explícame el teorema de Pitágoras',
    '¿Qué es una función derivada?',
    '¿Cómo calculo la probabilidad de un suceso?',
  ]}
/>
```

---

## 6. Common Math Formulas Reference

```tsx
export const MathFormulasReference = {
  algebra: {
    quadraticFormula: 'x = \\frac{-b \\pm \\sqrt{b^2-4ac}}{2a}',
    binomialSquare: '(a \\pm b)^2 = a^2 \\pm 2ab + b^2',
    differenceOfSquares: 'a^2 - b^2 = (a+b)(a-b)',
    binomialCube: '(a \\pm b)^3 = a^3 \\pm 3a^2b + 3ab^2 \\pm b^3',
  },

  geometry: {
    triangleArea: 'A = \\frac{b \\cdot h}{2}',
    circleArea: 'A = \\pi r^2',
    circleCircumference: 'C = 2\\pi r',
    pythagorean: 'a^2 + b^2 = c^2',
    sphereVolume: 'V = \\frac{4}{3}\\pi r^3',
    sphereSurface: 'A = 4\\pi r^2',
  },

  trigonometry: {
    sinDefinition: '\\sin\\theta = \\frac{\\text{opuesto}}{\\text{hipotenusa}}',
    cosDefinition: '\\cos\\theta = \\frac{\\text{adyacente}}{\\text{hipotenusa}}',
    tanDefinition: '\\tan\\theta = \\frac{\\sin\\theta}{\\cos\\theta}',
    pythagoreanIdentity: '\\sin^2\\theta + \\cos^2\\theta = 1',
    lawOfSines: '\\frac{a}{\\sin A} = \\frac{b}{\\sin B} = \\frac{c}{\\sin C}',
    lawOfCosines: 'c^2 = a^2 + b^2 - 2ab\\cos C',
  },

  calculus: {
    limitDefinition: '\\lim_{x \\to a} f(x) = L',
    derivativeDefinition: "f'(x) = \\lim_{h \\to 0} \\frac{f(x+h) - f(x)}{h}",
    powerRule: '\\frac{d}{dx}x^n = nx^{n-1}',
    chainRule: "\\frac{d}{dx}f(g(x)) = f'(g(x)) \\cdot g'(x)",
    integralPower: '\\int x^n dx = \\frac{x^{n+1}}{n+1} + C',
    fundamentalTheorem: '\\int_a^b f(x)dx = F(b) - F(a)',
  },

  statistics: {
    mean: '\\bar{x} = \\frac{1}{n}\\sum_{i=1}^{n} x_i',
    variance: '\\sigma^2 = \\frac{1}{n}\\sum_{i=1}^{n}(x_i - \\bar{x})^2',
    standardDeviation: '\\sigma = \\sqrt{\\sigma^2}',
    correlation: 'r = \\frac{\\sum(x_i-\\bar{x})(y_i-\\bar{y})}{\\sqrt{\\sum(x_i-\\bar{x})^2\\sum(y_i-\\bar{y})^2}}',
  },

  probability: {
    basicProbability: 'P(A) = \\frac{\\text{casos favorables}}{\\text{casos posibles}}',
    complement: "P(A') = 1 - P(A)",
    union: 'P(A \\cup B) = P(A) + P(B) - P(A \\cap B)',
    conditional: 'P(A|B) = \\frac{P(A \\cap B)}{P(B)}',
    bayes: 'P(A|B) = \\frac{P(B|A) \\cdot P(A)}{P(B)}',
  }
}
```
