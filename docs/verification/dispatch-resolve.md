# Typed dispatch resolution verification

Audience: maintainer verification.

This record supports the opt-in `bin/fm-dispatch-resolve.sh` contract owned by [`../configuration.md`](../configuration.md) ("Typed dispatch resolution") and the declared rule and profile fields owned there under "Crew dispatch profiles".
It records only facts that must be re-established when the typesafe.ai model, its API, or firstmate's dispatch rules change.
Task chronology, the captain's rules, and the briefs themselves stay in the private scout report.

## The API the tool depends on

Verified 2026-09-16 against `https://api.typesafe.ai`.
`GET /v1/models` listed `jev-latest` and `jev-preview`, both released 2026-09-10; a `jev-latest` request answered as `jev-1.13.0`.
`POST /v1/systemone` takes `{model, state, questions}`; a `choice` question returns `{choice, probabilities, confidence}` with the probabilities summing to 1.
Observed error shapes: 401 `authentication_error` for a bad key, 403 when the header is missing, 422 with a `detail[].loc` naming the offending field, 400 `api_usage_error` for an unknown model, 405 on GET.
No rate-limit headers were present on any response; every response carried `x-typesafe-request-id`.
Observed end-to-end latency from a Mac was 123 to 348 ms per request, with the server's own upstream time at 4 to 60 ms.

## Live rule match against real briefs

Run 2026-09-16 with the key injected for the one command through the vault (`av inject +TYPESAFE_API_KEY -- ...`), model `jev-latest`, confidence floor 0.6, timeout 5 s, one `quota-axi --json` snapshot for the whole run.
Rules: the captain's five-rule file with a captain-authored none option, one `approval: captain` rule, two rule floors on `model:fable`, and declared `provider` on the Pi profiles.
Briefs: 15 real briefs from this home's recent work plus 10 synthetic ones written to hit each rule.

| Measure | Result |
| --- | --- |
| Rule matched the hand label | 20 of 25 |
| Resolved to the hand-labeled profile | 20 of 25 |
| Outcomes: clear / ambiguous / escalate / error | 18 / 1 / 6 / 0 |
| Clear results with a wrong profile | 0 |
| API latency (min / median / max) | 152 / 214 / 348 ms |
| Wall time per call including jq (min / median / max) | 198 / 261 / 396 ms |
| Input tokens per brief (min / median / max) | 1,279 / 3,114 / 4,538 |
| Output tokens | 150 to 152 |
| API errors | 0 |

