## Example Calls with Tools Provided
Example to ollama to get the weather (standard example provided by OpenAI)
```
POST http://localhost:11434/api/chat

{
  "model": "qwen2.5:0.5b",
  "messages": [
    {
      "role": "user",
      "content": "What is the weather today where I live?"
    },
    {
      "role": "assistant",
      "content": "To provide you with the current weather in your location, I need to know your geographical coordinates. Could you please tell me your city or state, and a specific date for which you want to see the weather? If that's not possible, could you please specify an approximate date?"
    },
    {
      "role": "user",
      "content": "Sarasota, FL"
    }
  ],
  "stream": false,
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_current_weather",
        "description": "Get the current weather for a location",
        "parameters": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "The location to get the weather for, e.g. San Francisco, CA"
            },
            "format": {
              "type": "string",
              "description": "The format to return the weather in, e.g. 'celsius' or 'fahrenheit'",
              "enum": ["celsius", "fahrenheit"]
            }
          },
          "required": ["location", "format"]
        }
      }
    }
  ]
}
```

Example adding to to-do list:
```
POST http://localhost:11434/api/chat
{
    "model": "qwen2.5:0.5b",
    "stream": false,
    "messages":
    [
        {
            "role": "user",
            "content": "Add 'take out the trash' to my to-do list"
        }
    ],
    "tools":
    [
        {
            "type": "function",
            "function":
            {
                "name": "add_todo",
                "description": "Add an item to the user's to-do list.",
                "parameters":
                {
                    "type": "object",
                    "properties":
                    {
                        "name":
                        {
                            "type": "string",
                            "description": "The name of the to-do list item (for example 'pick up milk at the store')."
                        }
                    },
                    "required": ["name"]
                }
            }
        }
    ]
}
```

## Tool Calling where Data is Returned: Step-by-Step
In some tool calls, the model expects information back... and this information it will use in conversation to satisfy the user's query.

### Step 1: User asks question that a tool is needed for.
The request is:
```
POST http://localhost:11434/api/chat
{
    "model": "qwen2.5:0.5b",
    "stream": false,
    "messages":
    [
        {
            "role": "user",
            "content": "What is the temperature in Seattle?"
        }
    ],
    "tools":
    [
        {
            "type": "function",
            "function":
            {
                "name": "get_temperature",
                "description": "Get the temperature, in fahrenheit, for any city.",
                "parameters":
                {
                    "type": "object",
                    "properties":
                    {
                        "city":
                        {
                            "type": "string",
                            "description": "The name of the city that the user wants the temperature for (i.e. 'Atlanta' or 'Seattle')."
                        }
                    },
                    "required": ["city"]
                }
            }
        }
    ]
}
```

The model responds back like this (this is in Ollama):

```
{
  "model": "qwen2.5:0.5b",
  "created_at": "2025-02-17T20:13:04.8683278Z",
  "message": {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "function": {
          "name": "get_temperature",
          "arguments": {
            "city": "Seattle"
          }
        }
      }
    ]
  },
  "done_reason": "stop",
  "done": true,
  "total_duration": 478345200,
  "load_duration": 22058100,
  "prompt_eval_count": 188,
  "prompt_eval_duration": 15000000,
  "eval_count": 20,
  "eval_duration": 438000000
}
```

### Step 2: Make the function call!
The model used the `get_temperature` tool, so make the necessary API call to get that data.

### Step 3: Provide the data to the model
Now, for the model to respond naturally, we provide that information back to the model. But when we provide it, we also provide the original tool call message as well.

```
POST http://localhost:11434/api/chat
{
    "model": "qwen2.5:0.5b",
    "stream": false,
    "messages":
    [
        {
            "role": "user",
            "content": "What is the temperature in Seattle?"
        },
        {
            "model": "qwen2.5:0.5b",
            "created_at": "2025-02-17T20:13:04.8683278Z",
            "message": {
                "role": "assistant",
                "content": "",
                "tool_calls": [
                {
                    "function": {
                    "name": "get_temperature",
                    "arguments": {
                        "city": "Seattle"
                    }
                    }
                }
                ]
            },
            "done_reason": "stop",
            "done": true,
            "total_duration": 478345200,
            "load_duration": 22058100,
            "prompt_eval_count": 188,
            "prompt_eval_duration": 15000000,
            "eval_count": 20,
            "eval_duration": 438000000
        },
        {
            "role": "tool",
            "content": "temperature in Seattle: 63.4 degrees fahrenheit"    
        }
    ],
    "tools":
    [
        {
            "type": "function",
            "function":
            {
                "name": "get_temperature",
                "description": "Get the temperature, in fahrenheit, for any city.",
                "parameters":
                {
                    "type": "object",
                    "properties":
                    {
                        "city":
                        {
                            "type": "string",
                            "description": "The name of the city that the user wants the temperature for (i.e. 'Atlanta' or 'Seattle')."
                        }
                    },
                    "required": ["city"]
                }
            }
        }
    ]
}
```

