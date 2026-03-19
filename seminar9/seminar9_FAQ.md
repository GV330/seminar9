# Seminar 9 FAQ: Text as Data

*Answers to common questions from seminar9_text_analysis.Rmd. Last updated March 2026.*

---

## Setup and Basics

### 1. Why are we using `quanteda` rather than, say, `tidytext` or `tm`?

All three packages can build a document-feature matrix (DFM) — a table of word counts per document — but they differ in philosophy and scope. `tm` is the oldest and was the standard for many years, but its workflow is clunky by modern R standards. `tidytext` is elegant and integrates well with the tidyverse, but it is optimised for "one token per row" data rather than the matrix-based operations that downstream methods (like topic models and keyness statistics) expect. `quanteda` was written by political scientists, for political scientists: it ships with built-in support for corpora with document-level metadata, fast DFM operations, keyness statistics, and direct integration with `stm` and `quanteda.sentiment`. It is the dominant choice in computational political science research, which is why we use it here.

---

### 2. Why do we need `quanteda.sentiment` separately — isn't it part of `quanteda`?

`quanteda` is the core package: corpus construction, tokenisation, DFMs. Sentiment analysis is a more specialised downstream task, so it lives in a companion package, `quanteda.sentiment`, maintained by the same team. This modular design is common in R — it keeps the core package lean and lets the extension packages evolve independently. Crucially, `quanteda.sentiment` is not available on CRAN; it must be installed from GitHub using `remotes::install_github("quanteda/quanteda.sentiment")`. The setup chunk at the top of the seminar Rmd handles this automatically, but if you see an error saying the package cannot be found, that is why.

---

### 3. What does `eval=FALSE` mean in an R Markdown chunk?

