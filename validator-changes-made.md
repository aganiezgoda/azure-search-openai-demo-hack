# LLM Answer Validation Feature - Implementation Changes

## Overview

This document describes all changes made to implement an LLM-based answer validation feature for the Azure Search OpenAI demo application. The feature adds a second LLM agent that validates generated answers before returning them to users, checking for factual accuracy, completeness, source attribution, relevance, clarity, and appropriateness.

## Feature Description

When enabled via the "Validate answers with LLM" checkbox in Developer Settings, the application:
1. Generates an answer using the primary LLM (as usual)
2. Passes the answer, user query, and sources to a validation LLM
3. The validation LLM checks the answer quality and returns a JSON response
4. If issues are found, the corrected answer is used; otherwise, the original answer is returned
5. A validation thought step is added to the response context for transparency

## Files Created

### 1. `app/backend/approaches/prompts/validate_answer.prompty`

**Purpose**: Prompty template for the validation LLM call

**Content**:
- Defines the validation agent's role and responsibilities
- Specifies validation criteria (factual accuracy, completeness, source attribution, relevance, clarity, appropriateness)
- Expects variables: `user_query`, `generated_answer`, `sources`
- Returns JSON with: `is_valid`, `issues` (array), `corrected_answer`, `confidence`
- Uses `gpt-4` as the model
- Temperature set to 0.3 for more deterministic validation

## Files Modified

### 2. `app/backend/approaches/approach.py`

**Changes Made**:

#### Added `validate_answer()` method (lines ~1001-1098)
- Async method that takes user query, generated answer, sources, and overrides
- Renders the validation prompt using `prompt_manager.render_prompt()`
- Calls the LLM with the validation prompt
- Parses the JSON response from the validation LLM
- Returns a tuple: (validated_answer, validation_thought_step)
- Creates a ThoughtStep with validation details for transparency

**Key Implementation Detail**:
```python
validation_prompt = self.prompt_manager.render_prompt(
    self.validation_prompt,
    {
        "user_query": user_query,
        "generated_answer": generated_answer,
        "sources": sources,
    },
)
```

Note: Initially attempted to use `variables={...}` as keyword argument, which caused a TypeError. Fixed by passing the dict as the second positional parameter.

### 3. `app/backend/approaches/chatreadretrieveread.py`

**Changes Made**:

#### Modified `__init__()` method (line 92)
- Added loading of validation prompt template:
```python
self.validation_prompt = self.prompt_manager.load_prompt("validate_answer.prompty")
```

#### Modified `run_without_streaming()` method (lines ~128-138)
- Added validation call after answer extraction:
```python
if overrides.get("validate_answer"):
    content, validation_thought = await self.validate_answer(
        user_query=messages[-1]["content"],
        generated_answer=content,
        sources=sources,
        overrides=overrides,
    )
    extra_info["thoughts"].append(validation_thought)
```

### 4. `app/backend/approaches/retrievethenread.py`

**Changes Made**:

#### Modified `__init__()` method (line 81)
- Added loading of validation prompt template:
```python
self.validation_prompt = self.prompt_manager.load_prompt("validate_answer.prompty")
```

#### Modified `run()` method (lines ~148-156)
- Added validation call after answer generation:
```python
if overrides.get("validate_answer"):
    answer, validation_thought = await self.validate_answer(
        user_query=messages[-1]["content"],
        generated_answer=answer,
        sources=sources,
        overrides=overrides,
    )
    extra_info["thoughts"].append(validation_thought)
```

### 5. `app/frontend/src/api/models.ts`

**Changes Made**:
- Added `validate_answer?: boolean` to `ChatAppRequestOverrides` interface
- This allows the frontend to pass the validation setting to the backend

### 6. `app/frontend/src/components/Settings/Settings.tsx`

**Changes Made**:

#### Added `validateAnswer` prop to component interface
```typescript
validateAnswer: boolean;
```

#### Added checkbox UI (line ~408)
```typescript
<Checkbox
    id="validateAnswer"
    className={styles.settingsCheckbox}
    checked={validateAnswer}
    onChange={onChange}
    label={t("labels.validateAnswer")}
    onRenderLabel={onRenderLabel("labels.validateAnswer", "helpText.validateAnswer")}
/>
```

**Important**: Initially placed the checkbox inside conditional blocks (`streamingEnabled` and `!useWebSource`), which made it invisible in certain configurations. Fixed by placing it in the LLM Settings section outside these conditionals.

### 7. `app/frontend/src/pages/chat/Chat.tsx`

**Changes Made**:

#### Added state variable
```typescript
const [validateAnswer, setValidateAnswer] = useState<boolean>(false);
```

#### Added to request overrides
```typescript
const overrides: ChatAppRequestOverrides = {
    // ... other overrides
    validate_answer: validateAnswer,
};
```

