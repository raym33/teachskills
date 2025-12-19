# Gamificación Educativa - Tests y Ejercicios

Instrucciones especializadas para crear sistemas de evaluación gamificados que maximicen el engagement y la retención del aprendizaje.

---

## Principios de Gamificación Educativa

### Elementos Clave
1. **Progresión visible** - Barras de XP, niveles, desbloqueos
2. **Feedback inmediato** - Respuestas instantáneas con explicaciones
3. **Recompensas** - Badges, logros, rachas
4. **Competición sana** - Leaderboards opcionales, retos
5. **Narrativa** - Contexto de historia/misión
6. **Autonomía** - Múltiples caminos, elección de dificultad

---

## Sistema de Puntuación y Progresión

### Store de Gamificación (Zustand)

```tsx
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface Achievement {
  id: string
  name: string
  description: string
  icon: string
  unlockedAt?: Date
  category: 'mastery' | 'streak' | 'explorer' | 'speed' | 'perfect'
}

interface GamificationState {
  // Puntuación
  xp: number
  level: number
  streak: number
  lastActivityDate: string | null

  // Estadísticas
  totalQuestions: number
  correctAnswers: number
  perfectQuizzes: number
  timeSpentSeconds: number

  // Logros
  achievements: Achievement[]
  unlockedAchievements: string[]

  // Progreso por tema
  topicProgress: Record<string, {
    completed: number
    total: number
    bestScore: number
    attempts: number
  }>

  // Acciones
  addXP: (amount: number, reason?: string) => void
  incrementStreak: () => void
  resetStreak: () => void
  unlockAchievement: (achievementId: string) => void
  updateTopicProgress: (topicId: string, score: number, total: number) => void
  recordAnswer: (correct: boolean, timeMs: number) => void
}

// Cálculo de nivel basado en XP (curva logarítmica)
const calculateLevel = (xp: number): number => {
  return Math.floor(Math.sqrt(xp / 100)) + 1
}

// XP necesario para siguiente nivel
const xpForLevel = (level: number): number => {
  return Math.pow(level, 2) * 100
}

export const useGamificationStore = create<GamificationState>()(
  persist(
    (set, get) => ({
      xp: 0,
      level: 1,
      streak: 0,
      lastActivityDate: null,
      totalQuestions: 0,
      correctAnswers: 0,
      perfectQuizzes: 0,
      timeSpentSeconds: 0,
      achievements: defaultAchievements,
      unlockedAchievements: [],
      topicProgress: {},

      addXP: (amount, reason) => {
        set((state) => {
          const newXP = state.xp + amount
          const newLevel = calculateLevel(newXP)
          const leveledUp = newLevel > state.level

          // Mostrar notificación si subió de nivel
          if (leveledUp) {
            showLevelUpNotification(newLevel)
          }

          return {
            xp: newXP,
            level: newLevel,
          }
        })
      },

      incrementStreak: () => {
        const today = new Date().toDateString()
        const { lastActivityDate, streak } = get()

        if (lastActivityDate === today) return // Ya contado hoy

        const yesterday = new Date()
        yesterday.setDate(yesterday.getDate() - 1)

        const isConsecutive = lastActivityDate === yesterday.toDateString()

        set({
          streak: isConsecutive ? streak + 1 : 1,
          lastActivityDate: today,
        })

        // Verificar logros de racha
        const newStreak = isConsecutive ? streak + 1 : 1
        if (newStreak === 7) get().unlockAchievement('streak-week')
        if (newStreak === 30) get().unlockAchievement('streak-month')
      },

      resetStreak: () => set({ streak: 0 }),

      unlockAchievement: (achievementId) => {
        const { unlockedAchievements, achievements } = get()
        if (unlockedAchievements.includes(achievementId)) return

        const achievement = achievements.find(a => a.id === achievementId)
        if (achievement) {
          showAchievementNotification(achievement)
          set({
            unlockedAchievements: [...unlockedAchievements, achievementId],
          })
          // Bonus XP por logro
          get().addXP(50, `Logro: ${achievement.name}`)
        }
      },

      updateTopicProgress: (topicId, score, total) => {
        set((state) => {
          const current = state.topicProgress[topicId] || {
            completed: 0,
            total: 0,
            bestScore: 0,
            attempts: 0,
          }

          const percentage = (score / total) * 100

          return {
            topicProgress: {
              ...state.topicProgress,
              [topicId]: {
                completed: Math.max(current.completed, score),
                total,
                bestScore: Math.max(current.bestScore, percentage),
                attempts: current.attempts + 1,
              },
            },
          }
        })

        // Verificar logro de tema completado
        const percentage = (score / total) * 100
        if (percentage === 100) {
          get().unlockAchievement(`perfect-${topicId}`)
        }
      },

      recordAnswer: (correct, timeMs) => {
        set((state) => ({
          totalQuestions: state.totalQuestions + 1,
          correctAnswers: state.correctAnswers + (correct ? 1 : 0),
          timeSpentSeconds: state.timeSpentSeconds + timeMs / 1000,
        }))

        // XP por respuesta
        if (correct) {
          const speedBonus = timeMs < 5000 ? 5 : timeMs < 10000 ? 2 : 0
          get().addXP(10 + speedBonus)
        }
      },
    }),
    {
      name: 'gamification-storage',
    }
  )
)

// Logros predefinidos
const defaultAchievements: Achievement[] = [
  // Maestría
  { id: 'first-quiz', name: 'Primer Paso', description: 'Completa tu primer quiz', icon: '🎯', category: 'mastery' },
  { id: 'perfect-score', name: 'Perfección', description: 'Obtén 100% en un quiz', icon: '💯', category: 'perfect' },
  { id: 'ten-quizzes', name: 'Estudiante Dedicado', description: 'Completa 10 quizzes', icon: '📚', category: 'mastery' },
  { id: 'hundred-questions', name: 'Centenario', description: 'Responde 100 preguntas', icon: '💪', category: 'mastery' },

  // Rachas
  { id: 'streak-3', name: 'En Racha', description: '3 días consecutivos', icon: '🔥', category: 'streak' },
  { id: 'streak-week', name: 'Semana Perfecta', description: '7 días consecutivos', icon: '⭐', category: 'streak' },
  { id: 'streak-month', name: 'Mes Imparable', description: '30 días consecutivos', icon: '🏆', category: 'streak' },

  // Explorador
  { id: 'all-topics', name: 'Explorador', description: 'Visita todos los temas', icon: '🗺️', category: 'explorer' },
  { id: 'night-owl', name: 'Búho Nocturno', description: 'Estudia después de las 22:00', icon: '🦉', category: 'explorer' },
  { id: 'early-bird', name: 'Madrugador', description: 'Estudia antes de las 7:00', icon: '🐦', category: 'explorer' },

  // Velocidad
  { id: 'speed-demon', name: 'Veloz', description: 'Responde correctamente en menos de 3 segundos', icon: '⚡', category: 'speed' },
  { id: 'flash', name: 'Flash', description: 'Completa un quiz en menos de 1 minuto', icon: '🏃', category: 'speed' },
]
```

