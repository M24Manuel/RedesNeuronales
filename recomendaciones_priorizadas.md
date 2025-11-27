# Recomendaciones Priorizadas para xEriis/Compiladores

---

## 🎯 OBJETIVO

Transformar el proyecto de **excelente proyecto académico (95/100)** a **proyecto de nivel profesional (85/100 en estándares industriales)** mediante mejoras incrementales y priorizadas.

**Proyecto:** https://github.com/xEriis/Compiladores/tree/main/Proyecto
**Calificación Actual:** 95/100 (académico)
**Calificación Objetivo:** 85/100 (profesional)
**Tiempo Estimado Total:** 8-12 semanas

---

## 📊 MATRIZ DE PRIORIZACIÓN

| Mejora | Impacto | Esfuerzo | ROI | Prioridad |
|--------|---------|----------|-----|-----------|
| Tests Unitarios | CRÍTICO | Alto | 9/10 | P0 🔴 |
| Tracking Posición | CRÍTICO | Medio | 10/10 | P0 🔴 |
| README Completo | ALTO | Bajo | 10/10 | P0 🔴 |
| Errores Mejorados | ALTO | Alto | 7/10 | P1 🟠 |
| CI/CD | ALTO | Medio | 8/10 | P1 🟠 |
| Optimizaciones | MEDIO | Alto | 5/10 | P2 🟡 |
| Property Tests | MEDIO | Medio | 6/10 | P2 🟡 |
| Unicode | BAJO | Alto | 3/10 | P3 🟢 |
| Estados Léxicos | BAJO | Alto | 4/10 | P3 🟢 |

**ROI = Return on Investment (beneficio vs esfuerzo)**

---

## 🔴 PRIORIDAD P0 (CRÍTICA) - Semana 1-2

### P0.1: Tracking de Posición (Línea/Columna)

**Problema Actual:**
```haskell
error $ "Token no reconocido: " ++ take 10 s
-- Output: *** Exception: Token no reconocido: = x + 1
```

**No indica DÓNDE ocurrió el error.**

**Solución Implementada:**

```haskell
-- src/Position.hs
module Position
  ( Position(..)
  , initPosition
  , advanceChar
  , advanceLine
  ) where

data Position = Position
  { line :: !Int
  , column :: !Int
  , offset :: !Int
  } deriving (Show, Eq)

initPosition :: Position
initPosition = Position 1 1 0

advanceChar :: Position -> Position
advanceChar (Position l c o) = Position l (c + 1) (o + 1)

advanceLine :: Position -> Position
advanceLine (Position l _ o) = Position (l + 1) 1 (o + 1)
```

```haskell
-- src/Token.hs
module Token
  ( Token(..)
  , TokenType
  ) where

import Position

type TokenType = String

data Token = Token
  { tokenType :: !TokenType
  , lexeme :: !String
  , position :: !Position
  } deriving (Show, Eq)
```

```haskell
-- Modificar Lexer.hs
module Lexer (lexerM, LexError(..)) where

import Token
import Position

data LexError = LexError
  { errorPos :: Position
  , unexpected :: String
  , context :: String
  } deriving (Show)

lexerM :: MDD -> String -> Either LexError [Token]
lexerM mdd input = go mdd input initPosition
  where
    go :: MDD -> String -> Position -> Either LexError [Token]
    go _ [] _ = Right []

    go mdd s@(c:cs) pos
      | c == '\n' = go mdd cs (advanceLine pos)
      | isSpace c = go mdd cs (advanceChar pos)
      | otherwise = case prefijoMasLargo mdd s of
          Nothing -> Left $ LexError pos (take 20 s) (getContext s)
          Just (tokType, lex, rest) ->
            let token = Token tokType lex pos
                newPos = foldl' (\p _ -> advanceChar p) pos lex
            in fmap (token :) (go mdd rest newPos)

    getContext :: String -> String
    getContext s = take 40 s ++ if length s > 40 then "..." else ""
```

**Ejemplo de Output Mejorado:**
```
Error: Token no reconocido en línea 5, columna 7
  z = x + 1;
      ^
  Caracter inesperado: '='
```

**Tiempo:** 4-6 horas
**Impacto:** CRÍTICO - Base para todos los errores informativos
**Bloqueantes:** Ninguno

---

### P0.2: Tests Unitarios Básicos

**Problema Actual:**
```
test/
  └── .gitkeep  # Vacío, sin tests
```

**Riesgo:** Refactorizaciones pueden romper funcionalidad sin detectarlo

**Solución Paso a Paso:**

#### Paso 1: Configurar HSpec (30 min)

