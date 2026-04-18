---
date: '2026-04-18T20:00:00+01:00'
draft: false
title: "Mise en place d'un agent IA de FAQ avec FastAPI, LangChain et un RAG"
categories: ['AI', 'LangChain', 'FastAPI', 'RAG']
cover:
  image: '/images/ai-agent-cover.png'
  alt: 'Illustration d’un agent IA de FAQ'
  caption: 'Agent IA de FAQ combinant FastAPI, LangChain et un RAG pour des réponses précises basées sur la documentation interne.'
  focalPoint: 'center center'
---

Cet article détaille le processus de création d'un agent IA, développé avec **langchain** et combiné à un système de **Retrieval-Augmented Generation (RAG)**. L’ensemble est intégré dans une **API** Python pour faciliter l’interconnexion avec divers clients : **frontend** ou intégration dans des applications tierces via des **webhooks** (Slack ou Discord).

L’exemple présenté ici est simple, mais peut servir de socle à une API agentique étendue comportant des fonctionnalités avancées (streaming, outils multiples, etc.).

Par exemple, cet agent pourrait servir d'assistant interne pour une équipe DevOps, répondant aux questions sur la configuration d'outils **IaC** en s'appuyant sur la documentation interne de l'entreprise.

**⚠️ ATTENTION ⚠️** : Ce projet utilise Vertex AI (Google Cloud), un service payant. Les coûts dépendent du volume de requêtes et de la taille des documents indexés.

---
## Objectif

L'agent est conçu pour répondre aux questions des utilisateurs en basant ses réponses sur un **RAG** contenant la totalité de ma base de connaissance personnelle. L'objectif est de mettre à disposition un assistant pour les collaborateurs d'une entreprise qui aurait connaissance de la documentation interne pour fournir des réponses adaptées.

---
## Architecture

Le projet est composé de 3 éléments principaux :

-  **L'Agent (LangChain)** : Basé sur Gemini, il est configuré avec une personnalité via son **system prompt** et respecte certaines règles établie dans celui-ci. Il est par exemple obligé d'utiliser le RAG auquel il est connecté pour fournir ses réponses via un dispositif appelé un **tool** (outil), abstraction du RAG. 

-  **Le RAG (Retrieval-Augmented Generation)** : La source de vérité de l'agent qui lui est présentée en tant que **tool**. Cet outil se connecte à un corpus de documents préalablement indexé (par exemple, sur **Google Cloud Vertex AI**). Lorsqu'il est interrogé, il recherche et retourne les extraits de texte les plus pertinents par rapport à la question posée.

- **L'API (FastAPI)** : elle présente un **endpoint** permettant d'interroger l'agent en lui transmettant la question de l'utilisateur. La réponse du modèle est ensuite retournée à l'utilisateur (pas de *streaming* pour cette version).

Dès lors, le fonctionnement est le suivant :

![AI Agent Schema](/images/ai-agent-schema.png)

1.  Un utilisateur envoie sa question en tant que requête `POST` à un endpoint `/api/chat` exposé par l'API (depuis un client quelconque).
2.  La question est transmise à l'instance de l'agent **LangChain**.
3.  L'agent analyse la question et invoque son outil de recherche **RAG** pour fournir une réponse provenant de la documentation.
4.  L'outil RAG interroge la base documentaire vectorielle et renvoie les passages pertinents.
5.  Le LLM génère une réponse en se basant sur le contexte fourni par le **RAG**.
6.  La réponse finale est retournée à l'API, qui l'encapsule dans un objet **JSON** et la renvoie au client.

---
## Création de l'agent

### Configuration du modèle

Initialiser un agent IA avec **LangChain** en utilisant les modèles **Gemini** de *Google* se révèle assez simple et rapide. 

Dans un premier temps, il faut installer les dépendances python nécessaires :

```bash
pip install -U langchain
pip install -U langchain-google-genai
```

**Remarque** : il est recommandé d'utiliser un environnement virtuel python dédié pour la gestion des dépendances.