---

## Componentes de Quiz Gamificados

### Quiz Principal con Animaciones

```tsx
import React, { useState, useEffect, useRef } from 'react'
import { motion, AnimatePresence } from 'framer-motion'
import confetti from 'canvas-confetti'
import { useGamificationStore } from '../store/gamificationStore'

interface Question {
  id: string
  text: string
  options: string[]
  correctIndex: number
  explanation: string
  difficulty: 'easy' | 'medium' | 'hard'
  points: number
  hint?: string
  image?: string
  latex?: string // Para fórmulas
}

interface GamifiedQuizProps {
  questions: Question[]
  topicId: string
  title: string
  timeLimit?: number // segundos por pregunta (0 = sin límite)
  shuffleQuestions?: boolean
  shuffleOptions?: boolean
  showHints?: boolean
  allowRetry?: boolean
  onComplete?: (score: number, total: number) => void
}

export const GamifiedQuiz: React.FC<GamifiedQuizProps> = ({
  questions: initialQuestions,
  topicId,
  title,
  timeLimit = 30,
  shuffleQuestions = true,
  shuffleOptions = true,
  showHints = true,
  allowRetry = true,
  onComplete,
}) => {
  const [questions, setQuestions] = useState<Question[]>([])
  const [currentIndex, setCurrentIndex] = useState(0)
  const [selectedAnswer, setSelectedAnswer] = useState<number | null>(null)
  const [isRevealed, setIsRevealed] = useState(false)
  const [score, setScore] = useState(0)
  const [combo, setCombo] = useState(0)
  const [maxCombo, setMaxCombo] = useState(0)
  const [timeLeft, setTimeLeft] = useState(timeLimit)
  const [isComplete, setIsComplete] = useState(false)
  const [hintsUsed, setHintsUsed] = useState(0)
  const [showHint, setShowHint] = useState(false)
  const startTimeRef = useRef<number>(Date.now())

  const { addXP, recordAnswer, updateTopicProgress, unlockAchievement, incrementStreak } = useGamificationStore()

  // Inicializar y mezclar preguntas
  useEffect(() => {
    let q = [...initialQuestions]
    if (shuffleQuestions) {
      q = q.sort(() => Math.random() - 0.5)
    }
    if (shuffleOptions) {
      q = q.map(question => {
        const shuffled = question.options
          .map((opt, idx) => ({ opt, isCorrect: idx === question.correctIndex }))
          .sort(() => Math.random() - 0.5)
        return {
          ...question,
          options: shuffled.map(s => s.opt),
          correctIndex: shuffled.findIndex(s => s.isCorrect),
        }
      })
    }
    setQuestions(q)
  }, [initialQuestions, shuffleQuestions, shuffleOptions])

  // Timer
  useEffect(() => {
    if (timeLimit === 0 || isRevealed || isComplete) return

    const timer = setInterval(() => {
      setTimeLeft((prev) => {
        if (prev <= 1) {
          handleTimeout()
          return timeLimit
        }
        return prev - 1
      })
    }, 1000)

    return () => clearInterval(timer)
  }, [currentIndex, isRevealed, isComplete, timeLimit])

  const handleTimeout = () => {
    if (selectedAnswer === null) {
      setSelectedAnswer(-1) // Tiempo agotado
      setIsRevealed(true)
      setCombo(0)
      recordAnswer(false, timeLimit * 1000)
    }
  }

  const handleSelectAnswer = (index: number) => {
    if (isRevealed) return

    const answerTime = Date.now() - startTimeRef.current
    setSelectedAnswer(index)
    setIsRevealed(true)

    const question = questions[currentIndex]
    const isCorrect = index === question.correctIndex

    recordAnswer(isCorrect, answerTime)

    if (isCorrect) {
      // Calcular puntos con bonificaciones
      let points = question.points

      // Bonus por velocidad
      const speedMultiplier = timeLimit > 0
        ? 1 + (timeLeft / timeLimit) * 0.5
        : 1

      // Bonus por combo
      const comboMultiplier = 1 + (combo * 0.1)

      // Penalización por pista
      const hintPenalty = showHint ? 0.5 : 1

      const finalPoints = Math.round(points * speedMultiplier * comboMultiplier * hintPenalty)

      setScore(prev => prev + finalPoints)
      setCombo(prev => {
        const newCombo = prev + 1
        setMaxCombo(current => Math.max(current, newCombo))
        return newCombo
      })
      addXP(finalPoints)

      // Efectos visuales
      if (combo >= 2) {
        triggerComboEffect(combo + 1)
      }

      // Logros de velocidad
      if (answerTime < 3000) {
        unlockAchievement('speed-demon')
      }
    } else {
      setCombo(0)
      // Vibración en móvil
      if (navigator.vibrate) {
        navigator.vibrate(200)
      }
    }
  }

  const handleNext = () => {
    if (currentIndex < questions.length - 1) {
      setCurrentIndex(prev => prev + 1)
      setSelectedAnswer(null)
      setIsRevealed(false)
      setShowHint(false)
      setTimeLeft(timeLimit)
      startTimeRef.current = Date.now()
    } else {
      completeQuiz()
    }
  }

  const completeQuiz = () => {
    setIsComplete(true)

    const correctCount = score > 0 ? Math.round((score / questions.reduce((acc, q) => acc + q.points, 0)) * questions.length) : 0
    const percentage = (correctCount / questions.length) * 100

    updateTopicProgress(topicId, correctCount, questions.length)
    incrementStreak()

    // Logros
    unlockAchievement('first-quiz')
    if (percentage === 100) {
      unlockAchievement('perfect-score')
      triggerConfetti()
    }

    onComplete?.(correctCount, questions.length)
  }

  const triggerComboEffect = (comboCount: number) => {
    // Partículas según el combo
    confetti({
      particleCount: comboCount * 10,
      spread: 60,
      origin: { y: 0.7 },
      colors: ['#FFD700', '#FFA500', '#FF6347'],
    })
  }

  const triggerConfetti = () => {
    const duration = 3000
    const end = Date.now() + duration

    const frame = () => {
      confetti({
        particleCount: 5,
        angle: 60,
        spread: 55,
        origin: { x: 0 },
        colors: ['#4CAF50', '#8BC34A', '#CDDC39'],
      })
      confetti({
        particleCount: 5,
        angle: 120,
        spread: 55,
        origin: { x: 1 },
        colors: ['#4CAF50', '#8BC34A', '#CDDC39'],
      })

      if (Date.now() < end) {
        requestAnimationFrame(frame)
      }
    }
    frame()
  }

  const useHint = () => {
    setShowHint(true)
    setHintsUsed(prev => prev + 1)
  }

  if (questions.length === 0) return null

  const currentQuestion = questions[currentIndex]
  const progress = ((currentIndex + 1) / questions.length) * 100

  if (isComplete) {
    return (
      <QuizResults
        score={score}
        maxScore={questions.reduce((acc, q) => acc + q.points, 0)}
        correctCount={Math.round((score / questions.reduce((acc, q) => acc + q.points, 0)) * questions.length)}
        totalQuestions={questions.length}
        maxCombo={maxCombo}
        hintsUsed={hintsUsed}
        onRetry={allowRetry ? () => {
          setCurrentIndex(0)
          setScore(0)
          setCombo(0)
          setMaxCombo(0)
          setHintsUsed(0)
          setIsComplete(false)
          setSelectedAnswer(null)
          setIsRevealed(false)
          setTimeLeft(timeLimit)
        } : undefined}
      />
    )
  }

  return (
    <div className="gamified-quiz">
      {/* Header con progreso */}
      <div className="quiz-header">
        <h3>{title}</h3>
        <div className="quiz-stats">
          <span className="score">⭐ {score}</span>
          {combo > 1 && (
            <motion.span
              className="combo"
              initial={{ scale: 0 }}
              animate={{ scale: 1 }}
              key={combo}
            >
              🔥 x{combo}
            </motion.span>
          )}
        </div>
      </div>

      {/* Barra de progreso */}
      <div className="progress-bar-container">
        <motion.div
          className="progress-bar"
          initial={{ width: 0 }}
          animate={{ width: `${progress}%` }}
          transition={{ duration: 0.3 }}
        />
        <span className="progress-text">
          {currentIndex + 1} / {questions.length}
        </span>
      </div>

      {/* Timer */}
      {timeLimit > 0 && (
        <div className="timer-container">
          <motion.div
            className="timer-bar"
            initial={{ width: '100%' }}
            animate={{
              width: `${(timeLeft / timeLimit) * 100}%`,
              backgroundColor: timeLeft < 10 ? '#f44336' : timeLeft < 20 ? '#FF9800' : '#4CAF50'
            }}
            transition={{ duration: 0.5 }}
          />
          <span className={`timer-text ${timeLeft < 10 ? 'urgent' : ''}`}>
            ⏱️ {timeLeft}s
          </span>
        </div>
      )}

      {/* Pregunta */}
      <AnimatePresence mode="wait">
        <motion.div
          key={currentIndex}
          className="question-container"
          initial={{ opacity: 0, x: 50 }}
          animate={{ opacity: 1, x: 0 }}
          exit={{ opacity: 0, x: -50 }}
          transition={{ duration: 0.3 }}
        >
          {/* Dificultad */}
          <div className={`difficulty-badge ${currentQuestion.difficulty}`}>
            {currentQuestion.difficulty === 'easy' && '🟢 Fácil'}
            {currentQuestion.difficulty === 'medium' && '🟡 Media'}
            {currentQuestion.difficulty === 'hard' && '🔴 Difícil'}
            <span className="points">+{currentQuestion.points} pts</span>
          </div>

          {/* Imagen opcional */}
          {currentQuestion.image && (
            <img
              src={currentQuestion.image}
              alt="Imagen de la pregunta"
              className="question-image"
            />
          )}

          {/* Texto de la pregunta */}
          <h4 className="question-text">
            {currentQuestion.text}
          </h4>

          {/* Fórmula LaTeX opcional */}
          {currentQuestion.latex && (
            <div className="question-latex">
              <BlockMath math={currentQuestion.latex} />
            </div>
          )}

          {/* Opciones */}
          <div className="options-container">
            {currentQuestion.options.map((option, index) => (
              <motion.button
                key={index}
                className={`option-button ${
                  isRevealed
                    ? index === currentQuestion.correctIndex
                      ? 'correct'
                      : selectedAnswer === index
                      ? 'incorrect'
                      : 'dimmed'
                    : selectedAnswer === index
                    ? 'selected'
                    : ''
                }`}
                onClick={() => handleSelectAnswer(index)}
                disabled={isRevealed}
                whileHover={!isRevealed ? { scale: 1.02 } : {}}
                whileTap={!isRevealed ? { scale: 0.98 } : {}}
                initial={{ opacity: 0, y: 20 }}
                animate={{ opacity: 1, y: 0 }}
                transition={{ delay: index * 0.1 }}
              >
                <span className="option-letter">
                  {String.fromCharCode(65 + index)}
                </span>
                <span className="option-text">{option}</span>
                {isRevealed && index === currentQuestion.correctIndex && (
                  <span className="correct-icon">✓</span>
                )}
                {isRevealed && selectedAnswer === index && index !== currentQuestion.correctIndex && (
                  <span className="incorrect-icon">✗</span>
                )}
              </motion.button>
            ))}
          </div>

          {/* Pista */}
          {showHints && currentQuestion.hint && !isRevealed && (
            <button
              className="hint-button"
              onClick={useHint}
              disabled={showHint}
            >
              {showHint ? `💡 ${currentQuestion.hint}` : '💡 Ver pista (-50% puntos)'}
            </button>
          )}

          {/* Explicación tras responder */}
          <AnimatePresence>
            {isRevealed && (
              <motion.div
                className={`explanation ${selectedAnswer === currentQuestion.correctIndex ? 'correct' : 'incorrect'}`}
                initial={{ opacity: 0, height: 0 }}
                animate={{ opacity: 1, height: 'auto' }}
                exit={{ opacity: 0, height: 0 }}
              >
                <p className="explanation-header">
                  {selectedAnswer === currentQuestion.correctIndex
                    ? '✅ ¡Correcto!'
                    : selectedAnswer === -1
                    ? '⏰ ¡Tiempo agotado!'
                    : '❌ Incorrecto'}
                </p>
                <p className="explanation-text">{currentQuestion.explanation}</p>
              </motion.div>
            )}
          </AnimatePresence>
        </motion.div>
      </AnimatePresence>

      {/* Botón siguiente */}
      {isRevealed && (
        <motion.button
          className="next-button"
          onClick={handleNext}
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          whileHover={{ scale: 1.05 }}
          whileTap={{ scale: 0.95 }}
        >
          {currentIndex < questions.length - 1 ? 'Siguiente →' : '🏁 Ver resultados'}
        </motion.button>
      )}
    </div>
  )
}
```