```yaml
# package.yaml - agregar en test
dependencies:
  - hspec
  - hspec-discover
  - QuickCheck

test-arguments: --format=progress
```

#### Paso 2: Tests de Lexer (2 horas)

```haskell
-- test/LexerSpec.hs
module LexerSpec (spec) where

import Test.Hspec
import Lexer
import MDD
import TestUtils (buildTestMDD)

spec :: Spec
spec = describe "Lexer" $ do
  let mdd = buildTestMDD

  describe "Tokens básicos" $ do
    it "reconoce identificadores simples" $
      lexerM mdd "x" `shouldBe` Right [Token "id" "x" (Position 1 1 0)]

    it "reconoce números" $
      lexerM mdd "123" `shouldBe` Right [Token "num" "123" (Position 1 1 0)]

    it "reconoce asignación" $
      lexerM mdd ":=" `shouldBe` Right [Token "asign" ":=" (Position 1 1 0)]

  describe "Múltiples tokens" $ do
    it "tokeniza expresión simple" $ do
      let result = lexerM mdd "x := 10"
      result `shouldBe` Right
        [ Token "id" "x" (Position 1 1 0)
        , Token "asign" ":=" (Position 1 3 2)
        , Token "num" "10" (Position 1 6 5)
        ]

  describe "Manejo de espacios" $ do
    it "ignora espacios en blanco" $
      lexerM mdd "x   :=   10" `shouldSatisfy` isRight

    it "maneja tabuladores" $
      lexerM mdd "x\t:=\t10" `shouldSatisfy` isRight

    it "maneja saltos de línea" $ do
      let result = lexerM mdd "x := 10\ny := 20"
      fmap length result `shouldBe` Right 6

  describe "Palabras reservadas" $ do
    it "distingue 'if' de 'ifa'" $ do
      lexerM mdd "if" `shouldBe` Right [Token "res_cond" "if" (Position 1 1 0)]
      lexerM mdd "ifa" `shouldBe` Right [Token "id" "ifa" (Position 1 1 0)]

    it "reconoce todas las keywords de ciclos" $ do
      lexerM mdd "while" `shouldSatisfy` hasTokenType "res_cicle"
      lexerM mdd "for" `shouldSatisfy` hasTokenType "res_cicle"
      lexerM mdd "do" `shouldSatisfy` hasTokenType "res_cicle"

  describe "Casos edge" $ do
    it "maneja entrada vacía" $
      lexerM mdd "" `shouldBe` Right []

    it "maneja solo espacios" $
      lexerM mdd "   \t\n  " `shouldBe` Right []

    it "detecta tokens inválidos" $
      lexerM mdd "$invalid" `shouldSatisfy` isLeft

-- Helper functions
hasTokenType :: String -> Either LexError [Token] -> Bool
hasTokenType expected (Right tokens) = any (\t -> tokenType t == expected) tokens
hasTokenType _ (Left _) = False

isRight :: Either a b -> Bool
isRight (Right _) = True
isRight _ = False

isLeft :: Either a b -> Bool
isLeft = not . isRight
```

#### Paso 3: Tests de Autómatas (2 horas)

```haskell
-- test/AFDSpec.hs
module AFDSpec (spec) where

import Test.Hspec
import AFD
import AFN
import AFNe
import Regex

spec :: Spec
spec = describe "Autómatas" $ do
  describe "Minimización AFD" $ do
    it "preserva el lenguaje" $ do
      let afd = buildSampleAFD
      let min = minimiza afd
      acepta "abc" afd `shouldBe` acepta "abc" min

    it "reduce estados equivalentes" $ do
      let afd = buildAFDWithRedundantStates
      let min = minimiza afd
      length (estadosD min) `shouldSatisfy` (< length (estadosD afd))

  describe "Conversión AFN → AFD" $ do
    it "genera AFD determinista" $ do
      let afn = buildSampleAFN
      let afd = afn_to_AFD afn
      isDeterministic afd `shouldBe` True

    it "preserva lenguaje" $ do
      let afn = buildSampleAFN
      let afd = afn_to_AFD afn
      -- Probar con varias cadenas
      all (\s -> aceptaAFN s afn == acepta s afd) testStrings `shouldBe` True

  describe "Conversión AFN-ε → AFN" $ do
    it "elimina transiciones epsilon" $ do
      let afne = buildSampleAFNe
      let afn = afnEp_to_AFN afne
      hasEpsilonTransitions afn `shouldBe` False

  describe "Regex → AFN-ε" $ do
    it "convierte Term correctamente" $ do
      let expr = Term 'a'
      let afne = regex_to_AFNe expr
      length (estados afne) `shouldBe` 2

    it "convierte Concat correctamente" $ do
      let expr = Concat (Term 'a') (Term 'b')
      let afne = regex_to_AFNe expr
      aceptaAFNe "ab" afne `shouldBe` True
      aceptaAFNe "a" afne `shouldBe` False

    it "convierte Kleene correctamente" $ do
      let expr = Kleene (Term 'a')
      let afne = regex_to_AFNe expr
      aceptaAFNe "" afne `shouldBe` True
      aceptaAFNe "aaa" afne `shouldBe` True
      aceptaAFNe "b" afne `shouldBe` False

-- Helpers
isDeterministic :: AFD -> Bool
isDeterministic afd = all uniqueTransition (estadosD afd)
  where
    uniqueTransition q = all (atMostOne q) (alfabetoD afd)
    atMostOne q sym = length (filter (\(q', s, _) -> q' == q && s == sym) (transicionesD afd)) <= 1
```

