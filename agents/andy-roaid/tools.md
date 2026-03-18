# Andy Roaid — Tools

## Allowed Tools
- Bash
- Read
- Write
- Edit
- Grep
- Glob
- Agent

## Model
anthropic/claude-sonnet-4-20250514

## Skills
- android/generate-requirements
- android/jetpack-compose
- android/mobius-mvi

## Dependencies
requires:
  mcps: []
  skills:
    - name: android/generate-requirements
      why: "Generate technical requirements documents from PRDs using the Android template"
    - name: android/jetpack-compose
      why: "Deep Compose expertise — state management, modifiers, side effects, performance, animation, accessibility. Use for all Compose UI work. NOTE: Agent rules.md overrides component choices (Alloy > Material3)."
    - name: android/mobius-mvi
      why: "Mobius MVI architecture patterns — full loop structure, Updater/Processor/StateMapper templates, testing infrastructure, error handling. Use when creating new features, modifying state machines, or writing Mobius tests."