### Pantalla de Resultados

```tsx
interface QuizResultsProps {
  score: number
  maxScore: number
  correctCount: number
  totalQuestions: number
  maxCombo: number
  hintsUsed: number
  onRetry?: () => void
}

const QuizResults: React.FC<QuizResultsProps> = ({
  score,
  maxScore,
  correctCount,
  totalQuestions,
  maxCombo,
  hintsUsed,
  onRetry,
}) => {
  const percentage = Math.round((correctCount / totalQuestions) * 100)
  const { level, xp } = useGamificationStore()

  // Determinar rango/estrella
  const getStars = () => {
    if (percentage === 100) return 3
    if (percentage >= 70) return 2
    if (percentage >= 50) return 1
    return 0
  }

  const stars = getStars()

  const getMessage = () => {
    if (percentage === 100) return '¡PERFECTO! 🏆'
    if (percentage >= 80) return '¡Excelente! 🌟'
    if (percentage >= 60) return '¡Muy bien! 👏'
    if (percentage >= 40) return 'Buen intento 💪'
    return 'Sigue practicando 📚'
  }

  return (
    <motion.div
      className="quiz-results"
      initial={{ opacity: 0, scale: 0.8 }}
      animate={{ opacity: 1, scale: 1 }}
      transition={{ type: 'spring', duration: 0.5 }}
    >
      {/* Mensaje principal */}
      <h2 className="results-message">{getMessage()}</h2>

      {/* Estrellas */}
      <div className="stars-container">
        {[1, 2, 3].map((star) => (
          <motion.span
            key={star}
            className={`star ${star <= stars ? 'earned' : 'empty'}`}
            initial={{ scale: 0, rotate: -180 }}
            animate={{ scale: 1, rotate: 0 }}
            transition={{ delay: star * 0.2, type: 'spring' }}
          >
            {star <= stars ? '⭐' : '☆'}
          </motion.span>
        ))}
      </div>

      {/* Puntuación circular */}
      <div className="score-circle">
        <svg viewBox="0 0 100 100">
          <circle
            cx="50"
            cy="50"
            r="45"
            fill="none"
            stroke="#e0e0e0"
            strokeWidth="8"
          />
          <motion.circle
            cx="50"
            cy="50"
            r="45"
            fill="none"
            stroke={percentage >= 70 ? '#4CAF50' : percentage >= 40 ? '#FF9800' : '#f44336'}
            strokeWidth="8"
            strokeLinecap="round"
            strokeDasharray={`${2 * Math.PI * 45}`}
            strokeDashoffset={2 * Math.PI * 45}
            initial={{ strokeDashoffset: 2 * Math.PI * 45 }}
            animate={{ strokeDashoffset: 2 * Math.PI * 45 * (1 - percentage / 100) }}
            transition={{ duration: 1.5, ease: 'easeOut' }}
            transform="rotate(-90 50 50)"
          />
        </svg>
        <div className="score-text">
          <motion.span
            className="percentage"
            initial={{ opacity: 0 }}
            animate={{ opacity: 1 }}
            transition={{ delay: 0.5 }}
          >
            {percentage}%
          </motion.span>
          <span className="fraction">{correctCount}/{totalQuestions}</span>
        </div>
      </div>

      {/* Estadísticas */}
      <div className="stats-grid">
        <div className="stat-item">
          <span className="stat-icon">⭐</span>
          <span className="stat-value">{score}</span>
          <span className="stat-label">Puntos</span>
        </div>
        <div className="stat-item">
          <span className="stat-icon">🔥</span>
          <span className="stat-value">x{maxCombo}</span>
          <span className="stat-label">Mejor combo</span>
        </div>
        <div className="stat-item">
          <span className="stat-icon">💡</span>
          <span className="stat-value">{hintsUsed}</span>
          <span className="stat-label">Pistas</span>
        </div>
        <div className="stat-item">
          <span className="stat-icon">📊</span>
          <span className="stat-value">Nv. {level}</span>
          <span className="stat-label">{xp} XP</span>
        </div>
      </div>

      {/* Botones de acción */}
      <div className="results-actions">
        {onRetry && (
          <motion.button
            className="retry-button"
            onClick={onRetry}
            whileHover={{ scale: 1.05 }}
            whileTap={{ scale: 0.95 }}
          >
            🔄 Intentar de nuevo
          </motion.button>
        )}
        <motion.button
          className="continue-button"
          whileHover={{ scale: 1.05 }}
          whileTap={{ scale: 0.95 }}
        >
          ➡️ Siguiente tema
        </motion.button>
      </div>
    </motion.div>
  )
}
```