Of the five disagreements, one was a wrong hand label (the brief quoted the bug-fix rule's wording verbatim), three were real briefs the model read as the approval-gated design rule at 0.66 to 0.86 confidence and escalated by design, each of which the captain had in fact dispatched at the strongest-reasoning class, and one was a synthetic tweak that came back ambiguous at 0.41 confidence and was handed back to firstmate.
A lean request that asks only the rule Choice matched the full request (rule, profile, and status) on all 25 briefs, which is why the shipped tool asks one question and keeps every gate in code.
That table records the 2026-09-16 run with the captain-authored none option.
A second live run on 2026-09-17 used the same 25 briefs, held one quota snapshot constant through a fake `quota-axi`, and exercised a copy of this branch with the shipped neutral `No listed rule applies to this task.` option and option-free interface.

| Measure | Result |
| --- | --- |
| Rule matched the hand label | 20 of 25 |
| Resolved to the hand-labeled profile | 18 of 25 |
| Outcomes: clear / ambiguous / escalate / error | 17 / 2 / 6 / 0 |
| Clear results with a profile other than the hand label | 1 |
| API latency (min / median / max) | 137 / 220 / 1,795 ms |
| Input tokens per brief (min / median / max) | 754 / 2,589 / 4,013 |
| Output tokens | 60 to 62 |
| API errors | 0 |

The maximum latency was one outlier; the next slowest request was 309 ms.
The differing clear result was a synthetic small tweak that matched the simple-bug-fix rule at 0.90 and selected `cursor-grok-4.6-medium` instead of the hand-labeled `cursor-grok-4.6-high`: the tweak exemption removed from the none-option text belongs in that rule's own `when` text.
Two default-labeled briefs became ambiguous.

## Offline behavior

`tests/fm-dispatch-resolve.test.sh` drives the public interface with a fake `curl` that records argv, the request body, the header read from file descriptor 3, and whether the secret reached its environment, plus a fake `quota-axi` that performs the same environment check.
It proves firstmate can invoke the resolve path without a preflight, rules are snapshotted once from the isolated home's canonical `config/crew-dispatch.json`, and dynamic output fields are flattened to one line.
It proves the absent key (environment and `.env`) prints one stderr line, nothing on stdout, exits 0, and never invokes `curl` or `quota-axi`.
It proves absent, default-only, and empty-rules files return `no rules to match` without a model or quota request, while a broken rules-file symlink exits 2 as unreadable.
It proves the documented starter configuration resolves its Pi default through the declared Claude provider, a `.env` key turns the tool on, and the environment wins over it.
It proves the key is absent from child environments, never appears on `curl` argv, and arrives only as the bearer header on the descriptor.
It proves the request uses the fixed endpoint and the pinned versioned model ID, carries only the project, the brief's `# Task` section, and the rule Choice with one option per rule plus the fixed neutral none option, and never carries `why`, `use`, or quota.
It proves a brief with no `# Task` heading is sent whole, that a brief written by `bin/fm-brief.sh` is sent without the scaffold around its task section, and that a heading pasted into the captain's intent or a `# comment` line inside a code fence does not cut the task text.
It proves the clear, fixed-floor ambiguous with candidate evidence, escalate (approval with candidate evidence, unverifiable rule floor, tie, nothing rankable), known rule-floor fall-through, known and unverifiable profile-floor evidence, explicit-provider and provider-ID enforcement, authoritative Agy and explicit-provider Gemini routing, partial providers, eligible unranked candidates and their clear-result note, concrete quota vetoes and profile-floor shortfalls taking precedence over uncertainty, account-wide quota veto, limiting-bound ranking, missing-curl and quota-axi failures, HTTP 429 and 500, transport failure, malformed usage, zero-mass or malformed probabilities or confidence, malformed or duplicate profile, invalid selector, removed-option rejection, and out-of-range rule ID paths behave as the contract states, with configuration errors exiting 2 before any network call.
It proves an `approval: captain` rule carrying at least 0.2 of the probability escalates whether or not it is the top choice, including behind the none option and below the confidence floor, while a rule below that floor or without `approval` does not.
It proves each answered resolve appends one private journal line holding the status, rule, confidence, probabilities, answering model ID, project, brief path, tokens, and chosen profile but never brief text or the key, that off, error, and no-rule outcomes append nothing, that `FM_STATE_OVERRIDE` selects the directory, and that an unwritable journal costs one stderr line and changes no outcome.
`tests/fm-bootstrap.test.sh` proves bootstrap ignores resolver-only fields without the typed key, validates each malformed shape when the environment or home `.env` activates typed resolution, and prevents an environment-provided key from reaching child processes.

```console
$ bash tests/fm-dispatch-resolve.test.sh | tail -1
# all fm-dispatch-resolve tests passed
```

## Re-running the evidence

A live run needs a key and is not part of the suite.
[`../configuration.md`](../configuration.md) ("Typed dispatch resolution") owns when to run it.
The labeled corpus is private home data under `data/dispatch-eval/` and is never committed: the `# Task` section of past briefs with secrets removed, and a tab-separated `labels.tsv` of `<brief file><TAB><expected rule, such as rule_4 or default>`.
From the firstmate home, with the key in the environment or `.env`:

```sh
scratch=$(mktemp -d)
while IFS=$'\t' read -r brief want_rule; do
  FM_STATE_OVERRIDE="$scratch" bin/fm-dispatch-resolve.sh "data/dispatch-eval/$brief" --project eval 2>/dev/null |
    awk -v b="$brief" -v w="$want_rule" '
      /^  status:/  { st = $2 }
      /^  rule:/    { rule = $2; conf = $NF }
      /^  profile:/ { sub(/^  profile: /, ""); prof = $0 }
      END { printf "%s\twant=%s\tgot=%s\tconf=%s\tstatus=%s\t%s\n", b, w, rule, conf, st, prof }'
done < data/dispatch-eval/labels.tsv | tee /dev/stderr | awk -F'\t' '
  { n++ } $2 == "want=" substr($3, 5) { hit++ } $5 == "status=clear" { clear++ }
  END { printf "briefs=%d rule_match=%d clear=%d\n", n, hit, clear }'
rm -rf "$scratch"
```

Each line shows the brief, the expected and matched rule, the confidence, the status, and the `profile:` line, and the last line counts briefs, rule matches, and clear results.
Compare it with the run before the change, and add a dated table like the ones above only when a run establishes a new current guarantee.
The first live run after the model was pinned also confirms that the API accepts the versioned ID `jev-1.13.0`, which the vendor's Models page says is accepted whether or not it is listed; an unknown model answers 400 `api_usage_error`, which the tool reports as an error outcome.
