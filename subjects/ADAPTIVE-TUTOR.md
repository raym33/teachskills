# Tutor IA Adaptativo

Sistema de tutoría inteligente que analiza errores del estudiante, proporciona explicaciones personalizadas y se integra con los ejercicios gamificados.

---

## Arquitectura del Sistema

```
┌─────────────────────────────────────────────────────────────┐
│                    FLUJO DE APRENDIZAJE                      │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Ejercicio → Error → Análisis IA → Explicación → Refuerzo   │
│      ↑                                              │        │
│      └──────────────── Ejercicio adaptado ──────────┘        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Store de Errores y Patrones (Zustand)

```tsx
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface ErrorRecord {
  id: string
  timestamp: Date
  questionId: string
  topicId: string
  concept: string // Concepto relacionado (ej: "densidad", "ecuaciones")
  questionText: string
  correctAnswer: string
  userAnswer: string
  errorType: ErrorType
  timeSpentMs: number
  hintsUsed: boolean
  aiExplanationShown: boolean
  understoodAfter: boolean | null // null = no verificado aún
}

type ErrorType =
  | 'conceptual'      // No entiende el concepto
  | 'calculation'     // Error de cálculo
  | 'unit_conversion' // Error de conversión de unidades
  | 'formula'         // Fórmula incorrecta
  | 'reading'         // No leyó bien la pregunta
  | 'careless'        // Descuido (respuesta rápida incorrecta)
  | 'partial'         // Entendimiento parcial
  | 'unknown'         // No clasificado

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

interface AdaptiveTutorState {
  // Historial de errores
  errors: ErrorRecord[]

  // Dominio de conceptos
  conceptMastery: Record<string, ConceptMastery>

  // Sesión actual
  currentSessionErrors: string[] // IDs de errores
  consecutiveCorrect: number
  consecutiveErrors: number

  // Preferencias de aprendizaje detectadas
  learningProfile: {
    prefersVisual: boolean
    prefersStepByStep: boolean
    averageResponseTime: number
    bestTimeOfDay: string | null
    strongConcepts: string[]
    weakConcepts: string[]
  }

  // Acciones
  recordError: (error: Omit<ErrorRecord, 'id' | 'timestamp'>) => void
  recordCorrect: (questionId: string, topicId: string, concept: string) => void
  markUnderstood: (errorId: string, understood: boolean) => void
  updateConceptMastery: (conceptId: string) => void
  getWeakConcepts: () => ConceptMastery[]
  getRecommendedReview: () => ConceptMastery[]
  analyzeErrorPattern: (concept: string) => ErrorType | null
  resetSession: () => void
}