#### Paso 4: Tests de Comentarios (1 hora)

```haskell
-- test/CommentsSpec.hs
module CommentsSpec (spec) where

import Test.Hspec
import ReadFile (remove_comments_test)

spec :: Spec
spec = describe "Eliminación de comentarios" $ do
  describe "Comentarios de línea (//)" $ do
    it "elimina comentario simple" $
      remove_comments_test "x // comment" `shouldBe` "x "

    it "preserva código después de newline" $
      remove_comments_test "x // comment\ny" `shouldBe` "x \ny"

    it "maneja múltiples comentarios" $
      remove_comments_test "x // c1\ny // c2" `shouldBe` "x \ny "

  describe "Comentarios de bloque (/* */)" $ do
    it "elimina comentario de bloque simple" $
      remove_comments_test "x /* comment */ y" `shouldBe` "x  y"

    it "maneja comentarios multilínea" $
      remove_comments_test "x /* line1\nline2 */ y" `shouldBe` "x  y"

    it "maneja comentario sin cierre" $
      remove_comments_test "x /* unclosed" `shouldBe` "x "

  describe "Casos edge" $ do
    it "maneja operador división vs comentario" $
      remove_comments_test "a / b" `shouldBe` "a / b"

    it "no confunde */ en strings (si hubiera)" $
      remove_comments_test "x := \"*/\"" `shouldBe` "x := \"*/\""
```

#### Paso 5: Configurar test runner (30 min)

```bash
# stack.yaml - asegurar configuración
test:
  Proyecto-test:
    main: Spec.hs
    source-dirs: test
    ghc-options:
      - -Wall
      - -threaded
      - -rtsopts
      - -with-rtsopts=-N
    dependencies:
      - Proyecto
      - hspec
```

```haskell
-- test/Spec.hs (auto-discovery)
{-# OPTIONS_GHC -F -pgmF hspec-discover #-}
```

**Ejecutar:**
```bash
stack test
```

**Output esperado:**
```
Lexer
  Tokens básicos
    reconoce identificadores simples ✓
    reconoce números ✓
    reconoce asignación ✓
  Múltiples tokens
    tokeniza expresión simple ✓
  ...

Finished in 0.0234 seconds
45 examples, 0 failures
```

**Tiempo:** 6-8 horas
**Impacto:** CRÍTICO - Previene regresiones, documenta comportamiento
**Bloqueantes:** Ninguno

---

### P0.3: README Completo

**Problema Actual:**
```markdown
# Proyecto
```

**Solución:** Ver el README completo en `analisis_comparativo_detallado.md` sección 2.1.4

**Elementos Críticos del README:**
1. ✅ Descripción del proyecto (qué hace, por qué es especial)
2. ✅ Instalación paso a paso
3. ✅ Ejemplo de uso básico
4. ✅ Arquitectura (diagrama de pipeline)
5. ✅ Lista de tokens soportados
6. ✅ Cómo ejecutar tests
7. ✅ Estructura de directorios
8. ✅ Licencia y contribuciones

**Tiempo:** 2-3 horas
**Impacto:** ALTO - Primera impresión, onboarding
**Bloqueantes:** Ninguno

---

## 🟠 PRIORIDAD P1 (ALTA) - Semana 3-4

### P1.1: Manejo de Errores Mejorado

**Implementación:**