The model will respond like this:
```
{
  "model": "qwen2.5:0.5b",
  "created_at": "2025-02-17T20:17:36.3397071Z",
  "message": {
    "role": "assistant",
    "content": "The current temperature in Seattle is 63.4 degrees Fahrenheit."
  },
  "done_reason": "stop",
  "done": true,
  "total_duration": 367601700,
  "load_duration": 26841700,
  "prompt_eval_count": 214,
  "prompt_eval_duration": 22000000,
  "eval_count": 15,
  "eval_duration": 307000000
}
```

## Tool Calling where no data is returned: Step by Step
Some tools do not explicitly return data. For example, turning a light on or off. Other than confirmation that it worked, it is a one-way street!

### Step 1: Call to the model

```
POST http://localhost:11434/api/chat
{
    "model": "qwen2.5:0.5b",
    "stream": false,
    "messages":
    [
        {
            "role": "user",
            "content": "Add 'take out the trash' to my to-do list"
        }
    ],
    "tools":
    [
        {
            "type": "function",
            "function":
            {
                "name": "add_todo",
                "description": "Add an item to the user's to-do list.",
                "parameters":
                {
                    "type": "object",
                    "properties":
                    {
                        "name":
                        {
                            "type": "string",
                            "description": "The name of the to-do list item (for example 'pick up milk at the store')."
                        }
                    },
                    "required": ["name"]
                }
            }
        }
    ]
}
```

The model responds like this:
```
{
  "model": "qwen2.5:0.5b",
  "created_at": "2025-02-17T20:20:38.5389761Z",
  "message": {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "function": {
          "name": "add_todo",
          "arguments": {
            "name": "take out the trash"
          }
        }
      }
    ]
  },
  "done_reason": "stop",
  "done": true,
  "total_duration": 500538600,
  "load_duration": 24713300,
  "prompt_eval_count": 188,
  "prompt_eval_duration": 10000000,
  "eval_count": 23,
  "eval_duration": 457000000
}
```

### Step 2: Do what is necessary
The model indicated it wants to trigger the tool `add_todo`, so do what needs to happen.

### Step 3: Indicate success to model
We must now tell the model it worked. And after the model knows it work, it will confirm that with the user.

Append the tool call message itself (the `message` property from the initial response above) as well as a confirmation message to the messages, like this:
```
POST http://localhost:11434/api/chat
{
    "model": "qwen2.5:0.5b",
    "stream": false,
    "messages":
    [
        {
            "role": "user",
            "content": "Add 'take out the trash' to my to-do list"
        },
        {
            "role": "assistant",
            "content": "",
            "tool_calls": 
            [
                {
                    "function": 
                    {
                        "name": "add_todo",
                        "arguments": 
                        {
                            "name": "take out the trash"
                        }
                    }
                }
            ]
        },
        {
            "type": "tool",
            "content": "Item successfully added to to-do list."
        }
    ],
    "tools":
    [
        {
            "type": "function",
            "function":
            {
                "name": "add_todo",
                "description": "Add an item to the user's to-do list.",
                "parameters":
                {
                    "type": "object",
                    "properties":
                    {
                        "name":
                        {
                            "type": "string",
                            "description": "The name of the to-do list item (for example 'pick up milk at the store')."
                        }
                    },
                    "required": ["name"]
                }
            }
        }
    ]
}
```

And the model will respond with:
```
{
  "model": "qwen2.5:0.5b",
  "created_at": "2025-02-17T20:23:24.4401677Z",
  "message": {
    "role": "assistant",
    "content": "Your item has been added to your to-do list. You can see it now!"
  },
  "done_reason": "stop",
  "done": true,
  "total_duration": 455363100,
  "load_duration": 16638900,
  "prompt_eval_count": 214,
  "prompt_eval_duration": 13000000,
  "eval_count": 18,
  "eval_duration": 412000000
}
```