Every R Markdown code chunk can have options set in its header, like ` ```{r chunk-name, eval=FALSE} `. The `eval=FALSE` option tells knitr (the engine that knits Rmd files) to *display* the code but not *execute* it. The chunk appears in the output document as formatted code, but no results, plots, or errors are generated. This is useful when: (a) code requires a setup that cannot be assumed on all machines (like a Python environment); (b) code takes too long to run during normal knitting; or (c) you want to show example code without running it. You can still run an `eval=FALSE` chunk manually in RStudio by clicking the green run arrow. Note that `echo=FALSE` is the complementary option — it runs the code but hides it from the output.

---

### 5. What is a Python environment and why does running transformer models in R require Python?

The transformer models we use — from the Hugging Face library — are written in Python, not R. The `reticulate` package in R acts as a bridge: it allows R to call Python functions as if they were R functions, but a working Python installation must be present on the machine. Running `transforEmotion::setup_modules()` automates the process of creating an isolated Python environment with the right dependencies (PyTorch, Transformers, etc.) so you do not have to configure Python manually. An **isolated environment** means these packages are installed in a self-contained folder — they will not conflict with other Python software you may have installed. This is why Section 4 is set to `eval=FALSE`: not everyone attending the seminar will have Python configured, and running transformer inference on a CPU is slow enough to make live demonstration impractical.

---

## Section 1: Corpora and the DFM

### 6. What is a "bag of words" and what does it mean to say it ignores structure?

A **bag-of-words** representation treats a document as an unordered collection of words. You "throw" all the words from a document into a bag, shake it up, and count how many times each word appears — but you lose all information about the order those words appeared in. This is exactly what the DFM captures: counts, not sequences. The phrase "The dog bit the man" and "The man bit the dog" produce identical bags — and identical DFM rows — even though they describe very different events. Structure here means any kind of ordering: word order within a sentence (syntax), the position of a paragraph in a document, the way a negation ("not good") modifies the meaning of a following word. Bag-of-words methods are powerful and fast, but they are inherently limited by this assumption. Transformer models like BERT move beyond this by reading text sequentially and building representations that depend on context.

---

### 7. What is a DFM and how is it different from a TDM?

A **Document-Feature Matrix (DFM)** is a rectangular table where each row is a document and each column is a word (or other linguistic feature). The cell value is how often that word appears in that document. This is `quanteda`'s preferred term. A **Term-Document Matrix (TDM)** is exactly the same thing, just transposed: rows are terms, columns are documents. The two representations contain identical information; the difference is purely a matter of orientation. Most modern packages, including `quanteda` and `stm`, work with the DFM orientation because it is more natural to think of each document as an observation. When people say "bag-of-words model" they are referring to the assumption underlying both: word order is discarded, and only counts are retained.

---

### 8. What is stemming and why might it hurt sentiment analysis but help topic modelling?

**Stemming** strips words down to an approximate root form using simple rules: "climate", "climates", and "climatic" all become "climat"; "energy" and "energies" both become "energi". This is useful for topic modelling because it collapses variants of the same concept into one feature, reducing noise and improving the model's ability to detect co-occurrence patterns across documents. However, stemming can damage sentiment analysis because many dictionary entries are full, unstemmed words. If your sentiment dictionary contains "positive" but the stemmer reduces it to "posit", the match fails and the word goes unscored. Stemming can also destroy negation handling: the negated form "not good" relies on exact word matching that stemming may disrupt. This is why the seminar builds a separate, un-stemmed DFM specifically for sentiment analysis (Section 3).

---

### 9. Why do we use an un-stemmed DFM for sentiment analysis (Section 3) but a stemmed one for word frequencies (Section 2)?

The choice depends on what the DFM will be used for. In Section 2, we are counting and visualising word frequencies. Stemming there is desirable: we want "climate change", "climatic policy", and "climate action" to all contribute to the same "climat" count, giving a cleaner picture of the corpus's themes. In Section 3, we are matching words against a pre-built dictionary. That dictionary was compiled using the full, unstemmed words it expects to find in text — for example, the Lexicoder Sentiment Dictionary (LSD2015) contains entries like "catastrophic" and "hopeful", not "catastroph" and "hope". Stemming the text before dictionary matching therefore reduces the number of successful matches and degrades the sentiment scores. The general principle is: stem when your analysis is about themes and frequencies; do not stem when your analysis depends on exact word matching against an external vocabulary.

---

## Section 2: Word Frequencies and Word Clouds

### 10. The word cloud shows stemmed words like "climat" and "energi" — is that a mistake?

No — this is expected and intentional. The DFM used for the word cloud in Section 2 was built with `tokens_wordstem()` applied, which uses the Porter stemming algorithm. The Porter stemmer uses rule-based truncation to reduce words to their approximate root: "climate" becomes "climat", "energy" and "energies" both become "energi", "government" becomes "govern". These truncated forms look strange out of context, but they serve a real purpose: they ensure that all inflected forms of a word contribute to the same count, so the word cloud reflects the frequency of a concept rather than any particular grammatical form of it. If you want un-stemmed words in your word cloud, simply rebuild the DFM without the `tokens_wordstem()` step. For the sentiment analysis in Section 3, the Rmd deliberately avoids stemming precisely because the stemmed forms would not match the dictionary entries.

---

## Section 3: Dictionary-Based Sentiment

### 11. What is the Lexicoder Sentiment Dictionary (LSD) and why is it better for political text than a general dictionary?

The **Lexicoder Sentiment Dictionary (LSD)**, developed by Young and Soroka (2012), is a list of words classified as positive or negative that was built and validated specifically for political news text. Most general-purpose sentiment dictionaries (like Hu & Liu, or AFINN) were built from product reviews or social media and calibrated against consumer opinions. Political language has distinctive properties: words like "radical", "progressive", "liberal", "green", and "nationalist" carry valence in political contexts that differs from everyday usage. The LSD was compiled by political scientists who read and manually reviewed thousands of news articles about politics. It also includes negation handling — pairs like "not good" are treated as negative — which most simpler dictionaries ignore.

---

### 12. Why do the LSD and Hu & Liu dictionaries sometimes disagree?

The two dictionaries were built for different purposes and from different source material, so they encode different assumptions about what words mean. Hu & Liu was built from Amazon product reviews, where "radical" might mean something like "extreme" (negative) rather than "reformist" (positive). The LSD was validated on political news where the same word might carry a positive connotation in a Green Party context ("radical action on climate"). Proper nouns and political vocabulary like "Labour", "Conservative", "nuclear", and "Parliament" are often missing entirely from consumer lexica, or assigned weights that do not make sense in political contexts. This is precisely the point that Bleich and van der Veen (2021) make: no single dictionary is definitive, and averaging across multiple lexica is one way to hedge against idiosyncratic errors in any one of them.

---

## Section 4: Transformer-Based Sentiment

### 13. Why are the `transforEmotion` code chunks set to `eval=FALSE`?

There are two practical reasons. First, `transforEmotion` requires Python to be installed and configured via Miniconda — this is not something we can guarantee on every student machine, especially in a lab setting. Second, running transformer models is computationally expensive: inference on even a small sample of press releases can take several minutes on a CPU, which is too slow for a live seminar. Setting chunks to `eval=FALSE` means the code is displayed and explained but not executed when the document is knitted. You can still run the chunks manually in your R console (by clicking the green run arrow in RStudio) if you have the Python environment set up. The purpose of including the code is to show you what the workflow looks like, so you can implement it on your own research data later.

---

### 14. What is the difference between DistilBERT (SST-2) and BART-large-mnli for sentiment analysis?

These are the two models that may be used when you run `transformer_scores()` in `transforEmotion`. They differ in architecture, training, and what they are good at.

**DistilBERT (SST-2)**

DistilBERT is a compressed version of BERT. The SST-2 variant has been *fine-tuned* on the Stanford Sentiment Treebank, a dataset of movie reviews labelled positive or negative. It is a discriminative classifier: it was explicitly trained to predict sentiment, and it is fast and accurate within its domain. The limitation is that its training data (movie reviews) is quite different from political press releases, which can hurt its accuracy on our texts. Hugging Face model card: https://huggingface.co/distilbert/distilbert-base-uncased-finetuned-sst-2-english

**BART-large-mnli**

BART is a sequence-to-sequence model (encoder-decoder architecture), originally designed for text generation tasks (Lewis et al., 2020: https://arxiv.org/abs/1910.13461 / https://aclanthology.org/2020.acl-main.703/). The `facebook/bart-large-mnli` variant has been fine-tuned on the Multi-Genre NLI (Natural Language Inference) dataset, which trains it to judge whether a hypothesis follows from a premise. For sentiment classification, it asks: "does this text *entail* the label 'positive'?" — this is the zero-shot NLI approach. It is more flexible (it works with any labels you give it) and is the model we explicitly use in the seminar. The `facebook/` prefix in the name reflects the fact that Meta AI Research published it under their original Facebook organisation account on Hugging Face before the company rebranded. Hugging Face model card: https://huggingface.co/facebook/bart-large-mnli

**What BART-large-mnli returns**

`transformer_scores()` returns a probability distribution over the labels you supply (`positive`, `negative`, `neutral`). These three scores sum to 1.0 for each document, representing the model's confidence that the document belongs to each class. This is structurally different from the LSD score, which is a single continuous polarity value computed roughly as `(positive_words − negative_words) / (pos_words + neg_words)`. The two scales measure different things and should not be directly compared numerically — but examining where they *disagree* is a useful qualitative validation exercise.

**The key trade-off**

The literature consistently finds that fine-tuned models (like DistilBERT/SST-2) outperform zero-shot approaches when training data is available and matches the domain. Zero-shot BART-MNLI wins when you have no labelled data, a novel domain, or want flexibility in what the labels mean. For political text, neither model's training data is ideal — which is one reason dictionary methods like LSD remain competitive.

**Further reading**

For a political science perspective on this trade-off, two papers published in *Political Analysis* are directly relevant:

- "Political DEBATE: Efficient Zero-Shot and Few-Shot Classifiers for Political Text": https://www.cambridge.org/core/journals/political-analysis/article/political-debate-efficient-zero-shot-and-few-shot-classifiers-for-political-text/8D0B3E2AAF711F4812E42466DE503A13
- "Stay Tuned: Improving Sentiment Analysis and Stance Detection Using Large Language Models": https://www.cambridge.org/core/journals/political-analysis/article/stay-tuned-improving-sentiment-analysis-and-stance-detection-using-large-language-models/2D8F121012D3D1CB2259B6DD5EE32D0D

A broader benchmark showing fine-tuned smaller models often outperform zero-shot generative approaches: https://arxiv.org/html/2406.08660v1

---

## Section 5: Topic Modelling with STM

### 15. What does K=10 mean in the STM? How does one choose K?

In a **Structural Topic Model (STM)** — an extension of Latent Dirichlet Allocation (LDA) that allows document-level covariates to influence topic prevalence — K is the number of topics you ask the model to find. There is no single correct value; it is a modelling decision similar to choosing the number of clusters in k-means. Choosing a small K (say, 5) gives broad, interpretable topics but may lump distinct themes together. Choosing a large K (say, 50) gives finer-grained topics but they may be harder to interpret and some may be essentially duplicates. Common approaches to choosing K include: (1) fitting models with several values (e.g., 5, 10, 15, 20) and comparing held-out likelihood and semantic coherence scores using `searchK()` in the `stm` package; (2) reading the top words for each topic and checking whether K topics produce labels you can meaningfully distinguish; (3) consulting substantive knowledge about how many distinct themes you would expect in the corpus. For a corpus of Green Party press releases, K=10 is a reasonable starting point.

---

### 16. What is a prevalence covariate in STM and why is `year_decimal` a good choice here?

In an STM, a **prevalence covariate** is a document-level variable that is allowed to influence how much a given topic appears in a given document. Including `year_decimal` as a prevalence covariate means we are fitting a model where the proportion of each document devoted to each topic can shift systematically over time. After fitting the model, `estimateEffect()` regresses topic proportions on the covariate for each topic, letting us plot and test whether prevalence increases or decreases over time. We use `year_decimal` (a continuous decimal representation of the date, e.g. 2019.583 for late July 2019) rather than integer year because it preserves the within-year ordering of documents, giving the model more information than collapsing all press releases from the same calendar year to a single point.

---

### 17. How does STM compare to transformer-based topic modelling approaches?

STM and transformer-based topic models (such as BERTopic) are both topic models — algorithms that group documents into thematic clusters — but they work in fundamentally different ways.

**STM** is a bag-of-words model: it learns topics from word co-occurrence patterns (which words tend to appear together across documents). Its key strengths for social science are (1) **covariate integration** — document-level variables like year, party, or country can be used to explain variation in topic prevalence — and (2) **statistical inference** via `estimateEffect()`, which gives you coefficients, standard errors, and confidence intervals. This is what makes STM suitable for hypothesis testing, not just description.

**Transformer-based topic models** like BERTopic use dense semantic embeddings: documents are represented as high-dimensional vectors encoding meaning, and topics are formed by clustering similar documents in that space. This approach handles proper nouns, multi-word phrases, and semantic synonyms that STM would treat as unrelated features. The trade-off is that these models provide no built-in covariate integration and no standard errors, and they require Python and a set of additional dependencies (sentence-transformers, UMAP, HDBSCAN), which makes them harder to run reliably in a classroom setting.

We cover STM in this seminar because it is the workhorse method for hypothesis-driven political text analysis and runs entirely in R. Transformer-based topic modelling is a rapidly developing area worth being aware of, but we do not cover it explicitly here.

---