#### Added to Settings component props
```typescript
<Settings
    // ... other props
    validateAnswer={validateAnswer}
    onChange={onChangeHandler}
/>
```

#### Updated onChange handler
Added handling for the new setting in the existing onChange handler.

### 8. `app/frontend/src/pages/ask/Ask.tsx`

**Changes Made**:

#### Added state variable
```typescript
const [validateAnswer, setValidateAnswer] = useState<boolean>(false);
```

#### Added to request overrides
```typescript
const overrides: ChatAppRequestOverrides = {
    // ... other overrides
    validate_answer: validateAnswer,
};
```

#### Added to Settings component props
```typescript
<Settings
    // ... other props
    validateAnswer={validateAnswer}
    onChange={onChangeHandler}
/>
```

#### Updated onChange handler
Added handling for the new setting in the existing onChange handler.

### 9. `app/frontend/src/locales/en/translation.json`

**Changes Made**:

Added translations for the new UI elements:
```json
"labels": {
    "validateAnswer": "Validate answers with LLM"
},
"helpText": {
    "validateAnswer": "Use a second LLM agent to validate answer quality before returning to user"
}
```

**Note**: Translations should be added to other language files (da, es, fr, it, ja, nl, ptBR, tr) following the same pattern.

### 10. `tests/test_app.py`

**Changes Made**:

#### Added `test_ask_with_validation()` test
- Tests the /ask endpoint with `validate_answer: true` in overrides
- Verifies that a validation ThoughtStep is added to the response context
- Uses mocked OpenAI responses

#### Added `test_chat_with_validation()` test
- Tests the /chat endpoint with `validate_answer: true` in overrides
- Verifies that a validation ThoughtStep is added to the response context
- Uses mocked OpenAI responses

## Issues Encountered and Fixed

### Issue 1: AttributeError - Missing validation_prompt attribute

**Error**: `AttributeError: 'ChatReadRetrieveReadApproach' object has no attribute 'validation_prompt'`

**Root Cause**: Child approach classes (`ChatReadRetrieveReadApproach` and `RetrieveThenReadApproach`) override `__init__()` and don't call `super().__init__()`. The validation_prompt was only loaded in the base class `__init__()` method (line 278), but this code was never executed for child classes.

**Solution**: Added explicit loading of `validation_prompt` in both child classes' `__init__()` methods:
- `ChatReadRetrieveReadApproach.__init__()` at line 92
- `RetrieveThenReadApproach.__init__()` at line 81

### Issue 2: TypeError - Incorrect parameter name

**Error**: `PromptyManager.render_prompt() got an unexpected keyword argument 'variables'`

**Root Cause**: The `validate_answer()` method was calling `render_prompt()` with `variables={...}` as a keyword argument, but the method signature expects `render_prompt(prompt, data)` with data as a positional parameter.

**Solution**: Changed the call from:
```python
validation_prompt = self.prompt_manager.render_prompt(
    self.validation_prompt,
    variables={...},
)
```

To:
```python
validation_prompt = self.prompt_manager.render_prompt(
    self.validation_prompt,
    {...},
)
```

### Issue 3: Checkbox not visible in UI

**Error**: "Validate answers with LLM" checkbox was not visible in Developer Settings

**Root Cause**: The checkbox was initially placed inside conditional blocks:
- In `Chat.tsx`: wrapped in `{streamingEnabled && ...}` (duplicate checkbox)
- In both pages: wrapped in `{!useWebSource && ...}`

**Solution**: 
- Removed duplicate checkbox from the `streamingEnabled` section
- Moved the remaining checkbox outside the `{!useWebSource && ...}` conditional
- Placed it in the LLM Settings section where it's always visible

## How to Use

1. Open the chat or ask page in the application
2. Click on "Developer Settings" in the sidebar
3. Scroll to the "LLM Settings" section
4. Check the "Validate answers with LLM" checkbox
5. Ask a question or start a chat
6. The answer will be validated by a second LLM before being returned
7. View the "Thought process" tab to see validation details (issues found, confidence level, etc.)

## Deployment

To deploy these changes:

```powershell
azd deploy
```

This will deploy both frontend and backend changes to Azure Container Apps.

## Architecture Notes

- The validation is performed **after** answer generation, not during
- Validation uses the same `chatgpt_deployment` model as answer generation
- The validation prompt uses temperature 0.3 for more consistent validation
- Validation results are stored as a ThoughtStep for transparency
- If validation finds issues and provides a corrected answer, the corrected version is used
- The feature is opt-in via the Developer Settings checkbox (default: disabled)

## Future Enhancements

Potential improvements for this feature:
1. Add translations for all supported languages
2. Allow configuration of validation model (separate from answer generation model)
3. Add validation metrics to Application Insights
4. Add option to show validation details inline in the answer
5. Add configurable validation criteria via settings
6. Support validation in streaming mode
