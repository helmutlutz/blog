---
layout: post
title:  "LLM Agents: Crafting an Intelligent Assistant for Knowledge and Code"
date:   2024-03-07 13:00:00 +0100
category: Work
tags: MachineLearning LLMs Agents KnowledgeManagement
---
![AI's Stone](/images/obsidian-assistant/ai-s-stone.jpg)
*"Philosopher's Stone" (this image was created with the assistance of DALL·E 3)*

Today's post is about building a virtual assistant and the lessons learned along the way. It can answer questions from my Obsidian notes, search the web, and execute Python code.  
<!--more-->
  
Obsidian, in brief, is something like a personal wiki, a place to store notes from books, personal thoughts and things that I learned over time. It also contains information on personal projects, timelines, available resources, etc. - all in all, everything a personal assistant could wish for to make smart (or at least informed) guesses when prompted for advice.  
  
Here is the list of resources that I used and took inspiration from for this project: 
- Marc's [post][paeppers-llm-agent] on building an LLM agent from scratch
- Kai's [post][kaichaos-qa-bot] on building a Q&A bot to query a PDF
- Chanin's [post][st-llama2-chatbot] on building a Llama2 chatbot with replicate
- Eugene's [post][eugenes-obsidian-assistant] on building an a very similar Obsidian copilot

## Fundamental concepts
Virtual assistant is one way to call it. More commonly however, they are called LLM agents. These are programs that leverage Large Language Models (LLMs) to make decisions about when and how to utilize one tool out of their own toolbox in order to accomplish their task. Provided with the right tools, agents can interact with external systems, overcoming inherent limitations of LLMs such as knowledge cutoffs, hallucinations, or doing exact calculations. Tools can take different forms, ranging from API calls (think of searching the web) and Python functions to webhook-based plugins.  
  
So, how would the interaction with an agent look like? A straightforward method involves passing an entire description of the task to an LLM - with a curated list of tools, instructions and all other information it might need. Where not so long ago, you would write a code with many if-else statements to define the behavior of program, nowadays you can rely on LLMs to "understand" what you want from plain English.  
An example of such a prompt might resemble the following, potentially incorporating few-shot examples to enhance the LLM's accuracy in selecting the appropriate tool. There will be more detailed examples following.
```python
"""
Your task is to select a tool to answer a user question. You have access to the following tools.

search: search for an answer in FAQs
order: order items
noop: no tool is needed

{few shot examples: Provide a written example to the LLM of how it should behave}

Question: {input}
Tool:
"""
```
  
A typical LLM agent follows this sequence:  
- *User request*: The program receives a user input, e.g., "Where is my order 123456?" from a client application.  
- *Plan next action(s) and select tool(s) to use*: The program prompts the LLM to generate the next action and suggests a tool name, like OrdersAPI, from a predefined list. Alternatively, the LLM may generate an API call directly.  
- *Parse tool request*: Validate the tool/action prediction to ensure accuracy and prevent hallucinations. This may involve a separate LLM call.  
- *Invoke tool*: If tool names and parameters are valid, the tool is invoked through actions like HTTP requests or function calls.  
- *Parse output*: Process the tool's response, extracting relevant information in a standardized format.  
- *Interpret output*: Prompt the LLM to make sense of the output, deciding whether to generate a final answer or require additional actions.  
- *Terminate or continue to step 2*: Return a final answer or a default response in case of errors or timeouts.  
  
Agent frameworks may differ; for instance, ReAct combines tool selection and answer generation in a single prompt. The logic can run in a single pass or an agent loop that terminates upon generating a final answer, throwing an exception, or reaching a timeout. Despite variations, agents consistently leverage the LLM to orchestrate planning and tool invocations until task completion.

## Learning from a minimal example
For the Obsidian-chatbot I want to use LangChain, but it has already reached a point where it has become very complicated with lots of abstractions. Maybe we can understand how an LLM Agent works with a few simple lines of code. Doing some research, I came across the `llm_agents` [repository][paeppers-agent-repo] by Marc Päpper and found it to be a great resource for understanding some concepts. Let's take a look at some code passages and break down how an agent works under the hood.  
  
First, we want to have an entrypoint for running the agent. This entrypoint is the `run_agent.py` file. It takes user input, initializes an agent using the Agent class, and then runs the agent to generate a response. The agent is equipped with various tools such as PythonREPLTool, SerpAPITool, and HackerNewsSearchTool. The final answer from the agent is printed.  
```python
from llm_agents import Agent, ChatLLM, PythonREPLTool, SerpAPITool, HackerNewsSearchTool

if __name__ == '__main__':
    prompt = input("Enter a question / task for the agent: ")
    agent = Agent(llm=ChatLLM(), tools=[PythonREPLTool(), SerpAPITool(), HackerNewsSearchTool()])
    result = agent.run(prompt)

    print(f"Final answer is {result}")
```

### The Agent class - orchestrator of tools
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
How does the agent know when to stop and return a final answer? Again, this comes from the original prompt template. In the template, the LLM is instructed to "...repeat N times, use it until you are sure of the answer". When sure, it's instructed to print the following: "Thought: I now know the final answer; Final Answer: your final answer to the original input question"  

### The ChatLLM class - language model interaction
The ChatLLM class interacts with the language model (in this case, 'gpt-3.5-turbo') using the OpenAI API. It generates responses based on the provided prompt (which is updated in each step of the conversation), and the temperature parameter controls the "creativity" of the generated response.

### Providing the tools
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

## My version of a personal assistant
In the previous section we talked about the "high-level view" and how an agent works under the hood. In addition to the minimal example from before, I want the agent to be able to answer questions with information from my Obsidian notes.  
If you're not familiar with Obsidian, here is an excerpt from Wikipedia: "Obsidian is a personal knowledge base and note-taking software application that operates on Markdown files. It allows users to make internal links for notes and then to visualize the connections as a graph."  
So, if you are really consistent in taking notes, these files should be a wonderful resource for an agent who is presented not only with technical problems, but also serves as a personal advisor. My hope is, that eventually, the agent can help me to manage time, resources, and even act as a sparring partner for projects (brainstorming, programming etc.).   
One of the reasons I chose Obsidan, was that it stores everything in plain markdown files (essentially like `.txt` files). This "openness" makes it easy to process all the data at once.  
  
In this section, you will see how I built the "Obsidian Search Tool". It's of course only one way of building it. In principle you could also use the minimal agent library that I showed in the last section, Huggingface's Transformers Agents, or any other of the currently spawning agent libraries. Afterwards, we assemble the pieces: agent, tools, and all the wiring in between. 

### Building a vector store for the documents
Easy things first. I loaded all the documents in my Obsidian notes folder and split the documents into chunks.  
```python
def get_markdown_files(path):
    markdown_files = []
    
    for root, dirs, files in os.walk(path):

        for file in files:

            if "xRemove" in file:
                continue

            if file.endswith(".md"):
                full_file_path = os.path.join(root, file)
            
            markdown_files.append(os.path.relpath(full_file_path, path))
                
    return markdown_files
```
  
These markdown files need to be split into chunks and stored in a dictionary for better retrieval of long and short contexts:  
```python
def chunk_doc_to_dict(lines: list, min_chunk_lines=3) -> dict:
    """Create chunks from a document where each new paragraph / top-level bullet is a new chunk.

    :param lines:           List of lines in a document 
    :param min_chunk_lines: Minimum number of lines in a chunk

    :Return:                Dictionary of chunks.
    """

    chunks = defaultdict()
    current_chunk = []
    chunk_idx = 0
    current_header = None

    for line in lines:
        
        if line.startswith('\n'):  # Skip empty lines
            continue        
		
		# You may want to add more conditions to skip tags, sources, hyperlinks etc.

        if '##' in line:  # Chunk header = Section header
            current_header = line

        if line.startswith('- '):  # Top-level bullet
            
            if current_chunk:  # If chunks accumulated, add it to chunks
            
                if len(current_chunk) >= min_chunk_lines:
                    chunks[chunk_idx] = current_chunk
                    chunk_idx += 1
                current_chunk = []  # Reset current chunk
            
                if current_header:
                    current_chunk.append(current_header)

        current_chunk.append(line)

    # The previous loop only adds current_chunk to chunks when it encounters a new top-level bullet, and there are no more top-level bullets after the last line. So, we need to manually add the last chunk.
    if current_chunk:
        
        if len(current_chunk) > min_chunk_lines:
            chunks[chunk_idx] = current_chunk

    return chunks


def create_vault_dict(vault_path: str, paths: list) -> dict:
    """Iterate through all paths and create a vault dictionary

    Args:
        vault_path: Path to obsidian vault
        paths: Relative path of documents in obsidian vault

    Returns:
        Dictionary of full doccuments and chunks of documents
    """

    vault = dict()

    for filename in paths:
        with open(os.path.join(vault_path, filename), 'r', encoding='utf-8', errors='replace') as f:
            lines = f.readlines()
            chunks = chunk_doc_to_dict(lines)

            if len(chunks) > 0:  # Only add documents with chunks to the dict

                for chunk_id, chunk in chunks.items():
                    chunk_id = f'{filename}-{chunk_id}'

                    # Also add each chunk to the vault dict (for short contexts)
                    vault[chunk_id] = {'title': filename,
                                       'type': 'chunk',
                                       'path': str(filename),
                                       'chunk': ''.join(chunk)}

    return vault
```
  
Next is the so called "embedding": I will transform the chunks into vectors using the `all-MiniLM-L6-v2` sentence transformer model. A sentence transformer maps sentences & paragraphs to a 384-dimensional dense vector space and can be used for tasks like clustering or semantic search - and semantic search is what we want.  
Where do you typically store data? In a database, or a data warehouse. For vectors that is simply called *vector store*.  
> A vector is a data structure to store a sequence of numbers, and a vector *store* is a database for vectors. During the embedding, vectors are created from the data such that similar data points are close together in *vector space*. That is why similarity can be computed more efficiently and you can retrieve items even if they don't exactly match the query.  
  
The embedding and storing is both handled by the function `Chroma.from_documents(documents, embedding-transformer, persist_directory)` (Side note: *embedding-transformer* here refers to anything that computes, or *transforms* strings to vectors).   
```python
def convert_vaultdict_to_documents(vault: dict):
    docs = []
    for k, v in vault.items():
        content = v["chunk"]
        meta = {"origin": str(k), "type": v["type"]}
        docs.append(Document(page_content=content, metadata=meta))
    
    return docs


def create_vector_db(vault: dict, save_path) -> dict:
    """
    Convert the dictionary of chunks to a vector database.
    :param vault_dict:      dict
    :return:                ChromaDB instance
    """

    docs = convert_vaultdict_to_documents(vault)
    model_id = 'sentence-transformers/all-MiniLM-L6-v2'
    model_kwargs = {'device': 'cpu'}
    hf_embedding = HuggingFaceEmbeddings(
        model_name=model_id,
        model_kwargs=model_kwargs
    )
    db = Chroma.from_documents(
        documents=docs, 
        embedding=hf_embedding,
        persist_directory=save_path
    )

    return
```   
  
There is a plethora of vector store APIs available within LangChain, so how to choose the right one? Initially I came across these two examples: in-memory document arrays and Chroma. Chroma is a full-featured vector database for production usage. Key advantages are:  
- Persistent storage of vectors and metadata
- Indexing for fast similarity search
- Support for adding, deleting, updating documents
- Can scale to large document collections (not needed here but nice to have)
An in-memory document array is simpler, meant for testing/prototyping:
- Stores documents and vectors in memory, lost on restart
- No indexing for search, have to compare vectors directly
- Not built to scale  
In summary, Chroma is designed for production usage. In-memory document array is for simpler testing/prototyping. A good resource for making an informed decision is [this guide][choose-vector-store]
  
### Bringing it all together: Building the agent
Finally, we are ready to assemble the pieces: agent, tools, and all the wiring in between. First you will need to decide which *agent type* you want to use. With LangChain, there are a couple of agent types that are closely tied to OpenAI models. My rationale here was, I would rather not be immediately tied to one vendor, so I went with a simple but effective ReAct-type agent.  
> ReAct is a pattern to help LLMs solve tasks by generating both reasoning and actions. Reasoning helps the LLM to plan, update, and explain its actions, while actions help the model to interact with external sources, such as websites or environments, to get more information.  
  
To use it, you need this pattern:
```python
agent = create_react_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)
```
  
You see, we need three things, the LLM, toolbox, and a system prompt which is always used at the beginning of a chat. 
1. The LLM: I didn't want to use a particular proprietary model, but an *open* model that is available from a myriad of vendors. Llama2 is a good candidate here since it's available in Replicate, Amazon Bedrock, Huggingface, and many more. You can go to the [LangChain Docs here][langchain-llm-interfaces] and see if your chosen vendor is supported. As you can see in the code below, I chose Replicate and implemented it according to LangChain's documentation.
2. The toolbox: As explained in the section *Learning from a minimal example / Providing the tools*, you need to construct the tools in a certain way for the agent to be able to use them. The web-search and python REPL are built-in tools, they are documented [here][langchain-builtin-tools]. For the Obsidian-tool, I implemented a retrieval tool to extract information from the Obsidian vector store according to [this guide][conversational-retrieval-agents]. On top, I added Contextual Compression as described [here][contextual-compression].  
> The idea of contextual compression is to compress retrieved documents from the vector store, using the context of the given query, so that only the relevant information is returned.
> Under the hood, the agent sends the retrieved chunks to the LLM to extract relevant information. To do that it prepends each query with the following prompt:  
> """  
> Given the following question and context, extract any part of the context *AS IS* that is relevant to answer the question. If none of the context is relevant return NO_OUTPUT.  
>   
> Remember, *DO NOT* edit the extracted parts of the context.  
> """    
3. And last but not least, the prompt: You cannot simply pass a string as prompt. In LangChain, the prompt needs to be a certain object, because the agent needs to populate it with information about the available tools, the chat history, and the user query. The following does the trick:  
  
```python
from langchain.prompts import PromptTemplate

raw_template = """
    You are a helpful and honest assistant. If a question is not factually coherent, 
    explain why instead of answering something not correct. If you don't know the answer to a question, 
    just say so and don't share false information. You try to avoid fictional, irrelevant, 
    or repetitive sentences in your answers.\n\n
  
    You have access to the following tools:\n\n
    
    {tools}\n\n
    
    To use a tool, please use the following format:\n\n
    
    \n
    Thought: Do I need to use a tool? Yes\n
    Action: the action to take, should be one of [{tool_names}]\n
    Action Input: the input to the action\n
    Observation: the result of the action\n
    \n\n
    
    After using a tool, you MUST formulate your final answer, even if the obtained information is incomplete. 
    You MUST use the format:\n\n
    
    \n
    Thought: Do I need to use a tool? No\n
    Final Answer: [your response here]\n
    \n\n
    
    Begin!\n\n
    
    Previous conversation history:\n
    {chat_history}\n\n
    
    New input: {input}\n
    {agent_scratchpad}
"""

def create_prompt_template(rt=raw_template):
    input_variables = ["tools", "tool_names", "chat_history", "input", "agent_scratchpad"]

    return PromptTemplate(template=rt, input_variables=input_variables)
```  
  
Here is how it all comes together:   
```python
def initialize_agent():
    # Configure the LLM API connection
    
    model_args = {
        "temperature":temperature, 
        "top_p":top_p, 
    }
    llm = Replicate(
        model=llm_str, 
        model_kwargs=model_args
    )
    compressor = LLMChainExtractor.from_llm(llm)

    # Creating the toolbox that the agent will use
    
    ## Search
    search_tool = create_retriever_tool(
        retriever=TavilySearchAPIRetriever(k=3),
        name="SEARCH",
        description="Conduct a general internet search for information. "
        "Result will be an answer generated from the retrieved search results. ",
    )
    
    ## Obsidian vault DB
    obsidian_compression_retriever = ContextualCompressionRetriever(
        base_compressor=compressor, 
        base_retriever=vector_db.as_retriever()
    )
    obsidian_retriever_tool = create_retriever_tool(
        retriever=obsidian_compression_retriever,
        name="OBSIDIAN",
        description="Search the Obsidian vault. Input should be a question about notes in the Obsidian vault, "
        "Result will be an answer generated from the retrieved text snippets.",
    )

    ## Python REPL
    python_repl = PythonREPL()
    repl_tool = Tool(
        name="PYTHONSHELL",
        description="A Python shell. Use this to execute python commands. Input should be a valid python command. "
        "If you want to see the output of a value, you should print it out with `print(...)`.",
        func=python_repl.run,
    )

    tools = [obsidian_retriever_tool, search_tool, repl_tool]

    # Initializing the agent

    agent = create_react_agent(llm, tools, create_prompt_template())
    agent_executor = AgentExecutor(
        agent=agent, 
        tools=tools, 
        verbose=True, 
        max_iterations=5,
        handle_parsing_errors="""
        Check your output and make sure it is in the following format:\n 
        Thought: Do I need to use a tool? No\n
        Final Answer: [your response here]\n
        """
    )

    return agent_executor
```
  
## Assistant in action:
Here are some test inputs. The following question tests the Obsidian-retriever tool.  
**Query:**  
```  
Summarize my Obsidian notes on the repository architecture pattern.
```  
**Excerpt from the original Obsidian note:**  
```
- The repository pattern is an abstraction over data storage. It allows to decouple the model layer from the data layer
- The goal (and minimal viable product) here is to get to a web API as quickly as possible. We need a way to retrieve batch info from the database and instantiate our domain model objects from it. And we need a way of saving them back to the database
- Applying the Dependency Inversion Principle (DIP) to data access:
	- The classical layered architecture has every layer depend only on the layer below it: presentation layer -> business logic -> database layer (->: depends on)
	- We want the domain model to have *no dependencies whatsoever*. presentation layer -> business logic <- database layer, sometimes called the *onion architecture*
	- Inverting the dependency:
		- SQLAlchemy also allows for "classical" mapping, where you define your schema separately, and define an explicit mapper for how to convert between schema and domain model
 
[*... it goes on with some code examples (which are skipped by the Assistant) and finally gets to advantages and disadvantages (which are picked up by the Assistant)*]
```  
**Assistant's answer:**  
```
The repository pattern is a design pattern that decouples data access from the business logic of an application. It defines a simple interface between the application and the data storage, allowing the application to use different data storage solutions without changing the code.
 
The repository pattern is based on the dependency inversion principle (DIP), which states that high-level modules should not depend on low-level modules, but rather both should depend on abstractions. In the context of data access, this means that the domain model should not depend on the data storage technology, but rather both should depend on an abstraction, such as an interface or a port.
 
The advantages of the repository pattern include:
 
   - Simple interface between domain model and storage
   - Easy to unit test and swap out different storage solutions
   - Complete control over the mapping of objects to the database
 
The disadvantages of the repository pattern include:
 
   - An ORM already brings some decoupling
   - Maintaining ORM mappings requires work and extra code
   - Increases maintenance costs and is often met with incomprehension by others
 
If your app is just a simple create-read-update-delete (CRUD) wrapper around a database, you don't need a domain model or repository.
 
I hope this summary helps! Let me know if you have any further questions.
```
Another example, this time using the web search:  
  
**Query:**  
```  
Which websites provide CSV data on economic indicators of Germany's major industry sectors?
```  
**Assistant's answer:**  
```
The websites that provide CSV data on economic indicators of Germany's major industry sectors are:

    ifo Institute (https://www.ifo.de/en/node/51407)
    Federal Statistical Office (https://www.destatis.de/EN/Themes/Economy/Short-term-indicators.html)

Explanation: The ifo Institute provides a monthly survey of industrial enterprises in Germany, which includes information on production impediments caused by scarce raw materials and intermediate products. The Federal Statistical Office publishes detailed tables with monthly or quarterly results for various domains, including economic indicators, and provides the table data in CSV format for further processing.
```  
And a last example using the Python tool:  
  
**Query:**  
```
Write python code to generate 16 prime numbers in a for loop. Then store the numbers in a pandas dataframe  
```  
**Assistant's answer:**  
```
Sure, I can help you with that! Here's an example Python code that generates 16 prime numbers using a for loop and stores them in a Pandas DataFrame:
    ```python
    import numpy as np
    import pandas as pd

    # Generate 16 prime numbers using a for loop
    primes = []
    for i in range(16):
    is_prime = True
    for j in range(2, int(i ** 0.5) + 1):
        if i % j == 0:
            is_prime = False
            break
    if is_prime:
        primes.append(i)

    # Create a Pandas DataFrame with the generated prime numbers
    df = pd.DataFrame({'Prime Numbers': primes})

    print(df)
    ```
This code uses a for loop to iterate from 2 to 16, and for each number, it checks whether it is prime by iterating from 2 to its square root and checking if it has any factors. If it doesn't have any factors, it is considered prime, and its value is appended to a list called `primes`. Finally, the prime numbers are stored in a Pandas DataFrame called `df` with a column name of "Prime Numbers". The resulting DataFrame looks like this:
    ```  
    Prime Numbers
    0          2
    1          3
    2          5
    3          7
    4         11
    5         13
    6         17
    7         19
    8         23
    9         29
    10        31
    11        37
    12        41
    13        43
    14        47
    15        53
    16        59
    ```
```  
## Random tips and lessons
- Pay attention to use the correct prompt formatting needed for the respective model. In my case, Llama2 needed that the user input in a chat history be wrapped with `[INST]...[/INST]` tags: 
```python
if dict_message["role"] == "user":
    string_dialogue += "[INST] " + dict_message["content"] + " [/INST]\n"
```
- Leave out large documents in the vector store: In the beginning, I also stored chunks AND entire notes in the vector store. The Obsidian tool sends all similar documents retrieved to the LLM, with entire notes quickly exceeding the input token limit of 4096. In addition, it just causes unneccessary additional costs.
- I couldn't figure out how to apply compression to the result of search tools like SerpAPI (I guess this will apply to other search tools as well). I switched to using Tavily, which has been optimized for usage with LLMs. As far as I can see the search results are already processed internally by their own LLM. I saw that you can actually apply contextual compression but only if you use `TavilySearchAPIRetriever` - so the retriever version of the Tavily-wrapper.
- When a query triggers the wrong tool, the agent can continue forever. A simple safeguard to cap the max number of iterations, is to use `max_iterations` when setting up the `AgentExecutor`. 
- And why on earth does Llama2 sometimes continue in the background with the question "What is the meaning of life"? Well, turns out that when you simply enter a space " " in the query field and hit enter, the agent jumps into a new chain with exactly this question in mind. I'm not going down this rabbit hole today - I'll fix that on some other day ;)
- `llama-2-70b-chat` just cannot code at all. For coding tasks, I switched to `codellama-34b-instruct`. A strange habit of codellama is that it goes on with random questions like "What is the mean of the first row?", "What is the standard deviation of the second column?", or "What is the median of the third row". Another issue for another day.
- You might have noticed that I instructed the assistant to formulate a response immediately after using a tool, essentially preventing it from going from one tool to another. I did this because I noticed that it was forgetting the original system prompt, not sticking to the required output formatting, AND exceeding the token limit. There must be a way to limit what the agent stores in agent_scratchpad, but that is another point on my Todo-list. 
- Finally: Watch your costs! It's not a lot, but it quickly adds up when your dear assistant goes on pondering the question "What is the meaning of life?".
  - $0.15 for the query to summarize an Obsidian note. I was using `llama-2-70b-chat` (at that time the largest Llama2 on replicate).
  - $0.08 for a web search.
  - $0.17 for the small python task.