Il est ensuite nécessaire de s'authentifier auprès des services IA de *Google* pour les utiliser dans **LangChain**. Pour cela, plusieurs options existent comme l'utilisation d'une clé API générée dans **Gemini** ou l'utilisation de **Vertex AI**, la plateforme dédiée à l'IA de **Google Cloud Platform** (**GCP**). 

Pour ce projet, j'ai choisi la seconde option car je souhaite également utiliser d'autres services de la plateforme. L'authentification est donc effectuée de manière classique via la CLI **gcloud** :

```bash
gcloud auth login
gcloud auth application-default login
```

Il faut ensuite créer un nouveau projet puis activer l'API Vertex AI :

```absh
gcloud projects create ai-project
gcloud services enable vertex-ai
```

Dès lors, il est possible d'initialiser le modèle basé sur **Vertex** :

```python
from langchain_google_genai import ChatGoogleGenerativeAI

llm = ChatGoogleGenerativeAI(
    model="gemini-3.1-pro-preview",
    temperature=0,
    max_tokens=4096,
    timeout=None,
    max_retries=2,
)
```

L'initialisation du modèle est effectuée simplement grâce à l'appel de la classe **ChatGoogleGenerativeAI** qui permet de définir chaque aspect du modèle utilisé :

- **model** : modèle présent sur la plateforme Vertex
- **temperature** : la température est un indice, allant de 0 à 1, et correspondant au degré de créativité d'un modèle. Plus elle est importante, plus le modèle aura tendance à inventer des éléments de documentation dans le cas de données techniques basées sur un RAG. C'est pour cela qu'elle est fixée à 0 dans notre cas, afin d'écarter le plus possible le risque d'hallucination.
- **max_tokens** : permet de limiter le nombre de tokens retournés par le modèle
- **timeout** : délai au delà duquel le modèle cessera de tenter de générer une réponse
- **max_retries** : nombre de tentatives pour fournir une réponse

### Le system prompt

Ce modèle va pouvoir être combiné à un **system prompt** pour adapter le comportement de l'agent à nos intentions. Le but est de le configurer pour qu'il agisse en tant qu'expert de son domaine. 

Un prompt a donc été créé pour rédiger et ancrer la mission de l'agent, sa manière de répondre et le contraindre à utiliser les données provenant du **RAG** :

```python
SYSTEM_PROMPT = """
You are an AI assistant specialized in answering technical questions based **only** on the documentation indexed in the RAG engine.
Your sole purpose is to provide accurate, concise, and verifiable answers from the RAG database.

### Rules:
1. **Strict RAG Dependency**:
   - Every answer **must** be directly sourced from the RAG engine.
   - If the RAG engine does not contain the information, respond: *"I don’t have the information to answer this question in the current documentation."*

2. **No Hallucination**:
   - Never invent, extrapolate, or assume information.
   - If the RAG engine returns no relevant results, do not attempt to answer.

3. **Response Format**:
   - Use Markdown for clarity (code blocks, lists, bold for emphasis).
   - Always cite the source file(s) from the RAG engine (e.g., *"Source: `ansible/roles/nginx/README.md`"*).

4. **User Clarification**:
   - If the question is ambiguous, ask for clarification before answering.
   - Example: *"Could you clarify which part of the documentation you’re referring to? (e.g., Ansible roles, Terraform modules)"*

5. **Language**:
   - Respond in the same language as the user’s question (default: English).

6. **Error Handling**:
   - If the RAG engine fails or times out, respond: *"The documentation service is unavailable. Please try again later."*

### Example Interactions:
- **User**: *"How do I configure the Nginx role for load balancing?"*
  **You**: *"The Nginx role supports load balancing via the `load_balancer: true` parameter in `group_vars`. Example:
  ```yaml
  load_balancer: true
  upstream_servers:
    - server1.example.com
    - server2.example.com
