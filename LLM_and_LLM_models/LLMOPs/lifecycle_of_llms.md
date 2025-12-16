# An overview of the various stages in the lifecycle of LLM applications.

## 1. Ideation Phase or Planing Phase.

- **Data Sourcing**:
    is the process of identifying data needs, finding data sources and making them available.

 - **Base model selection**:
    is about selecting the right base LLM for our application's use-case.


## 2. Development Phase.
- **Prompt Engineering**:
    focus on crafting effective prompts to ensure the model produces accurate outputs.
- **Chains and Agents**:
    We should think about different architectures for shaping our application, such as chains and agents.
- **RAG vs Fine Tuning**:
    Various techniques exist that boost the model's performance, such as RAG and fine-tuning.
- **Testing**: 
    which is about determining when the application is ready for production.

## 3. Operational Phase.
- **Deployment**: 
    deploying the application to production.
- **Monitoring and observability**:
    Monitoring and observability help us keep track of how well everything is working.
- **Cost Management**: 
    which is about handling the expenses associated with using these models.
- **Governance and Security**:
    which includes access controls and measures to mitigate threats.

![LLM LifeCycle](/LLM_and_LLM_models/LLMOPs/images/llm_lifecycle.png)


## 1. Ideation Phase or Planning Phase

### 1. Data sourcing:
Providing the latest data is essential when developing our LLM application. This data serves as the fuel that powers the reasoning capabilities of the model. Data sourcing involves identifying needs, finding sources, and ensuring accessibility of the data we want to use. Three questions will guide us.

1. **Our first question is: Is the data relevant?** This involves identifying the right information from internal or external sources. 

2. **Our second question is: Is the data available?** Sometimes, we need to transform the data to make it ready. Additional databases might be needed for text data searchability. Evaluating costs, particularly for external data, is crucial as accessing it may incur charges. Finally, consider access limitations related to volume or frequency. 

3. **The last question is: Does the data meet standards?** These standards may concern quality and governance. If the data contains confidential or sensitive information, it could impact the choice of base model, since we might need to guarantee that it remains within the organization.

### Selecting the Best Model
After identifying data sources, the next step is selecting the base model. Most organizations choose pre-trained models, which already have been trained on significant amount of text data. The first step is determining whether to use a proprietary or an open-source model.

**Proprietary models (privately owned)**

Proprietary models, such ChatGPT, Cluade, GEMINI, are privately owned, while open-source ones are publicly accessible. A crucial consideration is whether exposing data to a third-party is acceptable, since proprietary models cannot be hosted within the confines of the organization. 

Advantages 
- ease of set-up and use, 
- quality assurance
- guarantees on reliability, speed and availability

Limitations:
- requires us to expose our data 
- offer limited customization.

**Open Source (Publickly accessible)**

Meta Llama 3, Downloadable models from Hugging faces.

Advantages
- In house hosting
- Transparency
- Full customizability

Limitations
- Limited Support
- Do not allow commercial use. 
- Customizing the models require dedicated AI engineers.

### Factors in model selection
After deciding between proprietary and open-source models, we need to narrow down our choice to reach our final selection. This means evaluating factors in four categories: performance, model characteristics, practical considerations, and less important secondary factors.

**Performance**:
- Response quality: we should consider the response quality, often better with the latest released models.
- Speed: Crucial for real-time applications.

**Model Characterstics**
- Data used to train the model: consider the data that was used to train the model, ranging from webpages to codebases, affecting the responses we might expect
- Context window size: refers to the number of words a model uses to predict a next word, influencing quality.
- Fine tunability: allows developers to optionally adjust the model for better performance.

**Practical Consideration**
- Licence: consider the type of license associated with the model, especially relevant for open-source models with commercial restrictions.
- Cost
- Environmental Impact.

**Secondary Factors**
- Number of parameters
- Popularity.

## 2. Development Phase
The development phase is a cyclic process of building and improving the application, known as the development cycle.At the heart of every application is the prompt, which instructs the LLM to generate desired outputs. We'll revisit this cycle as we progress, filling in new parts as we learn them.

![Development Cycle](/LLM_and_LLM_models/LLMOPs/images/development_cycle.png)

### Prompt Engineering