## Future Directions and Potential Improvements
In my next post I'll definitely look into adding memory to my assistant, but I'm sure that I will find many more potential improvements to make the quality of responses more consistent. As you continue your journey with building a personalized content chatbot using LangChain, consider the following future directions and potential enhancements:

1. Fine-Tuning and Customization:
  - Explore fine-tuning your ReAct-type agent to better align with your specific use case.
  - Customize the agent’s behavior, responses, and knowledge base to cater to your audience or domain.
2. Multi-Modal Capabilities:
  - Extend your chatbot beyond text by incorporating multi-modal capabilities.
  - Integrate image recognition, voice input, or other sensory inputs to enhance user interactions.
3. Contextual Understanding:
  - Enhance the agent’s ability to understand context across multiple turns in a conversation.
  - Investigate techniques for maintaining context and coherence over longer interactions.
4. User Feedback Loop:
  - Implement a feedback loop where users can provide corrections or additional context.
  - Use this feedback to improve the agent’s performance and adapt to user preferences.
5. Ethical Considerations:
  - Reflect on ethical implications related to privacy, bias, and fairness.
  - Ensure that your chatbot respects user privacy and avoids harmful or discriminatory behavior.
6. Integration with External APIs:
  - Connect your chatbot to external APIs, databases, or services.
  - Provide real-time information, weather updates, or personalized recommendations.

