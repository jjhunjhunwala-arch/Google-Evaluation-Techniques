Evaluation Techniques for Generative AI Images
A practical Jupyter notebook demonstrating how to evaluate AI-generated images using multiple complementary evaluation techniques rather than relying on a single score.

The notebook compares an image that closely follows a reference prompt with a deliberately misaligned image and evaluates both using:

SigLIP for fast image-text alignment scoring
Gemini as an LLM-as-a-Judge for multi-dimensional qualitative assessment
Gecko for granular rubric-based evaluation
The central idea is:

SigLIP for speed → Gemini/Gecko for depth → Humans for critical decisions

What this notebook demonstrates
Generative AI evaluation is difficult because "good" output can mean different things.

An image might:

look visually impressive but ignore the prompt
contain the requested objects but arrange them incorrectly
match the prompt semantically but have poor visual quality
appear correct to an automated similarity model while missing an important detail
This notebook demonstrates three ways of evaluating these differences.

Technique	Best for	Output
SigLIP	Fast semantic alignment checks	Single numerical score
Gemini LLM-as-a-Judge	Rich qualitative evaluation	Scores + explanations
Gecko	Prompt-element verification	Rubric-based results
Human review	Important or consequential decisions	Contextual judgement
The goal is not to identify one universally "best" evaluator.

Instead, the notebook demonstrates how different evaluation techniques answer different questions.

Evaluation workflow
The notebook follows this process:

Reference Prompt
      ↓
Generate Aligned Image
      ↓
Load Misaligned Image
      ↓
 ┌───────────────┐
 │    SigLIP     │ → Semantic alignment score
 └───────────────┘
      ↓
 ┌───────────────┐
 │    Gemini     │ → Multi-dimensional assessment
 └───────────────┘
      ↓
 ┌───────────────┐
 │     Gecko     │ → Rubric / element-level evaluation
 └───────────────┘
      ↓
Compare Results
      ↓
Human judgement where appropriate
Reference prompt
The experiment begins with a deliberately detailed prompt:

REFERENCE_PROMPT = """
A photorealistic golden retriever
puppy wearing a tiny red raincoat,
sitting on a wet cobblestone street
in Paris with the Eiffel Tower
visible in the background,
soft morning light
"""
This prompt contains several independently testable concepts:

golden retriever puppy
red raincoat
wet cobblestone street
Paris
Eiffel Tower
soft morning light
photorealistic style
This makes it useful for demonstrating how different evaluation approaches behave.

Creating the comparison
Aligned image
Gemini generates an image directly from the reference prompt:

response = client.models.generate_content(
    model=IMAGE_MODEL_ID,
    contents=REFERENCE_PROMPT,
    config=types.GenerateContentConfig(
        response_modalities=[
            "TEXT",
            "IMAGE"
        ],
    ),
)
This becomes the aligned image.

The objective is for the generated image to correspond closely to the reference prompt.

Misaligned image
A second image is loaded from:

misaligned_image.png
This represents an image that does not fully satisfy the reference prompt.

The two images are then evaluated using the same methods.

This creates a simple experiment:

Same prompt
   ↓
Aligned image       Misaligned image
       \               /
        \             /
         Evaluation
             ↓
          Compare
Part 1: SigLIP
What is SigLIP?
SigLIP is a vision-language model that can measure the relationship between an image and a text description.

The notebook uses:

google/siglip-base-patch16-224
through Hugging Face Transformers.

siglip_model = SiglipModel.from_pretrained(
    "google/siglip-base-patch16-224"
)

siglip_processor = SiglipProcessor.from_pretrained(
    "google/siglip-base-patch16-224"
)
Computing the score
The notebook processes the image and text together:

def compute_siglip_score(image, prompt):

    inputs = siglip_processor(
        text=[prompt],
        images=[image],
        return_tensors="pt",
        padding="max_length",
    )

    with torch.no_grad():
        outputs = siglip_model(**inputs)

    return torch.sigmoid**        outputs.logits_per_image
**  ).item()
Both images recei** a score:

aligned_sigl** = compute_siglip_score(
    alig**d_image,
    siglip_prompt
)

mis**igned_siglip = compute_siglip_sco**(
    misaligned_image,
    sigli**prompt
)
What SigLIP**ells us
SigLIP provides a fastumerical indication of image-textlignment.

The notebook uses thfollowing illustrative interpretaon:

