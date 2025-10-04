# Introduction to AI Agents: Tools and Autonomy

## 1. Introduction: What Are We Building and Why Should You Care?

### 1.1 What is an AI Agent?

You've probably used ChatGPT or similar AI tools. You ask a question, it gives an answer. That's great, but it's limited - the AI can only work with information it already knows.

An **AI agent** is different. It can:
- Use tools to get new information
- Make decisions about what to do next
- Work through multi-step problems
- Keep trying until it solves the task

Think of it like this:
- Regular AI = Someone who can only answer questions from memory
- AI Agent = Someone who can use Google, make phone calls, check databases, and figure out what they need to do to solve your problem

### 1.2 Why Does This Matter?

This is the fundamental technique behind most practical AI applications:
- Customer service bots that can check your order status
- AI assistants that can book appointments
- Automated systems that can analyze data and make decisions
- Research assistants that can look up information and synthesize it

### 1.3 About This Example

**IMPORTANT**: We're using a course scheduling example in this notebook, but that's just for demonstration. The *exact same technique* works for:
- Conducting systematic literature reviews
- Analyzing data and running statistical tests
- Planning travel itineraries
- Literally any task that requires decisions and actions

The example scenario doesn't matter. What matters is understanding:
1. How to give an AI access to tools
2. How to let it decide which tools to use
3. How to create a loop where it can keep working until the job is done

Think of this like learning to cook by making scrambled eggs. You're not learning about eggs - you're learning about heat control, timing, and technique. Once you understand that, you can cook anything.

Let's build this step by step.

---

## 2. Setting Up Our Environment

### 2.1 Importing Required Libraries

Every Python program starts with imports. These are just the libraries we need.
- `json`: For formatting data
- `OpenAI`: To talk to the AI
- `datetime`: For working with dates (we're not even really using this, but it's there)

```python
import json
from openai import OpenAI
from datetime import date

OPENAI_API_KEY=""
```

### 2.2 The Example Data (Remember: This Part Doesn't Matter!)

#### Our Example Scenario: Course Scheduling

We're pretending we have a student who needs to manage their course schedule. They have:
- Some courses they're already enrolled in
- Some courses they could add
- A maximum credit limit (like a budget)

**Again**: This could be anything. medical diagnosis workflows, financial portfolio management, event planning, inventory management, whatever. We picked course scheduling because it's relatable and simple.

The point is that we have:
1. Some data the AI needs to read
2. Some constraints it needs to respect (credit limit)
3. Some actions it can take (add/drop courses)

That pattern applies to almost any real-world problem.

```python
MAX_CREDITS = 18  # University credit limit per semester

# Courses the student is currently enrolled in
enrolled_courses = [
    {"name": "Introduction to Psychology", "code": "PSY101", "credits": 3},
    {"name": "Calculus I", "code": "MATH151", "credits": 4},
    {"name": "English Composition", "code": "ENG101", "credits": 3},
]

# Courses available to add during late registration
available_courses = [
    {"name": "Data Structures", "code": "CS201", "credits": 4},
    {"name": "World History", "code": "HIST101", "credits": 3},
    {"name": "Public Speaking", "code": "COMM110", "credits": 3},
    {"name": "Statistics", "code": "STAT200", "credits": 3},
]
```

The data is just lists of dictionaries. In a real application, this would come from a database or an API.

---

## 3. Building the Agent's Tools

### 3.1 Giving the AI Hands and Eyes

We're going to write normal Python functions that do things like:

- Check what courses are enrolled
- Add a course
- Drop a course
- Check the credit limit

Here's the key insight: **These functions are how the AI interacts with the world.**

The AI can't directly access our Python variables. It can't just read `enrolled_courses`. It needs to *call a function* to get that information. This is actually good because:

1. In real life, data lives in databases, APIs, etc. - not in Python variables
2. It gives us control over what the AI can and can't do
3. It teaches you the right pattern for building real systems

Think of these functions as the AI's hands and eyes. Without them, it's just talking into the void. With them, it can actually DO things.

We're keeping this simple with just 5 functions. You could add 50 more if you wanted.