export const useAdaptiveTutorStore = create<AdaptiveTutorState>()(
  persist(
    (set, get) => ({
      errors: [],
      conceptMastery: {},
      currentSessionErrors: [],
      consecutiveCorrect: 0,
      consecutiveErrors: 0,
      learningProfile: {
        prefersVisual: true,
        prefersStepByStep: true,
        averageResponseTime: 15000,
        bestTimeOfDay: null,
        strongConcepts: [],
        weakConcepts: [],
      },

      recordError: (errorData) => {
        const error: ErrorRecord = {
          ...errorData,
          id: `err-${Date.now()}-${Math.random().toString(36).substr(2, 9)}`,
          timestamp: new Date(),
        }

        set((state) => ({
          errors: [...state.errors, error],
          currentSessionErrors: [...state.currentSessionErrors, error.id],
          consecutiveErrors: state.consecutiveErrors + 1,
          consecutiveCorrect: 0,
        }))

        // Actualizar dominio del concepto
        get().updateConceptMastery(errorData.concept)
      },

      recordCorrect: (questionId, topicId, concept) => {
        set((state) => ({
          consecutiveCorrect: state.consecutiveCorrect + 1,
          consecutiveErrors: 0,
        }))

        get().updateConceptMastery(concept)
      },

      markUnderstood: (errorId, understood) => {
        set((state) => ({
          errors: state.errors.map((e) =>
            e.id === errorId ? { ...e, understoodAfter: understood } : e
          ),
        }))
      },

      updateConceptMastery: (conceptId) => {
        const { errors } = get()
        const conceptErrors = errors.filter((e) => e.concept === conceptId)
        const recentErrors = conceptErrors.filter(
          (e) => Date.now() - new Date(e.timestamp).getTime() < 7 * 24 * 60 * 60 * 1000
        )

        // Calcular nivel de dominio
        const totalAttempts = conceptErrors.length
        const correctRate = totalAttempts > 0
          ? 1 - (recentErrors.length / Math.max(totalAttempts, recentErrors.length * 2))
          : 0

        let masteryLevel: ConceptMastery['masteryLevel']
        if (correctRate >= 0.9) masteryLevel = 'mastered'
        else if (correctRate >= 0.7) masteryLevel = 'practicing'
        else if (correctRate >= 0.4) masteryLevel = 'learning'
        else masteryLevel = 'struggling'

        // Detectar patrones de error
        const errorPatterns = recentErrors.map((e) => e.errorType)
        const uniquePatterns = [...new Set(errorPatterns)]

        // Calcular próxima fecha de repaso (espaciado)
        const daysSinceLastError = recentErrors.length > 0
          ? (Date.now() - new Date(recentErrors[recentErrors.length - 1].timestamp).getTime()) / (24 * 60 * 60 * 1000)
          : 30

        const reviewInterval = masteryLevel === 'mastered' ? 14
          : masteryLevel === 'practicing' ? 7
          : masteryLevel === 'learning' ? 3
          : 1

        const nextReviewDate = new Date()
        nextReviewDate.setDate(nextReviewDate.getDate() + reviewInterval)

        set((state) => ({
          conceptMastery: {
            ...state.conceptMastery,
            [conceptId]: {
              conceptId,
              name: conceptId, // Se puede mapear a nombre legible
              totalAttempts,
              correctAttempts: totalAttempts - conceptErrors.length,
              lastAttempt: new Date(),
              masteryLevel,
              errorPatterns: uniquePatterns,
              needsReview: masteryLevel === 'struggling' || masteryLevel === 'learning',
              nextReviewDate,
            },
          },
        }))

        // Actualizar perfil de aprendizaje
        set((state) => {
          const weak = Object.values(state.conceptMastery)
            .filter((c) => c.masteryLevel === 'struggling' || c.masteryLevel === 'learning')
            .map((c) => c.conceptId)
          const strong = Object.values(state.conceptMastery)
            .filter((c) => c.masteryLevel === 'mastered')
            .map((c) => c.conceptId)

          return {
            learningProfile: {
              ...state.learningProfile,
              weakConcepts: weak,
              strongConcepts: strong,
            },
          }
        })
      },

      getWeakConcepts: () => {
        const { conceptMastery } = get()
        return Object.values(conceptMastery)
          .filter((c) => c.masteryLevel === 'struggling' || c.masteryLevel === 'learning')
          .sort((a, b) => a.correctAttempts / a.totalAttempts - b.correctAttempts / b.totalAttempts)
      },

      getRecommendedReview: () => {
        const { conceptMastery } = get()
        const now = new Date()
        return Object.values(conceptMastery)
          .filter((c) => c.nextReviewDate && new Date(c.nextReviewDate) <= now)
          .sort((a, b) =>
            new Date(a.nextReviewDate!).getTime() - new Date(b.nextReviewDate!).getTime()
          )
      },

      analyzeErrorPattern: (concept) => {
        const { errors } = get()
        const conceptErrors = errors
          .filter((e) => e.concept === concept)
          .slice(-10) // Últimos 10 errores

        if (conceptErrors.length < 3) return null

        // Contar tipos de error
        const counts: Record<ErrorType, number> = {
          conceptual: 0,
          calculation: 0,
          unit_conversion: 0,
          formula: 0,
          reading: 0,
          careless: 0,
          partial: 0,
          unknown: 0,
        }

        conceptErrors.forEach((e) => counts[e.errorType]++)

        // Encontrar el patrón dominante
        const dominant = Object.entries(counts)
          .sort(([, a], [, b]) => b - a)[0]

        return dominant[1] >= 2 ? (dominant[0] as ErrorType) : null
      },

      resetSession: () => {
        set({
          currentSessionErrors: [],
          consecutiveCorrect: 0,
          consecutiveErrors: 0,
        })
      },
    }),
    {
      name: 'adaptive-tutor-storage',
    }
  )
)
```

---

## Clasificador de Errores

```tsx
interface ErrorClassification {
  type: ErrorType
  confidence: number
  explanation: string
  suggestedApproach: string
}

