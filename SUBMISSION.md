# Submission

- **Name:** Mohammed Yusuf
- **Date submitted:** 14/11/2026
- **Repository link:** https://github.com/yusufshimer/pse-support-lab
- **Total time spent (real time):** about 6 hrs (3:00 - 9:00)

> Replies are drafts on the **exercise clock** (Monday 16 November 2026, Asia/Dubai). Times in the investigation logs are exercise-clock times. The real times are in the time log at the end.
> Evidence note: the first `make load` run (Incidents 1–5 reproduced, Prometheus graphs, restart and rate-limit counts) was done before I ran `make reset`. The batches shown in the portal screenshots come from the second run, after the reset. Both runs gave the same results.

- **Items I fully investigated, and why I chose them:** Incidents 2, 1, 4 and 3. Incident 2 is a production-wide P1 with a Wednesday go-live. Incident 1 is a month-end customer at risk of creating duplicates. Incident 4 turned out to affect every sender, not just Sahara. Incident 3 is a clear product defect with an auditor waiting. Incident 5 is a file-format issue on the customer's side with a quick fix, so it is last.

**Order I worked them:** 2 → 1 → 4 → 3 → 5. I raised Incident 4 from P2 to P1 once the logs showed it was not limited to Sahara.

---

## Incident 1 — `inbox/01-oasis-builders-upload-stuck.md`

### Acknowledgement

- **Sent at (exercise clock):** Mon 16 Nov 2026, 09:20 (received 09:05)
- **Reply:**

> Subject: Re: URGENT - monthly invoice upload stuck on "Processing"
>
> Hello Rashid,
>
> Thank you for writing. I understand your November file stayed on "Processing", you uploaded it again, and some invoices now say the document number already exists. With month-end close this week, I know you need a clear answer today.
>
> I'm Mohammed from Invoicely Support and I'm checking exactly which of your invoices were created and sent. **Please don't upload the file again for now**, because that is how those "already exists" messages appear. I'll write back by **11:30** with the status of each invoice and the safe way to send the rest.
>
> Regards,
> Mohammed Yusuf, Product Support Engineer

### Priority and justification

- **Priority:** P2
- **Why:** A subset of this customer's invoices is affected and they are at month-end close, but the portal is up and other customers can upload. Raise to P1 if no Oasis invoice reaches the ASP.
- **SLA deadlines I am working to:** first response 10:05 (done 09:20); next update 11:30, then every 2 hours; resolution 17:05.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 10:35 | Read the email and the sample file | The file has 600 rows making 8 invoices (5 distinct buyers) |
| 10:40 | `make load` (loadgen replays Oasis straight to the API) | Oasis upload took 121.8 s and finished Completed, 8 Accepted (`evidence/load-output.txt`) |
| 10:50 | Uploaded the file in the browser as Rashid did | After about 60 s the portal showed "504 Gateway Timeout. Please try again." (`evidence/01-oasis-504.png`) |
| 11:00 | Waited 2–3 minutes, then opened Batches without uploading again | A batch appeared by itself: 600 rows / 8 invoices, **Accepted 8**, processing time 121.6 s (`evidence/01-oasis-batch-created.png`). The upload had kept running after the error |
| 11:05 | Checked web logs | One 504 from the portal proxy (`evidence/01-proxy-timeout.txt`) |
| 11:10 | Uploaded the same file again (first run, before reset) and queried the database | All 8 invoices Rejected by the portal with AE-BR-09: "Document number 'OBS-2611-00x' already exists for this seller with status 'Accepted'." (`evidence/01-oasis-duplicate.txt`) |
| 11:15 | Read the code | The buyer lookup runs on every row, one after the other; the proxy timeout is 60 s; the API keeps working after the browser gives up |
| 11:20 | Dead end: looked at whether the ASP was slow at submitting | ASP responses took a fraction of a second in the logs. The delay is in validation, not delivery |

### Root cause

