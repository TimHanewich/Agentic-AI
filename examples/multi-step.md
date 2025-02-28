# Agentic AI Example: Agent Using Multi-Step Problem Solving via Tool Calling
Advanced large language models with tool calling capabilities often have the intelligence to perform multiple steps in a particular sequence to successfully fulfill the requests of the user/triggering agent.

I will demonstrate this in the following example. In the following, I am creating a lightweight *ghost writer* agent. This agent's purpose will be to help me by drafting articles for me to later review and potentially publish myself. However, I also want my agent to search the web for inspiration before it writes the article. And finally, I don't want the agent to just give me the article in the chat (for me to copy + paste out), I want it to programmatically save it to somewhere for me to later retrieve and review.

*The following example was performed on `GPT-4o-mini` via Azure OpenAI Service.*

## Step 1: You provide the agent with system instructions, available tools, and a task
In the below example, you'll see I am sending the agent several things:
1. In the `messages` array, the agent's system prompt is specified. It is given explicit instructions on its role, it's ultimate goal, but more importantly *how it will accomplish the goal*.
2. The agent is provided **tools** in the `tools` array. These are "tools" or capabilities that the agent will have available to itself in its task of fulfilling the user's request.
3. The agent is given a specific request. In the `messages` array, after the system message, you'll see a user message. This user message is the user's prompt (request) to the agent. You'll see that, in this case, the agent's task is to write an article about sharks.