// Clasificador basado en reglas + IA
export const classifyError = async (
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
): Promise<ErrorClassification> => {
  const { text, correctAnswer, userAnswer, timeSpentMs, averageTimeMs, hasFormula, hasUnits } = question

  // Heurísticas rápidas

  // 1. Respuesta muy rápida = descuido
  if (timeSpentMs < averageTimeMs * 0.3) {
    return {
      type: 'careless',
      confidence: 0.7,
      explanation: 'Respondiste muy rápido. Quizás no leíste bien la pregunta.',
      suggestedApproach: 'Tómate tu tiempo para leer cada pregunta con calma.',
    }
  }

  // 2. Error numérico cercano = cálculo
  const correctNum = parseFloat(correctAnswer.replace(/[^\d.-]/g, ''))
  const userNum = parseFloat(userAnswer.replace(/[^\d.-]/g, ''))

  if (!isNaN(correctNum) && !isNaN(userNum)) {
    const ratio = userNum / correctNum

    // Error de factor 10 = conversión de unidades
    if (Math.abs(Math.log10(ratio)) % 1 < 0.1 && Math.abs(Math.log10(ratio)) >= 0.9) {
      return {
        type: 'unit_conversion',
        confidence: 0.85,
        explanation: 'Tu respuesta difiere por un factor de 10. Revisa las unidades.',
        suggestedApproach: 'Verifica que todas las unidades estén en el mismo sistema antes de calcular.',
      }
    }

    // Cercano pero no exacto = error de cálculo
    if (ratio > 0.8 && ratio < 1.2 && ratio !== 1) {
      return {
        type: 'calculation',
        confidence: 0.75,
        explanation: 'Tu respuesta está cerca pero no es exacta. Revisa tus cálculos.',
        suggestedApproach: 'Repite las operaciones paso a paso, verificando cada resultado intermedio.',
      }
    }
  }

  // 3. Si hay fórmula y la respuesta es muy diferente = fórmula incorrecta
  if (hasFormula && !isNaN(correctNum) && !isNaN(userNum) && (userNum / correctNum < 0.5 || userNum / correctNum > 2)) {
    return {
      type: 'formula',
      confidence: 0.7,
      explanation: 'El resultado es muy diferente al esperado. Quizás usaste una fórmula incorrecta.',
      suggestedApproach: 'Revisa qué fórmula aplica a este tipo de problema.',
    }
  }

  // 4. Usar IA para clasificación más profunda si está disponible
  if (aiEndpoint) {
    try {
      const aiClassification = await classifyWithAI(aiEndpoint, question)
      if (aiClassification.confidence > 0.6) {
        return aiClassification
      }
    } catch (e) {
      console.warn('AI classification failed, using fallback')
    }
  }

  // 5. Por defecto: conceptual
  return {
    type: 'conceptual',
    confidence: 0.5,
    explanation: 'Parece que hay una confusión con el concepto.',
    suggestedApproach: 'Repasa la teoría antes de intentar más ejercicios.',
  }
}

// Clasificación con IA local
const classifyWithAI = async (
  endpoint: string,
  question: {
    text: string
    correctAnswer: string
    userAnswer: string
    concept: string
  }
): Promise<ErrorClassification> => {
  const prompt = `Analiza este error de un estudiante y clasifícalo.

PREGUNTA: ${question.text}
RESPUESTA CORRECTA: ${question.correctAnswer}
RESPUESTA DEL ESTUDIANTE: ${question.userAnswer}
CONCEPTO: ${question.concept}

Clasifica el error en UNA de estas categorías:
- conceptual: No entiende el concepto fundamental
- calculation: Error en las operaciones matemáticas
- unit_conversion: Error al convertir unidades
- formula: Usó la fórmula incorrecta
- reading: No leyó bien la pregunta
- careless: Descuido evidente
- partial: Entiende parcialmente

Responde SOLO en este formato JSON:
{
  "type": "categoria",
  "confidence": 0.0-1.0,
  "explanation": "Explicación breve del error",
  "suggestedApproach": "Cómo mejorar"
}`

  const response = await fetch(endpoint, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      model: 'local-model',
      messages: [{ role: 'user', content: prompt }],
      temperature: 0.3,
      max_tokens: 300,
    }),
  })

  const data = await response.json()
  const content = data.choices[0]?.message?.content || ''

  try {
    return JSON.parse(content)
  } catch {
    throw new Error('Invalid AI response format')
  }
}
```

---

## Generador de Explicaciones Adaptativas

```tsx
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
  visualAid?: {
    type: 'diagram' | 'formula' | 'animation' | 'example'
    content: string
  }
  steps?: string[]
  analogies?: string[]
  practiceQuestion?: {
    text: string
    answer: string
    hint: string
  }
  relatedConcepts?: string[]
}