- **What happened:** The November file has 600 rows. For every row the portal asks the Peppol directory whether the buyer is registered, one request at a time, so the import takes about 2 minutes (121.6 s). The portal's gateway gives up after 60 s and shows a 504, but the API keeps processing and creates all 8 invoices. Re-uploading then finds those invoice numbers already present (status Accepted) and rejects the copies with AE-BR-09. Nothing was issued twice.
- **Where:** `services/api/internal/validate/validate.go:166` (`v.dir.IsRegistered` called once per row); `web/app/api/[...path]/route.ts:6` (`UPSTREAM_TIMEOUT_MS = 60_000`); `services/api/internal/importer/importer.go:38` (work runs on `context.Background()`, so it survives the dropped connection).
- **Why it happens:** Lookups are per row (600) and not per unique buyer (5), they are not cached, and the upload request waits for the whole import. The duplicate check (`store.ExistingStatus`) correctly ignores Rejected invoices but counts every other status.
- **Evidence:** `evidence/01-oasis-504.png`, `01-oasis-batch-created.png`, `01-proxy-timeout.txt`, `01-oasis-import.txt`, `01-oasis-duplicate.txt`, `load-output.txt`.
- **Confidence and open questions:** High. The 504 after about 60 s and the batch appearing afterwards were both observed. The per-row lookup as the reason for the 2 minutes comes from reading the code and from the ASP mock's fixed 200 ms lookup delay (600 × 0.2 s ≈ 120 s); I did not time the lookups separately.

### Customer resolution reply OR escalation note

- **Which one and why:** Customer reply. The invoices were created, so there is nothing to repair. I would also pass the slowness to engineering as an improvement suggestion, not an incident.

> Hello Rashid,
>
> Here is what happened. Your file has 600 lines, and the portal checks the buyer on each line one at a time, so it needs about two minutes. The screen gave up waiting after one minute and showed an error, but the portal kept working in the background and **did create your 8 invoices from the first upload**, and all 8 show Accepted. Your later uploads found the same invoice numbers, which is why you saw "already exists". Nothing was issued twice.
>
> 1. Which invoices were sent: OBS-2611-001 to OBS-2611-008, all Accepted by the network. You can see them in the first Oasis batch in the portal.
> 2. What to do now: nothing. You don't need to upload the file again. Please don't upload it again, because it will only create more "already exists" messages.
>
> I'm sorry for the confusion. I've asked our engineers to make large files process faster. I'll check your batch again at 14:00 and confirm everything is still Accepted.
>
> Regards, Mohammed

### Workaround offered

Don't re-upload. For future large months, split the file into smaller files (for example 100–150 rows each) so each upload finishes within a minute.

### Prevention ideas

Look up each unique buyer once per import (or cache lookups). Process imports in the background (return "received" immediately and show progress). Raise the proxy timeout for uploads. Warn users "still processing, do not upload again". Alert when import time passes 30 s.

---

## Incident 2 — `inbox/02-alert-api-restarted-marina-bay.md`

### Acknowledgement

