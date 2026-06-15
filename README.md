# FitFindr — Starter Kit

This starter kit contains everything you need to begin Project 2.

## What's Included


ai201-project2-fitfindr-starter/
├── data/
│   ├── listings.json          # 40 mock secondhand listings
│   └── wardrobe_schema.json   # Wardrobe format + example wardrobe
├── utils/
│   └── data_loader.py         # Helper functions for loading the data
├── planning.md                # Your planning template — fill this out first
└── requirements.txt           # Python dependencies


## Setup

bash
pip install -r requirements.txt


Set your Groq API key in a .env file (get a free key at [console.groq.com](https://console.groq.com)):

GROQ_API_KEY=your_key_here


## The Mock Listings Dataset

data/listings.json contains 40 mock secondhand listings across categories (tops, bottoms, outerwear, shoes, accessories) and styles (vintage, y2k, grunge, cottagecore, streetwear, and more).

Each listing has: id, title, description, category, style_tags, size, condition, price, colors, brand, and platform.

Load it with:
python
from utils.data_loader import load_listings
listings = load_listings()


## The Wardrobe Schema

data/wardrobe_schema.json defines the format your agent uses to represent a user's existing wardrobe. It includes:

- schema: field definitions for a wardrobe item
- example_wardrobe: a sample wardrobe with 10 items you can use for testing
- empty_wardrobe: a starting template for a new user

Load an example wardrobe with:
python
from utils.data_loader import get_example_wardrobe
wardrobe = get_example_wardrobe()


## Where to Start

1. **Read planning.md and fill it out before writing any code.**
2. Verify the data loads correctly by running python utils/data_loader.py.
3. Build and test each tool individually before connecting them through your planning loop.

Your implementation files go in this same directory. There's no required file structure for your agent code — organize it however makes sense for your design.

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
<!-- Describe what this tool does in 1–2 sentences -->
     Searches available thrift listings for items matching a description, size, and price ceiling. Returns the top matches sorted by relevance.

**Input parameters:**
<!-- List each parameter, its type, and what it represents -->
- description (str): Natural  description of the item
- size (str): Clothing size
- max_price (float): Maximum price in USD; excludes listings above this amount

**What it returns:**
<!-- Describe the return value — what fields does a result contain? -->
     A list of matching listing dicts sorted by relevance. Each dict contains: id, title, description, category, style_tags (list), size, condition, price (float), colors (list), brand, and platform.

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
- new_item (dict): The listing object selected from search_listings
- wardrobe (dict): The user's existing wardrobe items

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
- outfit (str): The styled outfit suggestion returned by suggest_outfit

**What it returns:**
A single caption string in casual, first-person tone referencing the new item, price, platform, and key wardrobe pieces

**What happens if it fails or returns nothing:**
If the outfit string is empty or malformed, the agent skips the fit card and presents the search result and outfit suggestion without it

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?**
<!-- Describe the logic your planning loop uses. What does it look at? What conditions change its behavior? How does it know when it's done? -->

The agent follows a fixed sequence — search → suggest → create — gating each step on the previous output.

- If search_listings returns nothing: stop, explain, suggest adjustments.
- If suggest_outfit fails: stop after presenting search results.
- If create_fit_card fails: present search result + outfit suggestion without a caption.

The agent is done when create_fit_card returns, or any earlier step hits an unrecoverable dead end.

---

## State Management


**How does information from one tool get passed to the next?**

The agent holds all state in memory for the session. After each tool call, the result is stored and referenced directly in the next call:

- search_listings → top result stored as selected_item, passed as new_item to suggest_outfit
- suggest_outfit → result stored as outfit_suggestion, passed as outfit to create_fit_card
- The user's wardrobe description is captured from their initial message and held as wardrobe throughout


---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Tell the user nothing was found, suggest adjustments (raise price, broaden description, change size), and stop — do not call further tools |
| suggest_outfit | Wardrobe is empty | Ask the user to describe a few pieces they own, then retry once they respond |
| create_fit_card | Outfit input is missing or incomplete | Skip the fit card and present the search result and outfit suggestion without a caption — interaction is still complete |

---

## Spec Reflection

Defining the failure mode for each tool upfront clarified that search_listings needed to return an empty list rather than raise an exception — which shaped how the planning loop's early-exit condition was written. I planned to pause and ask the user for wardrobe items when the wardrobe was empty, but in practice suggest_outfit handles it by calling the LLM for general styling advice instead, since the Gradio interface has no back-and-forth input mechanism.


## AI Usage

**Instance 1: Implementing search_listings**
I gave Claude the Tool 1 spec from planning.md (input parameters, return value, failure mode) and asked it to explain how keyword scoring with set intersection worked. It explained the & operator and len() approach. I wrote the function myself using that logic, then asked Claude to check whether my price and size filters were applied in the right order. It confirmed the order was correct but pointed out I had a stray return [] outside the function body, which I removed.

**Instance 2: Debugging the wardrobe KeyError**
When suggest_outfit crashed with KeyError: 'title', I pasted the error and asked Claude what was wrong. It told me to check the actual key names in the wardrobe dict. I ran the data loader myself to inspect the output and found the key was 'name', not 'title'. I made that fix myself in tools.py.