With those three critical ingredients provided to the agent (explicit system instructions, tools to accomplish it's task, and an assigned task), the agent is asked to execute the task.

```
{
    "messages": 
    [
        {
            "role": "system",
            "content": "You are a ghost writer that works for me. Your job is to take ideas I have and use them as inspiration to write new articles. I will come to you and ask you to write an article for me on a given subject. First, you will search the web using your 'search_web' tool to search the web for information on this subject. You will then write the article using the information you learned from the web search. Then, you will save the article for me to later review using your 'save_article' tool. After this is all completed, tell me it is completed in a kind and sincere tone."
        },
        {
            "role": "user",
            "content": "Write me an article about sharks."
        }
    ],
    "tools": 
    [
        {
            "type": "function",
            "function": 
            {
                "name": "search_web",
                "description": "Search the web for content about a particular subject.",
                "parameters": 
                {
                    "type": "object",
                    "properties": 
                    {
                        "subject": 
                        {
                            "type": "string",
                            "description": "The subject or topic to search the web about."
                        }
                    },
                    "required": ["subject"]
                }
            }
        },
        {
            "type": "function",
            "function": 
            {
                "name": "save_article",
                "description": "Save an article about a topic as a draft.",
                "parameters": 
                {
                    "type": "object",
                    "properties": 
                    {
                        "article_name": 
                        {
                            "type": "string",
                            "description": "The name of the article."
                        },
                        "article_content": 
                        {
                            "type": "string",
                            "description": "The content (body) of the article."
                        }
                    },
                    "required": ["article_name", "article_content"]
                }
            }
        }
    ]
}
```

The above JSON body is provided to the agent via a POST request to the Chat Completions endpoint.

## Step 2: The Agent Determines What to Do: Search the Web
After sending this to the agent via POST request, the agent determines what to do and responds back (this is just the `message` property stripped out):

```
{
    "content": null,
    "refusal": null,
    "role": "assistant",
    "tool_calls": 
    [
        {
            "function": 
            {
                "arguments": "{\"subject\":\"sharks\"}",
                "name": "search_web"
            },
            "id": "call_sC6KDI43ZfCanSPdzG5Ux596",
            "type": "function"
        }
    ]
}
```

You'll see that, in order to accomplish it's assigned task, the agent has decided to first search the web for the term `sharks`!

As specified in the system prompt (the agent's explicit instructions), the agent is correctly first performing research on the topic before writing the article it was asked to write.

## Step 3: We, as Developers, Perform The Web Search
The agent has decided to search the web for the term "sharks". As developers, it is now our responsibility to set up the programmatic mechanism that searches the web (i.e. Google or Bing search with top web pages, like Wikipedia for example) and then provides that content back to the agent. However, for the sake of this demonstration, I'll just be providing some simple facts about sharks to keep things easy.

What I will provide to the agent (let's pretend this is the result of the web search):

```
Here are some interesting facts about sharks: 1. They’ve been around forever! Sharks have existed for over 400 million years—meaning they were cruising the oceans long before dinosaurs roamed the Earth. 2. They have incredible senses. Sharks can detect electrical fields produced by other animals through special sensory organs called ampullae of Lorenzini. This allows them to locate prey even when it’s buried in sand! 3. Not all sharks have to keep swimming to survive. While many species need to swim constantly to push oxygen-rich water over their gills, some, like nurse sharks, can pump water through their gills while resting on the ocean floor.
```

## Step 4: We send the results of the web search back to the agent
Now that we have our web search complete and data collected, we need to send this back to the agent!

To do this, we must append both **the agent's original tool call message** AND **the data itself that corresponds to the tool call**.

The body of the next POST request would look like this:

```
{
    "messages": 
    [
        {
            "role": "system",
            "content": "You are a ghost writer that works for me. Your job is to take ideas I have and use them as inspiration to write new articles. I will come to you and ask you to write an article for me on a given subject. First, you will search the web using your 'search_web' tool to search the web for information on this subject. You will then write the article using the information you learned from the web search. Then, you will save the article for me to later review using your 'save_article' tool. After this is all completed, tell me it is completed in a kind and sincere tone."
        },
        {
            "role": "user",
            "content": "Write me an article about sharks."
        },
        {
            "content": null,
            "refusal": null,
            "role": "assistant",
            "tool_calls": 
            [
                {
                    "function": 
                    {
                        "arguments": "{\"subject\":\"sharks\"}",
                        "name": "search_web"
                    },
                    "id": "call_sC6KDI43ZfCanSPdzG5Ux596",
                    "type": "function"
                }
            ]
        },
        {
            "role": "tool",
            "tool_call_id": "call_sC6KDI43ZfCanSPdzG5Ux596",
            "content": "Here are some interesting facts about sharks: 1. They’ve been around forever! Sharks have existed for over 400 million years—meaning they were cruising the oceans long before dinosaurs roamed the Earth. 2. They have incredible senses. Sharks can detect electrical fields produced by other animals through special sensory organs called ampullae of Lorenzini. This allows them to locate prey even when it’s buried in sand! 3. Not all sharks have to keep swimming to survive. While many species need to swim constantly to push oxygen-rich water over their gills, some, like nurse sharks, can pump water through their gills while resting on the ocean floor."
        }
    ],
    "tools": 
    [
        {
            "type": "function",
            "function": 
            {
                "name": "search_web",
                "description": "Search the web for content about a particular subject.",
                "parameters": 
                {
                    "type": "object",
                    "properties": 
                    {
                        "subject": 
                        {
                            "type": "string",
                            "description": "The subject or topic to search the web about."
                        }
                    },
                    "required": 
                    [
                        "subject"
                    ]
                }
            }
        },
        {
            "type": "function",
            "function": 
            {
                "name": "save_article",
                "description": "Save an article about a topic as a draft.",
                "parameters": 
                {
                    "type": "object",
                    "properties": 
                    {
                        "article_name": 
                        {
                            "type": "string",
                            "description": "The name of the article."
                        },
                        "article_content": 
                        {
                            "type": "string",
                            "description": "The content (body) of the article."
                        }
                    },
                    "required": ["article_name", "article_content"]
                }
            }
        }
    ]
}
```

In the above payload you'll see that in the `messages` array, we've provided the original tool call the agent made and the results of that tool call that we made (the results of the web search). Note that the `tool_call_id` must be specified in the response, directly correlating which tool call response corresponds to what tool call by the agent.

You send this to the agent via a POST request.

## Step 5: The Agent Determines What to Do: Save the Article
With the web search performed and the results of the web search used as a reference, the agent then goes onto the next step. 

The response to the POST request in step 4 looks like this (this is just the message stripped out):

```
{
    "content": null,
    "refusal": null,
    "role": "assistant",
    "tool_calls": 
    [
        {
            "function": 
            {
                "arguments": "{\"article_name\":\"The Fascinating World of Sharks\",\"article_content\":\"### The Fascinating World of Sharks\\n\\nSharks have long captivated humans with their majestic presence in the oceans. These remarkable creatures have existed for over 400 million years—cruising the seas long before dinosaurs roamed the Earth. This article explores the incredible characteristics of sharks, their adaptations, and their role in marine ecosystems.\\n\\n#### Ancient Survivors\\nOne of the most fascinating aspects of sharks is their resilience. As some of the oldest living species on the planet, they have survived multiple mass extinctions and have undergone minimal evolutionary change. Their success can be attributed to their adaptable nature and specialized hunting techniques.\\n\\n#### Incredible Senses\\nSharks possess extraordinary sensory capabilities that make them formidable predators. They can detect electrical fields produced by other marine animals through specialized sensory organs called ampullae of Lorenzini. This remarkable ability allows them to locate prey even when it is buried in sand, providing them an advantage during hunts.\\n\\n#### Swimming to Survive?\\nWhile many shark species need to swim continuously to survive—pushing oxygen-rich water over their gills—some, like nurse sharks, have adapted to pump water through their gills while resting on the ocean floor. This unique adaptation allows them to conserve energy while still meeting their respiratory needs.\\n\\n### Conclusion\\nSharks are not just fierce predators; they are vital members of marine ecosystems. Understanding their biology and behavior helps us appreciate their role in maintaining the health of ocean environments. As we learn more about these incredible creatures, we can better protect them and their habitats, ensuring that they continue to thrive in our oceans for generations to come.\"}",
                "name": "save_article"
            },
            "id": "call_ZTty8BJYMIzN4JtDDKXAllcu",
            "type": "function"
        }
    ]
}
```

Following the original system instructions, the agent next decides to write and save the article about sharks. As instructed, it calls to the `save_article` tool that we gave it. The agent follows the proper input schema of this tool, specifying two properties in the `arguments` property: **article_name** and **article_content**.

The article the agent wrote:

```
### The Fascinating World of Sharks

Sharks have long captivated humans with their majestic presence in the oceans. These remarkable creatures have existed for over 400 million years—cruising the seas long before dinosaurs roamed the Earth. This article explores the incredible characteristics of sharks, their adaptations, and their role in marine ecosystems.

#### Ancient Survivors
One of the most fascinating aspects of sharks is their resilience. As some of the oldest living species on the planet, they have survived multiple mass extinctions and have undergone minimal evolutionary change. Their success can be attributed to their adaptable nature and specialized hunting techniques.

#### Incredible Senses
Sharks possess extraordinary sensory capabilities that make them formidable predators. They can detect electrical fields produced by other marine animals through specialized sensory organs called ampullae of Lorenzini. This remarkable ability allows them to locate prey even when it is buried in sand, providing them an advantage during hunts.

#### Swimming to Survive?
While many shark species need to swim continuously to survive—pushing oxygen-rich water over their gills—some, like nurse sharks, have adapted to pump water through their gills while resting on the ocean floor. This unique adaptation allows them to conserve energy while still meeting their respiratory needs.

### Conclusion
Sharks are not just fierce predators; they are vital members of marine ecosystems. Understanding their biology and behavior helps us appreciate their role in maintaining the health of ocean environments. As we learn more about these incredible creatures, we can better protect them and their habitats, ensuring that they continue to thrive in our oceans for generations to come.
```

## Step 6: We, as Developers, Save the Article
The agent wrote us a nice article and correctly used its tool to save its work as an article! Similarly to how we performed the web search, we must now programatically save this article to wherever we intended to save it; that could be somewhere locally on your computer or even on somewhere online like Medium or your OneDrive. 

Again, for the sake of this demo, we'll pretend it is saved successfully. Like last time, to confirm this to the agent, we must append both **the agent's original tool call** AND **the confirmation that the tool call was successful**.

The new body that we will POST will now look like this:

```
{
    "messages": 
    [
        {
            "role": "system",
            "content": "You are a ghost writer that works for me. Your job is to take ideas I have and use them as inspiration to write new articles. I will come to you and ask you to write an article for me on a given subject. First, you will search the web using your 'search_web' tool to search the web for information on this subject. You will then write the article using the information you learned from the web search. Then, you will save the article for me to later review using your 'save_article' tool. After this is all completed, tell me it is completed in a kind and sincere tone."
        },
        {
            "role": "user",
            "content": "Write me an article about sharks."
        },
        {
            "content": null,
            "refusal": null,
            "role": "assistant",
            "tool_calls": 
            [
                {
                    "function": 
                    {
                        "arguments": "{\"subject\":\"sharks\"}",
                        "name": "search_web"
                    },
                    "id": "call_sC6KDI43ZfCanSPdzG5Ux596",
                    "type": "function"
                }
            ]
        },
        {
            "role": "tool",
            "tool_call_id": "call_sC6KDI43ZfCanSPdzG5Ux596",
            "content": "Here are some interesting facts about sharks: 1. They’ve been around forever! Sharks have existed for over 400 million years—meaning they were cruising the oceans long before dinosaurs roamed the Earth. 2. They have incredible senses. Sharks can detect electrical fields produced by other animals through special sensory organs called ampullae of Lorenzini. This allows them to locate prey even when it’s buried in sand! 3. Not all sharks have to keep swimming to survive. While many species need to swim constantly to push oxygen-rich water over their gills, some, like nurse sharks, can pump water through their gills while resting on the ocean floor."
        },
        {
            "content": null,
            "refusal": null,
            "role": "assistant",
            "tool_calls": 
            [
                {
                    "function": 
                    {
                        "arguments": "{\"article_name\":\"The Fascinating World of Sharks\",\"article_content\":\"### The Fascinating World of Sharks\\n\\nSharks have long captivated humans with their majestic presence in the oceans. These remarkable creatures have existed for over 400 million years—cruising the seas long before dinosaurs roamed the Earth. This article explores the incredible characteristics of sharks, their adaptations, and their role in marine ecosystems.\\n\\n#### Ancient Survivors\\nOne of the most fascinating aspects of sharks is their resilience. As some of the oldest living species on the planet, they have survived multiple mass extinctions and have undergone minimal evolutionary change. Their success can be attributed to their adaptable nature and specialized hunting techniques.\\n\\n#### Incredible Senses\\nSharks possess extraordinary sensory capabilities that make them formidable predators. They can detect electrical fields produced by other marine animals through specialized sensory organs called ampullae of Lorenzini. This remarkable ability allows them to locate prey even when it is buried in sand, providing them an advantage during hunts.\\n\\n#### Swimming to Survive?\\nWhile many shark species need to swim continuously to survive—pushing oxygen-rich water over their gills—some, like nurse sharks, have adapted to pump water through their gills while resting on the ocean floor. This unique adaptation allows them to conserve energy while still meeting their respiratory needs.\\n\\n### Conclusion\\nSharks are not just fierce predators; they are vital members of marine ecosystems. Understanding their biology and behavior helps us appreciate their role in maintaining the health of ocean environments. As we learn more about these incredible creatures, we can better protect them and their habitats, ensuring that they continue to thrive in our oceans for generations to come.\"}",
                        "name": "save_article"
                    },
                    "id": "call_ZTty8BJYMIzN4JtDDKXAllcu",
                    "type": "function"
                }
            ]
        },
        {
            "role": "tool",
            "tool_call_id": "call_ZTty8BJYMIzN4JtDDKXAllcu",
            "content": "Article saved successfully."
        }
    ],
    "tools": 
    [
        {
            "type": "function",
            "function": 
            {
                "name": "search_web",
                "description": "Search the web for content about a particular subject.",
                "parameters": 
                {
                    "type": "object",
                    "properties": 
                    {
                        "subject": 
                        {
                            "type": "string",
                            "description": "The subject or topic to search the web about."
                        }
                    },
                    "required": ["subject"]
                }
            }
        },
        {
            "type": "function",
            "function": 
            {
                "name": "save_article",
                "description": "Save an article about a topic as a draft.",
                "parameters": 
                {
                    "type": "object",
                    "properties": 
                    {
                        "article_name": 
                        {
                            "type": "string",
                            "description": "The name of the article."
                        },
                        "article_content": 
                        {
                            "type": "string",
                            "description": "The content (body) of the article."
                        }
                    },
                    "required": ["article_name", "article_content"]
                }
            }
        }
    ]
}
```

You'll see above we appended two messages to the `messages` array: the tool call by the agent to save the article and the confirmation that the article save was successful.

We send this to the agent via a POST request!

## Step 7: The Agent Confirms It was Successful
Remember, in the original system prompt, the agent was instructed to do the following:

```
After this is all completed, tell me it is completed in a kind and sincere tone.
```

After sending the body in step 6 to the agent, the agent responds accordingly (this is just the message stripped out):

```
{
    "role": "assistant",
    "content": "The article about sharks has been successfully written and saved for your review. If you have any more topics or further requests, feel free to ask!"
}
```

As instructed in the system prompt, the agent then kindly and sincerely confirms to the user that, after searching the web, writing the article, and saving the article, the task it was assigned is now complete!