Prompt engineering enhances prompts in three ways
- By giving clear instructions, we can improve performance by getting more accurate and helpful responses from LLM's
- By well-crafted prompts, we can gain more control over the Output.
- Good prompt engineering can avoid bias and hallucinations.

**How do we find the perfect prompts:?**

A typical prompts include 4 elements.
- An instruction for the model.
- Example/Additional context.
- Input Data
- Output Indicator.

Example: We want to predict calories of a dish. Instructions specify the task and format, examples are other dishes' calories, input is the actual dish, and the output indicator guides what the model should produce.

![Good Prompt](/LLM_and_LLM_models/LLMOPs/images/prompts_example.png)

**Prompt Management**
- Crucial for efficiency, Re-producibility and collaboration.
- Important to track to visit and re-use the past results
    - Prompt
    - Output
    - A details about Model and Settings.
- Use prompt manager or version control
- Begin generating a collection of good input-output pairs for evaluation. 

**Prompt template**

Once we've gathered a collection of promising prompts, we're ready to proceed. We can always return to refine our prompt engineering later on. Right now, it's crucial to start developing prompt templates. In our application, we have input data that we need to transform to output. These templates use placeholders for input and work like recipes for different tasks, fitting any kind of data. They're essential for making reusable prompts, making our application more flexible and efficient. For example, with our dish calorie prediction template, we can predict calories for any type of dish. This will be even more helpful as we create advanced applications.

![Prompt Template](/LLM_and_LLM_models/LLMOPs/images/prompt_template.png)

### Chains and Agents
While prompts are central to any application, chains and agents provide its flow and structure.

Let's go back to our calorie prediction template. It requires examples, and an input which is the dish description. But how do we find examples? Suppose we have a database of dishes. We need to find similar dishes based on the description. Also, the application's output should be converted to a number for calorie calculation. To achieve this, we'll go through a few steps: 
- receiving input, 
- searching examples, 
- prompt creation, 
- output retrieval, 
- and output parsing. 

We can use chains or agents to build this functionality.

**Chains**

A chain, also known as a pipeline or a flow, consists of connected steps that take inputs and produce outputs. 

For our example, we start with a dish description. First, we find similar dishes in the database. Next, we combine the dish description and examples with the prompt template. This prompt is then given to the LLM. From the output, we extract a number.This process demonstrates a chain in action.

![Chain](/LLM_and_LLM_models/LLMOPs/images/chain.png)

The need for chain

- Develop sophisticated applications that interface with our own systems. 
- Establish a modular design, enhancing scalability and operational efficiency as our system grows. 
- Unlock endless possibilities for customization.

**Agents** 

Say our dish calorie prediction isn't giving optimal results, likely due to insufficient details like ingredients and quantities. Suppose we have the option to look up more information about a dish. We now have two actions: searching for extra examples, or retrieving additional dish information. This is where agents shine.

Agent consist of 
- Multiple actions(or tools)
- an LLM deciding which action to take 

Useful when
- There are many actions.
- The optimal sequence of steps is unknown.
- Uncertain about the inputs.

![Agent](/LLM_and_LLM_models/LLMOPs/images/agents.png)

Diffent between agents and chains.
|                   |  Chains               |     Agent  |
|------------        |--------------------   |------------|
| Nature        | deterministic         | Adaptive   |
| Complexity    | Low                   | Hight      |
| flexiability  | Low                   | Hight      |
| Risk          | Lower due t0Predictability                | Higher due to adaptibility    |

So the development cycle now like following

### Rag vs Finetuning.
RAG is a common LLM design pattern, combining the model's reasoning abilities with external factual knowledge.

RAG consists of three steps in a chain: 
- Retrieve related documents, 
- Augment the prompt with these documents, 
- Generate the output

This allows the LLM to use external data for better results. The retrieve step is crucial, as dealing with large knowledge bases can be challenging. Thankfully, there are ready-made solutions, often implemented using vector databases.

**RAG-chain with vector database**

Let's create a RAG-chain with vector databases. 

Retrieve.
- First, we convert the input into a numerical representation called an embedding, which captures its meaning. Similar meanings yield similar embeddings. Embeddings are created using pre-trained models. 
- Next, we search our vector database containing all embeddings. We compare the input embedding with those of the documents and calculate their similarity. 
- Finally, we retrieve the most similar documents.

Augment
- combines the input with these documents to create the final prompt.