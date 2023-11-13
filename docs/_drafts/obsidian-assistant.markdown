---
layout: post
title:  "Getting into LLM Agents: Building an Assistant (aka ChatGPT) for my Knowledgebase"
date:   2023-12-29 16:00:00 +0100
category: Work
tags: MachineLearning
---
![AI's Stone](/images/obsidian-assistant/ai-s-stone.jpg)
*"AI's Stone" (this image was created with the assistance of DALL·E 3)*

Explain your motivation for wanting to build a personalized content chatbot using your markdown notes from Obsidian.
<!--more-->
    
Resources: 
- [llm agents][llm-agent]
- [content chatbot][content-chatbot].
  
## Establishing the concepts
Briefly introduce the concept of LLM agents and their applications. Discuss the history and development of LLM agents, including their evolution from simple chatbots to more complex agents.

## Learning from a "minimal example"
For the obsidian-chatbot I want to use langchain, but it has already reached a point where it has become very complicated with lots of abstractions. Maybe we can understand how an llm-agent works with a few simple lines of code. Doing some research, I came across the [llm_agents repository by Marc Päpper] and found it to be a great resource for understanding some concepts. Let's take a look at some code passages and break down how an agent works under the hood.  
  
First of all, we want to have an entrypoint for running the agent. This entrypoint is the `run_agent.py` file. It takes user input, initializes an agent using the Agent class, and then runs the agent to generate a response. The agent is equipped with various tools such as PythonREPLTool, SerpAPITool, and HackerNewsSearchTool. The final answer from the agent is printed.  
```python
from llm_agents import Agent, ChatLLM, PythonREPLTool, SerpAPITool, HackerNewsSearchTool

if __name__ == '__main__':
    prompt = input("Enter a question / task for the agent: ")
    agent = Agent(llm=ChatLLM(), tools=[PythonREPLTool(), SerpAPITool(), HackerNewsSearchTool()])
    result = agent.run(prompt)

    print(f"Final answer is {result}")
```
### The Agent class - coordinating tools
The Agent class orchestrates the interaction between the language model (provided with the variable `ChatLLM`) and various tools. It has methods to run the agent, decide the next action based on the generated prompt, and parse the output to determine the next tool to use. Everything in the Agent class operates on a prompt template:  
```python
PROMPT_TEMPLATE = """Today is {today} and you can use tools to get new information. Answer the question as best as you can using the following tools: 

{tool_description}

Use the following format:

Question: the input question you must answer
Thought: comment on what you want to do next
Action: the action to take, exactly one element of [{tool_names}]
Action Input: the input to the action
Observation: the result of the action
... (this Thought/Action/Action Input/Observation repeats N times, use it until you are sure of the answer)
Thought: I now know the final answer
Final Answer: your final answer to the original input question

Begin!

Question: {question}
Thought: {previous_responses}
"""
```  
The heart of the Agent is a loop that decides, based on the previous response, which tool to use next. Then the `use()` method of the chosen tool is called and the result is appended to the prompt as "Observation":  
```python
while num_loops < self.max_loops:
    num_loops += 1
    curr_prompt = prompt.format(previous_responses='\n'.join(previous_responses))
    generated, tool, tool_input = self.decide_next_action(curr_prompt)
    if tool == 'Final Answer':
        return tool_input
    if tool not in self.tool_by_names:
        raise ValueError(f"Unknown tool: {tool}")
    tool_result = self.tool_by_names[tool].use(tool_input)
    generated += f"\n{OBSERVATION_TOKEN} {tool_result}\n{THOUGHT_TOKEN}"
    print(generated)
    previous_responses.append(generated)
```  
And how does the Agent actually know when the final answer is reached? Again, this comes from the original prompt template. In the template, the LLM is instructed to "...repeat N times, use it until you are sure of the answer". When sure, it's instructed to print the following: "Thought: I now know the final answer; Final Answer: your final answer to the original input question"  

### The ChatLLM class - language model interaction
The ChatLLM class interacts with the language model (in this case, 'gpt-3.5-turbo') using the OpenAI API. It generates responses based on the provided prompt (which is updated in each step of the conversation), and the temperature parameter controls the "creativity" of the generated response.

### The tools
I'm only going to briefly describe two tools that are implemented, the REPL tool and the google search tool:  
- The PythonREPLTool is a tool that simulates a Python REPL (Read-Eval-Print Loop). It allows the agent to execute Python commands provided by the user. The `use` method of this tool runs a Python command and returns the output.  
- The SerpAPITool is a tool for fetching specific information from Google search results. It uses the SerpAPI library to perform a Google search based on the user's input and returns relevant information from the search results.  
  
To construct tools with a consistent usage pattern, each time that a tool-class is created, the ToolInterface class is used as an abstract base class to define a common interface.  
```python
from pydantic import BaseModel

class ToolInterface(BaseModel):
    name: str
    description: str
    
    def use(self, input_text: str) -> str:
        raise NotImplementedError("use() method not implemented")  # Implement in subclass
```  
Each tool (e.g., PythonREPLTool, SerpAPITool) must implement the `use` method, specifying how it processes input. This is expressed in code by having an abstract `use()` method in the ToolInterface base class which raises a NotImplementedError. As such, the base class enforces that all subclasses must implement this method. This ensures that every tool, regardless of its specific functionality, must adhere to a common interface.  
  
In summary, the agent combines a language model with various tools to provide a versatile response system. What it really does is to provide a framework of a thought process. Then it guides the LLM step by step through this thought process until the LLM has enough information to reach a final answer.  

## Building a personalized assistant for notes
Introduce the content-chatbot repository and its purpose.
Explain how this chatbot differs from a traditional website-based chatbot.
Describe your vision of using Obsidian and markdown notes as the knowledge base for your chatbot.
Explain how you plan to utilize the markdown format for training the chatbot.
Highlight the advantages of using a personal knowledge base in this context.

## Time to code
Share the process of adapting the content-chatbot code to work with your Obsidian notes.
Discuss any challenges you faced and how you overcame them.
Provide code snippets or examples to illustrate key steps in the implementation.

## Testing and Fine-Tuning
Describe your experience testing the chatbot with different queries and scenarios.
Discuss any initial results and observations.
Explain how you plan to fine-tune the chatbot's responses for better accuracy.

## Future Directions and Potential Improvements
Share your thoughts on potential enhancements or extensions to the project.
Discuss any additional features or functionalities you might consider adding in the future.

## Closing thoughts
Summarize your journey of learning about LLM agents and building a personalized content chatbot.
Reflect on the insights and skills you gained throughout the process.
Encourage readers to explore the possibilities of LLM agents for their own projects.
Share any final thoughts, acknowledgments, or resources for further reading on LLM agents.


[llm-agent]: https://github.com/mpaepper/llm_agents
[content-chatbot]: https://github.com/mpaepper/content-chatbot