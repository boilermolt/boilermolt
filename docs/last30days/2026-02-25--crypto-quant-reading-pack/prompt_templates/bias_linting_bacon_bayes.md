# Prompt template: Bias linting (Bacon) + Bayesian update check

## Task
Before approving a trade thesis, identify cognitive bias risks and show a minimal Bayesian update.

## Output
1) **Bias linting (Bacon’s idols)**
   - Idols of the Tribe (human nature/statistical fallacies): …
   - Idols of the Cave (personal preferences/data selection): …
   - Idols of the Marketplace (language/labels confusions): …
   - Idols of the Theatre (model ideology/over-trusting frameworks): …

2) **Hypotheses**
   - H1 (edge is real) vs H0 (noise/overfit)

3) **Evidence table**
   - Evidence items Ei with direction (+/−), reliability score, and whether it is independent.

4) **Bayesian update (toy but explicit)**
   - Prior P(H1)
   - Likelihood ratio for key evidence
   - Posterior P(H1|E)
   - Sensitivity: show posterior under 2 alternative priors.

5) **Decision rule**
   - Trade only if posterior exceeds threshold AND guardrail checks pass.

## References
- Bacon, Novum Organum: https://www.gutenberg.org/ebooks/45988
- Bayes (1763) (Royal Society PDF): https://royalsocietypublishing.org/doi/pdf/10.1098/rstl.1763.0053?download=true
