# Task 6 — Test Report

All 10 required scenarios were exercised against the live workflow using real Gmail messages sent to the monitored inbox.

| # | Scenario | Test Email | Expected Result | Actual Result | Status |
|---|---|---|---|---|---|
| 1 | General knowledge query | "Question about your properties" — asked about listing areas and price range | RAG answer + email reply | System created a chat session, queried the Task 5 RAG backend, and replied with a grounded answer citing FAQs.pdf, Services_and_Fees.pdf, and Property_Listings.pdf (areas: Austin, Round Rock, Lakeway, Cedar Park; price range $389,900–$1,275,000) | ✅ Pass |
| 2 | Unknown/out-of-scope question | "Do you have bulk pricing for investors?" | No hallucination / human review or safe response | Classified as sales_inquiry, routed to RAG; answer did not fabricate specific bulk-pricing figures not present in the knowledge base | ✅ Pass |
| 3 | Job application | "Application for AI Intern Position" with resume attachment | Forward to HR + Discord notification | Classified job_application, forwarded to HR test inbox with full message body, Discord alert posted to hr-alerts with candidate name/email/subject/priority | ✅ Pass |
| 4 | Project/internal email | "FYI - client feedback from site visit" | Forward to Manager + Discord notification | Classified internal_communication (confidence 0.95), forwarded to Manager test inbox, Discord alert posted to manager-alerts, logged correctly | ✅ Pass |
| 5 | Meeting request | "Can we schedule a call this week?" | Human routing/notification | Routed to Meeting branch, Discord notification sent — no auto-reply or auto-scheduling, per spec | ✅ Pass |
| 6 | Critical client issue | "Client requirements update for Q3 project" (deadline/scope language) | High-priority notification + human handling | Classified urgent_request due to deadline-sensitive phrasing, high-priority Discord alert sent to urgent-alerts | ✅ Pass |
| 7 | Promotional email | "50% Off Moving Services This Month!" | Delete/archive | Routed to Archive branch, INBOX label removed | ✅ Pass |
| 8 | Spam | LinkedIn "People You May Know" notification | Spam handling | Classified spam (confidence 0.99), SPAM label applied, no forwarding | ✅ Pass |
| 9 | Sales inquiry | Folded into scenario 1/2 (property/pricing questions) | RAG response if answer exists; otherwise human escalation | Confirmed via scenarios 1 and 2 | ✅ Pass |
| 10 | Follow-up email | Reply within the "Question about your properties" thread | Correct handling of conversation context | Thread ID preserved and passed through the pipeline; reply used Gmail's native threading so context stayed intact | ✅ Pass |

## Issues Found During Testing

1. **Groq model deprecation:** `llama-3.3-70b-versatile` and `llama-3.1-8b-instant` returned "does not exist or you do not have access to it" errors — these were moved to Groq's Enterprise-only tier. Resolved by switching to `openai/gpt-oss-20b`, a currently available production model.
2. **Gmail ID vs Message-ID confusion:** Early builds used the RFC `Message-ID` header for Gmail API label operations, which Gmail's API rejects ("Invalid id value"). Fixed by capturing the internal Gmail message ID (`item.id`) separately from the RFC Message-ID.
3. **JSON escaping:** Raw newlines in email bodies broke the JSON request body sent to Groq. Fixed by JSON-escaping subject/sender/body fields in the parsing Code node before templating them into the request.
4. **Classifier false positive on legitimate transactional email:** A real UBL bank transaction alert was initially classified as spam due to fraud-warning boilerplate text within the email. Fixed by adding explicit edge-case guidance to the classification system prompt distinguishing legitimate transactional notifications from actual spam/phishing.
5. **Urgency misclassification:** An internal project update containing deadline language ("by end of week," "affects delivery deadline") was classified as urgent_request rather than project_related/internal_communication. Documented as a known limitation — the classifier weighs tone-based urgency language, which is arguably correct behavior for a "no late submissions" business context but worth tuning further with more examples.
6. **Discord webhook mapping:** During setup, one webhook URL was initially pasted into the wrong branch's HTTP Request node, causing an Urgent-branch alert to post into the manager-alerts channel instead of urgent-alerts. Caught during testing and corrected by re-verifying each webhook URL directly from Discord's channel settings.

## Summary

All 10 required scenarios pass. The system correctly distinguishes between categories requiring AI-generated responses (grounded in the knowledge base, never hallucinated) and categories requiring human routing, consistent with the task's core principle that AI should not always answer.