---

## Tipos de Ejercicios Gamificados

### 1. Arrastrar y Soltar (Drag & Drop)

```tsx
import { DndContext, useDraggable, useDroppable } from '@dnd-kit/core'

interface DragDropExerciseProps {
  items: { id: string; content: string }[]
  targets: { id: string; label: string; correctItemId: string }[]
  instruction: string
  onComplete: (correct: number, total: number) => void
}

export const DragDropExercise: React.FC<DragDropExerciseProps> = ({
  items,
  targets,
  instruction,
  onComplete,
}) => {
  const [placements, setPlacements] = useState<Record<string, string>>({})
  const [isChecked, setIsChecked] = useState(false)
  const { addXP } = useGamificationStore()

  const handleDragEnd = (event: DragEndEvent) => {
    const { active, over } = event
    if (over) {
      setPlacements(prev => ({
        ...prev,
        [over.id]: active.id as string,
      }))
    }
  }

  const checkAnswers = () => {
    setIsChecked(true)
    const correct = targets.filter(t => placements[t.id] === t.correctItemId).length
    addXP(correct * 15)
    onComplete(correct, targets.length)
  }

  return (
    <div className="drag-drop-exercise">
      <p className="instruction">{instruction}</p>

      <DndContext onDragEnd={handleDragEnd}>
        {/* Elementos arrastrables */}
        <div className="draggable-items">
          {items
            .filter(item => !Object.values(placements).includes(item.id))
            .map(item => (
              <DraggableItem key={item.id} id={item.id}>
                {item.content}
              </DraggableItem>
            ))}
        </div>

        {/* Zonas de destino */}
        <div className="drop-targets">
          {targets.map(target => (
            <DroppableTarget
              key={target.id}
              id={target.id}
              label={target.label}
              isCorrect={isChecked && placements[target.id] === target.correctItemId}
              isIncorrect={isChecked && placements[target.id] && placements[target.id] !== target.correctItemId}
            >
              {placements[target.id] && (
                <span>{items.find(i => i.id === placements[target.id])?.content}</span>
              )}
            </DroppableTarget>
          ))}
        </div>
      </DndContext>

      {!isChecked && Object.keys(placements).length === targets.length && (
        <motion.button
          className="check-button"
          onClick={checkAnswers}
          whileHover={{ scale: 1.05 }}
        >
          ✓ Comprobar
        </motion.button>
      )}
    </div>
  )
}
```

### 2. Completar Huecos (Fill in the Blanks)

