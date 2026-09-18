# Escalation & Recovery

## What the handout asks for

Per the *Technical Challenge – FDE* PDF, Topic 2 is **"Escalation & Recovery (Operational Leadership and Innovation)"**, 
a 25-minute segment including Q&A. The scenario: a large customer's search platform is intermittently failing under peak 
traffic and reporting rapid business impact. Four things need to be covered:

1. A root-cause analysis approach
2. A short-term remediation plan to stabilize the customer
3. Communications to executives
4. A plan to prevent recurrence


💡 Unlike Topic 1, the handout doesn't repeat "format is your decision" for Topic 2 explicitly, but per the format decision 
already made, this doc (plus the RCA decision tree, built separately in Miro given how dense it is) is the chosen format for this topic.

---

## 1. Root-Cause Analysis Approach

### Framing

https://miro.com/app/live-embed/uXjVHnle7k8=/?embedMode=view_only_without_ui&moveToViewport=-90%2C1288%2C3152%2C1997&embedId=963579768507

The key diagnostic question for "intermittent + peak traffic" is whether this is a **capacity/limit ceiling** being hit 
or a genuine **performance bug**, the two look identical to end users but need completely different fixes. Three possible 
shapes of the problem:

- **Capacity ceiling problem**: incoming query volume exceeds the org's licensed QPS/QPM allowance, Coveo returns `429 Too Many Requests`. Shows up as **over-limit events** on the **System Performance page's Queries tab**, correlated with the outage windows. *(Verified against Coveo's own "Monitor system performance" documentation — over-limit events are exactly the term Coveo uses for this.)*
- **Functional bug**: the request reaches Coveo and gets a response, but something in the platform configuration (an ML pipeline condition firing on every query, an overly broad Content Security rule) adds excessive cost or wrong behavior.
- **Client-side problem**: the request never reaches Coveo successfully — it fails or times out earlier, in the browser, the Atomic/Headless integration, or the network/CDN layer.

**Decision tree, not a checklist** — the panel should see you reason about which branch to test first: rate-limit events, since it's the cheapest check and matches "intermittent under peak" almost exactly.

### Step 1: Confirm the shape of the problem

**Check**: System Performance page → **Queries tab**. Look for rate-limit / over-limit events correlated with the outage windows.
`https://platform.cloud.coveo.com/admin/#/laurapokemonchallengemcfix5o4/organization/system-performance/`

- If they spike exactly when the platform fails → capacity ceiling problem, not a functional bug. → **Step 2**.
- If they don't correlate at all → shifts toward client-side or query-pipeline issues instead. → **Step 3**.

### Step 2: What is driving the query volume? *(if Step 1 = Yes)*

**Bot traffic suspected**
Reference: `docs.coveo.com/en/pbrf2018/coveo-analytics/about-bot-traffic`
Check **Analytics → Data Health** and **Reports**:
- Dashboard → Summary → **Activity** tab: **Search Event Count** — a bot burst shows as a sharp, isolated spike with no matching Unique Visits rise.
- **Relevancy** tab: **Click-Through Over Time** — Coveo colors it green (>60%), black (40–60%), red (<40%). Bots search but rarely click, so clickthrough crashes toward/below red during the suspicious window.
- **Content Gaps** tab: repeated no-result/no-click queries piling up.

**Organic growth suspected**
Real users generating more traffic than before, creeping past the org's contracted entitlement — not an attack or bug.
- Public/anonymous interfaces (like the Pokémon Atomic page, same Anonymous Search key model) run on a **queries per month (QPM)** quota, reset on the 1st.
- Authenticated interfaces use per-user entitlements with no hard query cap.
- Check: **License & Usage** page (Quotas tab) + **System Performance** page (Queries tab) — compare current QPS usage against quota, cross-check the trend. A steady, weeks-long climb toward the QPS ceiling confirms organic growth; a sudden spike with no matching site activity points back to bot traffic instead.
- Docs: "Review your license and usage" (`docs.coveo.com/en/q2ik0227`), "Monitor search consumption" (`docs.coveo.com/en/1855`), "Review organization settings and limits" (`docs.coveo.com/en/1562`).

### Step 3: Causes beyond rate limits *(if Step 1 = No)*

**A) Coveo ML model overhead suspected**
- Coveo Platform → **Query Pipelines** → pipeline's **Overview** tab → **Performance** (query response time) → **Machine Learning** tab (model + condition).
- Root cause: a Coveo ML model (RGA, ART, QS, SE, or similar) set to a broad condition firing on nearly every query, elevating average pipeline response time.
- Coveo ML model overhead suspected" still doesn't state that RGA's Query is not empty condition is broad by design (mandatory for the model), not a misconfiguration.