- **Sent at (exercise clock):** 09:55 (alert 09:42, Priya's forward 09:50)
- **Sent to:** Priya Nair, cc support-oncall
- **Reply:**

> Subject: Re: Fwd: [FIRING:1] Invoicely API restarted
>
> Hi Priya,
>
> Thanks for flagging this. I believe the alert and what your customers see (502 errors on upload, invoices waiting in "Validated") are the same problem: the API restarting. I own this and I'm confirming the cause now.
>
> Until I tell you otherwise, please ask customers not to re-upload, and to use "Retry submission" on invoices that stay in Validated. I'll update you by **10:30**, and again before your afternoon call with Marina Bay so you can say where we are for Wednesday.
>
> Mohammed

### Priority and justification

- **Priority:** P1
- **Why:** The production API is restarting and customers cannot reliably get invoices to the ASP. Invoices are stuck. Marina Bay goes live on Wednesday. The policy says to page engineering once impact is confirmed, without waiting for the full root cause.
- **SLA deadlines I am working to:** first response 10:20 (done 09:55); updates every 30–60 min; resolution 13:42.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 10:00 | Baseline: uploaded the clean Desert Rose file | 5 of 5 Accepted in 2.5 s (normal) |
| 10:03 | `make load` (240 peak uploads, 8 at a time) | 22 uploads failed: 15 × `EOF`, 4 × connection refused, 2 × DNS lookup of `api` failed, 1 × read error (`evidence/load-output.txt`). The API was going down during the load |
| 10:06 | `grep 'api starting'` in the API logs | Starts at 09:20:35 (boot), then restarts at **10:42:20, 10:42:45 and 10:48:39** (UTC). 3 restarts within about 15 minutes (`evidence/02-api-starts.txt`) |
| 10:09 | `docker inspect … OOMKilled` | `OOMKilled=false`, `restarts=2` at the time. **Dead end / misleading:** this flag describes only the current run and resets after a restart |
| 10:12 | Searched logs for `panic|fatal|out of memory|goroutine` | Nothing. The logs just stop at 10:42:19 and the process starts again at 10:42:20 |
| 10:15 | `docker events` filtered for oom/die | Nothing printed. **Dead end:** Docker Desktop does not keep that history |
| 10:18 | Prometheus: `container_memory_working_set_bytes` for the api container | Memory about 35–40 MB until 10:38, then up to about 130 MB at 10:41–10:42, right at the 128 MB limit, with drops at the restarts (`evidence/02-prometheus-memory.png`) |
| 10:21 | Prometheus: `container_oom_events_total` for api | **4** (`evidence/02-prometheus-oom.png`). Direct proof that the kernel killed the container for memory |
| 10:24 | Read the code and compose file | `docker-compose.yml:60` sets `mem_limit: 128m`. `importer/sources.go:31-34` keeps every upload in a map and never removes it |
| 10:27 | Database: invoice and import status counts | Parsed 33, Validated 51; 9 imports stuck in Processing. The oldest unfinished invoices were updated at 10:42:16, four seconds before the first restart (`evidence/04-submitted-split.txt`) |

### Root cause

- **What happened:** The API container ran out of memory and was killed and restarted three times (10:42:20, 10:42:45 and 10:48:39 UTC). While it was down the portal returned 502/connection errors. Each restart also wiped the in-memory submission queue, so work that was in progress was lost: 9 imports stayed in Processing and 33 invoices in Parsed and 51 in Validated never moved on.
- **Where:** `docker-compose.yml:60` (128 MB limit); `services/api/internal/importer/sources.go:31-34` (`SourceCache.Put`, no eviction); `services/api/internal/submit/submit.go:65` (in-memory queue, nothing re-queues unfinished invoices at startup).
- **Why it happens:** Every uploaded file (raw bytes and the parsed workbook) is kept in memory for the life of the process, so memory grows with traffic until it hits the limit. The third restart at 10:48:39 happened after the load had stopped and memory was still high, which is consistent with memory not being released.
- **Evidence:** `evidence/02-prometheus-oom.png` (OOM events = 4), `02-prometheus-memory.png`, `02-api-starts.txt`, `02-api-restarts.txt`, `04-submitted-split.txt`, `load-output.txt`.
- **Confidence and open questions:** High that the container was OOM-killed (4 OOM events, memory at the limit). Medium that the unbounded source cache is the main cause: the code shows nothing is ever deleted, but I did not profile memory or test a fix. Open: whether the 128 MB limit alone is too small for expected traffic. `restarts=3` matches the alert; the OOM counter (4) counts kernel events, so it can differ from the restart count.

### Customer resolution reply OR escalation note

- **Which one and why:** Escalation to L3. A code and infrastructure defect, a P1, and support cannot repair the stuck imports through the product.

### Escalation: API container OOM-killed repeatedly under load; imports and invoices stuck

**Priority:** P1: production API restarting, uploads failing, invoices stuck; Marina Bay Hospitality goes live Wed 18 Nov.
**Customer / seller:** All customers; reported by Priya Nair (CSM) and Grafana alert.
**Reported by / at:** Grafana Alerting 09:42; Priya Nair 09:50.
**Affected:** 9 imports stuck in Processing, 33 invoices in Parsed, 51 in Validated; 22 of 240 upload requests failed in the load test; all senders uploading around the restarts.

**Symptoms (customer view):** 502/connection errors on upload; invoices stay in Validated.

**Evidence:**
- Metrics: Prometheus `container_oom_events_total{…service="api"}` = 4; `container_memory_working_set_bytes` reaches about 130 MB against a 128 MB limit (`evidence/02-prometheus-*.png`).
- Logs: `api starting` at 10:42:20, 10:42:45 and 10:48:39 UTC; no panic before any of them (`evidence/02-api-starts.txt`).
- Database: Parsed 33, Validated 51; imports Processing 9, Completed 233.
- Load result: 218 ok, 22 failed (`evidence/load-output.txt`).

**Steps to reproduce:** `make load`; watch the api container's memory and `docker compose logs api | grep 'api starting'`.

**What I have already tried / ruled out:** No panic or fatal error in the logs. `docker inspect` showed `OOMKilled=false`, but that flag is reset on restart, so Prometheus was the reliable signal. The database and ASP were healthy.

**Suspected cause (with confidence):** `SourceCache` never evicts (`importer/sources.go:31-34`) against a 128 MB limit (`docker-compose.yml:60`). Medium-high.

**Workaround in place:** Yes for stuck Validated invoices: Retry submission / "Retry stuck invoices". Not for Parsed invoices or Processing imports; Retry only accepts Validated or Failed.

**Ask:** (1) Confirm the cause. (2) Add a size limit or expiry to the source cache. (3) Re-queue unfinished invoices at startup. (4) Raise the memory limit short-term. (5) Repair the 9 stuck imports. I need a mitigation before end of day Tuesday 17 Nov so Marina Bay can be tested before Wednesday.
**Next customer update due:** 10:30.

> **Update to Priya (10:30):** Cause found: the API is running out of memory when many files are uploaded, so it restarts, and invoices that were in progress get stuck. Invoices stuck in "Validated" can be retried from the portal, and I'm doing that for the ones I can see. Engineering has the details; I'm asking for a mitigation before Tuesday evening so we can test Marina Bay's volume before they go live, and I'll confirm what they commit to. Next update at 12:00 and before your call.

### Workaround offered

Retry stuck invoices from the batch page; ask customers not to re-upload; restart the container if it is down.

### Prevention ideas

Evict or cap the source cache. Recover unfinished work at startup (or use a durable queue). Alert on memory use well before the limit. Load-test Marina Bay's expected volume before go-live.

---

## Incident 3 — `inbox/03-al-noor-fuel-br-co-15.md`

### Acknowledgement

- **Sent at (exercise clock):** 10:45 (received 10:15)
- **Reply:**

> Dear Fatima,
>
> Thank you for your message. Some of your November fuel invoices were rejected with rule BR-CO-15, which checks that an invoice's total with VAT equals the total without VAT plus the VAT. I understand your auditor needs a written explanation.
>
> I'm comparing the rejected invoices with the accepted ones to see why only some are affected. I'll reply with an explanation and next steps by **14:00** today. Please don't change your figures yet.
>
> Regards, Mohammed Yusuf, Invoicely Support

### Priority and justification

- **Priority:** P2
- **Why:** A subset of invoices is rejected by the ASP and an external auditor is waiting. No workaround is confirmed.
- **SLA deadlines I am working to:** first response 11:15 (done 10:45); update every 2–4 hours; resolution 18:00.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 12:45 | Replayed the Al Noor file (`make load`) | 6 invoices: **Accepted 3, Rejected 3** |
| 12:55 | Opened a rejected invoice in the portal | ANF-2611-0042: ASP rule BR-CO-15, "Got 15022.42, expected 15022.43" (`evidence/03-brco15.png`) |
| 13:05 | Checked the totals in the file with a short script | Net 14,307.08 + VAT 715.35 = 15,022.43, but the portal's total with VAT is 15,022.42. Invoices ANF-2611-0044 and -0045 show the same one-fil gap |
| 13:15 | Compared accepted and rejected invoices | The rejected invoices have unit prices with three decimals (2.917, 2.585, 2.835). The accepted ones don't |
| 13:20 | Read the code | `pint.go:171` computes the total with VAT from quantity × price × 1.05, not from net + VAT |
| 13:25 | Dead end: customer's arithmetic | Their line amounts are within the portal's 0.01 tolerance (AE-BR-08), so the file is valid by the portal's own rules |

### Root cause

- **What happened:** The portal adds up the "total with VAT" in a different way from the other two totals, so on some invoices it comes out one fil lower than net + VAT. The ASP checks net + VAT = total and rejects the invoice.
- **Where:** `services/api/internal/pint/pint.go:171`: `gross += l.Quantity * l.UnitPrice * (1 + l.VATRate/100)`.
- **Why it happens:** The net total and the VAT come from the rounded line amounts the customer entered, but the total with VAT is recalculated from unrounded quantity × price. With prices like 2.917 per litre the two sets of numbers drift apart by a fil.
- **Evidence:** `evidence/03-brco15.png` ("Got 15022.42, expected 15022.43"); batch showing Accepted 3 / Rejected 3.
- **Confidence and open questions:** High. Open: whether the fix should simply use net + VAT (what the rule requires), and whether other customers with 3-decimal prices are affected.

### Customer resolution reply OR escalation note

- **Which one and why:** Escalation to L3: a product defect, which support cannot fix or work around safely. The customer did nothing wrong, and the reply says so.

### Escalation: BR-CO-15 rejections caused by how the portal computes the VAT-inclusive total

**Priority:** P2: a subset of invoices rejected; auditor review in progress.
**Customer / seller:** Al Noor Fuel & Lubricants LLC.
**Reported by / at:** Fatima Khoury, email, 10:15.
**Affected:** 3 of 6 invoices in the November file (ANF-2611-0042, -0044, -0045); possibly any seller using 3-decimal unit prices.

**Symptoms (customer view):** Some invoices rejected by the ASP with BR-CO-15.

**Evidence:**
- ASP response: 422, BR-CO-15: "Got 15022.42, expected 15022.43" for ANF-2611-0042 (`evidence/03-brco15.png`).
- Worked numbers: net 14,307.08 + VAT 715.35 = 15,022.43; portal sent 15,022.42.
- Code: `pint.go:171`.

**Steps to reproduce:** `make replay CUSTOMER=al-noor`; open the batch and a Rejected invoice.

**What I have already tried / ruled out:** The customer's figures are correct and pass the portal rules.

**Suspected cause (with confidence):** `pint.go:171`. High.

**Workaround in place:** No safe one on the customer side.

**Ask:** Make the total with VAT equal net + VAT (in fils), release the fix, and tell me when the rejected invoices can be re-uploaded (the same invoice numbers are fine, since rejected numbers are not used up). Needed today.
**Next customer update due:** 14:00.

> **Customer update (14:00):** Fatima, the rejection comes from how our system added up the VAT-inclusive total on invoices with fuel prices at three decimals: it came out one fil lower than it should (15,022.42 instead of 15,022.43 on ANF-2611-0042). Your figures are correct and nothing needs changing in your file. For your auditor: rule BR-CO-15 requires total with VAT = total without VAT + VAT, and our calculation did not meet it on these invoices. Our engineers have the details. Once the fix is released I'll ask you to re-upload only the rejected invoices (the same invoice numbers are fine), and I'll write by 17:00 with the status.

### Workaround offered

None that I would recommend to the customer. Once the fix is released, re-upload the three rejected invoices.

### Prevention ideas

Calculate totals from the rounded line amounts and VAT, and check BR-CO-15 in the portal before submitting. Add test files with 3-decimal prices.

---

## Incident 4 — `inbox/04-bluepeak-sahara-not-received.md`

### Acknowledgement

- **Sent at (exercise clock):** 11:45 (received 11:30)
- **Reply:**

> Hi Daniel,
>
> Thanks for the clear report. I understand the 40 Sahara Medical invoices show "Submitted", many have no ASP reference, and the hospital hasn't received most of them. I'm checking where each invoice is now.
>
> **Please hold off resending the batch** until I confirm, so we don't create duplicates at the hospital. I'll reply with the status of all 40 and what to do next by **13:30**.
>
> Mohammed Yusuf, Invoicely Support

### Priority and justification

- **Priority:** Started as P2. **Raised to P1** after the logs showed it affects every sender that submits a burst, not only Sahara (see log).
- **Why:** Invoices look submitted but never reach the network, silently, and the hospital's payment run is this week. Once it was clear that other customers were hit too, the "invoices are not reaching the ASP" test for P1 applies.
- **SLA deadlines I am working to:** P2 first response 12:30 (done 11:45); after raising to P1: updates every 30–60 min, resolution 4 h from the raise. I would tell Daniel and Priya about the change and why.

### Investigation log

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 11:50 | Replayed the Sahara file | 40 invoices created. **Accepted 8, Submitted 32** (`evidence/load-output.txt`) |
| 12:00 | Opened the Sahara batch | Invoice SMS-1116-009 shows Submitted with no ASP reference (–) (`evidence/04-sahara-submitted.png`) |
| 12:10 | Read the ASP mock | Limit per sender: a burst of 8, then 2 per second (`asp-mock/main.go:36-37`); beyond that it answers HTTP 429 |
| 12:20 | Searched the ASP logs for "rate limit exceeded" | **753** throttled events (`evidence/04-rate-limit-count.txt`, sample lines in `04-rate-limit.txt`) |
| 12:30 | API logs | Lines like `asp response … status_code":429` for invoices from other customers too |
| 12:35 | Database: Submitted invoices by batch | 32 from the Sahara batch and **724 from other senders** during the peak load (`evidence/04-submitted-split.txt`); total 756, close to the 753 throttled events |
| 12:40 | Read `submit.go` | The `switch` handles 200, 422 and 500/502/503/504, but there is no case for 429, so the invoice stays Submitted and is never retried |
| 12:45 | Dead end: wrong or unregistered buyer | The Sahara file has only 2 distinct buyers and none were rejected by the ASP |

### Root cause

- **What happened:** The portal sent the invoices to the ASP faster than the ASP's fair-use limit allows (8 at once, then 2 per second). The ASP answered "too many requests" (HTTP 429) for the rest. The portal doesn't handle that answer, so those invoices stay "Submitted" with no ASP reference and are never retried. The connector's 201 was correct; the failure is after the hand-off.
- **Where:** `services/api/internal/submit/submit.go:105` (`submit`) and `:133` (`switch resp.StatusCode` has no 429 case; `Retry-After` is ignored); the status is set to Submitted before sending.
- **Why it happens:** Four parallel workers send at full speed, and an unhandled status code leaves the invoice in its in-flight state forever.
- **Evidence:** `evidence/04-sahara-submitted.png`, `04-rate-limit-count.txt` (753 throttled), `04-submitted-split.txt` (32 + 724 stuck), `load-output.txt` (Submitted 32 / Accepted 8).
- **Confidence and open questions:** High. The numbers are close (753 throttled vs 756 stuck). The 3 extra stuck invoices could have been mid-send when the API restarted, but I did not verify that. Open: whether any "Submitted" invoice actually reached the hospital (to be checked with the ASP's document list for the sender).

### Customer resolution reply OR escalation note

- **Which one and why:** Escalation to L3. A product defect. I clicked "Retry stuck invoices" on the Sahara batch. The 32 invoices stayed in Submitted, because Retry only works on Validated or Failed invoices. So support cannot fix this through the portal.

### Escalation: invoices stuck in "Submitted" after the ASP returns 429 (rate limit)

**Priority:** P1: affects every sender that sends more than about 8 invoices in a burst.
**Customer / seller:** First reported by Sahara Medical Supplies FZE via Bluepeak ERP.
**Reported by / at:** Daniel Okafor, email, 11:30.
**Affected:** Sahara: 32 of 40 invoices. Others: 724 invoices from other senders in the peak test. Total 756 in Submitted.

**Symptoms (customer view):** Status "Submitted", empty ASP reference, buyer did not receive them; no error shown.

**Evidence:**
- ASP: 753 "rate limit exceeded" events (HTTP 429 with `Retry-After`).
- Database: 756 invoices in Submitted, 32 from the Sahara batch.
- Code: no 429 case at `submit.go:133`.
- Portal: `evidence/04-sahara-submitted.png`.

**Steps to reproduce:** `make replay CUSTOMER=sahara`; open the batch.

**What I have already tried / ruled out:** Not a buyer or validation rejection; not an ASP outage.

**Suspected cause (with confidence):** Missing 429 handling in `submit.go`. High.

**Workaround in place:** No. "Retry stuck invoices" did not move the 32 Submitted invoices, so support cannot repair them from the portal.

**Ask:** (1) Handle 429: wait for `Retry-After` and retry, or mark the invoice Failed so Retry works. (2) Limit sending to the ASP's rate per sender. (3) Repair the stuck invoices (resend only those without an ASP message id). Today, because of Sahara's payment run.
**Next customer update due:** 13:30.

> **Customer update (13:30):** Daniel, thank you for holding off. Invoices were not lost: our system sent them faster than the network provider allows, the provider asked it to slow down, and our system did not retry those, leaving them "Submitted" without a reference. 8 of the 40 went through; 32 did not. Your 201 response was correct, and nothing is wrong on your connector. **Please still don't resend the batch**, because that would duplicate the 8 that were delivered. I've given engineering the exact list so only the missing ones are resent. I'll update you by 16:00 and confirm delivery so you can tell Sahara and the hospital.

### Workaround offered

None for the customer. Support, after engineering repairs the invoices: check every invoice has an ASP reference and compare against the ASP's list for the sender.

### Prevention ideas

Handle every unexpected status code explicitly (default → Failed). Respect `Retry-After` and rate-limit per sender. Alert on invoices in Submitted for more than 15 minutes. Chart 429s in Grafana.

---

## Incident 5 — `inbox/05-ghaf-cafe-date-error.md`

### Acknowledgement

- **Sent at (exercise clock):** 12:50 (received 12:10)
- **Reply:**

> Dear Hessa,
>
> Thank you for writing, and I'm sorry for the trouble. I understand your invoices were rejected because of the date format, and your customers are waiting. I can see the message in your file and I'll send you a simple fix by **14:30** today.
>
> Mohammed Yusuf, Invoicely Support

### Priority and justification

- **Priority:** P3
- **Why:** One customer, one file-format issue with a simple fix on her side. Nothing reached the ASP, so nothing was sent twice.
- **SLA deadlines I am working to:** first response 16:10 (done 12:50); update daily; resolution within 3 days.

### Investigation log (short; I did not escalate this one)

| Time (exercise clock) | What I checked / did | What I found |
|---|---|---|
| 13:35 | Replayed the Ghaf file | 4 invoices, **Rejected 4** by the portal (`evidence/05-date-error.png`) |
| 13:45 | Looked at the Errors column | 4, 4, 6 and 2 errors: exactly 2 per source row, so both date columns are affected on every row (confirmed on invoice GTC-118-HU6P: both the issue date and the payment due date show the AE-BR-04 error) |
| 13:50 | Source rows in the batch page | Dates are written as 13/11/2026, 14/11/2026, 15/11/2026 (day/month/year text) |
| 14:00 | Read `sheet.go` `ParseDate` | Accepts only YYYY-MM-DD text or a real Excel date. Day/month text is rejected by rule AE-BR-04 |

### Not escalated. Why

The product is working as documented. The fix is on the customer's side.

> **Reply to Hessa (by 14:30):**
>
> Dear Hessa,
>
> Your date is correct. The 13th of November is exactly what you meant. The system just needs dates in one specific form, and it can't read the day/month form that is typed as text. Both the **issue date** and the **payment due date** columns need changing.
>
> To fix it in two steps: (1) in Excel, select both date columns, and use Format Cells → Date, then re-type the dates (or type them as 2026-11-13); (2) upload the file again. Your invoices were not sent, so nothing is duplicated, and the same invoice numbers can be used again.
>
> If it still doesn't work, send me the file and I'll check it with you. I'll look at your upload again at 16:00 to make sure it went through.
>
> Mohammed

### Workaround offered

Re-type the dates as real Excel dates or as YYYY-MM-DD.

### Prevention ideas

Make the error message say how to fix it ("use 2026-11-13, or format the column as Date"). Accept day/month/year text, or show a warning for it. Add a date example to the template's help text.

---

## Overall

### Prioritisation

I read all five first. Incident 2 went first because it is a production-wide P1 with Marina Bay going live on Wednesday. Incident 1 was second because a customer at month-end could create more duplicates. Incident 4 was third; I started it at P2 and raised it to P1 once the logs showed it affects every sender that sends a burst, not just Sahara. Incident 3 was fourth, and Incident 5 last because it is a simple file fix on the customer's side.

### What I would do differently

I would check Prometheus's OOM counter first instead of trusting the OOMKilled flag from docker inspect, which was misleading because it resets after a restart. I would take screenshots while an error is on screen, and I would keep a timeline as I go instead of rebuilding it later. I would also reproduce each customer's problem on a fresh system before reading the code, so the code is confirming something I have already seen.

### What I learned / what I would need to learn

I had not used Prometheus or cAdvisor before, and I learned that OOM events show up in container_oom_events_total even when docker inspect says OOMKilled=false. I learned how HTTP 429 and rate limits work, and how a missing case in a status-code switch can leave invoices stuck in "Submitted" with no error. I learned how a timeout in one layer (the portal proxy at 60 seconds) can leave work running in another (the API), which is why Oasis's upload failed on screen but still created the invoices. I got more comfortable with Docker Compose, container logs and psql. Next I would study Go memory profiling, Grafana alert rules and the PINT AE specification.

### Time log (real time)

| Date | From–to | What I worked on |
|---|---|---|
| 14 Nov 2026 | 3:00 PM – 3:30 PM | Running the stack with make up, baseline upload (Desert Rose), checking the portal and logs |
| 14 Nov 2026 | 3:30 PM – 4:45 PM | Reading the code, running make load, first investigation of Incident 2 (restarts, OOMKilled, logs) |
| 14 Nov 2026 | 4:45 PM – 6:15 PM | Prometheus queries (memory and OOM counter), database queries, rate-limit counts (Incident 4), Al Noor error (Incident 3) |
| 14 Nov 2026 | 6:15 PM – 7:00 PM | make reset, reproducing the Oasis 504 in the browser (Incident 1) |
| 14 Nov 2026 | 7:00 PM – 9:00 PM | Screenshots for Incidents 3, 4 and 5, writing SUBMISSION.md, pushing to GitHub |

### AI usage disclosure

I used Claude (Anthropic's AI assistant) throughout this exercise. It helped me read the repository code and sample files, suggested hypotheses for each incident, told me which commands and queries to run, helped me interpret my results, and drafted the structure and wording of this document, which I then edited. I ran the lab myself on my own Mac, produced all the evidence in `evidence/` (logs, SQL output, Prometheus graphs, portal screenshots), and checked each hypothesis against the running system. Where the AI's first guesses were incomplete or wrong, I tested them and corrected them. For example, it first suggested that `docker inspect`'s `OOMKilled=false` and the missing `docker events` output made a memory kill unlikely; Prometheus's OOM counter (4) showed otherwise, and I recorded this as a dead end in Incident 2. Likewise, the stuck-invoice count was first thought to be partly restart casualties, but the rate-limit count (753 throttled vs 756 stuck) showed it was mostly the missing 429 handling.
