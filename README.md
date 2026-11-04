# disaster-crisis-triage
Crisis Traffic: multilingual & code-mixed disaster-post triage with web-traffic analytics
Course context: Web and Social Media Analytics (text preprocessing, NLP, social-media analytics, visualization and reporting).

What it does
For every disaster tweet the pipeline answers three questions:

What language / script is it? English, Devanagari Hindi, Romanized Hindi, or code-mixed (token-level lexicon + script detection).
What does a responder need to do? 5 response-oriented classes: urgent_need, offering_help, infrastructure_status, informational, irrelevant (remapped from HumAID's 11 humanitarian labels, see src/crisistraffic/labels.py).
When does the traffic arrive? Posting volume over time per event, burst hours, peak timing, and how the share of urgent needs changes across the event. Timestamps are recovered from tweet IDs (Twitter snowflake encoding), so no API is needed.
Research gaps this addresses
Gap	Where in the code
Romanized / code-mixed Indian text not handled	preprocess.py (language ID, transliteration, variant normalization) and the ablation in evaluate.ablation
English-only data, broad non-actionable labels	labels.py (5 actionable classes)
Poor generalization to a new disaster	evaluate.loeo (leave-one-event-out)
Quick start (Windows)
pip install -r requirements.txt
python tests/test_preprocess.py                 # unit tests should print "ok"
python tests/test_translate.py                  # translation tests (no model needed)
python run_pipeline.py                          # uses the HumAID path in config.yaml
Open results/report.html in your browser. To use another path: python run_pipeline.py --data "C:\path\to\events_set1". When you get HumAID set2 (adds Kerala floods etc.): --data PATH_SET1 PATH_SET2.

No data yet? python tests/make_fake_humaid.py then python run_pipeline.py --data tests/fake_data/events_set1 (random text; only proves the code runs).

Situation-room dashboard
python run_pipeline.py also writes results/situation.json. Double-click dashboard/situation_room.html to open it in your browser, then choose that JSON file. It shows, as of any moment in an event (time slider or Replay button): active urgent requests, offers of help, posts mentioning camps/shelters, infrastructure issues, help needed vs offered by type (rescue / medical / water / food / shelter), and the latest posts. Click "View illustrative demo" to see the layout with invented data.

Run everything on your own computer (no internet needed after install)
Windows: double-click run_all.bat. It installs the packages, runs the pipeline and opens the dashboard and report. Later, double-click open_dashboard.bat to reopen the dashboard. The dashboard loads its data automatically from dashboard/situation_data.js (written by the pipeline) and uses a bundled copy of Chart.js, so it works offline. Manual equivalent: pip install -r requirements.txt, python run_pipeline.py, then open dashboard/situation_room.html. Your data never leaves your machine.

Dashboards (dashboard/)
Double-click to open in a browser, then choose the files from results/:

situation_room.html (interface in English, Hindi, Kannada, Tamil, Bengali; filter posts by language) loads results/situation.json: active urgent requests, offers, camp/shelter mentions, requests vs offers by help type, infrastructure alerts, with an "as of" time slider and replay.
analytics_dashboard.html loads traffic_hourly.csv, traffic_event_summary.csv, loeo_summary.csv.
Outputs (results/)
report.html (everything, with charts), situation.json (feeds the situation room), loeo_summary.csv, loeo_per_class.csv, mapping_table.csv, situation.json, sentiment_by_class.csv / sentiment_by_event.csv / sentiment_hourly.csv, predictions_<file>.csv / predictions_summary_<file>.csv, traffic_hourly.csv, traffic_event_summary.csv, traffic_class_share_by_phase.csv, and, when a labeled Indian-language file is supplied, gap1_ablation.csv and language_mix.csv, plus translation_quality.csv / translation_samples.csv when translating.

The code-mixed experiment (needs your labeled data)
HumAID is English only, so it cannot test code-mixing. You need Hindi / Romanized / code-mixed posts (e.g. FIRE 2017 IRMiDis, the tweets from the Nepal earthquake):

python run_pipeline.py --make-annotation-sheet fire_tweets.csv sheet.csv --n 300   # samples 70% non-English
# hand-label the 'label' column with the 5 classes (two annotators if possible)
python run_pipeline.py --kappa annotator_a.csv annotator_b.csv                     # agreement
python run_pipeline.py --labeled sheet_labeled.csv --steps ablation report
The ablation compares raw, norm, norm_translit, norm_translit_langtag and reports macro-F1 per language group, both with cross-validation inside your labeled set and zero-shot from HumAID.

data/demo_codemixed_SYNTHETIC.csv is 60 posts I wrote to test the code. Never report numbers from it.

Adding datasets (e.g. the Indian multilingual set) and translating
Any labeled CSV can be added next to the HumAID disasters; it becomes one more selectable event. data/indian_crisis_testset.csv is already registered in config.yaml under data.extra (double-click run_indian.bat, or python run_pipeline.py --extra data/indian_crisis_testset.csv).

Choosing it

Pipeline: --events indian_crisis_testset scores only that set; the other event names work the same way.
Dashboard: it is in the event drop-down as indian_crisis_testset (order only, no real timestamps). The file has no timestamps, so posts are laid out in file order (15 fake "minutes" apart). Traffic analytics skip it; only the dashboard uses the fake order.
Your own file: add an entry under data.extra (path, and cols naming your id / text / label / language columns), or just run --extra my_file.csv if the columns are called id, text, label. Labels must be the 5 classes (infrastructure is accepted for infrastructure_status).
How it trains (three recipes, all from one command)

What	How	Where you see it
Zero-shot: train on all HumAID, test on the Indian set	default (train: false)	loeo_summary.csv row indian_crisis_testset; gap1_ablation.csv setting zero_shot_from_humaid
Cross-validation inside the Indian set (per language)	default, --steps ablation	gap1_ablation.csv setting cv4_on_indian_set, one row per lang_code
Train on HumAID and the Indian set	set train: true in config.yaml	the Indian set is then in every other event's training pool
The Indian file is a mock/test set: keep train: false when you want an honest test number. It writes the same scenario in ~12 languages, so the cross-validation keeps every scenario inside one fold (group_regex in config.yaml); a plain random split scored 0.55 macro-F1 where the grouped split scores 0.39 (on stand-in training data), because the same sentence leaked between train and test.

Translation (any language, any CSV)

python run_pipeline.py --extra data/indian_crisis_testset.csv --translate --prep trans_norm, or double-click run_indian_translate.bat. Non-English posts are translated to English by NLLB-200 (one offline model, ~200 languages), then classified by the English model. English posts, including all of HumAID, are never sent to the translator. Every translation is cached in data/translation_cache.jsonl, so each post is translated once.
One-time install: pip install torch transformers sentencepiece (the model, ~2.5 GB, downloads on first use).
Translate a file without training anything: python run_pipeline.py --translate-file in.csv out.csv --text-col text --lang-col lang_code (--lang-col is optional). Output adds source_lang, text_en, translated.
Source language: the language column if you give one (best), otherwise detected from the script, and Hindi / Marathi / Nepali are told apart with the small offline py3langid package. Latin-script text with no language column is only translated if it is confidently detected as a non-English language (Spanish, French, ...).
--prep trans_norm / trans_norm_langtag are two more rows in the preprocessing ablation, so you can compare them against raw/norm/... per language. If the file has an english_reference column, translation_quality.csv scores the translator against it per language (that column is used only for this check, never as classifier input).
Romanized text (Tamil/Malayalam/Hindi typed in Latin letters) is not translated by default: NLLB was not trained on it and mostly copies it back. translate.roman: true tries anyway (experimental, check translation_quality.csv). Without translation those posts keep going through norm_translit / character n-gram models, which is what the ablation compares.
No GPU needed, but CPU translation is slow for thousands of posts; the cache means you only pay once. labse_lr / minilm_lr are the translation-free alternative: multilingual embeddings read all of these languages directly.
Models (--model)
tfidf_lr (default, fast), tfidf_char_lr (character n-grams, robust to spelling variation), tfidf_wc_lr, and optional minilm_lr / labse_lr sentence embeddings (pip install sentence-transformers; downloads a model; the embedding models were not run in testing).

Multilingual interface
The situation room has a language switcher (English / हिन्दी / ಕನ್ನಡ / தமிழ் / বাংলা) and a post-language filter (English, Hindi Devanagari, Hindi Roman, code-mixed, other Indian language). The UI translations were machine-written: have a native speaker review them before showing them publicly. With HumAID alone every post is English, so the language filter only becomes interesting once you add FIRE / your own Hindi and code-mixed posts.

Data collection (web scraping)
python run_pipeline.py --collect "Kerala floods" collects public news headlines (Google News RSS) into data/collected_posts.csv. Other sources: --collect-url URL --collect-kind rss for any RSS/Atom feed, and --collect-kind html for the headlines and paragraphs of a page (--collect-selector changes which elements are kept). Use --collect-lang hi-IN for Hindi news. Re-running appends and removes duplicates. Rules built in: HTML pages are fetched only when robots.txt allows it, requests are rate-limited, and social-media sites are not scraped (their terms forbid it; use an official API or a dataset such as HumAID). Code: src/crisistraffic/collect.py.

Then label the collected posts with the model trained on HumAID: python run_pipeline.py --predict-file data/collected_posts.csv (add --translate --prep trans_norm for non-English posts). Output: results/predictions_<file name>.csv and predictions_summary_<file name>.csv (one pair per input file, so labeling a second file never overwrites the first). These classes are model guesses, not ground truth, so they are never used to score the model.

Unlabeled Indian-language posts (data/indian_crisis_posts_unlabeled.csv, 180 posts, columns id,text) are listed under predict.files in config.yaml, so a plain python run_pipeline.py (or run_all.bat) labels them and adds them to the dashboard. To run only that part: double-click run_predict_indian.bat, or python run_pipeline.py --extra data/indian_crisis_testset.csv --predict-file data/indian_crisis_posts_unlabeled.csv --predict-train-extras --steps predict situation report. The posts then appear in the dashboard drop-down as indian_crisis_posts_unlabeled (model-predicted) (order only, no real timestamps): the classes there are model guesses. --predict-train-extras makes the model learn from the labeled Indian set as well as HumAID; without it, only HumAID (English) is used, which reads Tamil/Telugu/Bengali/... posts very poorly unless you also add --translate --prep trans_norm. Add a lang column to the file (e.g. ta, hi-Latn) if you know each post's language; it improves translation. Expect rough labels: with only the 260 labeled Indian posts to learn from, the scenario-grouped cross-validation macro-F1 is about 0.4, so spot-check the predictions before trusting them.

Sentiment analysis
The sentiment step (on by default) scores English text with VADER and writes sentiment_by_class.csv, sentiment_by_event.csv and sentiment_hourly.csv; the report gets a sentiment section. Install: pip install vaderSentiment (a few KB, no model download). Non-English posts are scored on their English translation when you run with --translate; otherwise they are marked not scored (never neutral). Code: src/crisistraffic/sentiment.py.

Website, Google Analytics 4 and SEO
python build_site.py (or double-click build_site.bat) builds site/: a landing page, both dashboards and the report, each with SEO tags (description, canonical, Open Graph, JSON-LD on the landing page, sitemap.xml, robots.txt) and GA4 tracking of page views, clicks, dropdown choices (which disaster event is opened) and file loads. Fill the site: block in config.yaml first. Step-by-step setup, how to verify GA4, and the privacy note: docs/WEB_ANALYTICS_AND_SEO.md.

Honest limitations
Language ID and Romanized-Hindi detection are lexicon-based heuristics, not trained models; extend data/variants_extra.tsv as you find new spellings. Tamil/Kannada/Bengali are only detected by script.
Devanagari transliteration is rule-based and approximate (no full schwa deletion); the variant map covers common cases.
A TF-IDF model trained on English cannot read Hindi words it never saw, so expect norm_translit to help only when you also have Indian-language training data or multilingual embeddings.
The 5-class mapping is coarse: "displaced", "injured/dead" and "missing" are mapped to informational although some are urgent. The relabeled sample is how you measure that error.
Translation quality depends on NLLB; short, slangy or code-mixed posts translate worst. The english_reference check measures it, but it is a word-overlap proxy, not a human judgement. Marathi and Hindi are only reliably told apart when a language column is given.
Tweet-ID timestamps are creation times; they say nothing about location.
VADER is an English rule-based lexicon: it misses sarcasm and reads disaster vocabulary as negative, so use sentiment to compare classes and events, not as an absolute measure.
Collected news headlines are not tweets; the model was trained on tweets, so expect lower accuracy on them. Live collection depends on the source site and was only tested here on sample feed/page text.
GA4 numbers come from your own Google Analytics account after you publish the site; nothing in the pipeline can produce them.
Layout
run_pipeline.py        single entry point
config.yaml            default data path / model / prep
src/crisistraffic/     data, labels, lexicon, preprocess, translate, models, evaluate, traffic, situation, sentiment, collect, predict, report
build_site.py          website builder (SEO + GA4);  web/index.html is its landing-page template;  docs/ has the setup guide
data/                  variants_extra.tsv (editable), demo_codemixed_SYNTHETIC.csv, indian_crisis_testset.csv (mock, labeled), indian_crisis_posts_unlabeled.csv (unlabeled)
tests/                 unit tests + fake-data generator
Do not commit the datasets (see .gitignore); HumAID's Licensing.txt has the terms of use.