export const generateExplanation = async (
  endpoint: string,
  request: ExplanationRequest
): Promise<AdaptiveExplanation> => {
  const {
    concept,
    errorType,
    question,
    correctAnswer,
    userAnswer,
    studentLevel,
    previousErrors,
    prefersVisual,
    prefersStepByStep
  } = request

  // Construir contexto de errores previos
  const errorContext = previousErrors.length > 0
    ? `El estudiante ha cometido estos errores previos en ${concept}: ${
        previousErrors.slice(-3).map(e => e.errorType).join(', ')
      }`
    : ''

  // Adaptar el estilo según preferencias
  const styleInstructions = []
  if (prefersVisual) {
    styleInstructions.push('Incluye diagramas ASCII o describe visualizaciones')
  }
  if (prefersStepByStep) {
    styleInstructions.push('Explica paso a paso, numerando cada paso')
  }
  if (studentLevel === 'beginner') {
    styleInstructions.push('Usa lenguaje muy simple, evita jerga técnica')
  }

  const prompt = `Eres un profesor experto y paciente. Un estudiante cometió un error.

CONCEPTO: ${concept}
TIPO DE ERROR: ${errorType}
PREGUNTA: ${question}
RESPUESTA CORRECTA: ${correctAnswer}
RESPUESTA DEL ESTUDIANTE: ${userAnswer}
NIVEL DEL ESTUDIANTE: ${studentLevel}
${errorContext}

ESTILO DE EXPLICACIÓN:
${styleInstructions.join('\n')}

Genera una explicación personalizada que:
1. NO critique al estudiante
2. Explique por qué la respuesta correcta es correcta
3. Identifique el error específico sin ser condescendiente
4. Proporcione una analogía del mundo real
5. Sugiera un ejercicio de práctica similar

Responde en JSON:
{
  "mainExplanation": "Explicación clara y empática",
  "steps": ["Paso 1...", "Paso 2..."],
  "analogies": ["Analogía 1..."],
  "practiceQuestion": {
    "text": "Pregunta de práctica",
    "answer": "Respuesta",
    "hint": "Pista"
  },
  "relatedConcepts": ["concepto1", "concepto2"]
}`

  try {
    const response = await fetch(endpoint, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        model: 'local-model',
        messages: [{ role: 'user', content: prompt }],
        temperature: 0.7,
        max_tokens: 800,
      }),
    })

    const data = await response.json()
    const content = data.choices[0]?.message?.content || ''

    return JSON.parse(content)
  } catch (error) {
    // Fallback a explicación genérica
    return generateFallbackExplanation(request)
  }
}

// Explicaciones pre-generadas por tipo de error
const generateFallbackExplanation = (request: ExplanationRequest): AdaptiveExplanation => {
  const explanations: Record<ErrorType, Partial<AdaptiveExplanation>> = {
    conceptual: {
      mainExplanation: `Parece que hay una confusión con el concepto de ${request.concept}. Vamos a repasarlo desde el principio.`,
      steps: [
        'Primero, asegúrate de entender la definición básica',
        'Luego, identifica las variables involucradas',
        'Finalmente, aplica el concepto al problema',
      ],
    },
    calculation: {
      mainExplanation: 'Hubo un error en los cálculos. No te preocupes, es muy común. Vamos a revisar paso a paso.',
      steps: [
        'Identifica los datos del problema',
        'Escribe la operación que vas a realizar',
        'Realiza el cálculo despacio',
        'Verifica el resultado sustituyendo',
      ],
    },
    unit_conversion: {
      mainExplanation: 'El error está en la conversión de unidades. Es crucial que todas las unidades sean compatibles.',
      steps: [
        'Identifica las unidades de cada dato',
        'Convierte todo al mismo sistema (SI recomendado)',
        'Realiza el cálculo',
        'Asegúrate de que el resultado tenga las unidades correctas',
      ],
    },
    formula: {
      mainExplanation: `Parece que se usó una fórmula incorrecta. Repasemos cuándo aplicar cada fórmula de ${request.concept}.`,
    },
    reading: {
      mainExplanation: 'El error parece venir de no leer completamente la pregunta. Tómate tu tiempo para entender qué se pide.',
      steps: [
        'Lee la pregunta completa',
        'Subraya los datos importantes',
        'Identifica qué te piden calcular',
        'Resuelve y verifica que respondes lo que preguntan',
      ],
    },
    careless: {
      mainExplanation: 'Fue un pequeño descuido. Respondiste muy rápido. La próxima vez, tómate unos segundos extra para verificar.',
    },
    partial: {
      mainExplanation: 'Entiendes parte del concepto, pero falta completar el entendimiento. Vamos a reforzar lo que falta.',
    },
    unknown: {
      mainExplanation: 'Vamos a revisar este ejercicio juntos para entender dónde estuvo el error.',
    },
  }

  return {
    mainExplanation: explanations[request.errorType].mainExplanation || '',
    steps: explanations[request.errorType].steps,
    relatedConcepts: [request.concept],
  }
}
```

---

## Componente de Feedback Integrado

```tsx
import React, { useState, useEffect } from 'react'
import { motion, AnimatePresence } from 'framer-motion'
import { useAdaptiveTutorStore } from '../store/adaptiveTutorStore'
import { classifyError, generateExplanation } from '../utils/adaptiveTutor'

interface AdaptiveFeedbackProps {
  question: {
    id: string
    text: string
    correctAnswer: string
    concept: string
    hasFormula?: boolean
    hasUnits?: boolean
  }
  userAnswer: string
  isCorrect: boolean
  timeSpentMs: number
  onContinue: () => void
  onPracticeMore: (concept: string) => void
  aiEndpoint?: string
}