**B) Quota suspected**

- Check: **License & Usage** page: look at **Generative Queries per Month (GQPM)** and the equivalent recommendation-query entitlement.
- Symptom that distinguishes this from (A): the generative answer *specifically* fails or errors while core search results 
keep working, this is a different signature than pipeline-wide latency.
- Root cause: GQPM (or the recommendation-query entitlement) exhausted for the month.

**Client-side / network suspected**

Suggested subsection:
- Check: browser Developer Tools, **Console** tab (JS errors) and **Network** tab (blocked or failed requests to the 
Coveo Search API, CORS or CDN failures).
- Root cause: client-side or network issue between the search UI (Atomic/Headless) and Coveo's Search API, so the request
never lands on Coveo's side at all.

**Origin/source infrastructure suspected**

- Sources page in Coveo, Activity panel for that source and note the exact timestamp of the last crawl
- Separately, check the customer's own server and API monitoring, which lives entirely outside Coveo, for any spike in 
load or errors. 
- Correlate the two: does the crawl timestamp line up with that spike in load or errors on the customer's origin systems? 
If they overlap, that confirms the origin servers were straining under the crawl and real user traffic at the same time. 
So the root cause is origin infrastructure strain, not a Coveo configuration issue.

---

## 2. Short-Term Remediation Plan