```

Ce long jeu d'instruction permet de limiter les hallucinations, d'influencer les messages fournies par l'agent et d'assurer que le contenu soit conforme au contenu de la documentation interne contenue dans le RAG.

**Remarque** : la rédaction du prompt a été déléguée à un LLM. Le développeur peut spécifier directement les intentions de comportement de l'agent à un autre **LLM** qui se chargera de les mettre en forme pour maximiser la prise en compte du contexte par l'IA.

---
## Le RAG

Le **Retrieval-Augmented Generation (RAG)** permet à un modèle d’IA de répondre en s’appuyant sur une base de connaissances externe, plutôt que sur ses seules connaissances générales. Pour ce projet, l’objectif est d’interroger une documentation technique stockée dans un système RAG, afin d’obtenir des réponses précises et sourcées.

Implémenter un RAG manuellement implique plusieurs étapes techniques : découper les documents en morceaux (*chunks*), générer des représentations vectorielles (*embeddings*), et gérer une base de données vectorielle. Ces étapes, bien que nécessaires, ajoutent une complexité qui n’est pas toujours justifiée pour un premier prototype.

Pour cette raison, j’ai choisi **Vertex AI Rag Engine**, une solution managée proposée par Google. Ce service prend en charge automatiquement le découpage des documents, la génération des *embeddings*, et le stockage dans une base vectorielle. 

Il suffit alors d’uploader les fichiers dans un dossier **Google Cloud Storage** ou **Google Drive**. **Vertex AI** prend alors en charge automatiquement l'ensemble des opérations précitées.

La création du **RAG** est entièrement réalisable avec **python**, ce qui permet d'automatiser la collecte et l'import des documents. 

**Exemple de script pour l'import dans le RAG** :

```python
from vertexai import rag
from vertexai.generative_models import GenerativeModel, Tool
import vertexai

vertexai.init(project=<project_id>, location="<region>")

embedding_model_config = rag.RagEmbeddingModelConfig(
    vertex_prediction_endpoint=rag.VertexPredictionEndpoint(
        publisher_model="publishers/google/models/text-embedding-005"
    )
)

rag_corpus = rag.create_corpus(
    display_name=display_name,
    backend_config=rag.RagVectorDbConfig(
        rag_embedding_model_config=embedding_model_config
    ),
)

rag.import_files(
	"projects/YOUR_GCP_PROJECT/locations/YOUR_REGION/ragCorpora/YOUR_CORPUS_ID",
    paths = ["gs://bucket/folder/"],
    # Optional
    transformation_config=rag.TransformationConfig(
        chunking_config=rag.ChunkingConfig(
            chunk_size=512,
            chunk_overlap=100,
        ),
    ),
    max_embedding_requests_per_min=1000,  # Optional
)
```

**Remarque** : Il est intéressant d'utiliser un **folder** dans le cas de **Cloud Storage** car la limite d'ingestion via **Cloud Storage** est à 25 fichiers, alors qu'elle est à plusieurs milliers via l'usage d'un folder. 
### Combiner le RAG et l'agent LangChain

Une fois le RAG configuré, il peut être lié à l'agent en tant qu'outil utilisable par celui-ci. Cet outil est une fonction **Python**, surmontée d'un décorateur `@tool` propre à **LangChain** qui permet de spécifier un outil pouvant être utilisé par les agents. 

L'idée est alors de créer un outil permettant d'interroger le RAG et de retourner les informations pertinentes :

```python
from langchain.tools import tool
from vertexai import rag

@tool
def knowledge_base_retrieval_tool(query: str) -> str:
    response = rag.retrieval_query(
        rag_resources=[rag.RagResource(rag_corpus="YOUR_CORPUS_ID")],
        text=query,
        rag_retrieval_config=rag.RagRetrievalConfig(top_k=3)
    )
    return " ".join([context.text for context in response.contexts.contexts])