## Closing Thoughts
My journey into the world of large language models (LLMs) has been both exciting and fun. I encourage you to dive deeper into LLM agents, experiment, and create your own applications and experiences! 🌟🤖

Acknowledgments: I’d like to express my gratitude to the LangChain community, fellow developers, and the creators of LLMs. Your collective efforts inspire us all. Happy building! 🙌

[eugenes-obsidian-assistant]: https://eugeneyan.com/writing/obsidian-copilot/
[paeppers-llm-agent]: https://www.paepper.com/blog/posts/intelligent-agents-guided-by-llms/
[paeppers-agent-repo]: https://github.com/mpaepper/llm_agents
[kaichaos-qa-bot]: https://randomdotnext.com/q-a-bot-using-langchain-huggingface-embedding-openai/
[st-llama2-chatbot]: https://blog.streamlit.io/how-to-build-a-llama-2-chatbot/
[choose-vector-store]: https://js.langchain.com/docs/modules/data_connection/vectorstores/
[langchain-llm-interfaces]: https://python.langchain.com/docs/integrations/llms/
[langchain-agent-quickstart]: https://python.langchain.com/docs/modules/agents/quick_start
[langchain-builtin-tools]: https://python.langchain.com/docs/integrations/tools
[conversational-retrieval-agents]: https://python.langchain.com/docs/use_cases/question_answering/conversational_retrieval_agents
[contextual-compression]: https://python.langchain.com/docs/modules/data_connection/retrievers/contextual_compression/