## Download Python

## Install Poetry (Package Manager) in Powershell
```
(Invoke-WebRequest -Uri https://raw.githubusercontent.com/python-poetry/poetry/master/get-poetry.py -UseBasicParsing).Content | python -
```

Check poetry using below
```
poetry --version
```

Create poetry instance and this creates .toml file
```
poetry init
```

Give details of your service and give no for dependencies auto generation. Finally click yes

Create virtual env using below
```
poetry shell
```

Create the src folder

```
mkdir src
```

```
cd src
```

Create "__init__.py" and "main.py" inside src/  folder

## Adding Spacy 

```
poetry add spacy
```

Add spacy ner model in "pyproject.toml" file under [tool.poetry.dependencies]

```
en_core_web_sm = { url = "https://github.com/explosion/spacy-models/releases/download/en_core_web_sm-3.0.0/en_core_web_sm-3.0.0.tar.gz" }

```

```
poetry update
```

Update "main.py" to check if the model works fine with below code

```
import spacy

nlp = spacy.load("en_core_web_sm")
doc = nlp("Apple is looking at buying U.K. startup for $1 billion")

for ent in doc.ents:
    print(ent.text, ent.start_char, ent.end_char, ent.label_)
```

## Adding FastAPI 

Let's move to [FastAPI](https://fastapi.tiangolo.com/) to create APIs for the ner service

```
poetry add fastapi
```

```
poetry add uvicorn
```

Update "main.py" to add fastapi

```
import spacy
from fastapi import FastAPI

app = FastAPI()

nlp = spacy.load("en_core_web_sm")

doc = nlp("Apple is looking at buying U.K. startup for $1 billion")

for ent in doc.ents:
    print(ent.text, ent.start_char, ent.end_char, ent.label_)

```

## Adding FastAPI Post Request 

Create a ner-service post request

```
import spacy
from fastapi import FastAPI

app = FastAPI()
nlp = spacy.load("en_core_web_sm")


@app.post('/ner-service')
async def get_ner(payload: Payload):
    
    doc = nlp("Apple is looking at buying U.K. startup for $1 billion")

    for ent in doc.ents:
        print(ent.text, ent.start_char, ent.end_char, ent.label_)

    return ''
```

## Adding Pydantic Models

### Creating models for handling the Payload


Now, add the [Pydantic](https://pydantic-docs.helpmanual.io/) models (Content, Payload) in 'main.py'

```
from typing import List
from pydantic import BaseModel

class Content(BaseModel):
    post_url: str
    content: str

class Payload(BaseModel):
    data: List[Content]
```

Update the main.py with below to merge the spacy with pydantic models

```
@app.post('/ner-service')
async def get_ner(payload: Payload):
    
    tokenize_content: List[spacy.tokens.doc.Doc] = [
        nlp(content.content) for content in payload.data
    ]

    for doc in tokenize_content:
        response = [ {'text': ent.text, 'entity_type': ent.label_} for ent in doc.ents ]
        print(response)

    return ''
```

### Running FastAPI Locally

Run uvicorn to check the response in FastAPI

```
uvicorn main:app --reload
```

Open http://127.0.0.1:8000/docs to access the FastAPI Swagger UI

Click POST /ner-service to open up the Request Editor

Click Try it out and check the Request like below

```
{
  "data": [
    {
      "post_url": "string",
      "content": "My name is Mark and I love Facebook"
    }
  ]
}
```

Click Execute and you should see the predicted Entities in your terminal

Update the service again like below to return the response rather than printing

```
@app.post('/ner-service')
async def get_ner(payload: Payload):
    
    tokenize_content: List[spacy.tokens.doc.Doc] = [
        nlp(content.content) for content in payload.data
    ]
    document_enities = []
    for doc in tokenize_content:
        response = [ {'text': ent.text, 'entity_type': ent.label_} for ent in doc.ents ]
        document_enities.append(response)

    return document_enities
```

Try executing the API again and you should get the Response in the Swagger UI  http://127.0.0.1:8000/docs


### Creating models for handling Multiple Entities

Add below models in main.py
```
class SingleEntity(BaseModel):
    text: str
    entity_type: str

class Entities(BaseModel):
    post_url: str
    entities: List[SingleEntity]

```

Now modify the return statement with Entities model

```
@app.post('/ner-service')
async def get_ner(payload: Payload):
    
    tokenize_content: List[spacy.tokens.doc.Doc] = [
        nlp(content.content) for content in payload.data
    ]
    document_enities = []
    for doc in tokenize_content:
        response = [ {'text': ent.text, 'entity_type': ent.label_} for ent in doc.ents ]
        document_enities.append(response)

    return [
        Entities(post_url=post.post_url, entities=ents)
        for post, ents in zip(payload.data, document_enities)
    ]

```

Try executing the API again http://127.0.0.1:8000/docs and you should get the Response is denoted with post_url and entities.

