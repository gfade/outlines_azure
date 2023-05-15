<div align="center">

<img src="./docs/source/_static/logo.png" alt="Outlines Logo" width=300></img>

# Outlines

Build _reliable_ workflows based on interactions with generative models.

</div>


## Prompt management

Outlines makes it easier to write and manage prompts by encapsulating templates
inside "template functions". These functions make it possible to neatly separate
the prompt logic from the general program logic; they can be imported from other
modules and libraries.

Template functions use the Jinja2 templating engine to help build complex
prompts (like few-shot examples) in a concise manner:

``` python
import outlines.text as text


@text.prompt
def few_shot_examples(question, examples):
    """Something something

    {% for example in examples %}
    EXAMPLE: {{ example }}
    {% endfor %}

    QUESTION: {{ question }}
    Let's think step by step.

    """
```

Functions can also be _partially evaluated_ just like any function, which can be useful when building agents:

``` python
import functools as ft
import outlines.text as text


@text.prompt
def my_agent(name, goals):
    """Your name is {{ name }}.

    GOALS:
    {% for goal in goals %}
    {{ loop.counter }}. {{ goal }}
    {% endfor %}
    """


jarvis = ft.partial(my_agent, "JARVIS")
```

The template contained in template functions remains accessible:

``` python
import outlines.text as text


@text.prompt
def prompt():
    "I am accessible"


prompt.template
# I am accessible
```

### Tools

Prior work has shown that we can teach language models to call external functions to get additional informations or perform tasks, by encoding the functions' description in the prompt. To avoid duplicating information between the function definition and the description passed to the prompt we define custom Jinja filters that can extract the function's name, description, signature and source:


``` python
from typing import Callable, List
import outlines.text as text


def google_search(query: str):
    """Google Search"""
    pass


def wikipedia_search(query: str):
    """Wikipedia Search"""
    pass


@text.prompt
def my_commands(tools: List[Callable]):
    """AVAILABLE COMMANDS:

    {% for tool in tools %}
    TOOL
    {{ tool | name }}, {{ tool | description }}, args: {{ tool | signature }}
    {{ tool | source }}
    {% endfor %}
    """


prompt = my_commands([google_search, wikipedia_search])
```

### Response models

We can instruct models to return their output in a pre-defined format, often JSON. To avoid duplicating information between the function definition and the description passed to the prompt we define a custom Jinja filter that can extract the expected response's schema:

``` python
from pydantic import BaseModel
import outlines.text as text


class Joke(BaseModel):
    joke: str
    explanation: str


@text.prompt
def joke_ppt(response_model):
    """Tell a joke and explain why the joke is funny.

    RESPONSE FORMAT:
    {{ response_model | schema }}
    """


joke_ppt(Joke)
# Tell a joke and explain why the joke is funny.
#
# RESPONSE FORMAT:
# {
#    "joke": "The joke"
#    "explanation": "The explanation of why the joke is funny"
#  }
```


## Natural language functions

Large language models can be prompted so their output can be parsed into a data structure that can be manipulated by programming languages. The combination prompt + model call + output parser can thus be thought as a "natural language" function.

``` python
import json
import outlines.text as text
import outlines.models as models


@text.prompt
def prime_numbers(n: int):
    """Return a list that contains all prime numbers between 1 and {{ n }}.

    The output must be parsable as a Python list.
    """


def parse(result):
    return json.loads(result)


get_prime_numbers = text.function(
   models.text_completion.openai("gpt-3.5-turbo"),
   prime_numbers,
   parse
)


get_prime_numbers(10)
# [2, 3, 5, 7]
```

For more complex outputs one can pass a Pydantic model to `text.function`, which will be used to parse the output:

``` python
from pydantic import BaseModel
import outlines.text as text


class Joke(BaseModel):
    joke: str
    explanation: str


@text.prompt
def joke_ppt(response_model):
    """Tell a joke and explain why the joke is funny.

    RESPONSE FORMAT:
    {{ response_model | schema }}
    """

tell_a_joke = text.function(
   models.text_completion.openai("gpt-3.5-turbo"),
   joke_ppt,
   Joke
)

tell_a_joke(Joke)
# [2, 3, 5, 7]
```

# Controlled generation

Outlines offers mechanisms to specify high-level constraints on the text generations:

- `stop_at` allows to stop the generation once a particular word, sequence of symbol had been generated;
- `is_in` allows to constrain the model to generate an answer chosen among a set of possible answers;
- `type` allows to constrain the model's output to either `"int"`s or `"float"`s;

Coming:

- Ability to constrain the output to a JSON with a given structure;
- Ability to constrain the output to a List;
- Ability to constrain the output to be Python code;

# Examples

- [Pick the odd one out](https://github.com/normal-computing/outlines/blob/main/examples/pick_odd_one_out.py)
- [Meta prompting](https://github.com/normal-computing/outlines/blob/main/examples/meta_prompting.py)
- [ReAct](https://github.com/normal-computing/outlines/blob/main/examples/meta_prompting.py)
- [Generate code to solve math problems](https://github.com/normal-computing/outlines/blob/main/examples/dust/math-generate-code.py)
- [BabyAGI](https://github.com/normal-computing/outlines/blob/main/examples/babyagi.py)