export const AdaptiveFeedback: React.FC<AdaptiveFeedbackProps> = ({
  question,
  userAnswer,
  isCorrect,
  timeSpentMs,
  onContinue,
  onPracticeMore,
  aiEndpoint = 'http://localhost:1234/v1/chat/completions',
}) => {
  const [explanation, setExplanation] = useState<AdaptiveExplanation | null>(null)
  const [isLoading, setIsLoading] = useState(false)
  const [showPractice, setShowPractice] = useState(false)
  const [practiceAnswer, setPracticeAnswer] = useState('')
  const [practiceChecked, setPracticeChecked] = useState(false)

  const {
    recordError,
    recordCorrect,
    markUnderstood,
    errors,
    learningProfile,
    consecutiveErrors,
  } = useAdaptiveTutorStore()

  useEffect(() => {
    if (isCorrect) {
      recordCorrect(question.id, 'topic', question.concept)
      return
    }

    // Clasificar y registrar error
    const analyzeError = async () => {
      setIsLoading(true)

      const classification = await classifyError({
        text: question.text,
        correctAnswer: question.correctAnswer,
        userAnswer,
        concept: question.concept,
        hasFormula: question.hasFormula || false,
        hasUnits: question.hasUnits || false,
        timeSpentMs,
        averageTimeMs: learningProfile.averageResponseTime,
      }, aiEndpoint)

      recordError({
        questionId: question.id,
        topicId: 'topic',
        concept: question.concept,
        questionText: question.text,
        correctAnswer: question.correctAnswer,
        userAnswer,
        errorType: classification.type,
        timeSpentMs,
        hintsUsed: false,
        aiExplanationShown: true,
        understoodAfter: null,
      })

      // Generar explicación adaptativa
      const previousConceptErrors = errors.filter(e => e.concept === question.concept)

      const adaptiveExplanation = await generateExplanation(aiEndpoint, {
        concept: question.concept,
        errorType: classification.type,
        question: question.text,
        correctAnswer: question.correctAnswer,
        userAnswer,
        studentLevel: previousConceptErrors.length > 5 ? 'intermediate' : 'beginner',
        previousErrors: previousConceptErrors.slice(-5),
        prefersVisual: learningProfile.prefersVisual,
        prefersStepByStep: learningProfile.prefersStepByStep,
      })

      setExplanation(adaptiveExplanation)
      setIsLoading(false)
    }

    analyzeError()
  }, [isCorrect, question, userAnswer])

  const handleUnderstood = (understood: boolean) => {
    const lastError = errors[errors.length - 1]
    if (lastError) {
      markUnderstood(lastError.id, understood)
    }

    if (!understood && explanation?.practiceQuestion) {
      setShowPractice(true)
    } else {
      onContinue()
    }
  }

  const checkPractice = () => {
    setPracticeChecked(true)
  }

  if (isCorrect) {
    return (
      <motion.div
        className="adaptive-feedback correct"
        initial={{ opacity: 0, y: 20 }}
        animate={{ opacity: 1, y: 0 }}
      >
        <div className="feedback-icon">✅</div>
        <h3>¡Correcto!</h3>

        {consecutiveErrors === 0 && (
          <p className="encouragement">
            {Math.random() > 0.5
              ? '¡Sigue así!'
              : '¡Excelente trabajo!'}
          </p>
        )}

        <button className="continue-btn" onClick={onContinue}>
          Continuar →
        </button>
      </motion.div>
    )
  }

  return (
    <motion.div
      className="adaptive-feedback incorrect"
      initial={{ opacity: 0, y: 20 }}
      animate={{ opacity: 1, y: 0 }}
    >
      <div className="feedback-icon">💭</div>

      {isLoading ? (
        <div className="loading-explanation">
          <div className="loading-spinner" />
          <p>Analizando tu respuesta...</p>
        </div>
      ) : explanation ? (
        <AnimatePresence mode="wait">
          {!showPractice ? (
            <motion.div
              key="explanation"
              initial={{ opacity: 0 }}
              animate={{ opacity: 1 }}
              exit={{ opacity: 0 }}
            >
              {/* Respuesta correcta */}
              <div className="correct-answer-box">
                <span className="label">Respuesta correcta:</span>
                <span className="answer">{question.correctAnswer}</span>
              </div>

              {/* Explicación principal */}
              <div className="explanation-main">
                <p>{explanation.mainExplanation}</p>
              </div>

              {/* Pasos si existen */}
              {explanation.steps && explanation.steps.length > 0 && (
                <div className="explanation-steps">
                  <h4>📝 Paso a paso:</h4>
                  <ol>
                    {explanation.steps.map((step, i) => (
                      <motion.li
                        key={i}
                        initial={{ opacity: 0, x: -10 }}
                        animate={{ opacity: 1, x: 0 }}
                        transition={{ delay: i * 0.15 }}
                      >
                        {step}
                      </motion.li>
                    ))}
                  </ol>
                </div>
              )}

              {/* Analogía */}
              {explanation.analogies && explanation.analogies.length > 0 && (
                <div className="explanation-analogy">
                  <h4>💡 Piénsalo así:</h4>
                  <p>{explanation.analogies[0]}</p>
                </div>
              )}

              {/* Conceptos relacionados */}
              {explanation.relatedConcepts && explanation.relatedConcepts.length > 0 && (
                <div className="related-concepts">
                  <span>Conceptos relacionados:</span>
                  {explanation.relatedConcepts.map((concept, i) => (
                    <button
                      key={i}
                      className="concept-chip"
                      onClick={() => onPracticeMore(concept)}
                    >
                      {concept}
                    </button>
                  ))}
                </div>
              )}

              {/* Verificación de comprensión */}
              <div className="understanding-check">
                <p>¿Te ha quedado claro?</p>
                <div className="understanding-buttons">
                  <button
                    className="understood-btn yes"
                    onClick={() => handleUnderstood(true)}
                  >
                    ✓ Sí, lo entiendo
                  </button>
                  <button
                    className="understood-btn no"
                    onClick={() => handleUnderstood(false)}
                  >
                    🤔 Necesito practicar más
                  </button>
                </div>
              </div>
            </motion.div>
          ) : (
            <motion.div
              key="practice"
              className="practice-section"
              initial={{ opacity: 0 }}
              animate={{ opacity: 1 }}
            >
              <h4>🎯 Ejercicio de refuerzo</h4>

              {explanation.practiceQuestion && (
                <>
                  <p className="practice-question">
                    {explanation.practiceQuestion.text}
                  </p>

                  <input
                    type="text"
                    className="practice-input"
                    value={practiceAnswer}
                    onChange={(e) => setPracticeAnswer(e.target.value)}
                    placeholder="Tu respuesta..."
                    disabled={practiceChecked}
                  />

                  {!practiceChecked ? (
                    <>
                      <button
                        className="check-practice-btn"
                        onClick={checkPractice}
                        disabled={!practiceAnswer.trim()}
                      >
                        Comprobar
                      </button>
                      <p className="hint">
                        💡 Pista: {explanation.practiceQuestion.hint}
                      </p>
                    </>
                  ) : (
                    <div className={`practice-result ${
                      practiceAnswer.toLowerCase().trim() ===
                      explanation.practiceQuestion.answer.toLowerCase().trim()
                        ? 'correct'
                        : 'incorrect'
                    }`}>
                      {practiceAnswer.toLowerCase().trim() ===
                       explanation.practiceQuestion.answer.toLowerCase().trim() ? (
                        <>
                          <span>✅ ¡Correcto! Ya lo vas pillando.</span>
                          <button onClick={onContinue}>Continuar →</button>
                        </>
                      ) : (
                        <>
                          <span>
                            La respuesta era: {explanation.practiceQuestion.answer}
                          </span>
                          <button onClick={() => onPracticeMore(question.concept)}>
                            Practicar más sobre {question.concept}
                          </button>
                        </>
                      )}
                    </div>
                  )}
                </>
              )}
            </motion.div>
          )}
        </AnimatePresence>
      ) : (
        <p>La respuesta correcta es: {question.correctAnswer}</p>
      )}

      {/* Indicador de errores consecutivos */}
      {consecutiveErrors >= 3 && (
        <motion.div
          className="consecutive-errors-warning"
          initial={{ opacity: 0, scale: 0.9 }}
          animate={{ opacity: 1, scale: 1 }}
        >
          <p>
            💪 Llevas {consecutiveErrors} errores seguidos.
            ¿Quieres repasar la teoría de {question.concept}?
          </p>
          <button onClick={() => onPracticeMore(question.concept)}>
            Sí, repasar teoría
          </button>
        </motion.div>
      )}
    </motion.div>
  )
}
```

---

## Integración con GamifiedQuiz

```tsx
// En GamifiedQuiz.tsx, reemplazar el feedback simple por AdaptiveFeedback

