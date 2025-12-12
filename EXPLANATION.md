## 1. Agent Workflow

The agent processes user input through a sequential pipeline:

### Step-by-Step Process

1. **Receive User Input**
   - User uploads fridge/pantry image via Colab file widget
   - User enters text constraints (e.g., "low-carb dinner")

2. **Vision Analysis** (No memory retrieval - session starts fresh)
   - Send image + detailed prompt to Gemini 2.5 Flash
   - Model identifies ingredients with quantities
   - Organizes results by category (proteins, vegetables, dairy, etc.)

3. **Planning Sub-tasks**
   - Agent receives ingredients + constraints
   - Plans to generate 3 diverse recipe options:
     - **Task A**: Find fastest recipe
     - **Task B**: Find recipe with highest ingredient usage
     - **Task C**: Find recipe with fewest missing items
   - Strategy: Single comprehensive prompt to Gemini

4. **Execute Recipe Generation**
   - Single API call generates all 3 options in structured JSON
   - For each recipe:
     - Match constraints (dietary, time, budget)
     - Maximize use of available ingredients
     - Calculate waste reduction score
     - Generate cooking instructions
     - Request Unsplash image via search query
   - Parse JSON response and validate

5. **User Selection**
   - Display 3 recipe cards with images
   - User selects preferred option (1, 2, or 3)
   - Retrieve full recipe details

6. **Return Final Output**
   - Display complete recipe with:
     - Ingredients (have vs. need)
     - Step-by-step instructions
     - Nutrition information
     - Chef's tips
   - Provide shopping list export options

## 2. Key Modules

### Vision Agent (`detect_ingredients()`)
**Location**: Cell 6-7

**Function**: Analyzes images using Gemini's multimodal capabilities

**Key Logic**:
```python
def detect_ingredients(image, additional_context=""):
    model = genai.GenerativeModel('models/gemini-2.5-flash')
    prompt = """
    Analyze this fridge/pantry image.
    Identify ALL visible ingredients with quantities.
    Organize by category.
    """
    response = model.generate_content([prompt, image])
    return response.text
```

**Output**: Structured text list of ingredients by category

---

### Recipe Planner (`generate_recipe_options()`)
**Location**: Cell 8

**Function**: Generates 3 diverse recipe options

**Planning Strategy**:
- Instructs model to create recipes with different optimization goals
- Enforces diversity via cuisine style variation
- Uses structured JSON output for reliable parsing

**Key Logic**:
```python
def generate_recipe_options(detected_ingredients, user_constraints):
    prompt = f"""
    Create 3 DIFFERENT recipe options using: {detected_ingredients}
    Constraints: {user_constraints}
    
    Vary by: cuisine style, cooking method, optimization goal
    Output JSON with: recipe_name, prep_time, calories, 
                      ingredients_needed, instructions, etc.
    """
    response = model.generate_content(prompt)
    return json.loads(clean_response(response.text))
```

**Output**: List of 3 recipe dictionaries

---

### Display Manager (`display_recipe_options()`, `display_full_recipe()`)
**Location**: Cell 8

**Function**: Renders recipes in user-friendly formats

**Capabilities**:
- HTML/CSS cards for visual appeal
- Fetches recipe images from Unsplash
- Color-coded badges for quick scanning
- Formatted text output with Unicode symbols

---

### Session Memory (Python Variables)
**Location**: Global notebook scope

**Storage**:
```python
# Session state
uploaded_image = None          # PIL.Image
detected_ingredients = None    # String
generated_recipes = None       # List[Dict]
selected_recipe = None         # Dict
```

**Limitations**: 
- No persistence across sessions
- No learning from past interactions
- Resets on kernel restart

## 3. Tool Integration

### Gemini API (Vision + Generation)
**Function**: `google.generativeai.GenerativeModel()`

**Usage**:
```python
# Configure API
genai.configure(api_key=os.environ['GEMINI_API_KEY'])

# Vision call
model = genai.GenerativeModel('models/gemini-2.5-flash')
response = model.generate_content([text_prompt, image])

# Text generation call
response = model.generate_content(text_prompt)
```

**Why This Tool**:
- Native multimodal support (no separate vision API)
- Structured output enforcement
- Fast response times (Flash model)
- Cost-effective for hackathon scale

---

### Unsplash Source API (Images)
**Function**: `get_recipe_image(search_query)`

**Usage**:
```python
def get_recipe_image(search_query):
    base_url = "https://source.unsplash.com/400x300/"
    return f"{base_url}?{search_query},food,dish"

# Example
image_url = get_recipe_image("chicken stir fry")
# Returns: Random relevant food photo URL
```

**Why This Tool**:
- No API key required
- High-quality food photography
- Simple URL-based access
- Automatic search relevance

---

### Google Search (Planned - Cell 10)
**Function**: Price lookup for missing ingredients