```python
def total_credits(courses):
    """
    Helper function to calculate total credits.
    This isn't a tool the AI uses - just a helper for our other functions.
    """
    return sum(course["credits"] for course in courses)


def get_current_schedule():
    """
    Tool 1: Check Current Schedule

    This lets the AI see what courses are currently enrolled.

    In a real app, this would query a database or call an API.
    Here, it just returns our list.
    """
    return {"count": len(enrolled_courses), "courses": enrolled_courses}


def get_available_courses():
    """
    Tool 2: See What's Available

    The AI can call this to see what courses could be added.
    """
    return {"available_courses": available_courses}


def add_course(code: str):
    """
    Tool 3: Add a Course

    When the AI calls this, it:
    1. Finds the course by code
    2. Adds it to enrolled_courses
    3. Removes it from available_courses
    4. Returns info about what happened

    Notice we return a dictionary with lots of info. This helps the AI understand
    what happened and decide what to do next.
    """
    match = next((course for course in available_courses if course["code"] == code), None)
    if match is None:
        return {"code": code, "added": False, "message": "Course not available."}

    enrolled_courses.append(match)
    available_courses.remove(match)

    return {
        "code": code,
        "name": match["name"],
        "added": True,
        "new_total_credits": total_credits(enrolled_courses),
        "new_course_count": len(enrolled_courses),
        "remaining_available": [c["code"] for c in available_courses],
    }


def drop_course(code: str):
    """
    Tool 4: Drop a Course

    The opposite of add_course. Removes a course from the schedule.
    """
    match = next((course for course in enrolled_courses if course["code"] == code), None)
    if match is None:
        return {"code": code, "dropped": False, "message": "Course not in schedule."}

    enrolled_courses.remove(match)
    available_courses.append(match)

    return {
        "code": code,
        "name": match["name"],
        "dropped": True,
        "credits_freed": match["credits"],
        "new_total_credits": total_credits(enrolled_courses),
        "new_course_count": len(enrolled_courses),
        "added_to_available": True,
    }


def check_credit_limit():
    """
    Tool 5: Check Credit Status

    This lets the AI see if the student is within the credit limit.

    Think of this like checking your bank balance before making a purchase.
    The AI needs to know: "Can I add another course, or am I at the limit?"
    """
    current_credits = total_credits(enrolled_courses)
    return {
        "current_credits": current_credits,
        "max_credits": MAX_CREDITS,
        "available_credits": MAX_CREDITS - current_credits,
        "within_limit": current_credits <= MAX_CREDITS,
    }
```

Notice what we did NOT do:

- We didn't write any fancy AI logic
- We didn't teach the AI when to use which function
- We didn't create a decision tree

We just wrote normal Python functions that do normal things. The AI will figure out when and how to use them.

### 3.2 Describing Our Tools to the AI

This is a critical step: The AI doesn't automatically know what our functions do. We need to DESCRIBE each function to it.

That's what this `tools` list does. For each function, we tell the AI:

1. The name of the function
2. What it does (description)
3. What parameters it needs
4. What type each parameter is

This is formatted in a specific way that OpenAI's API understands. The format looks complicated, but it's just a detailed description. Look at the "description" field in each tool - that's the important part. The AI reads that to understand what the function does.

**Tip**: The quality of these descriptions matters A LOT. If you write vague descriptions, the AI won't know when to use the tool. Be specific and clear.

```python
tools = [
    {
        "type": "function",
        "name": "fetch_schedule",
        "description": "Get the student's current enrolled courses with credit information.",
        "strict": True,
        "parameters": {"type": "object", "properties": {}, "required": [], "additionalProperties": False},
    },
    {
        "type": "function",
        "name": "fetch_available_courses",
        "description": "List courses available to add with credit values.",
        "strict": True,
        "parameters": {"type": "object", "properties": {}, "required": [], "additionalProperties": False},
    },
    {
        "type": "function",
        "name": "add_course",
        "description": "Enroll in a course. Removes the course from available courses.",
        "strict": True,
        "parameters": {
            "type": "object",
            "properties": {"code": {"type": "string", "description": "Course code to add (e.g., CS201)."}},
            "required": ["code"],
            "additionalProperties": False,
        },
    },
    {
        "type": "function",
        "name": "drop_course",
        "description": "Drop a course from the schedule. Course becomes available again.",
        "strict": True,
        "parameters": {
            "type": "object",
            "properties": {"code": {"type": "string", "description": "Course code to drop from schedule."}},
            "required": ["code"],
            "additionalProperties": False,
        },
    },
    {
        "type": "function",
        "name": "check_credit_limit",
        "description": "Calculate current credit load versus the maximum allowed credits.",
        "strict": True,
        "parameters": {"type": "object", "properties": {}, "required": [], "additionalProperties": False},
    },
]
```

One more thing to notice: The function names in this list don't exactly match our Python function names.

- We call the Python function `get_current_schedule()`
- But we tell the AI it's called `fetch_schedule`

Why? Because we need to map between what the AI calls and what actually runs. You'll see that mapping happen in the agent loop below.

This separation is actually useful - it means you can change your Python function names without having to retrain the AI or update prompts everywhere.

---

## 4. Helper Functions

### 4.1 Making the Output Pretty

This function is optional but helpful. When the AI calls a tool, it gets back a bunch of JSON data. That's great for the AI, but hard for us humans to read.

This function takes the results and formats them into nice, readable strings so we can watch what the AI is doing.

This is purely for debugging and demonstration. In a production app, you might log this to a file or display it in a UI instead.