> 0.5       Strong
0** - 0.5   Moderate
< 0.2       Wea**```

These thresholds should be t**ated as **demonstration heuristic**rather than universal quality thr**holds**. In a production evaluati** pipeline, thresholds should be c**ibrated against representative ex**ples and human-labelled data for **e specific use case.

### Strengt**
SigLIP is useful when you need:
** a fast automated signal
- large-**ale comparisons
- ranking of gene**ted images
- image-text alignment**hecks
- regression testing

### L**itation

A single score cannot te** you **why** an image scored well**r poorly.

For example:

```text
**ignment score: 0.61
does not**ell you whether the image:

omi**ed the red raincoat
showed the **ong dog breed
failed to depict **ris
omitted the Eiffel Tower
**ed the wrong lighting
This motives the second evaluation techniqu

Part 2: Gemini as an LLM**s-a-Judge
The second evaluation rategy uses Gemini as a multimodaevaluator.

Rather than asking:

"How similar is this image to thitext?"

the notebook asks Gemini ** evaluate several dimensions indi**dually.

Evaluation crite**a
Four criteria are used:

CRITERIA = [
    "prompt_adherence",
    "visual_quality",
    "coherence",
    "creativity",
]
ach criterion receives a score fr:

1 → 5
along with **short explanation.

Evalu**ion prompt
Gemini receives the ige together with an evaluation ruic:

EVAL_PROMPT_TEMPLA** = """
You are an expert image ev**uator.

Score this image against:**{prompt}"

Rate each criterion 1-**and explain briefly.

Return ONLY**alid JSON:

{
  "prompt_adherence** {
    "score": X,
    "explanati**": "..."
  },

  "visual_quality"**{
    "score": X,
    "explanatio**: "..."
  },

  "coherence": {
  **"score": X,
    "explanation": ".**"
  },

  "creativity": {
    "sc**e": X,
    "explanation": "..."
 **,

  "overall_score": X,

  "summ**y": "..."
}
"""
Why s**uctured JSON matters
Requesting ON makes the evaluator's output eier to:

parse
compare
visu**ise
store
analyse programmati**lly
integrate into evaluation p**elines
For example:

{
 **prompt_adherence": {
    "score":**,
    "explanation": "..."
  },
 **visual_quality": {
    "score": 4**    "explanation": "..."
  },
  "**herence": {
    "score": 5,
    "**planation": "..."
  },
  "creativ**y": {
    "score": 4,
    "explan**ion": "..."
  },
  "overall_score** 4.5,
  "summary": "..."
}
T** notebook converts Gemini's respo**e using:

json.loads(te**)
so the results can be anal**ed programmatically.

What**LM-as-a-Judge adds
Gemini provid** something the SigLIP score canno**

Score + explanation
```**Instead of:

```text
0.61
yocan obtain evaluations across mulple dimensions:

Prompt a**erence: 5/5
Visual quality:    4/**Coherence:         5/5
Creativity**       4/5
with explanations**or each result.

This makes LLM-a**a-Judge particularly useful for: ** exploratory evaluation

qualita**ve analysis
design review
ide**ifying recurring failure modes
**aluating subjective criteria
co**aring prompt variants
Lim**ations of LLM-as-a-Judge
LLM evaation is still model-generated evuation.

Results may vary between**uns and can be influenced by:

**aluator prompt wording
rubric d**initions
model version
image **mplexity
scoring scale
orderi** effects
ambiguity in evaluatio**criteria
For important evaluatio**pipelines, consider using:

exp**cit scoring rubrics
repeated ev**uations
multiple evaluators
r**resentative benchmark datasets
**man-labelled examples
human rev**w for consequential decisions
--**

Part 3: Rubric-based evaluatio**with Gecko
The third approach us** Gecko-style text-to-image evaltion** through Vertex AI's evalua**on capabilities.

Rather than reding the complete prompt to one silarity score, the objective is tovaluate whether individual requirents from the prompt are satisfie**

Preparing the evaluatio**dataset
The aligned and misalign** images are converted into evalua**on rows:

gecko_eval_da**set = pd.DataFrame([
    make_gecko_row(
        REFERENCE_PROMPT,
        aligned_image
    ),

    ma**_gecko_row(
        REFERENCE_PRO**T,
        misaligned_image
    )**])
The resulting dataset con**ins two cases:

Row 0 → A**gned image

Row 1 → Misaligned im**e
Both are evaluated against**he same reference prompt.

--**

Generating rubrics
The notebo** asks Vertex AI to generate rubri** using:

vtx_types.Rubr**Metric.GECKO_TEXT2IMAGE
Exam**e:

dataset_with_rubric**= (
    vtx_client.evals.generate**ubrics(
        src=gecko_eval_da**set,
        rubric_group_name="l**son3_lab_rubrics",
        predef**ed_spec_name=(
            vtx_ty**s.RubricMetric.GECKO_TEXT2IMAGE
 **     ),
    )
)
The detailedrompt can therefore be decomposednto more specific requirements.

**nceptually, instead of asking:

`**text Does this image match the prompt?


the evaluation can reason about individual elements such as:

```text
Is there a golden retriever puppy?

Is the puppy wearing a red raincoat?

Is the scene on a cobblestone street?

Does the scene represent Paris?

Is the Eiffel Tower visible?

Does the lighting resemble soft morning light?
This provides a much more diagnostic view of prompt adherence.

Running the Gecko evaluation
The generated rubrics are then used to evaluate the images:

gecko_result = vtx_client.evals.evaluate(
    dataset=dataset_with_rubrics,
    metrics=[
        vtx_types.RubricMetric.GECKO_TEXT2IMAGE(
            rubric_group_name=(
                "lesson3_lab_rubrics"
            )
        )
    ],
)
The result helps show which parts of the prompt were successfully represented and which were missed.

Why rubric-based evaluation matters
Consider two generated images.

Image A
Golden retriever puppy ✓
Red raincoat ✓
Wet cobblestone ✓
Paris ✓
Eiffel Tower ✓
Morning light ✓
Image B
Golden retriever puppy ✓
Red raincoat ✗
Wet cobblestone ✓
Paris ?
Eiffel Tower ✗
Morning light ✓
A single similarity score might tell you:

Image A = 0.68
Image B = 0.53
A rubric tells you why Image B is weaker.

That distinction becomes particularly useful when diagnosing prompts or evaluating model behaviour systematically.

Comparing the three techniques
Technique	Answers	Strength	Limitation
SigLIP	"How well do the image and text align?"	Fast and scalable	Little explanation
Gemini	"How good is this image across several dimensions?"	Rich explanations	Non-deterministic
Gecko	"Which specific prompt requirements were satisfied?"	Granular diagnosis	Less suited to highly subjective criteria
Human	"Is this actually acceptable for our intended use?"	Context and judgement	Expensive and slower
A layered evaluation strategy
One of the most important lessons from this notebook is that AI evaluation does not need to rely on a single evaluator.

A more robust evaluation pipeline can be layered.

Layer 1: Automated screening
Use SigLIP for:

Fast
↓
Cheap
↓
Scalable
↓
Alignment screening
Large numbers of generations can be evaluated quickly.

Layer 2: Rich evaluation
Use Gemini for:

Multiple quality dimensions
+
Explanations
+
Failure analysis
This is particularly useful when you need to understand why an output performs poorly.

Layer 3: Requirement-level evaluation
Use Gecko when you need to understand:

Which prompt requirements passed?

Which requirements failed?
This becomes useful for systematic prompt and model testing.

Layer 4: Human review
Use humans where:

decisions are consequential
evaluation criteria are highly subjective
domain expertise is required
safety or reputational risks exist
automated evaluators disagree
edge cases require contextual judgement
The resulting strategy is:

SigLIP
   ↓
Fast automated signal

Gemini
   ↓
Explainable multi-dimensional assessment

Gecko
   ↓
Granular requirement-level evaluation

Human
   ↓
Critical or contextual judgement
Visualising evaluation results
The notebook also compares results using a radar visualisation:

plot_radar_comparison(
    CRITERIA,
    aligned_image,
    aligned_gemini,
    aligned_siglip,
    misaligned_image,
    misaligned_gemini,
    misaligned_siglip
)
Visual comparisons can make it easier to identify differences across dimensions such as:

prompt adherence
visual quality
coherence
creativity
They can be especially valuable when comparing multiple prompting approaches or model versions.

Models and technologies
The notebook uses:

IMAGE_MODEL_ID = "gemini-3.1-flash-image"
TEXT_MODEL_ID = "gemini-3-flash-preview"
It also uses:

Google Gemini
Vertex AI
Vertex AI Evaluation
Gecko
SigLIP
Hugging Face Transformers
PyTorch
Pandas
Pillow
Jupyter Notebook
Requirements
Key Python packages include:

google-genai
google-cloud-aiplatform
transformers
torch
pandas
pillow
ipython
Install the repository dependencies with:

pip install -r requirements.txt
Google Cloud setup
The notebook expects access to a Google Cloud project configured for the relevant Vertex AI capabilities.

Authentication is handled using:

from helper import authenticate

credentials, project_id = authenticate()
The Gemini client is then created:

client = genai.Client(
    project=project_id,
    location="global",
    credentials=credentials,
    http_options=types.HttpOptions(
        base_url=os.getenv(
            "GOOGLE_VERTEX_BASE_URL"
        )
    ),
)
The evaluation client uses:

vtx_client = vtx_sdk.Client(
    project=project_id,
    location="us-central1",
    credentials=credentials,
    http_options=types.HttpOptions(
        base_url=os.getenv(
            "GOOGLE_VERTEX_BASE_URL"
        )
    ),
)
Ensure that your Google Cloud environment has the required APIs, permissions and model access before running the notebook.

Repository structure
A typical repository could look like:

.
├── README.md
├── evaluation-techniques.ipynb
├── helper.py
├── requirements.txt
└── misaligned_image.png
The helper.py file is expected to provide functions including:

authenticate()
extract_image()
make_gecko_row()
plot_radar_comparison()
Running the notebook
1. Clone the repository
git clone <repository-url>
cd <repository-name>
2. Create a virtual environment
python -m venv .venv
macOS/Linux
source .venv/bin/activate
Windows
.venv\Scripts\Activate.ps1
3. Install dependencies
pip install -r requirements.txt
4. Configure Google Cloud authentication
Ensure the environment can authenticate to the Google Cloud project used by the notebook.

5. Add the misaligned test image
Place:

misaligned_image.png
in the notebook directory.

6. Launch Jupyter
jupyter notebook
Run the notebook cells sequentially.

Some generation and evaluation operations may take several minutes.

Interpreting evaluation results
A key lesson is:

Do not treat an evaluation score as objective truth.

Evaluation metrics are signals.

For example, a strong SigLIP score tells you that the image and text are semantically aligned according to that model.

It does not automatically establish:

accuracy
safety
fairness
appropriateness
accessibility
brand compliance
domain correctness
Similarly, an LLM-generated score should not automatically become a pass/fail gate without first validating the evaluator against examples judged by people.

Evaluation for production systems
For experimentation, comparing two images manually is useful.

For production systems, a stronger evaluation framework might look like:

Representative Test Dataset
          ↓
Automated Metrics
          ↓
Rubric-based Evaluation
          ↓
LLM-based Evaluation
          ↓
Human-labelled Benchmark
          ↓
Threshold Calibration
          ↓
Regression Testing
          ↓
Production Monitoring
This transforms evaluation from a one-off test into an ongoing quality assurance process.

Responsible evaluation
Evaluation becomes particularly important when generative AI outputs are used in real-world settings.

Consider adding explicit evaluation criteria for:

factual accuracy
harmful or inappropriate content
stereotyping and bias
privacy
copyright
accessibility
brand requirements
domain-specific rules
representation of people
hallucinated visual content
The appropriate criteria will depend on the use case.

For high-impact or consequential applications, automated evaluation should support rather than replace expert human review.

Important implementation note
There is a small issue worth correcting in the supplied notebook.

In the misaligned Gemini evaluation section, the code currently prints:

print(f"Summary: {aligned_gemini['summary']}")
It should likely be:

print(f"Summary: {misaligned_gemini['summary']}")
Otherwise the aligned image's summary is shown while displaying the misaligned image's criterion scores.

Compatibility update
The notebook uses:

gemini-3.1-flash-image
rather than the retired preview identifier:

gemini-3.1-flash-image-preview
Model identifiers, preview endpoints and evaluation APIs can change, so check current Vertex AI documentation when reproducing the notebook.

Resources
Gecko research paper\n)\n)\n)\n)\n)\n)\n)\n)\n)
Vertex AI Gen AI Evaluation Service\n)\n)\n)\n)\n)\n)\n)\n)
Gecko text-to-image evaluation notebook\n)\n)\n)\n)\n)\n)\n)
Gecko text-to-video evaluation notebook
Key takeaway
There is no single perfect metric for generative AI quality.

Each evaluation approach provides a different signal:

SigLIP
"How aligned are the image and text?"

Gemini
"How does the image perform across different quality dimensions?"

Gecko
"Which specific requirements were satisfied?"

Human evaluation
"Is this output appropriate and good enough for the actual use case?"
Used together, these techniques provide a much richer picture of AI quality than any single score.

Measure quickly. Diagnose deeply. Keep humans involved where judgement matters.