```tsx
interface FillBlanksExerciseProps {
  text: string // Usa {{blank:id:answer}} para huecos
  hints?: Record<string, string>
  caseSensitive?: boolean
  onComplete: (correct: number, total: number) => void
}

export const FillBlanksExercise: React.FC<FillBlanksExerciseProps> = ({
  text,
  hints = {},
  caseSensitive = false,
  onComplete,
}) => {
  const [answers, setAnswers] = useState<Record<string, string>>({})
  const [isChecked, setIsChecked] = useState(false)
  const [results, setResults] = useState<Record<string, boolean>>({})

  // Parsear texto y extraer huecos
  const blanks = useMemo(() => {
    const regex = /\{\{blank:(\w+):([^}]+)\}\}/g
    const matches: { id: string; answer: string; index: number }[] = []
    let match
    while ((match = regex.exec(text)) !== null) {
      matches.push({ id: match[1], answer: match[2], index: match.index })
    }
    return matches
  }, [text])

  const checkAnswers = () => {
    setIsChecked(true)
    const newResults: Record<string, boolean> = {}

    blanks.forEach(blank => {
      const userAnswer = answers[blank.id] || ''
      const correct = caseSensitive
        ? userAnswer.trim() === blank.answer
        : userAnswer.trim().toLowerCase() === blank.answer.toLowerCase()
      newResults[blank.id] = correct
    })

    setResults(newResults)
    const correctCount = Object.values(newResults).filter(Boolean).length
    onComplete(correctCount, blanks.length)
  }

  // Renderizar texto con inputs
  const renderText = () => {
    let lastIndex = 0
    const elements: React.ReactNode[] = []

    blanks.forEach((blank, i) => {
      // Texto antes del hueco
      elements.push(
        <span key={`text-${i}`}>
          {text.slice(lastIndex, blank.index)}
        </span>
      )

      // Input del hueco
      elements.push(
        <span key={`blank-${blank.id}`} className="blank-container">
          <input
            type="text"
            className={`blank-input ${
              isChecked ? (results[blank.id] ? 'correct' : 'incorrect') : ''
            }`}
            value={answers[blank.id] || ''}
            onChange={(e) => setAnswers(prev => ({ ...prev, [blank.id]: e.target.value }))}
            disabled={isChecked}
            placeholder={hints[blank.id] || '...'}
          />
          {isChecked && !results[blank.id] && (
            <span className="correct-answer">({blank.answer})</span>
          )}
        </span>
      )

      lastIndex = blank.index + `{{blank:${blank.id}:${blank.answer}}}`.length
    })

    // Texto final
    elements.push(<span key="text-end">{text.slice(lastIndex)}</span>)

    return elements
  }

  return (
    <div className="fill-blanks-exercise">
      <p className="exercise-text">{renderText()}</p>

      {!isChecked && (
        <motion.button
          className="check-button"
          onClick={checkAnswers}
          whileHover={{ scale: 1.05 }}
          disabled={Object.keys(answers).length < blanks.length}
        >
          ✓ Comprobar
        </motion.button>
      )}
    </div>
  )
}
```

### 3. Ordenar Secuencia

```tsx
import { DndContext, closestCenter } from '@dnd-kit/core'
import { SortableContext, verticalListSortingStrategy, useSortable } from '@dnd-kit/sortable'
import { CSS } from '@dnd-kit/utilities'

interface OrderExerciseProps {
  items: { id: string; content: string }[]
  correctOrder: string[] // IDs en orden correcto
  instruction: string
  onComplete: (isCorrect: boolean) => void
}

export const OrderExercise: React.FC<OrderExerciseProps> = ({
  items: initialItems,
  correctOrder,
  instruction,
  onComplete,
}) => {
  const [items, setItems] = useState(() =>
    [...initialItems].sort(() => Math.random() - 0.5)
  )
  const [isChecked, setIsChecked] = useState(false)
  const [isCorrect, setIsCorrect] = useState(false)

  const handleDragEnd = (event: DragEndEvent) => {
    const { active, over } = event
    if (!over || active.id === over.id) return

    setItems((items) => {
      const oldIndex = items.findIndex(i => i.id === active.id)
      const newIndex = items.findIndex(i => i.id === over.id)
      return arrayMove(items, oldIndex, newIndex)
    })
  }

  const checkOrder = () => {
    setIsChecked(true)
    const currentOrder = items.map(i => i.id)
    const correct = JSON.stringify(currentOrder) === JSON.stringify(correctOrder)
    setIsCorrect(correct)
    onComplete(correct)
  }

  return (
    <div className="order-exercise">
      <p className="instruction">{instruction}</p>

      <DndContext collisionDetection={closestCenter} onDragEnd={handleDragEnd}>
        <SortableContext items={items.map(i => i.id)} strategy={verticalListSortingStrategy}>
          <div className="sortable-list">
            {items.map((item, index) => (
              <SortableItem
                key={item.id}
                id={item.id}
                disabled={isChecked}
                isCorrect={isChecked && correctOrder[index] === item.id}
                isIncorrect={isChecked && correctOrder[index] !== item.id}
              >
                <span className="order-number">{index + 1}</span>
                <span className="order-content">{item.content}</span>
              </SortableItem>
            ))}
          </div>
        </SortableContext>
      </DndContext>

      {!isChecked && (
        <motion.button className="check-button" onClick={checkOrder}>
          ✓ Comprobar orden
        </motion.button>
      )}

      {isChecked && (
        <motion.div
          className={`result-message ${isCorrect ? 'correct' : 'incorrect'}`}
          initial={{ opacity: 0, y: 10 }}
          animate={{ opacity: 1, y: 0 }}
        >
          {isCorrect ? '✅ ¡Orden correcto!' : '❌ Revisa el orden'}
        </motion.div>
      )}
    </div>
  )
}
```

### 4. Emparejar (Matching)