```python
def format_tool_result(name, result):
    """Format tool results so we can easily see what happened"""
    if name == "fetch_schedule":
        courses = [f"{c['code']}: {c['name']} ({c['credits']} cr)" for c in result.get('courses', [])]
        return f"  → {result.get('count', 0)} courses: {', '.join(courses)}"

    elif name == "fetch_available_courses":
        courses = [f"{c['code']}: {c['name']} ({c['credits']} cr)" for c in result.get('available_courses', [])]
        return f"  → Available: {', '.join(courses)}"

    elif name == "check_credit_limit":
        return f"  → Credits: {result.get('current_credits', 0)} | Max: {result.get('max_credits', 0)} | Available: {result.get('available_credits', 0)}"

    elif name == "add_course":
        if result.get('added'):
            return f"  → Added {result.get('code')} ({result.get('name')})! New credit total: {result.get('new_total_credits', 0)}"
        else:
            return f"  → Failed to add {result.get('code')}: {result.get('message', 'Unknown error')}"

    elif name == "drop_course":
        if result.get('dropped'):
            return f"  → Dropped {result.get('code')} ({result.get('name')})! Freed {result.get('credits_freed', 0)} credits | New total: {result.get('new_total_credits', 0)}"
        else:
            return f"  → Failed to drop {result.get('code')}: {result.get('message', 'Unknown error')}"

    else:
        return f"  → {json.dumps(result, indent=2)}"
```

---

## 5. The Agent Loop

### 5.1 The Core Concept: The Agentic Loop

Here's how a normal chatbot works:

1. User asks a question
2. AI gives an answer
3. Done

Here's how an agent works:

1. User gives a task
2. AI thinks: "What information do I need?"
3. AI calls a tool to get information
4. AI thinks: "What should I do with this information?"
5. AI calls another tool to take action
6. AI thinks: "Did that solve the problem?"
7. If not, go back to step 2
8. Once solved, give final answer

See the difference? The agent LOOPS. It keeps working until the job is done.

### 5.2 How The Loop Works (Step by Step)

Here's what happens in this function:

1. **Start with a message**: We send the AI a task (like "add courses until I have 15 credits")