```

Cet outil permet à l'IA de fournir des réponses basées sur l'ensemble des documents indexés dans le RAG. 
### Création de l'agent

L'ensemble des éléments configurés jusqu'ici sont alors combinés pour initialiser l'agent dans une fonction dédiée : le LLM, l'outil RAG et le prompt système.

**Code d'initialisation de l'agent** :

```python
from langchain.agents import create_agent
from langchain_google_genai import ChatGoogleGenerativeAI
import vertexai

def agent_setup():   

	llm = ChatGoogleGenerativeAI(
	    model="gemini-3.1-pro-preview",
	    temperature=0,
	    max_tokens=4096,
	    timeout=None,
	    max_retries=2,
	)
	
    agent = create_agent(
        model=llm,
        tools=[knowledge_base_retrieval_tool],
        system_prompt=SYSTEM_PROMPT,
    )
    
    return agent
```

L'agent initialisé, on peut simplement l'invoquer en appelant la fonction `agent_setup` :

```python
agent = agent_setup()

response = agent.invoke(
	{"messages": [{"role": "user", "content": "<user_message>"}]}
)

print(response)
```

---
## Exposition de l'agent via une API

L'API est le point d'entrée public. Son code est volontairement simple et se concentre sur l'exposition de l'agent.

```python
# Fichier : main.py
from fastapi import FastAPI
from pydantic import BaseModel
from agent_logic import agent_setup
from fastapi.middleware.cors import CORSMiddleware
import logging

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

# Initialize the FastAPI app and the agent
app = FastAPI(
    title="Knowledge Base API",
    description="API to interact with an AI agent based on RAG and LangChain.",
    version="0.1.0",
)

agent = agent_setup()

# CORS configuration (adjust according to your environment)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://localhost:5173"],  # Replace with your allowed origins
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

class Query(BaseModel):
    """Request model to validate the user's question."""
    question: str

@app.get("/", summary="Check API status")
async def root():
    """Health check endpoint to verify the API is running."""
    return {"message": "Knowledge Base API is running", "status": "ok"}

@app.post("/api/chat", summary="Ask a question to the agent")
async def ask_question(query: Query):
    """
    Endpoint to query the AI agent.

    Args:
        query (Query): Object containing the user's question.

    Returns:
        dict: Agent's response in the format {"response": str}.
    """
    logger.info(f"Received question: {query.question}")

    try:
        response = agent.invoke(
            {"messages": [{"role": "user", "content": query.question}]}
        )

        logger.info(f"Response generated: {response['output'][:100]}...")  # Log first 100 chars
        return {"response": response['output']}

    except Exception as e:
        logger.error(f"Error processing question: {str(e)}", exc_info=True)
        return {"error": "An error occurred while processing your request."}, 500
```

**Fonctionnalités de l'API :**

- **Interrogation de l'agent IA** : Endpoint POST `/api/chat` pour poser des questions
- **Validation des entrées** : Vérification du format de la question (Pydantic)
- **Réponses basées sur RAG** : Utilisation de Vertex AI pour des réponses sourcées
- **Gestion des erreurs** : Retourne des messages clairs en cas d'échec
- **Logging complet** : Trace des questions/réponses et erreurs
- **CORS configuré** : Autorise les requêtes depuis des frontends spécifiques
- **Endpoint de santé** : GET / pour vérifier le statut de l'API

**Quelques remarques** :

- La structure de `response['output']` peut varier selon la version de **Langchain** ou le modèle utilisé. 

- Cependant, dans le cas où l'on souhaite utiliser des interfaces utilisateurs comme [assistant-ui](https://www.assistant-ui.com/), il faut introduire des notions de [streaming](https://docs.langchain.com/oss/python/langchain/streaming) pour la réponse de l'agent dans l'interfaces utilisateur. Cela nécessite alors d'adapter l'API en utilisant [StreamingResponse](https://fastapi.tiangolo.com/advanced/custom-response/#streamingresponse) et le mot clé `yield` qui permet (en `async`) de retourner les réponses au fil de la génération par le modèle.

---
## Déploiement et Utilisation

### Installation des dépendances

Un fichier `requirements.txt` est nécessaire pour installer les dépendances :

```txt
fastapi
uvicorn[standard]
pydantic
langchain
langchain-google-genai
google-cloud-aiplatform
```

### Lancement du serveur

Le serveur API peut être lancé localement avec Uvicorn :

```bash
uvicorn main:app --reload --host 0.0.0.0 --port 8000
```

### Exemple de requête

Une fois le serveur démarré, il peut être interrogé à l'aide d'un client HTTP comme `curl` :

```bash
curl -X POST "http://127.0.0.1:8000/api/chat" \
-H "Content-Type: application/json" \
-d '{
    "question": "<Question>"
}'
```

La réponse sera un objet JSON contenant la réponse formatée générée par l'agent.
### Création d'une image de conteneur

Afin de rendre l'agent portable et déployable facilement dans tous types d'environnements, il est préférable de créer une image **docker** qui permettra d'unifier l'ensemble des éléments de ce projet. 

```dockerfile
FROM python:3.13.13-slim


WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

EXPOSE 8000

# Commande de démarrage
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

Une bonne pratique est aussi l'ajout d'un `.dockerignore` pour exclure les fichiers inutiles ou indésirables :

```dockerfile
__pycache__
*.pyc
.env
.git
```

Construction et teste de l'image :

```bash
docker build -t rag-fastapi-agent .
docker run -p 8000:8000 rag-fastapi-agent
```

Cette image peut alors être poussées vers le registre d'un fournisseur cloud tel que **GCP** afin d'exécuter l'image dans une instance **Cloud Run** par exemple.
### Conclusion

Cette architecture permet de débuter la construction d'un agent assistant proposant des réponses fiable dans un contexte professionnel puisque basées sur la documentation interne.

En confinant l'agent à une source de vérité unique via un outil RAG et en lui imposant des règles strictes via un *system prompt* , on réduit fortement le risque d'hallucination et d'impertinence.

L'utilisation de **FastAPI** offre une interface standard et performante qui se révèle très facile à intégrer avec des applications existantes ou des types de clients variés (applications web, outils en CLI ou des **webhooks**). 

Pour aller plus loin, on pourrait évidemment augmenter la fiabilité et sécurité de l'API en ajoutant des fonctions d'authentification ou la surveillance des usages de l'agent pour éviter une surconsommation de tokens.

Enfin, on pourrait améliorer cet agent pour qu'en plus du **RAG**, il intègre un d'autres outils. Par exemple un `tool` **GitLab** lui permettant de parcourir les projets de l'entreprise afin de fournir des réponses directement tirées de la *codebase* interne.

## Références

- [From Prompt to Production: Dockerizing a LangChain Agent with FastAPI - DEV Community](https://dev.to/moni121189/from-prompt-to-production-dockerizing-a-langchain-agent-with-fastapi-1pe3)
- [Présentation du moteur RAG Vertex AI  |  Generative AI on Vertex AI  |  Google Cloud Documentation](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/rag-engine/rag-overview?hl=fr)
- [Guide de démarrage rapide RAG  |  Generative AI on Vertex AI  |  Google Cloud Documentation](https://docs.cloud.google.com/vertex-ai/generative-ai/docs/rag-engine/rag-quickstart?hl=fr)
- [RAG API. 30 lines of code is all you need for… | by Sascha Heyer | Google Cloud - Community | Medium](https://medium.com/google-cloud/google-cloud-rag-api-c7e3c9931b3e)
- [LangChain overview - Docs by LangChain](https://docs.langchain.com/oss/python/langchain/overview)
- [Fast API Tutorial for AI Engineers | by Tom Odhiambo | Medium](https://medium.com/@odhitom09/fast-api-tutorial-for-ai-engineers-576bd14e4ddf)
- [Create a RAG Chatbot with FastAPI & LangChain](https://blog.futuresmart.ai/building-a-production-ready-rag-chatbot-with-fastapi-and-langchain)
