# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
     Searches available thrift listings for items matching a description, size, and price ceiling. Returns the top matches sorted by relevance.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `description` (str): Natural  description of the item
- `size` (str): Clothing size
- `max_price` (float): Maximum price in USD; excludes listings above this amount

**What it returns:**
<!-- Describe the return value — what fields does a result contain? -->
     A list of up to 3 listing objects, each containing title, price, size, source and condition

**What happens if it fails or returns nothing:**
<!-- What should the agent do if no listings match? -->
     - The agent tells the user no matches were found and stops — it does not call suggest_outfit or create_fit_card with empty input.

---

### Tool 2: suggest_outfit

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
     - Takes the selected thrift find and the user's existing wardrobe and returns a specific outfit pairing with brief styling notes on how to wear the combination.


**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- `new_item` (dict): The listing object selected from search_listings
- `wardrobe` (dict): The user's existing wardrobe items

**What it returns:**
<!-- Describe the return value -->
     A string describing which wardrobe pieces to pair with the new item

**What happens if it fails or returns nothing:**
<!-- What should the agent do if the wardrobe is empty or no outfit can be suggested? -->
     If the wardrobe is empty, the agent asks the user to describe a few pieces they own before proceeding

---

### Tool 3: create_fit_card

**What it does:**
Generates a short, social-media-ready caption for the full outfit centered on the thrift find 

**Input parameters:**
- `outfit` (str): The styled outfit suggestion returned by suggest_outfit

**What it returns:**
A single caption string in casual, first-person tone referencing the new item and the key wardrobe pieces it's paired with.

**What happens if it fails or returns nothing:**
If the outfit string is empty or malformed, the agent skips the fit card and presents the search result and outfit suggestion without it 

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?**
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->

The agent follows a fixed sequence — search → suggest → create — gating each step on the previous output.

- If `search_listings` returns nothing: stop, explain, suggest adjustments.
- If `suggest_outfit` fails: stop after presenting search results.
- If `create_fit_card` fails: present search result + outfit suggestion without a caption.

The agent is done when `create_fit_card` returns, or any earlier step hits an unrecoverable dead end.

---

## State Management


**How does information from one tool get passed to the next?**

The agent holds all state in memory for the session. After each tool call, the result is stored and referenced directly in the next call:

- `search_listings` → top result stored as `selected_item`, passed as `new_item` to `suggest_outfit`
- `suggest_outfit` → result stored as `outfit_suggestion`, passed as `outfit` to `create_fit_card`
- The user's wardrobe description is captured from their initial message and held as `wardrobe` throughout


---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Tell the user nothing was found, suggest adjustments (raise price, broaden description, change size), and stop — do not call further tools |
| suggest_outfit | Wardrobe is empty | Ask the user to describe a few pieces they own, then retry once they respond |
| create_fit_card | Outfit input is missing or incomplete | Skip the fit card and present the search result and outfit suggestion without a caption — interaction is still complete |

---

## Architecture


User query
     |
     v
Planning Loop
     |
     +--> search_listings(description, size, max_price)
               |
               |-- results=[] --> [ERROR] No listings found, suggest adjustments --> STOP
               |
               |-- results=[item, ...]
                         |
                         v
               Session: selected_item = results[0]
                         |
                         v
               suggest_outfit(selected_item, wardrobe)
                         |
                         |-- wardrobe empty --> [PAUSE] Ask user for wardrobe --> retry
                         |
                         |-- outfit returned
                                   |
                                   v
                         Session: outfit_suggestion = "..."
                                   |
                                   v
                         create_fit_card(outfit_suggestion, selected_item)
                                   |
                                   |-- fails --> [SKIP] show search + outfit, no card --> STOP
                                   |
                                   |-- fit_card returned
                                             |
                                             v
                                   Session: fit_card = "..."
                                             |
                                             v
                                   Return to user: listings + outfit + fit card

---

## AI Tool Plan


**Milestone 3 — Individual tool implementations:**

**search_listings:** Give Claude the Tool 1 block and ask it to explain how to filter listings by the three parameters. I'll then ask Claude to review it and check the empty-results case.

**suggest_outfit:** Give Claude the Tool 2 block and ask it to suggest how to match wardrobe items to a new piece. I'll write the function, then test the empty wardrobe case myself.

**create_fit_card:** Give Claude the Tool 3 block and ask it to suggest tone/voice for the caption. I'll verify it includes item name, price, and source.

**Milestone 4 — Planning loop and state management:**

Give Claude the Architecture diagram and ask it to explain how to structure the conditional logic. I'll using Claude to spot-check that state passes correctly between tools and that the error branches work as designed.
---

## A Complete Interaction (Step by Step)

**Example user query:** "I'm looking for a vintage graphic tee under $30. I mostly wear baggy jeans and chunky sneakers. What's out there and how would I style it?"

**Step 1:**
The agent calls search_listings("vintage graphic tee", max_price=30.0).
Returns 3 results:
1. Faded Band Tee — $22, Depop, Good condition
2. Vintage Nirvana Tee — $28, Poshmark, Fair condition
3. 90s Graphic Tee — $25, eBay, Good condition

**Step 2:**
The agent stores selected_item = results[0] (Faded Band Tee, $22) and calls:
suggest_outfit(new_item=<Faded Band Tee>, wardrobe={"bottoms": ["baggy jeans"], "shoes": ["chunky sneakers"]})
Returns: "Pair this with your baggy jeans and chunky sneakers for a 90s streetwear look. Do a small front tuck to keep it from swamping your frame."

**Step 3:**
The agent stores outfit_suggestion and calls:
create_fit_card(outfit="Pair this with your baggy jeans...", new_item=<Faded Band Tee>)
Returns: "thrifted this faded band tee off depop for $22 and my baggy jeans have never been happier 🖤 chunky sneakers doing the heavy lifting as always"

**Final output to user:**
All 3 search results with prices and sources, the outfit suggestion for the top pick, and the fit card caption ready to copy.