import { AdaptiveFeedback } from './AdaptiveFeedback'

// Dentro del componente GamifiedQuiz, en la sección de revelación:

{isRevealed && (
  <AdaptiveFeedback
    question={{
      id: currentQuestion.id,
      text: currentQuestion.text,
      correctAnswer: currentQuestion.options[currentQuestion.correctIndex],
      concept: currentQuestion.concept || topicId,
      hasFormula: !!currentQuestion.latex,
      hasUnits: currentQuestion.text.includes('unidad') ||
                currentQuestion.text.includes('m/s') ||
                currentQuestion.text.includes('kg'),
    }}
    userAnswer={currentQuestion.options[selectedAnswer] || 'Sin respuesta'}
    isCorrect={selectedAnswer === currentQuestion.correctIndex}
    timeSpentMs={Date.now() - startTimeRef.current}
    onContinue={handleNext}
    onPracticeMore={(concept) => {
      // Navegar a ejercicios de refuerzo del concepto
      console.log('Practicar más:', concept)
    }}
    aiEndpoint="http://localhost:1234/v1/chat/completions"
  />
)}
```

---

## Panel de Diagnóstico del Estudiante

```tsx
export const StudentDiagnosticPanel: React.FC = () => {
  const {
    conceptMastery,
    learningProfile,
    getWeakConcepts,
    getRecommendedReview,
    errors,
  } = useAdaptiveTutorStore()

  const weakConcepts = getWeakConcepts()
  const reviewConcepts = getRecommendedReview()

  return (
    <div className="diagnostic-panel">
      <h2>📊 Tu Progreso de Aprendizaje</h2>

      {/* Resumen general */}
      <div className="summary-cards">
        <div className="summary-card">
          <span className="card-icon">📚</span>
          <span className="card-value">{Object.keys(conceptMastery).length}</span>
          <span className="card-label">Conceptos estudiados</span>
        </div>
        <div className="summary-card">
          <span className="card-icon">✅</span>
          <span className="card-value">
            {Object.values(conceptMastery).filter(c => c.masteryLevel === 'mastered').length}
          </span>
          <span className="card-label">Dominados</span>
        </div>
        <div className="summary-card">
          <span className="card-icon">🎯</span>
          <span className="card-value">{errors.length}</span>
          <span className="card-label">Errores analizados</span>
        </div>
      </div>

      {/* Conceptos que necesitan repaso */}
      {reviewConcepts.length > 0 && (
        <div className="review-section">
          <h3>📅 Repaso recomendado hoy</h3>
          <div className="review-list">
            {reviewConcepts.map((concept) => (
              <div key={concept.conceptId} className="review-item">
                <span className="concept-name">{concept.name}</span>
                <span className={`mastery-badge ${concept.masteryLevel}`}>
                  {concept.masteryLevel}
                </span>
                <button className="review-btn">Repasar</button>
              </div>
            ))}
          </div>
        </div>
      )}

      {/* Áreas de mejora */}
      {weakConcepts.length > 0 && (
        <div className="weak-concepts-section">
          <h3>💪 Áreas de mejora</h3>
          {weakConcepts.map((concept) => (
            <div key={concept.conceptId} className="weak-concept-card">
              <div className="concept-header">
                <span className="concept-name">{concept.name}</span>
                <span className="accuracy">
                  {Math.round((concept.correctAttempts / concept.totalAttempts) * 100)}% aciertos
                </span>
              </div>

              {/* Patrones de error */}
              {concept.errorPatterns.length > 0 && (
                <div className="error-patterns">
                  <span>Errores frecuentes:</span>
                  {concept.errorPatterns.map((pattern, i) => (
                    <span key={i} className={`pattern-tag ${pattern}`}>
                      {errorTypeLabels[pattern]}
                    </span>
                  ))}
                </div>
              )}

              <button className="practice-btn">
                Practicar {concept.name}
              </button>
            </div>
          ))}
        </div>
      )}

      {/* Fortalezas */}
      {learningProfile.strongConcepts.length > 0 && (
        <div className="strengths-section">
          <h3>⭐ Tus fortalezas</h3>
          <div className="strengths-list">
            {learningProfile.strongConcepts.map((concept) => (
              <span key={concept} className="strength-chip">
                ✓ {concept}
              </span>
            ))}
          </div>
        </div>
      )}
    </div>
  )
}

