# Adverse Drug Event Extraction Report using Generative NLP Models: CADEC Dataset 
## Overview
- Developed an NLP system that extracts adverse drug events (ADEs) from patient forum posts.
- Used generative language models from Hugging Face for medical abbreviation expansion and entity extraction.
- Standardized extracted entities using the UMLS API.
- Implemented a verification system with iterative correction capability.

## Technical details
### Data Preprocessing
- Load the unannotated text posts from ```CADEC.v2/cadec/text```
- Load the annotated posts from ```CADEC.v2/cadec/original```. Parse them to get ground truth dictionaries.

### Generative NLP
#### Model Choice
Chose ```Phi-4-mini-instruct``` with 4-bit quantization for a good balance of inference speed and output quality. Experimented with ```Qwen2.5``` and ```Gemma-3``` models, but ```Phi-4``` gave the best performance.
Used ```batch_size = 20```, ```temperature = 0```, ```max_new_tokens = 512```.
 
#### Abbreviation Expansion and Entity Extraction
Employed Hugging Face pipelines to expand medical abbreviations and extract entities from posts in batched mode for efficiency.

###  Entity Standardization
Standardized the extracted entities using the UMLS API. Mapped drugs to **RxNorm** and ADEs/Symptoms to **SNOMED CT**. Retrieved best match, otherwise returned original.

Implemented an ```LRU Cache``` and parallel API requests for faster performance.

### Verification System
1. JSON parsing
2. Completeness check: Used fuzzy matching to compare extracted entities with ground truth.
3. Similarity check: Used sentence embeddings (```MiniLM-L6-v2```) for semantic similarity verification
- Note: To account for the confusing difference between "symptoms/diseases" and "adverse drug events", partial credit is given for entities matched across these categories.

### Iterative Correction
- Implemented a feedback loop with up to 3 attempts.
- On failure, the generative model extraction was called again with the generated feedback and previous attempt.
- Feedback was generated from the verification system
- Log all successes, failures, and feedback.

### Results
Achieved a success rate of 81% (including retries) on a sample of 400 posts. 

A "success" is defined as extracted entities passing the verification system within 3 attempts.

Saved to ```extracted_entities.json```

### Setup and Usage
- Folder paths: ```CADEC.v2/cadec/text``` and ```CADEC.v2/cadec/original```, if run locally. Otherwise, save ```Datasets/CADEC.v2.zip``` to your Google Drive.
- Add your UMLS API key (```UMLS_API_KEY```)

### Note
Due to GPU limitations on Colab, only 400 out of 1250 posts were fully processed.
To run on the full dataset, set ```SAMPLE_SIZE``` to 1250

### Examples
```
Extracted:  {'drugs': ['propain', 'mypaid forte', 'ibuprofen', 'acetaminophen 500 MG / metoclopramide 5 MG Oral Tablet'], 'ades': ['Xerostomia'], 'symptoms_diseases': ['Pain']}

Ground Truth:  {'drugs': ['propain', 'mypaid forte', 'Paracetamol 325 mg', 'ibuprofen 400 MG'], 'ades': ['Xerostomia'], 'symptoms_diseases': ['Pain']}
```

```
Extracted:  {'drugs': [], 'ades': ['Stomach ache', 'excessive drowsiness'], 'symptoms_diseases': ['Pain in left foot', 'Pain', 'Drowsiness']}

Ground Truth:  {'drugs': [], 'ades': ['Stomach ache', 'excessive drowsiness'], 'symptoms_diseases': ['Pain', 'hobbling', 'foot was hurting']}
```

```
Extracted:  {'drugs': [], 'ades': ['burning/numb sensation in hands & feet', 'Arthralgia', 'Blood Pressure', 'cholesterol', 'Diabetes Mellitus'], 'symptoms_diseases': ['Blood Pressure', 'cholesterol', 'Diabetes Mellitus']}

Ground Truth:  {'drugs': [], 'ades': ['Burning sensation quality', 'Arthralgia', 'Numb chin syndrome'], 'symptoms_diseases': ['cholesterol', 'Blood Pressure', 'Diabetes Mellitus']}
```

```
Extracted:  {'drugs': ['Lipitor', 'Simcor', 'Bio-Statin'], 'ades': ['agonizing pain', 'crippling', 'Arthralgia of the ankle and/or foot'], 'symptoms_diseases': ['pain in feet and legs', 'Disability', 'Myocardial Infarction']}

Ground Truth:  {'drugs': ['Lipitor', 'Simcor', 'Simcor'], 'ades': ['pain in feet and legs', 'Arthralgia of the ankle and/or foot', 'Feet still hurt'], 'symptoms_diseases': ['Pain', 'Myocardial Infarction', 'Registered disabled']}
```

#### Iterative loop example

```
Processing post 384/400: LIPITOR.290.ann
------------------------------------
Attempt 1: Raw JSON string - {
  "drugs": [],
  "ades": ["pain in shoulder and arm", "could not sleep because of pain"],
  "symptoms_diseases": ["pain in shoulder and arm", "could not sleep because of pain"]
}

Parsed successfully

Extracted:  {'drugs': [], 'ades': ['pain in shoulder and arm', 'could not sleep because of pain'], 'symptoms_diseases': ['pain in shoulder and arm', 'could not sleep because of pain']}
Ground Truth:  {'drugs': ['omega-3 fatty acids', 'coq10'], 'ades': ['Pain in muscle of shoulder', 'Pain', 'Pain', 'could not lift arm', 'Pain in left arm'], 'symptoms_diseases': []}

Attempt 1 failed: similarity verification failed

Retrying with feedback: Feedback: Try extracting more 'drugs' (expected 2, got 0), more 'ades' (expected 5, got 2), fewer 'symptoms_diseases' (expected 0, got 2).

Attempt 2: Raw JSON string - {
  "drugs": ["omega 3", "coq10"],
  "ades": ["pain in shoulder and arm", "could not sleep because of pain", "pain subsided", "cholesterol levels went down 100 points", "pain subsided"],
  "symptoms_diseases": []
}

Parsed successfully
Extracted:  {'drugs': ['omega-3 fatty acids', 'coq10'], 'ades': ['pain in shoulder and arm', 'could not sleep because of pain', 'pain subsided', 'cholesterol levels went down 100 points', 'pain subsided'], 'symptoms_diseases': []}
Ground Truth:  {'drugs': ['omega-3 fatty acids', 'coq10'], 'ades': ['Pain in muscle of shoulder', 'Pain', 'Pain', 'could not lift arm', 'Pain in left arm'], 'symptoms_diseases': []}
```