**Planned Usage**:
```python
def lookup_prices(items):
    for item in items:
        query = f"price of {item} grocery store"
        results = google_search_tool(query)
        prices = extract_price_data(results)
    return prices
```

**Integration Method**: Gemini function calling with `web_search` tool

## 4. Observability & Testing

### Logging Strategy

**Console Output**:
- Progress indicators for each step (🔍, ✅, 📋)
- Success/failure status messages
- Partial API response previews
- Error details with troubleshooting hints

**Example Log**:
```
🔍 Analyzing image with Gemini Vision...
--------------------------------------------------
✅ Analysis complete!
==================================================

📋 DETECTED INGREDIENTS:
Proteins: 6 eggs, 1 chicken breast
Vegetables: 2 red bell peppers...
```

### Error Handling

**API Failures**:
```python
try:
    response = model.generate_content([prompt, image])
except Exception as e:
    print(f"❌ Error during vision analysis: {e}")
    print("🔍 Troubleshooting tips:")
    print("  1. Check API key validity")
    print("  2. Verify image format (JPEG/PNG)")
```

**JSON Parsing**:
```python
try:
    recipe_data = json.loads(response_text)
except json.JSONDecodeError as e:
    print(f"⚠️ JSON parsing error: {e}")
    print("Raw response:", response.text[:500])
    # Attempt cleanup and retry
    cleaned_text = remove_markdown_artifacts(response_text)
    recipe_data = json.loads(cleaned_text)
```

### Testing Approach

**Manual Testing** (Current):
1. Run cells sequentially 1-8
2. Upload test image (known ingredients)
3. Verify ingredient detection accuracy
4. Test various constraint inputs
5. Check recipe quality and feasibility
6. Validate JSON parsing success

**Test Cases**:
- ✅ Well-stocked fridge (15+ items)
- ✅ Nearly empty fridge (3-4 items)
- ✅ Ambiguous constraints ("healthy meal")
- ✅ Conflicting constraints ("low-carb pasta")
- ✅ Invalid image formats
- ✅ Empty constraint input

**Validation Checklist**:
- [ ] Ingredients detected match image
- [ ] All 3 recipes respect constraints
- [ ] Shopping lists are accurate
- [ ] Cooking instructions are safe and logical
- [ ] Nutrition estimates are reasonable
- [ ] Images display correctly

### Tracing Decisions

**Step-by-step visibility**:
1. Image upload → Shows image preview
2. Vision analysis → Prints full ingredient list
3. Constraint input → Echoes user input
4. Recipe generation → Shows summary of 3 options
5. Selection → Confirms choice
6. Full recipe → Displays complete details

**Judges can trace**:
- Which ingredients were detected
- How constraints influenced recipes
- Why certain ingredients were/weren't used
- How waste reduction was calculated

## 5. Known Limitations

### 1. Ingredient Detection Accuracy
**Issue**: Vision AI may misidentify items (~10% error rate)

**Impact**: 
- Suggests recipes with ingredients you don't have
- Misses ingredients you do have

**Mitigation**: 
- Use high-confidence detections only
- Future: Add user verification step

---

### 2. Constraint Ambiguity
**Issue**: Vague terms like "healthy" or "quick" are subjective

**Impact**: Generated recipes may not match user expectations

**Mitigation**:
- Agent interprets based on common definitions
- Explains interpretation in chef_notes
- Future: Ask clarifying questions

---

### 3. No Price Data (Current)
**Issue**: Cannot show real grocery costs yet

**Impact**: Users must manually check prices

**Workaround**: Show estimated costs based on averages

**Solution**: Cell 10 will add Google Search integration

---

### 4. Session-Only Memory
**Issue**: No learning across sessions

**Impact**: 
- Repeats same questions each time
- Can't remember preferences
- No personalization

**Future**: Add local storage or database for user profiles

---

### 5. API Rate Limits
**Issue**: Gemini free tier has request limits

**Impact**: May fail during heavy demo usage

**Mitigation**: 
- Use efficient prompts (minimize tokens)
- Cache results when possible
- Upgrade to paid tier if needed

---

### 6. Long API Response Times
**Issue**: Vision + generation can take 5-10 seconds

**Impact**: User waits for results

**Mitigation**:
- Use Flash model (faster than Pro)
- Show progress indicators
- Future: Async processing with streaming

---

## Performance Characteristics

**Typical Session Timeline**:
- Image upload: 5 sec (user action)
- Vision analysis: 3-5 sec (API)
- Constraint input: 10 sec (user action)
- Recipe generation: 5-8 sec (API)
- Display: <1 sec
- Selection: 5 sec (user action)
- Full recipe: <1 sec

**Total**: ~30-40 seconds (including user interaction)

**API Costs per Session** (Gemini 2.5 Flash):
- Vision analysis: ~$0.002
- Recipe generation: ~$0.003
- **Total**: ~$0.005 per user session