```tsx
interface MatchingExerciseProps {
  pairs: { left: string; right: string }[]
  leftTitle?: string
  rightTitle?: string
  onComplete: (correct: number, total: number) => void
}

export const MatchingExercise: React.FC<MatchingExerciseProps> = ({
  pairs,
  leftTitle = 'Concepto',
  rightTitle = 'Definición',
  onComplete,
}) => {
  const [selectedLeft, setSelectedLeft] = useState<number | null>(null)
  const [matches, setMatches] = useState<Record<number, number>>({})
  const [isChecked, setIsChecked] = useState(false)

  // Mezclar el lado derecho
  const shuffledRight = useMemo(() =>
    pairs.map((_, i) => i).sort(() => Math.random() - 0.5),
    [pairs]
  )

  const handleLeftClick = (index: number) => {
    if (isChecked || matches[index] !== undefined) return
    setSelectedLeft(index)
  }

  const handleRightClick = (rightIndex: number) => {
    if (isChecked || selectedLeft === null) return
    if (Object.values(matches).includes(rightIndex)) return

    setMatches(prev => ({ ...prev, [selectedLeft]: rightIndex }))
    setSelectedLeft(null)
  }

  const checkMatches = () => {
    setIsChecked(true)
    let correct = 0
    Object.entries(matches).forEach(([left, right]) => {
      if (shuffledRight[right] === parseInt(left)) {
        correct++
      }
    })
    onComplete(correct, pairs.length)
  }

  return (
    <div className="matching-exercise">
      <div className="matching-columns">
        <div className="left-column">
          <h4>{leftTitle}</h4>
          {pairs.map((pair, index) => (
            <motion.div
              key={`left-${index}`}
              className={`match-item left ${
                selectedLeft === index ? 'selected' : ''
              } ${matches[index] !== undefined ? 'matched' : ''} ${
                isChecked && matches[index] !== undefined
                  ? shuffledRight[matches[index]] === index
                    ? 'correct'
                    : 'incorrect'
                  : ''
              }`}
              onClick={() => handleLeftClick(index)}
              whileHover={!isChecked && matches[index] === undefined ? { scale: 1.02 } : {}}
            >
              {pair.left}
            </motion.div>
          ))}
        </div>

        <div className="right-column">
          <h4>{rightTitle}</h4>
          {shuffledRight.map((originalIndex, displayIndex) => (
            <motion.div
              key={`right-${displayIndex}`}
              className={`match-item right ${
                Object.values(matches).includes(displayIndex) ? 'matched' : ''
              }`}
              onClick={() => handleRightClick(displayIndex)}
              whileHover={
                !isChecked && !Object.values(matches).includes(displayIndex)
                  ? { scale: 1.02 }
                  : {}
              }
            >
              {pairs[originalIndex].right}
            </motion.div>
          ))}
        </div>
      </div>

      {/* Líneas de conexión SVG */}
      <svg className="connection-lines">
        {Object.entries(matches).map(([left, right]) => (
          <line
            key={`line-${left}-${right}`}
            className={
              isChecked
                ? shuffledRight[right] === parseInt(left)
                  ? 'correct'
                  : 'incorrect'
                : ''
            }
          />
        ))}
      </svg>

      {!isChecked && Object.keys(matches).length === pairs.length && (
        <motion.button className="check-button" onClick={checkMatches}>
          ✓ Comprobar
        </motion.button>
      )}
    </div>
  )
}
```

### 5. Verdadero/Falso con Explicación

```tsx
interface TrueFalseQuestion {
  statement: string
  isTrue: boolean
  explanation: string
}

interface TrueFalseExerciseProps {
  questions: TrueFalseQuestion[]
  onComplete: (correct: number, total: number) => void
}

export const TrueFalseExercise: React.FC<TrueFalseExerciseProps> = ({
  questions,
  onComplete,
}) => {
  const [answers, setAnswers] = useState<Record<number, boolean | null>>({})
  const [isChecked, setIsChecked] = useState(false)

  const handleAnswer = (index: number, answer: boolean) => {
    if (isChecked) return
    setAnswers(prev => ({ ...prev, [index]: answer }))
  }

  const checkAnswers = () => {
    setIsChecked(true)
    const correct = questions.filter((q, i) => answers[i] === q.isTrue).length
    onComplete(correct, questions.length)
  }

  return (
    <div className="true-false-exercise">
      {questions.map((question, index) => (
        <motion.div
          key={index}
          className="tf-question"
          initial={{ opacity: 0, y: 20 }}
          animate={{ opacity: 1, y: 0 }}
          transition={{ delay: index * 0.1 }}
        >
          <p className="statement">{question.statement}</p>

          <div className="tf-buttons">
            <motion.button
              className={`tf-button true ${
                answers[index] === true ? 'selected' : ''
              } ${
                isChecked && question.isTrue ? 'correct' : ''
              } ${
                isChecked && answers[index] === true && !question.isTrue ? 'incorrect' : ''
              }`}
              onClick={() => handleAnswer(index, true)}
              whileHover={!isChecked ? { scale: 1.05 } : {}}
              whileTap={!isChecked ? { scale: 0.95 } : {}}
            >
              ✓ Verdadero
            </motion.button>

            <motion.button
              className={`tf-button false ${
                answers[index] === false ? 'selected' : ''
              } ${
                isChecked && !question.isTrue ? 'correct' : ''
              } ${
                isChecked && answers[index] === false && question.isTrue ? 'incorrect' : ''
              }`}
              onClick={() => handleAnswer(index, false)}
              whileHover={!isChecked ? { scale: 1.05 } : {}}
              whileTap={!isChecked ? { scale: 0.95 } : {}}
            >
              ✗ Falso
            </motion.button>
          </div>

          {isChecked && (
            <motion.div
              className="explanation"
              initial={{ opacity: 0, height: 0 }}
              animate={{ opacity: 1, height: 'auto' }}
            >
              <span className={answers[index] === question.isTrue ? 'correct' : 'incorrect'}>
                {answers[index] === question.isTrue ? '✅' : '❌'}
              </span>
              {question.explanation}
            </motion.div>
          )}
        </motion.div>
      ))}

      {!isChecked && Object.keys(answers).length === questions.length && (
        <motion.button className="check-button" onClick={checkAnswers}>
          ✓ Comprobar todas
        </motion.button>
      )}
    </div>
  )
}
```

---

## Sistema de Logros y Badges

### Panel de Logros

```tsx
export const AchievementsPanel: React.FC = () => {
  const { achievements, unlockedAchievements } = useGamificationStore()

  const categories = ['mastery', 'streak', 'explorer', 'speed', 'perfect'] as const
  const categoryNames = {
    mastery: '🎯 Maestría',
    streak: '🔥 Rachas',
    explorer: '🗺️ Explorador',
    speed: '⚡ Velocidad',
    perfect: '💯 Perfección',
  }

  return (
    <div className="achievements-panel">
      <h2>🏆 Logros</h2>
      <p className="achievement-count">
        {unlockedAchievements.length} / {achievements.length} desbloqueados
      </p>

      {categories.map(category => (
        <div key={category} className="achievement-category">
          <h3>{categoryNames[category]}</h3>
          <div className="achievements-grid">
            {achievements
              .filter(a => a.category === category)
              .map(achievement => {
                const isUnlocked = unlockedAchievements.includes(achievement.id)
                return (
                  <motion.div
                    key={achievement.id}
                    className={`achievement-card ${isUnlocked ? 'unlocked' : 'locked'}`}
                    whileHover={{ scale: 1.05 }}
                  >
                    <span className="achievement-icon">
                      {isUnlocked ? achievement.icon : '🔒'}
                    </span>
                    <span className="achievement-name">{achievement.name}</span>
                    <span className="achievement-desc">
                      {isUnlocked ? achievement.description : '???'}
                    </span>
                  </motion.div>
                )
              })}
          </div>
        </div>
      ))}
    </div>
  )
}
```