```haskell
-- src/LexError.hs
module LexError
  ( LexError(..)
  , formatError
  , withSuggestion
  ) where

import Position
import Data.List (minimumBy)
import Data.Function (on)

data LexError = LexError
  { errorPosition :: Position
  , unexpectedToken :: String
  , errorContext :: String
  , suggestion :: Maybe String
  } deriving (Eq)

instance Show LexError where
  show = formatError

-- Formato bonito del error
formatError :: LexError -> String
formatError (LexError pos unexp ctx sug) =
  unlines
    [ "Error léxico en línea " ++ show (line pos) ++ ", columna " ++ show (column pos) ++ ":"
    , "  " ++ contextLine
    , "  " ++ replicate (column pos - 1) ' ' ++ "^"
    , "  Token inesperado: " ++ show (take 10 unexp)
    , suggestionLine
    ]
  where
    contextLine = take 60 ctx
    suggestionLine = case sug of
      Just s -> "  Sugerencia: " ++ s
      Nothing -> ""

-- Calcular distancia de Levenshtein
levenshtein :: String -> String -> Int
levenshtein s1 s2 = dist !! length s1 !! length s2
  where
    dist = [[calcDist i j | j <- [0..length s2]] | i <- [0..length s1]]
    calcDist i 0 = i
    calcDist 0 j = j
    calcDist i j
      | s1 !! (i-1) == s2 !! (j-1) = dist !! (i-1) !! (j-1)
      | otherwise = 1 + minimum
          [ dist !! (i-1) !! j       -- deletion
          , dist !! i !! (j-1)       -- insertion
          , dist !! (i-1) !! (j-1)   -- substitution
          ]

-- Sugerir token similar
suggestToken :: String -> [String] -> Maybe String
suggestToken input validTokens =
  let distances = [(tok, levenshtein input tok) | tok <- validTokens]
      sorted = filter (\(_, d) -> d <= 2) distances
  in case sorted of
    [] -> Nothing
    _ -> Just $ fst $ minimumBy (compare `on` snd) sorted

-- Agregar sugerencia a un error
withSuggestion :: LexError -> [String] -> LexError
withSuggestion err@(LexError _ unexp _ _) validTokens =
  err { suggestion = fmap mkSuggestion (suggestToken (take 10 unexp) validTokens) }
  where
    mkSuggestion tok = "¿Quisiste decir '" ++ tok ++ "'?"
```

**Integración en Lexer:**
```haskell
-- Lexer.hs modificado
lexerM :: MDD -> String -> Either LexError [Token]
lexerM mdd input = go mdd input initPosition (lines input)
  where
    go _ [] _ _ = Right []
    go mdd s@(c:cs) pos allLines
      | ... -- similar a antes
      | otherwise = case prefijoMasLargo mdd s of
          Nothing ->
            let ctx = getContextFromLines pos allLines
                err = LexError pos (take 20 s) ctx Nothing
                validTokens = ["if", "else", "while", ":=", "+", "-", ...]
            in Left $ withSuggestion err validTokens
          Just (tokType, lex, rest) -> ...

getContextFromLines :: Position -> [String] -> String
getContextFromLines pos allLines =
  if line pos > 0 && line pos <= length allLines
    then allLines !! (line pos - 1)
    else ""
```

**Ejemplo de Output:**
```
Error léxico en línea 5, columna 3:
  z = x + 1;
    ^
  Token inesperado: "= x + 1;"
  Sugerencia: ¿Quisiste decir ':='?
```

**Tiempo:** 4-5 horas
**Impacto:** ALTO - Mejora significativa de UX

---

### P1.2: CI/CD con GitHub Actions

**Implementación:**

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build-and-test:
    name: Build and Test
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup Haskell
      uses: haskell/actions/setup@v2
      with:
        ghc-version: '9.2.5'
        enable-stack: true
        stack-version: 'latest'

    - name: Cache dependencies
      uses: actions/cache@v3
      with:
        path: |
          ~/.stack
          .stack-work
        key: ${{ runner.os }}-stack-${{ hashFiles('stack.yaml.lock') }}
        restore-keys: |
          ${{ runner.os }}-stack-

    - name: Build dependencies
      run: stack build --only-dependencies

    - name: Build project
      run: stack build --pedantic

    - name: Run tests
      run: stack test --coverage

    - name: Generate coverage report
      run: stack hpc report --all

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage/hpc_index.html

    - name: Check formatting
      run: |
        stack install ormolu
        ormolu --mode check $(find app src test -name '*.hs')

  lint:
    name: Lint
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v3
    - uses: haskell/actions/setup@v2
      with:
        ghc-version: '9.2.5'
        enable-stack: true

    - name: Run HLint
      run: |
        stack install hlint
        hlint app src test
