API keys are like disasters waiting to happen. Someone might share it on email, chat or write it down, meaning the key may get compromised. Then, you end up with a big bill because of a compromised key.

[Using Keyless Auth with Azure AI Services by Microsoft](https://www.youtube.com/watch?v=IkDcQvKoQ8k) by Pamela Fox and Marlene Mhangami.

Just in case you're getting a "badly formatted request error" from the OpenAI service, see this article by [Luke Murray](https://luke.geek.nz/azure/openai-request-badly-formatted/) on custom sub-domains.

#### When running agents locally on a workstation 

```powershell
az login --scope https://management.core.windows.net//.default

az login --scope https://cognitiveservices.azure.com/.default

```

#### Using LangChain and LangGraph

```python
from azure.identity import AzureCliCredential, get_bearer_token_provider

from langchain_openai import AzureChatOpenAI, AzureOpenAIEmbeddings

import os


credential = AzureCliCredential(subscription=os.environ['AZURE_SUBSCRIPTION'])
token_provider = get_bearer_token_provider(credential, os.environ['AZURE_BEARER_TOKEN_PROVIDER_ENDPOINT'])


def get_llm() -> AzureChatOpenAI:
    llm = AzureChatOpenAI(
        azure_ad_token_provider=token_provider,
        azure_endpoint=os.environ['AZURE_OPENAI_ENDPOINT'],
        azure_deployment=os.environ['OPENAI_DEPLOYMENT'],
        openai_api_version=os.environ['OPENAI_API_VERSION'],
    )

    return llm


def get_embedding_model() -> AzureOpenAIEmbeddings:
    embedding_generator = AzureOpenAIEmbeddings(
        azure_ad_token_provider=token_provider,
        azure_endpoint=os.environ['AZURE_OPENAI_ENDPOINT'],
        azure_deployment=os.environ['EMBEDDING_MODEL'],
        openai_api_version=os.environ['EMBEDDING_API_VERSION']
    )

    return embedding_generator


```

#### Using OpenAI Agents SDK

```python
from azure.identity import AzureCliCredential, get_bearer_token_provider
from openai import AsyncAzureOpenAI

import os


api_version=os.environ['OPENAI_API_VERSION']
deployment=os.environ['OPENAI_DEPLOYMENT']
endpoint=os.environ['AZURE_OPENAI_ENDPOINT']


credential = AzureCliCredential(subscription=os.environ['AZURE_SUBSCRIPTION'])
token_provider = get_bearer_token_provider(credential, os.environ['AZURE_BEARER_TOKEN_PROVIDER_ENDPOINT'])


openai_client = AsyncAzureOpenAI(
    api_version=api_version,
    azure_endpoint=endpoint,
    azure_ad_token_provider=token_provider,
    azure_deployment=deployment,    
)


set_default_openai_client(openai_client, use_for_tracing=False)

```

#### Running on a cloud environment need an Azure service principal.
