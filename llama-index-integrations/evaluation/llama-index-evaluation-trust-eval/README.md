# LlamaIndex Evaluation Integration: Trust-Eval

Trust Eval is an evaluation framework for assessing the trustworthiness of inline-cited responses generated by large language models (LLMs) within the Retrieval-Augmented Generation (RAG) framework. Our suite of metrics measures correctness, citation quality, and groundedness. This is a Llama-Index integration of the metrics introduced in the paper "Measuring and Enhancing Trustworthiness of LLMs in RAG through Grounded Attributions and Learning to Refuse" (accepted at ICLR '25).

For more information, visit [Trust-Eval github](https://github.com/shanghongsim/trust_eval/tree/main)

## Background

![Inline cited response](./assets/example_inline_cited_response.png)
*Figure 1: An example of an inline cited response.*

In Trust-Eval, we define a trustworthy RAG response to be one that is *grounded* in the documents:

- **Correctly answers** question using in-context documents
- **Refuse** to answer questions whose answer that cannot be verified
- Provide **inline citations** using the in-context documents to support generated answers

## Metrics

**Correctness**

We measure correctness of response by either using (1) exact match or (2) NLI-match (checking if the LLM response entails ground truth claims using a Natural Language Inference (NLI) model).

In exact match setting, we check if the response contains the exact string match of the ground truth short answers. For example, to the question "Who has the highest goals in world football?" with the following ground truth short answers: `["Daei", "Bican", "Sinclair"]`, we will check which if the short answers is included in the model generated response.  

Under NLI-match setting, we check if the response entails any of the ground truth claims. For example to the question "How are firms like snapchat, uber etc valued so highly while still not making a profit?" with the following ground truth claims:

```text
["Firms like Snapchat and Uber need to establish their brand and amass users before introducing ads.",
"Introducing ads too early can deter potential users.",
"Uber is reinvesting a lot of money to make their service better."]
```

We will check if the model's response entails the claim. Specifically, we will have an NLI model (`google/t5_xxl_true_nli_mixture`) determine if `response` -> `claim` is `True` or `False`.

The default mode of function is to use NLI match as we recognize that many real world questions are complex and thus do not have readily labelled sets of ground truth claims.

**Refusal**

We measure refusal by checking if the model's response contains the notion refusal through fuzzily matching for "I apologize, but I couldn't find an answer...".

**Quality of citations**

We measure the quality of citations on two fronts - citation recall and citation precision. Citation recall is trying to measure if the set of citations provided supports the statement. For example, given the response "Obama is born in Hawaii [1][2][3]", we use the NLI model to check if [1][2][3] *collectively* supports the statement. Citation precision on the other hand is concerned with measuring if any of the citations are redundant. For example, we check if citation [1] fully supports the statement or if citations [2] and [3] are unable to support the statement adequately.

## Installation

```bash
llama-index-evaluation-trust-eval
```

## Usage

```python
from llama_index.evaluation.trust_score import TrustScoreEvaluator

# Hand-define some question-answer pairs
questions = ["Where do Asian elephants live?", 
         "Can Orthodox Jewish people eat shellfish?", 
         "How many species of sea urchin are there?", 
         "How heavy is an adult golden jackal?",
         "Who owns FX network?",
         "How many letters are used in the French alphabet?"]
answers = ["Asian elephants live in the forests and grasslands of South and Southeast Asia, including India, Sri Lanka, Thailand, and Indonesia.",
          "No, Orthodox Jewish people do not eat shellfish because it is not kosher according to Jewish dietary laws.",
          "There are about 950 species of sea urchins worldwide.",
          "An adult golden jackal typically weighs between 6 and 14 kilograms (13 to 31 pounds).",
          "FX network is owned by The Walt Disney Company.",
          "The French alphabet uses 26 letters, the same as the English alphabet."]

# Instantiate the evaluator object
evaluator = TrustScoreEvaluator()

# Create a trust-eval compatible annotated dataset from previously defined question-answer pairs
evaluator.create_dataset(questions=questions, answers=answers)

# Generate LLM responses
evaluator.run()

# Evaluate LLM responses
result = evaluator.evaluate()

print(result)
```

## Output

```text
{ // refusal response: "I apologize, but I couldn't find an answer..."
    
    // Basic statistics
    "num_samples": 6,
    "answered_ratio": 50.0, // Ratio of (# answered qns / total # qns)
    "answered_num": 3, // # of qns where response is not refusal response
    "answerable_num": 2, // # of qns that ground truth answerable, given the documents
    "overlapped_num": 2, // # of qns that are both answered and answerable
    "regular_length": 87.0, // Average length of all responses
    "answered_length": 90.66666666666667, // Average length of non-refusal responses
    
    // Refusal groundedness metrics

    // # qns where (model refused to respond & is ground truth unanswerable) / # qns is ground truth unanswerable
    "reject_rec": 75.0,

    // # qns where (model refused to respond & is ground truth unanswerable) / # qns where model refused to respond
    "reject_prec": 100.0,

    // F1 of reject_rec and reject_prec
    "reject_f1": 85.71428571428571,

    // # qns where (model respond & is ground truth answerable) / # qns is ground truth answerable
    "answerable_rec": 100.0,

    // # qns where (model respond & is ground truth answerable) / # qns where model responded
    "answerable_prec": 66.66666666666667,

    // F1 of answerable_rec and answerable_prec
    "answerable_f1": 80.0,

    // Avg of reject_rec and answerable_rec
    "macro_avg": 87.5,

    // Avg of reject_f1 and answerable_f1
    "macro_f1": 82.85714285714286,

    // Response correctness metrics

    // Regardless of response type (refusal or answered), check if ground truth claim is in the response. 
    "regular_claims_nli": 33.33333333333333,
    
    // Only for qns with answered responses, check if ground truth claim is in the response. 
    "answered_claims_nli": 33.33333333333333,

    // Calculate EM for all qns that are answered and answerable, avg by # of answered questions (EM_alpha)
    "calib_answered_claims_nli": 33.33333333333333,

    // Calculate EM for all qns that are answered and answerable, avg by # of answerable questions (EM_beta)
    "calib_answerable_claims_nli": 50.0,

    // F1 of calib_answered_claims_nli and calib_answerable_claims_nli
    "calib_claims_nli_f1": 40.0,

    // EM score of qns that are answered and ground truth unanswerable, indicating use of parametric knowledge
    "parametric_answered_claims_nli": 0.0,

    // Citation quality metrics

    // (Avg across all qns) Does the set of citations support statement s_i? 
    "regular_citation_rec": 32.5,

    // (Avg across all qns) Any redundant citations? (1) Does citation c_i,j fully support statement s_i? (2) Is the set of citations without c_i,j insufficient to support statement s_i? 
    "regular_citation_prec": 63.33333333333333,

    // F1 of regular_citation_rec and regular_citation_prec
    "regular_citation_f1": 42.95652173913043,

    // (Avg across answered qns only)
    "answered_citation_rec": 50.0,

    // (Avg across answered qns only)
    "answered_citation_prec": 76.66666666666666,

    // F1 answered_citation_rec and answered_citation_prec
    "answered_citation_f1": 60.526315789473685,

    // Avg (macro_f1, calib_claims_nli_f1, answered_citation_f1)
    "trust_score": 61.127819548872175
}
```
