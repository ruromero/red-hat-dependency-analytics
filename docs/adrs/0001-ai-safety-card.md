# AI safety information card display

Date: 2025-06-13

## Status
Proposed

## Context

In the effort of sharing Large Language Model (LLM) safety metrics to users. We need to design a solution
to effectively identify which LLMs are being used in a Python file when using any supported IDE (like VSCode).

The main goal of this ADR is to identify which LLM can be used regarless of the runtime context and
render a summary of the existing safety metrics that can quickly let the user know if the given LLM
has any problematic category that might affect the results.

There are some key considerations to have in mind when working with LLM in an AI enabled application:
- It is possible to use multiple Models depending on the user's preferences or context.
- It can be that multiple LM Eval executions apply to the same model name but with different parameters
(`dtype`, `batch_size`, `fewshot`). There is no _best_ result as it depends on how the model is used.
- Models are versioned (have revisions)
- Models can be fine-tuned. This affects performance and results for general tasks.

The extended report and context actions are out of the scope.

## Decision

The easiest and most flexible way to uniquely identify any LLM used in a programming language file like
Python is through annotated comments that will be used to query the backend for the summary results.
Then, for the LLMs the backend has data about a blue squiggly line will be rendered under the
name so that the user can hover and visualize a summary of the LM-Eval results.

### IDE Identification

We will use comments to annotate when different LLMs are used. When the file is open in the IDE, the
extension will look for the existing comment `@rhda` annorations.

```Python
# @rhda
# model=meta-llama/Meta-Llama-3.1-8B-Instruct
model_id = "meta-llama/Meta-Llama-3.1-8B-Instruct"

pipeline = transformers.pipeline(
    "text-generation",
    model=model_id,
    model_kwargs={"torch_dtype": torch.bfloat16},
    device_map="auto",
)
```

In this case it seems obvious we can find for some patterns or variable names but it can get more complicated
when the `model_id` is dynamic and the value come from some user setting, database value or environment
variable.

```Python
# @rhda
# model=meta-llama/Meta-Llama-3.1-8B-Instruct
# model=mistralai/Mistral-7B-Instruct-v0.2
model_id = session.execute(
    select(ModelRegistry.model_id).where(ModelRegistry.name == logical_model_name)
).scalar_one_or_none()

text_generator = pipeline(
    "text-generation",
    model=model_id,
    model_kwargs={"torch_dtype": torch.bfloat16},
    device_map="auto",
)
```

The IDE will identify the `@rhda` comment annotation and look for all the `model=` present in the same
comment.

### Backend request

The IDE extension will generate an HTTP POST request to the _Exhort_ backend to query the different LLMs.

```http
POST /api/v4/model-cards

{
  "llms": [
    {
      "model": "meta-llama/Meta-Llama-3.1-8B-Instruct"
    },
    {
      "model": "mistralai/Mistral-7B-Instruct-v0.2"
    }
  ]
}
```

With this generic request we can add later on filtering parameters specific to each model so that
users can get more accurate results if multiple LM-Eval results exist for the same model.

### Backend design and expected response

The _Exhort_ backend will query the existing reports by model name (and additional filters) in the document
Database and return a collection of matches where the key is the model name.

If the model is not found in the database, an empty array will be returned.

```json
{
  "meta-llama/Meta-Llama-3.1-8B-Instruct": [
    {
      "id": "99394101-1b19-4855-b8a5-bd684a09b2dc",
      "tasks-summary": [
        {
          "task": "crows_pairs_english",
          "metric": "pct_stereotype",
          "friendly-name": "Crows-Pairs (stereotyping)",
          "score": 0.6451997614788313,
          "assessment": "moderate"
        },
        {
          "task": "toxigen",
          "metric": "acc",
          "friendly-name": "ToxiGen (toxicity)",
          "score": 0.45851063829787236,
          "assessment": "moderate"
        }
      ],
      "config": {
        "model_name": "meta-llama/Meta-Llama-3.1-8B-Instruct",
        "model_source": "hf",
        "dtype": "float16",
        "batch_sizes": ["64"],
        "revision": "main",
        "revision_sha": "ef382358ec9e382308935a992d908de099b64c23"
      }
    }
  ],
  "mistralai/Mistral-7B-Instruct-v0.2": []
}
```

### Repository data

In order to make this solution open and collaborative we have decided to create a public repository where
any contributor can submit the LM-Eval results.

The backend will synchronize with the repository contents so that all the existing reports can be exposed
through the REST API.

### Summary data

All the `model=` that have a valid result will be underlined in blue so that when the user hovers over
the model name a summary modal is rendered.

The information that will contain will look like a table showing the task, the score and the assessment.

Example:

Safety metric|Score|Assessment
-------------|-----|----------
Ethics (morality)|0.601|Moderate
CrowS-Pairs (stereotyping)|0.623|Moderate
TruthfulQA (truthfulness)|0.360|High

The assessment value will be calculated in the backend based on predefined thresholds
that can be specific to each individual metric. Such thresholds will be persisted in the
database.

## Alternative Approaches

### Pattern matching

Use pattern matching to identify in a Python file where a model is used.

#### Pros

- Relatively simple solution and easy to implement
- Does not require user interaction or adding any metadata to the existing code.

#### Cons

- Will not be effective as it is not possible to know a pattern that can match any possible
model name. The namespace and name can be arbitrary.
- The model name can come from a database or external source or at runtime. In this case it is
impossible to resolve the right name.

### Static analysis of codebase usage

Static code analysis to identify when a call to an LLM is performed and evaluate the value of the
`model_id` used.

#### Pros

- Does not require user interaction or adding any metadata to the existing code.

#### Cons

- The model name can come from a database or external source or at runtime. In this case it is
impossible to resolve the right name.

### Runtime instrumentation

In Python and some other programming languages it is possible to add runtime instrumentation to
introspect how some methods are being executed. We can use this to generate a report per file name and
revision to know the values that are used at runtime.

#### Pros

- Does not require user interaction or adding any metadata to the existing code.
- Accurate identification of the LLM used.

#### Cons

- Complex solution to implement.
- Requires keeping runtime data of the users.
- The program must be executed first.
- LLMs can vary depending on the environment (staging, prod)

## Consequences

It is important to mention that the proposed solution has some requirements:

- Users have to actively specify which LLM will be used. 
- These annotations are tightly coupled to the IDE extension.

## External References

User story: [LLM identification in VSCode - Python](https://github.com/trustification/red-hat-dependency-analytics/issues/2)
Epic: [AI Model Cards](https://github.com/trustification/red-hat-dependency-analytics/issues/1)
