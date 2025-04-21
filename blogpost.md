---

## Building an AI Interviewer Agent with Google Gemini and LangGraph

Let's be honest: preparing for tech interviews can be tough. You grind LeetCode, watch mock interviews on YouTube, but something's often missing. If you're like me, maybe you don't have tech buddies available 24/7 for practice, or you worry if the feedback you *do* get is actually helpful. Solving problems on your own is one thing, but explaining your thought process under pressure? That's a whole different skill.

This exact feeling sparked an idea for my Google GenAI Capstone project: **What if AI could be that reliable, on-demand practice partner?** An AI that doesn't just throw questions at you, but actually interviews you, guides you when you're stuck, and gives detailed feedback?

So, I built one! It's an **AI Mock Technical Interviewer**, created using Google's powerful Gemini AI model and a cool framework called LangGraph.

![AI Mock Interviewer App UI](https://raw.githubusercontent.com/MaxDatex/AI-Mock-Interviewer/main/static/first-page-2.png)

In this post, I'll share:

*   **The "Why":** The problem I wanted to solve (based on my own prep!).
*   **The "What":** What my AI interviewer actually does.
*   **The "How":** The tools I used (Gemini, LangGraph, etc.) and why.
*   **Behind the Scenes:** Some key moments, challenges, and code snippets from the build.
*   **What's Next:** Where I see this project going.

### The Problem: Difficulties in Self-Led Interview Training

Like many developers, I'm currently preparing for technical interviews. I quickly realized that just solving coding problems isn't enough. Real interviews involve:

*   **Talking through your logic:** Explaining *why* you chose a certain approach.
*   **Asking clarifying questions:** Making sure you understand the problem.
*   **Handling hints:** Getting unstuck gracefully.
*   **Getting feedback:** Understanding your strengths and weaknesses.

Finding someone who has the time, expertise, and willingness to conduct realistic mock interviews and provide solid feedback is hard! This is where the idea for an AI interviewer really took shape.

### The Solution: An AI Practice Partner

<p align="center">
<img src="https://raw.githubusercontent.com/MaxDatex/AI-Mock-Interviewer/main/static/logo.jpg" alt="AI Mock Technical Interviewer logo" width="500"/>
</p>

My goal was to create an AI that feels less like a static quiz and more like a helpful interviewer. Here’s what it does:

1.  **Conducts the Interview:** It asks coding questions (from a predefined list, no random stuff!).
2.  **Guides and Hints:** If you get stuck, it can offer subtle hints, just like a real interviewer might.
3.  **Understands Whiteboarding:** You can *draw* your ideas on a digital whiteboard (using Gradio's Sketchpad), and the AI uses Gemini's image understanding to "see" and discuss your drawing!
4.  **Listens and Remembers:** Using LangGraph, the AI keeps track of the conversation, so it understands the context.
5.  **Gives Detailed Feedback:** After you finish, it analyzes the entire conversation and your code to provide a structured report on your technical skills, problem-solving approach, and communication.
6.  **Creates a Learning Plan:** It even identifies areas you need to work on and uses Google Search (via Gemini Grounding) to suggest relevant learning resources!

All of this is powered by Google's Gemini AI models, which handle the talking, understanding text and images, structuring the feedback, and even searching the web for relevant study materials.

### The Toolkit: Why I Chose These Tools

![Toolkit logos](https://raw.githubusercontent.com/MaxDatex/AI-Mock-Interviewer/main/static/logos_blogpost.png)

I got introduced to these tools during a Google GenAI course, and they felt like a great fit for this project:

*   **Google Gemini:** This is the AI "brain." It's great at understanding natural language, looking at images (the whiteboard feature!), generating human-like text, following complex instructions (like giving specific feedback), and using other tools (like function calling and search).
*   **LangGraph:** Think of this as the "director" of the interview. An interview has steps: start, ask question, get answer, give hint (maybe), end, evaluate. LangGraph lets you define this flow clearly, managing the conversation state step-by-step.
*   **LangChain:** Provides helpful building blocks for working with AI models and tools, making it easier to connect everything.
*   **Gradio:** This made creating the interactive web interface surprisingly easy. It provides ready-made components for text chat, a code editor, and even the sketchpad/whiteboard.
*   **Kaggle Notebooks:** This project was built for the Google course, which required using Kaggle. It turned out to be a convenient place to code, manage secrets (like the API key), and share the project.

### Building It: Highs, Lows, and Code Snippets

Creating this involved a lot of trial and error, but also some really cool moments:

**Challenge & "Aha!" Moment #1: Teaching the AI to Interview**

Getting the AI to behave like a *good* interviewer took the most effort. I wrote a long initial prompt telling it how to act (be supportive, give hints, ask probing questions). But you can't predict every situation!

I'd test it, and the AI would do something unexpected – maybe give a hint too quickly, or misunderstand the whiteboard. Each time, I'd go back and tweak the prompt, adding more specific instructions or examples (this is called few-shot learning). It was a constant back-and-forth, but each improvement felt like a mini-victory, teaching the AI a bit more about the nuances of interviewing.

**Challenge & "Aha!" Moment #2: Tools That Change Things**

LangGraph makes it easy for the AI to use "tools" – functions that fetch data or perform actions. Most tools just return information, like listing available questions. But I needed tools that actually *changed* the interview **state**. Think of the "state" as the interview's memory – it holds the conversation history, the current question, the code you've written, and whether the interview has finished.

Here is visualization of the Interview Flow with LangGraph
<p align="center">
<img src="https://raw.githubusercontent.com/MaxDatex/AI-Mock-Interviewer/main/static/graph-visualisation.png" alt="Interview Graph structure" width="500"/>
</p>

This required a special setup in LangGraph. The `select_question` tool itself is simple (it just defines *that* the action exists): 

```python
# Tool definition for selecting a question

class SelectQuestionArgs(BaseModel):
    """Input schema for the select_question tool."""
    ID: Literal[tuple(IDS)] = Field(description="ID of the question")

@tool(args_schema=SelectQuestionArgs)
def select_question(ID: str) -> str:
    """Shows user question with provided ID.
    ALWAYS use this tool when the candidate confirms selected question.
    You can start interview process ONLY after using this tool.
    """
    # This tool itself doesn't contain the main logic...
    pass # ...the logic happens in the custom node below
```

The *real* work happens in a custom LangGraph *node* called `question_selection_node`. This node runs *after* the AI decides to use the `select_question` tool:

```python
# Simplified logic inside the 'question_selection_node' function

def question_selection_node(state: InterviewState) -> InterviewState:
    # 1. Get the last AI message (which contains the tool call)
    tool_msg: AIMessage = state.get("messages", [])[-1]

    # 2. Loop through tool calls in that message (usually just one here)
    for tool_call in tool_msg.tool_calls:
        if tool_call["name"] == "select_question":
            # 3. Extract the Question ID the AI provided
            ID: int = int(tool_call["args"]["ID"])

            # 4. Find the question details in the question database 
            selected_question: pd.Series = df[df.id==ID].iloc[0]
            question_content = selected_question.content
            question_code = selected_question.code

            # 5. Prepare a message confirming the selection
            response = f"Okay, let's work on '{selected_question.problem_name}'. Here's the description:\n{question_content}\nInitial code:\n{question_code}"

            # Create a message containing the response for the AI
            tool_message = ToolMessage(content=response, name=tool_call["name"], tool_call_id=tool_call["id"])

    # 6. CRITICAL STEP: Update the state dictionary with the new question and code!
    # The '|' merges the old state with the new values.
    return state | {
        "messages": [tool_message], # Add the tool result message
        "question": question_content, # Update the active question
        "code": question_code         # Update the starter code
    }
```
Making these state-changing tools work correctly felt like properly wiring up the AI's brain and hands – tricky, but essential!

**Explaining LangGraph's Tool Magic (Simply):**

LangGraph provides powerful ways to structure AI interactions, but some of its abstractions, like how tools are called, can be a bit confusing at first glance, requiring a deeper dive into its underlying implementation.

Here's the simple version of how the *standard* `ToolNode` (used for tools that *don't* change the state directly, like getting the list of topics) works:

1.  You chat with the AI interviewer.
2.  Based on the conversation and its instructions (especially after using `.bind_tools(tools)` to tell the model which tools are available), the AI (Gemini) might decide it needs to use a tool. It outputs a special `AIMessage` containing `tool_calls`, like: `AIMessage(content="", tool_calls=[{"name": "get_topic_categories", "args": {}, "id": "call_123"}])`.
3.  LangGraph routes the flow to the `ToolNode`.
4.  `ToolNode` looks at the `tool_calls` in that last `AIMessage`.
5.  It finds the matching Python function (e.g., our `get_topic_categories` function decorated with `@tool`).
6.  It runs that function.
7.  It takes the result (the list of topics) and wraps it in a `ToolMessage`, linking it back to the original call ID: `ToolMessage(content=['Arrays', 'Linked Lists', ...], name='get_topic_categories', tool_call_id='call_123')`.
8.  This `ToolMessage` gets added to the state's message list.
9.  The flow usually goes back to the AI, which now sees the tool result and can form its next response, like, "Okay, we have questions on these topics: Arrays, Linked Lists..."

For tools like `select_question` that *do* need to change the state, we bypass the standard `ToolNode` and use a custom node (`question_selection_node`) as shown in the previous section.

**Surprise! The AI Gives Good Feedback?**

I was genuinely impressed by the quality of the feedback the AI generated. It didn't just say "good job" or "try harder." It pointed to specific parts of the conversation, analyzed my approach, commented on communication, and created that personalized learning plan. It highlighted things I wouldn't notice just by solving problems alone. That showed me the real value beyond just question-answering.

It also surprised me with its flexibility. I could ask "give me beginner array problems" and it would understand and use the `list_questions` tool with the right filters (`difficulty='Easy'`, `category='Array Manipulation'`). Switching questions mid-interview was also smooth, thanks to the state management and tools.

**Getting the Structured Feedback:**

To make sure the final report was consistent and easy to use, I told Gemini exactly what format to use. I defined a structure using Python's Pydantic library, like a blueprint for the report.

```python
# Example Pydantic Model (Blueprint for part of the feedback)
class StrengthItem(BaseModel):
    """Represents a single observed strength."""
    point: str = Field(..., description="Concise statement of the strength")
    evidence: str = Field(..., description="Specific examples or quotes")

class EvaluationOutput(BaseModel):
    """Root object for the complete evaluation."""
    overall_summary: str = Field(..., alias="overallSummary")
    strengths: List[StrengthItem] = Field(...)
    # ... other fields like areas_for_development, detailed_analysis, etc.
```

Then, when calling the Gemini API to generate the report, I told it to follow this blueprint:

```python
# Inside the function that generates the report (create_report_node)

evaluation_response = client.models.generate_content(
    model='gemini-2.0-flash',
    contents=CANDIDATE_EVALUATION_PROMPT.format(...), # Prompt asking for evaluation
    config={
        'response_mime_type': 'application/json', # Tell it we want JSON
        'response_schema': EvaluationOutput,     # Tell it the *exact* structure/blueprint
    },
)
# Now, evaluation_response.parsed contains the data neatly structured
evaluation_data: EvaluationOutput = evaluation_response.parsed
```
This ensures the output is always organized the same way, making it reliable.

*Example of the part of the candidate_report.md*
![Candidate report example](https://raw.githubusercontent.com/MaxDatex/AI-Mock-Interviewer/main/static/report-screenshot.png)

### Limitations and What's Next

This is a great start, but it's not perfect:

*   **It Can't *Run* Your Code:** The AI analyzes the code you type as text, but it doesn't actually execute it to check for errors or test edge cases.
*   **Human Nuance:** While the AI provides remarkably detailed analysis based on the transcript, it might not catch every subtle non-verbal cue or unspoken hesitation a human interviewer might notice. However, it offers a very strong and consistent approximation of a real evaluation process focused on what was said and coded. 

**My Next Steps:**

If I had more time, the top two things I'd add are:

1.  **Voice Interaction:** Using speech-to-text and text-to-speech would make it feel much more like a real, immersive conversation. This is my #1 priority!
2.  **Code Execution:** Letting the AI actually run the code against test cases would provide even more valuable feedback.

### Who Is This For?

Honestly, I think anyone preparing for technical interviews could find this useful:

*   **Beginners:** Can get practice explaining their thoughts and benefit from the guided hints and personalized learning plans.
*   **Experienced Devs:** Can refresh their knowledge and practice articulating complex solutions clearly.
*   **FAANG Aspirants:** Can get targeted practice for the kind of algorithmic and problem-solving questions common in those interviews.

### Final Thoughts

I'm really proud of how this project turned out. It started as a personal need – wanting better interview practice – and grew into a functional tool that combines several cool AI technologies. Building the conversational flow with LangGraph, integrating the whiteboard feature with Gemini, and getting detailed, grounded feedback feels like a significant step towards making AI a genuinely helpful partner in interview preparation.

**Want to see it in action or peek at the code?**

You can find the complete project in my Kaggle Notebook:

[**>>> Mock Technical Interviewer with Gemini & LangGraph <<<**](https://www.kaggle.com/code/maxymrim/mock-technical-interviewer-with-gemini-langgraph)

[**>>> Video Project Breakdown + Demo <<<**](https://www.youtube.com/watch?v=8ct_uJ772Zk)

Give it a try! I'd be thrilled to hear your feedback, suggestions, or ideas in the comments on Kaggle or below! What features would *you* find most helpful?