const errorTypeLabels: Record<ErrorType, string> = {
  conceptual: 'Conceptuales',
  calculation: 'De cálculo',
  unit_conversion: 'Conversión de unidades',
  formula: 'Fórmulas',
  reading: 'Lectura',
  careless: 'Descuidos',
  partial: 'Parciales',
  unknown: 'Otros',
}
```

---

## Estilos CSS

```css
/* Feedback adaptativo */
.adaptive-feedback {
  padding: var(--spacing-lg);
  border-radius: var(--radius-lg);
  margin-top: var(--spacing-md);
}

.adaptive-feedback.correct {
  background: linear-gradient(135deg, rgba(76, 175, 80, 0.1), rgba(139, 195, 74, 0.1));
  border: 2px solid #4CAF50;
}

.adaptive-feedback.incorrect {
  background: linear-gradient(135deg, rgba(33, 150, 243, 0.05), rgba(156, 39, 176, 0.05));
  border: 2px solid #2196F3;
}

.feedback-icon {
  font-size: 2.5rem;
  text-align: center;
  margin-bottom: var(--spacing-sm);
}

.correct-answer-box {
  background: rgba(76, 175, 80, 0.1);
  padding: var(--spacing-sm) var(--spacing-md);
  border-radius: var(--radius-md);
  margin-bottom: var(--spacing-md);
  display: flex;
  gap: var(--spacing-sm);
  align-items: center;
}