2. **The AI responds**: It either:
   - Calls a tool (like "check_credit_limit"), OR
   - Gives a final answer (if it thinks it's done)

3. **If it called a tool**:
   - We run the actual Python function
   - We send the results back to the AI
   - We go back to step 2 (loop continues)

4. **If it gave a final answer**:
   - We're done! Print the answer and exit

5. **Safety valve**: We have a max loop count (20) so it can't run forever

### 5.3 Why This Matters

This pattern - call tool, get result, decide what to do next, repeat - is the foundation of agent-based AI.

You could build:

- A research assistant that searches multiple sources and synthesizes findings
- A customer service bot that checks order status, processes refunds, updates tickets
- A data analysis tool that queries databases, performs calculations, generates reports

All using this exact same pattern. Just different tools.

```python
def run_agent_query(model, query, client):
    """
    Run the agent loop

    Args:
        model: Which AI model to use
        query: The task you want the agent to complete
        client: The OpenAI client

    Returns:
        The final answer from the agent
    """

    # Step 1: Set up the initial conversation
    # We give the AI a "system message" that explains its role
    messages = [
        {
            "role": "system",
            "content": (
                "You are a course scheduling assistant. Help students manage their course schedule by checking "
                "their current courses, finding available courses to add, adding or dropping courses as needed, and ensuring "
                "they stay within the credit limit. Always check the credit limit before finishing. "
                "Once the schedule meets the requirements and you can summarize the plan, answer and end with DONE."
            ),
        },
        {"role": "user", "content": query},
    ]

    # Step 2: Initialize the loop
    loop = 0
    max_loops = 20  # Safety limit

    # Step 3: Start looping!
    while loop < max_loops:
        loop += 1
        print(f"\n### Loop {loop} ###")

        # Step 4: Send messages to the AI and get a response
        # We pass in our tools so the AI knows what it can do
        response = client.responses.create(
            model=model,
            input=messages,
            tools=tools,
            parallel_tool_calls=False,  # Do one thing at a time (easier to follow)
        )

        # Step 5: Add the AI's response to our message history
        # This is how the AI "remembers" what it's done
        messages.extend(response.output)

        # Step 6: Check if the AI called any tools
        used_tool = False
        for item in response.output:
            if item.type != "function_call":
                continue

            used_tool = True
            args = json.loads(item.arguments)

            # Step 7: Map the AI's function call to our actual Python function
            # This is where we connect the AI's request to real code
            if item.name == "fetch_schedule":
                result = get_current_schedule()
            elif item.name == "fetch_available_courses":
                result = get_available_courses()
            elif item.name == "add_course":
                result = add_course(args["code"])
            elif item.name == "drop_course":
                result = drop_course(args["code"])
            elif item.name == "check_credit_limit":
                result = check_credit_limit()

            # Step 8: Show what happened (for our benefit)
            print(f"Tool call → {item.name}({args})")
            print(format_tool_result(item.name, result))

            # Step 9: Send the tool result back to the AI
            # The AI will use this information to decide what to do next
            messages.append(
                {
                    "type": "function_call_output",
                    "call_id": item.call_id,
                    "output": json.dumps(result),
                }
            )

        # Step 10: If the AI called a tool, loop back and let it go again
        if used_tool:
            continue

        # Step 11: If we get here, the AI didn't call any tools
        # That means it gave a final answer. We're done!
        final_answer = response.output_text.strip()
        print("\nAgent answer:\n", final_answer)
        return final_answer

    # Step 12: If we hit the loop limit, something went wrong
    if loop == max_loops:
        print("\nLoop limit hit without completion.")
        return None
```

### 5.4 What Just Happened?

- The AI decides which tools to use. We don't tell it.
- The AI decides when it's done. We don't tell it.
- We just provide tools and let it work.

This is autonomous AI. We gave it:

1. A goal
2. Some tools
3. A loop to work in

And it figures out the rest.

That's powerful. And also a little scary. Which is why we have:

- The loop limit (so it can't run forever)
- Clear tool descriptions (so it knows what it's doing)
- A good system prompt (so it knows its job)

---

## 6. Running The Agent

### 6.1 Let's Actually Run This Thing

Time to see it in action. We'll:
1. Create an OpenAI client
2. Give the agent a task
3. Watch it work

The task we're giving it: "You have 10 credits, need at least 15, can't exceed 18. Add courses to get there."

Watch how it:

- Checks current schedule
- Sees available courses
- Checks credit limit
- Adds courses
- Checks again
- Confirms it's done

All without us telling it the specific steps.

```python
# Initialize the OpenAI client
client = OpenAI(api_key=OPENAI_API_KEY)

# Define our task
goal = (
    "I'm currently enrolled in 3 courses but I need at least 15 credits this semester to maintain full-time status. "
    "Check my current schedule, see how many credits I have, and add courses from the available list until I reach at least 15 credits. "
    "Make sure I don't go over the 18 credit maximum. Then summarize my final schedule."
)

# Run it!
model = "gpt-5"
result = run_agent_query(model, goal, client)
```

---

## 7. What You Just Learned (And What's Next)

### 7.1 What You Just Built

Congratulations! You just built an AI agent. Let's recap what that means:

1. **Tool Use**: You gave an AI the ability to call functions and get real data
2. **Decision Making**: The AI decided which tools to use and when
3. **Looping**: The AI kept working until the task was complete
4. **Autonomy**: You gave a goal, not step-by-step instructions

### 7.2 Why This Pattern Matters

This same pattern works for anything:

- **Data Analysis**: Tools to query databases, run calculations, generate charts
- **Research**: Tools to search papers, read documents, synthesize information
- **Customer Support**: Tools to check orders, process refunds, send emails
- **Business Ops**: Tools to check inventory, place orders, update records

The scenario changes. The technique doesn't.

### 7.3 Things to Think About

As you build your own agents, consider:

1. **What tools does my agent need?**
   - What information does it need to read?
   - What actions does it need to take?
   - Start small (like we did - just 5 tools)

2. **How do I keep it safe?**
   - Loop limits (so it doesn't run forever)
   - Approval steps (for critical actions)
   - Clear boundaries (what it can and can't do)

3. **How do I make it reliable?**
   - Good tool descriptions (so it knows what to use)
   - Good system prompts (so it knows its job)
   - Good error handling (so it fails gracefully)

### 7.4 What to Add Next

Want to extend this? Try adding:

- `check_time_conflicts()` - Make sure courses don't overlap
- `check_prerequisites()` - Verify the student can take a course
- `get_professor_ratings()` - Fetch RateMyProfessor scores
- `optimize_schedule()` - Find the best combination of courses

Each new tool is just:

1. Write a Python function
2. Add it to the tools list
3. Add it to the mapping in the loop

The AI will automatically figure out when to use it.

---

## 8. Bonus: More Example Queries You Can Try

Try running these different queries:

```python
# Just check current status
result = run_agent_query(model, "What courses am I currently enrolled in?", client)

# Add a specific course
result = run_agent_query(model, "Add CS201 to my schedule and check if I'm under the credit limit", client)

# Handle a constraint
result = run_agent_query(model, "I need to drop a 3-credit course to make room for CS201. What should I do?", client)

# Complex multi-step task
result = run_agent_query(model, "I need exactly 16 credits. Add or drop courses to get there.", client)
```

See how the same agent handles different tasks? That's the flexibility of this approach.
