You are screening math questions for a retrieval benchmark. Each question will be shown to a search system ON ITS OWN, without the web page it came from. Decide for each question whether it is SELF-CONTAINED: a competent reader can tell exactly what is being asked and what a correct answer must establish, using only the question text.

Mark self_contained = false when any of these holds, and give every code that applies:
- missing_figure: the question depends on a figure, diagram, graph, table or image that the text does not describe.
- missing_values: it refers to quantities, data, conditions, answer options, definitions or earlier parts that are not stated ("the data above", "the following choices", "the circles shown", a sample whose numbers are not given).
- external_reference: it depends on a specific study, paper, dataset, course exercise or other source whose content is not given.
- unclear_target: what must be computed, proved or explained is not determined.
- not_math: answering it needs no mathematical, statistical or quantitative reasoning at all (for example pure trivia, product or game questions, programming syntax).

Do not penalize:
- standard named theorems, definitions, notation and well-known constants;
- assumptions a typical textbook solver makes without being told, such as uniform acceleration, starting from rest, a sinusoidal alternating current, constant pressure in a gas-law problem, standard conditions, or two parents in a family-age problem;
- questions that ask for a general method or explanation, as long as the subject is specified;
- informal wording, typos or unusual notation, as long as the meaning is recoverable.

Judge each question independently. Do not try to solve it. Keep "note" under 20 words. When self_contained is true, reason_codes must be empty.