.correct-answer-box .label {
  color: #666;
  font-size: 0.875rem;
}

.correct-answer-box .answer {
  font-weight: 600;
  color: #4CAF50;
}

.explanation-main {
  font-size: 1rem;
  line-height: 1.6;
  margin-bottom: var(--spacing-md);
}

.explanation-steps {
  background: #f8f9fa;
  padding: var(--spacing-md);
  border-radius: var(--radius-md);
  margin-bottom: var(--spacing-md);
}

.explanation-steps h4 {
  margin: 0 0 var(--spacing-sm) 0;
  font-size: 0.9rem;
}

.explanation-steps ol {
  margin: 0;
  padding-left: 1.5rem;
}

.explanation-steps li {
  margin-bottom: var(--spacing-xs);
  line-height: 1.5;
}

.explanation-analogy {
  background: linear-gradient(135deg, #fff8e1, #fff3e0);
  padding: var(--spacing-md);
  border-radius: var(--radius-md);
  border-left: 4px solid #FF9800;
  margin-bottom: var(--spacing-md);
}

.understanding-check {
  text-align: center;
  padding-top: var(--spacing-md);
  border-top: 1px solid #e0e0e0;
}

.understanding-buttons {
  display: flex;
  gap: var(--spacing-sm);
  justify-content: center;
  margin-top: var(--spacing-sm);
}

.understood-btn {
  padding: var(--spacing-sm) var(--spacing-md);
  border-radius: var(--radius-md);
  border: none;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.2s;
}

.understood-btn.yes {
  background: #4CAF50;
  color: white;
}

.understood-btn.no {
  background: #f5f5f5;
  color: #666;
}

.understood-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 4px 12px rgba(0, 0, 0, 0.15);
}

.consecutive-errors-warning {
  margin-top: var(--spacing-md);
  padding: var(--spacing-md);
  background: linear-gradient(135deg, #fff3e0, #ffe0b2);
  border-radius: var(--radius-md);
  text-align: center;
}

/* Panel de diagnóstico */
.diagnostic-panel {
  padding: var(--spacing-lg);
  background: white;
  border-radius: var(--radius-lg);
}

.summary-cards {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: var(--spacing-md);
  margin-bottom: var(--spacing-lg);
}

.summary-card {
  text-align: center;
  padding: var(--spacing-md);
  background: #f8f9fa;
  border-radius: var(--radius-md);
}

.summary-card .card-icon {
  font-size: 1.5rem;
  display: block;
}

.summary-card .card-value {
  font-size: 2rem;
  font-weight: 700;
  display: block;
  color: var(--color-primary);
}

.summary-card .card-label {
  font-size: 0.75rem;
  color: #666;
}

.mastery-badge {
  padding: 2px 8px;
  border-radius: 12px;
  font-size: 0.7rem;
  font-weight: 600;
  text-transform: uppercase;
}

.mastery-badge.mastered { background: #c8e6c9; color: #2e7d32; }
.mastery-badge.practicing { background: #fff9c4; color: #f9a825; }
.mastery-badge.learning { background: #bbdefb; color: #1565c0; }
.mastery-badge.struggling { background: #ffcdd2; color: #c62828; }

.pattern-tag {
  padding: 2px 6px;
  border-radius: 8px;
  font-size: 0.7rem;
  margin-left: 4px;
}

.pattern-tag.conceptual { background: #e3f2fd; color: #1565c0; }
.pattern-tag.calculation { background: #fce4ec; color: #c2185b; }
.pattern-tag.unit_conversion { background: #f3e5f5; color: #7b1fa2; }
.pattern-tag.formula { background: #e8f5e9; color: #2e7d32; }
```

---

## Uso Completo

```tsx
// En una página de ejercicios
import { GamifiedQuiz } from './components/GamifiedQuiz'
import { StudentDiagnosticPanel } from './components/StudentDiagnosticPanel'
import { useAdaptiveTutorStore } from './store/adaptiveTutorStore'

const ExercisePage: React.FC = () => {
  const { getWeakConcepts } = useAdaptiveTutorStore()
  const weakConcepts = getWeakConcepts()

  // Preguntas adaptadas a las debilidades del estudiante
  const adaptedQuestions = useMemo(() => {
    const baseQuestions = [...allQuestions]

    // Añadir más preguntas de conceptos débiles
    weakConcepts.forEach(weak => {
      const reinforcement = allQuestions.filter(q => q.concept === weak.conceptId)
      baseQuestions.push(...reinforcement.slice(0, 2))
    })

    return baseQuestions
  }, [weakConcepts])

  return (
    <div>
      <GamifiedQuiz
        questions={adaptedQuestions}
        topicId="physics-mechanics"
        title="Mecánica - Quiz Adaptativo"
      />

      <StudentDiagnosticPanel />
    </div>
  )
}
```