### Notificación de Logro

```tsx
export const AchievementNotification: React.FC<{ achievement: Achievement }> = ({
  achievement,
}) => {
  return (
    <motion.div
      className="achievement-notification"
      initial={{ x: 300, opacity: 0 }}
      animate={{ x: 0, opacity: 1 }}
      exit={{ x: 300, opacity: 0 }}
      transition={{ type: 'spring', damping: 15 }}
    >
      <div className="notification-content">
        <span className="notification-icon">{achievement.icon}</span>
        <div className="notification-text">
          <span className="notification-title">🎉 ¡Logro desbloqueado!</span>
          <span className="notification-name">{achievement.name}</span>
          <span className="notification-desc">{achievement.description}</span>
        </div>
      </div>
      <div className="notification-xp">+50 XP</div>
    </motion.div>
  )
}
```

---

## Barra de Progreso y Nivel

```tsx
export const ProgressBar: React.FC = () => {
  const { xp, level } = useGamificationStore()

  const currentLevelXP = Math.pow(level - 1, 2) * 100
  const nextLevelXP = Math.pow(level, 2) * 100
  const progressXP = xp - currentLevelXP
  const neededXP = nextLevelXP - currentLevelXP
  const percentage = (progressXP / neededXP) * 100

  return (
    <div className="level-progress">
      <div className="level-badge">
        <span className="level-number">{level}</span>
      </div>

      <div className="progress-container">
        <div className="progress-info">
          <span>Nivel {level}</span>
          <span>{progressXP} / {neededXP} XP</span>
        </div>
        <div className="progress-bar-bg">
          <motion.div
            className="progress-bar-fill"
            initial={{ width: 0 }}
            animate={{ width: `${percentage}%` }}
            transition={{ duration: 0.5 }}
          />
        </div>
      </div>

      <div className="next-level">
        <span>→ Nv. {level + 1}</span>
      </div>
    </div>
  )
}
```

---

## Estilos CSS para Gamificación

```css
/* Quiz gamificado */
.gamified-quiz {
  background: white;
  border-radius: var(--radius-lg);
  padding: var(--spacing-lg);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.1);
}

.quiz-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: var(--spacing-md);
}

.quiz-stats {
  display: flex;
  gap: var(--spacing-md);
  align-items: center;
}

.score {
  font-size: 1.25rem;
  font-weight: 700;
  color: var(--color-primary);
}

.combo {
  background: linear-gradient(135deg, #FF6B00, #FF9800);
  color: white;
  padding: 0.25rem 0.75rem;
  border-radius: 20px;
  font-weight: 700;
  animation: pulse 0.5s ease-in-out;
}

@keyframes pulse {
  0%, 100% { transform: scale(1); }
  50% { transform: scale(1.2); }
}

/* Timer */
.timer-container {
  position: relative;
  height: 8px;
  background: #e0e0e0;
  border-radius: 4px;
  margin-bottom: var(--spacing-md);
  overflow: hidden;
}

.timer-bar {
  height: 100%;
  border-radius: 4px;
  transition: background-color 0.3s;
}

.timer-text {
  position: absolute;
  right: 8px;
  top: -24px;
  font-size: 0.875rem;
  font-weight: 600;
}

.timer-text.urgent {
  color: #f44336;
  animation: blink 0.5s infinite;
}

@keyframes blink {
  0%, 100% { opacity: 1; }
  50% { opacity: 0.5; }
}

/* Opciones */
.option-button {
  display: flex;
  align-items: center;
  gap: var(--spacing-md);
  width: 100%;
  padding: var(--spacing-md);
  border: 2px solid #e0e0e0;
  border-radius: var(--radius-md);
  background: white;
  cursor: pointer;
  transition: all 0.2s;
  text-align: left;
}

.option-button:hover:not(:disabled) {
  border-color: var(--color-primary);
  background: rgba(76, 175, 80, 0.05);
}

.option-button.selected {
  border-color: var(--color-primary);
  background: rgba(76, 175, 80, 0.1);
}

.option-button.correct {
  border-color: #4CAF50;
  background: rgba(76, 175, 80, 0.2);
}

.option-button.incorrect {
  border-color: #f44336;
  background: rgba(244, 67, 54, 0.1);
}

.option-button.dimmed {
  opacity: 0.5;
}

.option-letter {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: #f5f5f5;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 700;
  color: #666;
}

/* Dificultad */
.difficulty-badge {
  display: inline-flex;
  align-items: center;
  gap: 0.5rem;
  padding: 0.25rem 0.75rem;
  border-radius: 20px;
  font-size: 0.75rem;
  font-weight: 600;
  margin-bottom: var(--spacing-sm);
}

.difficulty-badge.easy {
  background: rgba(76, 175, 80, 0.1);
  color: #4CAF50;
}

.difficulty-badge.medium {
  background: rgba(255, 152, 0, 0.1);
  color: #FF9800;
}

.difficulty-badge.hard {
  background: rgba(244, 67, 54, 0.1);
  color: #f44336;
}

/* Resultados */
.quiz-results {
  text-align: center;
  padding: var(--spacing-xl);
}

.stars-container {
  display: flex;
  justify-content: center;
  gap: var(--spacing-md);
  margin: var(--spacing-lg) 0;
}

.star {
  font-size: 3rem;
}

.star.empty {
  opacity: 0.3;
  filter: grayscale(1);
}

.score-circle {
  position: relative;
  width: 150px;
  height: 150px;
  margin: var(--spacing-lg) auto;
}

.score-circle svg {
  width: 100%;
  height: 100%;
}

.score-text {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  text-align: center;
}

.score-text .percentage {
  display: block;
  font-size: 2rem;
  font-weight: 700;
}

.score-text .fraction {
  font-size: 0.875rem;
  color: #666;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: var(--spacing-md);
  margin: var(--spacing-lg) 0;
}

.stat-item {
  text-align: center;
}

.stat-icon {
  font-size: 1.5rem;
}

.stat-value {
  display: block;
  font-size: 1.25rem;
  font-weight: 700;
}

.stat-label {
  font-size: 0.75rem;
  color: #666;
}

/* Logros */
.achievement-notification {
  position: fixed;
  top: 20px;
  right: 20px;
  background: linear-gradient(135deg, #FFD700, #FFA500);
  color: #333;
  padding: var(--spacing-md);
  border-radius: var(--radius-lg);
  box-shadow: 0 8px 32px rgba(255, 165, 0, 0.4);
  z-index: 1000;
}

.achievement-card {
  padding: var(--spacing-md);
  border-radius: var(--radius-md);
  text-align: center;
  transition: all 0.3s;
}

.achievement-card.unlocked {
  background: linear-gradient(135deg, rgba(255, 215, 0, 0.1), rgba(255, 165, 0, 0.1));
  border: 2px solid #FFD700;
}

.achievement-card.locked {
  background: #f5f5f5;
  border: 2px solid #e0e0e0;
  opacity: 0.6;
}

.achievement-icon {
  font-size: 2rem;
  display: block;
  margin-bottom: var(--spacing-sm);
}

/* Nivel y progreso */
.level-progress {
  display: flex;
  align-items: center;
  gap: var(--spacing-md);
  padding: var(--spacing-md);
  background: white;
  border-radius: var(--radius-lg);
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.1);
}

.level-badge {
  width: 48px;
  height: 48px;
  border-radius: 50%;
  background: linear-gradient(135deg, var(--color-primary), var(--color-primary-dark));
  display: flex;
  align-items: center;
  justify-content: center;
  color: white;
  font-weight: 700;
  font-size: 1.25rem;
}

.progress-container {
  flex: 1;
}

.progress-bar-bg {
  height: 12px;
  background: #e0e0e0;
  border-radius: 6px;
  overflow: hidden;
}

.progress-bar-fill {
  height: 100%;
  background: linear-gradient(90deg, var(--color-primary), #8BC34A);
  border-radius: 6px;
}
```