```

**Badges para README:**
```markdown
[![CI](https://github.com/xEriis/Compiladores/workflows/CI/badge.svg)](https://github.com/xEriis/Compiladores/actions)
[![codecov](https://codecov.io/gh/xEriis/Compiladores/branch/main/graph/badge.svg)](https://codecov.io/gh/xEriis/Compiladores)
```

**Tiempo:** 3-4 horas
**Impacto:** ALTO - Automatización, calidad continua

---

## 🟡 PRIORIDAD P2 (MEDIA) - Semana 5-6

### P2.1: Property-Based Testing

```haskell
-- test/PropertySpec.hs
module PropertySpec (spec) where

import Test.Hspec
import Test.QuickCheck
import AFD
import Lexer

spec :: Spec
spec = describe "Properties" $ do
  describe "Minimización" $ do
    it "preserva lenguaje" $ property $
      \afd input -> acepta input afd == acepta input (minimiza afd)

    it "no aumenta estados" $ property $
      \afd -> length (estadosD (minimiza afd)) <= length (estadosD afd)

  describe "Lexer" $ do
    it "preserva longitud (sin espacios)" $ property $
      \input -> let tokens = lexerM testMDD input
                in case tokens of
                  Right ts -> sum (map (length . lexeme) ts) <= length input
                  Left _ -> True

    it "nunca retorna lexema vacío" $ property $
      \input -> case lexerM testMDD input of
                  Right ts -> all (not . null . lexeme) ts
                  Left _ -> True
```

**Tiempo:** 4-5 horas
**Impacto:** MEDIO - Mayor confianza en correctitud

---

### P2.2: Optimizaciones de Performance

**ByteString:**
```haskell
import qualified Data.ByteString.Char8 as BS

lexerM :: MDD -> BS.ByteString -> Either LexError [Token]
```

**Strict fields:**
```haskell
data MDD = MDD
  { estadosM :: ![String]
  , alfabetoM :: ![Char]
  , transicionesM :: ![TransM]
  , inicialM :: !String
  , finalesM :: ![(String, String)]
  }
```

**HashMap para transiciones:**
```haskell
import qualified Data.HashMap.Strict as HM

type TransMap = HM.HashMap (String, Char) String
```

**Tiempo:** 6-8 horas
**Impacto:** MEDIO - 2-5x speedup

---

## 🟢 PRIORIDAD P3 (BAJA) - Semana 7+

### P3.1: Soporte Unicode
### P3.2: Estados Léxicos (Modes)
### P3.3: Integración con Parser

*Detalles en análisis comparativo*

---

## 📈 PLAN DE EJECUCIÓN SUGERIDO

### Sprint 1 (Semana 1-2): Fundamentos
- [ ] Día 1-2: Implementar tracking de posición
- [ ] Día 3-5: Crear suite de tests básica (45+ tests)
- [ ] Día 6-7: Escribir README completo
- **Entregable:** v0.2.0 con tests y documentación

### Sprint 2 (Semana 3-4): Robustez
- [ ] Día 1-3: Mejorar manejo de errores
- [ ] Día 4-5: Configurar CI/CD
- [ ] Día 6-7: Alcanzar 80% cobertura
- **Entregable:** v0.3.0 con CI/CD funcionando

### Sprint 3 (Semana 5-6): Calidad
- [ ] Día 1-3: Property-based tests
- [ ] Día 4-7: Optimizaciones de performance
- **Entregable:** v0.4.0 con optimizaciones

### Sprints Opcionales (Semana 7+)
- Características avanzadas según necesidad

---

## 🎯 MÉTRICAS DE ÉXITO

### Después de Sprint 1:
- ✅ >45 tests pasando
- ✅ README completo
- ✅ Errores con línea/columna

### Después de Sprint 2:
- ✅ CI/CD verde
- ✅ >80% cobertura
- ✅ Errores con sugerencias

### Después de Sprint 3:
- ✅ >100 tests pasando
- ✅ 2-5x más rápido
- ✅ Property tests

---

## 💡 CONSEJOS FINALES

1. **No hacer todo a la vez**: Implementar incrementalmente
2. **Tests primero**: Cada feature nueva con sus tests
3. **Documentar mientras codificas**: README actualizado siempre
4. **Medir antes de optimizar**: Benchmarks antes de cambios de performance
5. **Pedir reviews**: Code review mejora calidad

**El camino de 95 (académico) a 85 (profesional) es factible en 8-12 semanas de trabajo enfocado.**

---

**Autor:** Claude (Sonnet 4.5)
**Fecha:** 2025-11-27
**Versión:** 1.0