| Root cause | Coveo documentation | Immediate stabilization step                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
|---|---|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Bot traffic** driving rate-limit ceiling | ["Handling bot traffic"](https://docs.coveo.com/en/pbrf2018/) | Every API key has an editable deny-list of IPs in the Admin Console. If the bot traffic is coming from a narrow, identifiable IP range, add it there. But Coveo's own docs caution this only helps against unsophisticated bots; real bot traffic usually rotates across large IP pools specifically to dodge this kind of block, so it's not a substitute for the WAF/proxy fix.                                                                                                                                                                                                                    |
| **Organic growth** exceeding QPM | ["Review your license and usage"](https://docs.coveo.com/en/q2ik0227/), ["Quota or entitlement vs. limit"](https://docs.coveo.com/en/q39f0167/) | Temporary QPM increase from Coveo Support / Account team, because there's no self-service toggle for the customer to do this in the Admin Console.                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| **ML model overhead** on the query pipeline | ["Associate/Edit/Dissociate an RGA model with a query pipeline"](https://docs.coveo.com/en/nb6a0104/), ["Manage model associations with query pipelines"](https://docs.coveo.com/en/2816/) | Fully dissociate the RGA model from the pipeline as a clean stopgap.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **GQPM/RQPM quota exhausted** | ["Generative queries per month (GQPM)"](https://docs.coveo.com/en/nc5e0379/), ["Quota or entitlement vs. limit"](https://docs.coveo.com/en/q39f0167/) | Fully dissociate the running model from the pipeline as a clean stopgap. Disabling RGA on the affected pipeline stops the generative call from erroring out while core search keeps returning results.                                                                                                                                                                                                                                                                                                                                                                                               |
| **Client-side/network** issue | ["Troubleshoot query error codes"](https://docs.coveo.com/en/1471/) | Check the browser's Network tab to see if the request reaches Coveo at all (one independently doable step). If it does and errors, the linked doc decodes the code; if it doesn't (CORS, blocked, timeout), that's the frontend. From there it's coordination, not configuration: loop in the customer's web-dev/IT team immediately, since the fix is theirs to make. If a recent frontend change triggered the outage, recommend rolling back to the last known-good build and purging the CDN cache rather than debugging live, standard on their own CI/CD, and something they execute, not you. |
| **Origin/source infrastructure** strain | ["Manage sources"](https://docs.coveo.com/en/3390/), ["Schedule a source update"](https://docs.coveo.com/en/1933/), ["About crawling speed"](https://docs.coveo.com/en/2078/) | Pause or reschedule the crawl to off-peak hours, this is Coveo's own stated leading practice.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |

---

## 3. Communications to Executives

### Part A: Immediate acknowledgment (before root cause is known)

Parallel:
- **Customer-facing**: Minimal conent + immediate reaction message (secure trust without stating something we do not know yet)
> *We're aware that search on [customer]'s site is experiencing intermittent slowdowns during peak traffic, and our 
> team is actively investigating the cause. No customer data has been affected. We'll follow up with a status update as 
> soon as we know more." 

- **Internal escalation**: The account team / internal escalation chain is notified. 

### Part B: Root cause found, remediation applied

**Framework: BLUF (Bottom Line Up Front)** Lead with resolution status, not the problem, because that's what an executive audience needs first.

> *"We've identified the cause of the intermittent slowdowns on [customer]'s search and expect full stability restored by [time]. No customer data has been affected. [One plain-language sentence naming the root cause]. We'll follow up with a full incident summary within 24 hours."*


### Part C: Exec conversation, after short-term remediation with prevention strategy (within 24h)

**Framework: Matt Abrahams' "What? So What? Now What?"** (Stanford GSB, *Think Fast, Talk Smart*). Since this is a spoken, unscripted conversation, so it needs a structure that generates the right content live rather than a fixed script.

- **What?** Plain-language statement of the root cause, translated out of Coveo/technical jargon.
- **So What?** Why it mattered to this specific customer's business.
- **Now What?** The concrete prevention steps from Section 4.

This is a technique already practiced regularly for these conversations — used here as the deliberate live counterpart to the written BLUF update, not a new framework introduced just for this presentation. It also closes the loop on the 24-hour promise made in Part B.

💡 Scope decision: exact ownership of who sends each message (this candidate, account exec, CSM, on-call) is 
deliberately left out of this doc. Without visibility into Coveo's actual internal escalation structure, naming a 
specific owner would be a guess dressed up as an answer, not a real recommendation.

---

## 4. Plan to Prevent Recurrence

| Root cause | Coveo documentation | Long-term prevention                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
|---|---|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Bot traffic** driving rate-limit ceiling | ["About bot traffic"](https://docs.coveo.com/en/pbrf2018/) | Discuss with customer to deploy a **WAF** (Cloudflare, Akamai, AWS WAF) or a **reverse proxy** on the customer's own infrastructure to identify and block bot traffic before it reaches Coveo.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Organic growth** exceeding QPM | ["Review your license and usage"](https://docs.coveo.com/en/q2ik0227/)  | Two complementary fixes: (1) Right-size the QPM entitlement with the account team and set up proactive usage alerting on the License & Usage page. (Verified: Coveo's own docs frame this monitoring as how you prevent disruptions, and recommend contacting your rep once usage regularly nears the entitlement.) (2) Build a CDN edge rule or reverse proxy on the customer's infrastructure to cap active search sessions and show overflow visitors a "come back soon" message, customer-side, not a Coveo feature, so Coveo never sees the turned-away requests. Like the WAF above, it takes real build time, which is why it belongs here and not in the short-term plan. |
| **ML model overhead** on the query pipeline | ["Manage model associations with query pipelines"](https://docs.coveo.com/en/2816/) | Use Coveo's built-in **A/B-testing** mechanism for model associations to permanently route only a chosen percentage of traffic to the model instead of 100%, comparing pipeline performance with and without it before scaling back up.                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| **GQPM/RQPM quota exhausted** | ["Quota or entitlement vs. limit"](https://docs.coveo.com/en/q39f0167/) | Discuss a permanent GQPM/RQPM increase with the customer's account team. *(Note: unlike a hard-capped platform Limit, RQPM is a contractual entitlement/quota, so per Coveo's own docs, it can potentially be increased on request once the sustained need is established.)*                                                                                                                                                                                                                                                                                                                                                                                                      |
| **Client-side/network** issue | ["Evaluate your implementation before going live (checklist)"](https://docs.coveo.com/en/3012/) | Discuss with customers web-dev team: Pre-deployment validation of frontend changes against the live Coveo integration (a staging check before anything reaches production), plus ongoing client-side monitoring: Synthetic checks or real-user monitoring on the search UI itself, so a bad deploy is caught before or immediately after release, not only once the customer reports an outage.                                                                                                                                                                                                                                                                                   |
| **Origin/source infrastructure** strain | ["Manage sources"](https://docs.coveo.com/en/3390/), ["Schedule a source update"](https://docs.coveo.com/en/1933/) | Permanently reschedule the crawl's cadence to off-peak hours.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

---