---

## Integración con Asignaturas

### Ejemplo: Quiz de Física Gamificado

```tsx
const physicsQuiz: Question[] = [
  {
    id: 'phy-1',
    text: '¿Cuál es la unidad de fuerza en el Sistema Internacional?',
    options: ['Julio (J)', 'Newton (N)', 'Pascal (Pa)', 'Watt (W)'],
    correctIndex: 1,
    explanation: 'El Newton (N) es la unidad de fuerza. 1 N = 1 kg·m/s²',
    difficulty: 'easy',
    points: 10,
    hint: 'Piensa en la segunda ley de Newton: F = m·a',
  },
  {
    id: 'phy-2',
    text: 'Si un objeto cae libremente, ¿cuál es su aceleración aproximada?',
    latex: 'g \\approx ?\\, \\text{m/s}^2',
    options: ['5 m/s²', '10 m/s²', '15 m/s²', '20 m/s²'],
    correctIndex: 1,
    explanation: 'La aceleración de la gravedad en la Tierra es aproximadamente 9.8 m/s² ≈ 10 m/s²',
    difficulty: 'easy',
    points: 10,
  },
  {
    id: 'phy-3',
    text: '¿Qué tipo de energía tiene un objeto en movimiento?',
    options: ['Energía potencial', 'Energía cinética', 'Energía térmica', 'Energía química'],
    correctIndex: 1,
    explanation: 'La energía cinética es la energía asociada al movimiento: Ec = ½mv²',
    difficulty: 'easy',
    points: 10,
    latex: 'E_c = \\frac{1}{2}mv^2',
  },
  {
    id: 'phy-4',
    text: 'Un coche acelera de 0 a 100 km/h en 10 segundos. ¿Cuál es su aceleración?',
    options: ['2.78 m/s²', '10 m/s²', '27.8 m/s²', '100 m/s²'],
    correctIndex: 0,
    explanation: '100 km/h = 27.78 m/s. a = Δv/Δt = 27.78/10 = 2.78 m/s²',
    difficulty: 'medium',
    points: 20,
    hint: 'Primero convierte km/h a m/s dividiendo entre 3.6',
  },
  {
    id: 'phy-5',
    text: '¿Cuánta energía potencial gravitatoria tiene un objeto de 5 kg a 20 m de altura?',
    latex: 'E_p = mgh',
    options: ['100 J', '500 J', '1000 J', '2000 J'],
    correctIndex: 2,
    explanation: 'Ep = mgh = 5 kg × 10 m/s² × 20 m = 1000 J',
    difficulty: 'medium',
    points: 20,
  },
]

// Uso
<GamifiedQuiz
  questions={physicsQuiz}
  topicId="physics-energy"
  title="Quiz de Energía y Movimiento"
  timeLimit={30}
  showHints={true}
/>
```

### Ejemplo: Ejercicio de Química con Drag & Drop

```tsx
const periodicTableExercise = (
  <DragDropExercise
    instruction="Arrastra cada elemento a su grupo correcto de la tabla periódica"
    items={[
      { id: 'na', content: 'Na (Sodio)' },
      { id: 'cl', content: 'Cl (Cloro)' },
      { id: 'he', content: 'He (Helio)' },
      { id: 'fe', content: 'Fe (Hierro)' },
    ]}
    targets={[
      { id: 'alkali', label: 'Metales alcalinos', correctItemId: 'na' },
      { id: 'halogen', label: 'Halógenos', correctItemId: 'cl' },
      { id: 'noble', label: 'Gases nobles', correctItemId: 'he' },
      { id: 'transition', label: 'Metales de transición', correctItemId: 'fe' },
    ]}
    onComplete={(correct, total) => console.log(`${correct}/${total} correctos`)}
  />
)
```

### Ejemplo: Matemáticas con Completar Huecos

```tsx
const algebraExercise = (
  <FillBlanksExercise
    text="Para resolver la ecuación 2x + {{blank:a:6}} = 14, primero restamos {{blank:b:6}} de ambos lados, obteniendo 2x = {{blank:c:8}}. Luego dividimos entre {{blank:d:2}}, resultando x = {{blank:e:4}}."
    hints={{
      a: 'número que se suma',
      b: 'lo mismo que restamos',
      c: '14 menos 6',
      d: 'coeficiente de x',
      e: 'resultado final',
    }}
    onComplete={(correct, total) => console.log(`${correct}/${total}`)}
  />
)
```

---

## Consideraciones de Diseño

### Accesibilidad
- Todos los elementos interactivos deben ser accesibles por teclado
- Usar `aria-live` para anunciar puntuaciones y feedback
- Proporcionar alternativas visuales para efectos de sonido
- Respetar `prefers-reduced-motion` para animaciones

### Motivación Equilibrada
- No penalizar en exceso los errores (aprender del fallo)
- Ofrecer múltiples intentos
- Mostrar progreso incluso en fallos parciales
- Evitar comparaciones competitivas obligatorias

### Persistencia
- Guardar progreso con Zustand persist
- Sincronizar entre dispositivos (opcional)
- Permitir "modo invitado" sin persistencia

---

## Dependencias Recomendadas

```json
{
  "dependencies": {
    "zustand": "^4.5.0",
    "framer-motion": "^11.0.0",
    "canvas-confetti": "^1.9.0",
    "@dnd-kit/core": "^6.1.0",
    "@dnd-kit/sortable": "^8.0.0",
    "react-katex": "^3.0.1"
  }
}
```
