# Pending WhatsApp templates — the full list and how to create each one

> **Version:** 1.1 · **2026-10-22** · **Owner:** Rahul K
>
> **What this answers:** *"Which WhatsApp templates does Kanha still have to get approved at Meta,
> and what exactly do I click and paste to create them?"* — and, from v1.1, *"what is every WhatsApp
> template Kanha has ever had or planned, and what is the Hindi (Devanagari) version of each?"*
> **Part A** = the 42 to submit now (§1–§6). **Part B** = everything else to date (§7). **Part C** = the
> Devanagari set and what the code needs before it is used (§8–§9).
>
> **Verification level — read this first.**
> - **Code-verified 2026-10-22:** the list of templates, their categories, bodies, variable counts and
>   what sends each one. Read from every `whatsapp_template` insert in
>   `kanha-ai/src/main/resources/db/migration` (V2, V26, V46, V49, V102, V140, V141) and from the
>   Java that calls them (`OrderNotificationOrchestrator`, `ComplaintNotificationOrchestrator`,
>   `CustomerReminderService`, `ScheduledMessageService`, `AccountNotificationListener`,
>   `CustomerSecurityNotificationListener`, `StaffAttendanceWhatsapp`, `WhatsappReplyEventListener`).
> - **NOT live-verified:** what Meta says today. I did not query Meta or the production database.
>   The last live reading in these docs is **2026-10-04**: the brand WABA `2041802123192471` held
>   only `kfcsignup` and `hello_world` ([`26`](26-NUMBER-ROLES-AND-ROUTING-PLAN.md) §1.3). Before
>   you start, run the status query in [§5](#5-check-what-is-pending-right-now) — it takes a minute and
>   tells you which rows below are already done.
> - Meta's form layout changes. The *fields* below are stable; button positions are not.
>
> **Companions.** Rules and rejection causes: [`08`](08-TEMPLATES-AND-ASSETS.md) §2–§4 (this doc does
> not repeat them in full). Tracker: [`13`](13-ONBOARDING-TRACKER.md) Phase C.

---

## 1. The short version

- **42 templates to submit now:** 41 seeded in the database as `PENDING`, plus `delivery_on_the_way`
  (§3.8), which the transport module sends but no migration seeds. One more optional family
  (`product_carousel_N`) is not seeded either. Everything else that has ever been named, planned or seen
  at Meta is in §7.
- **None of them can reach a customer outside the 24-hour chat window until Meta approves it** on the
  WABA the number lives on. Until then the app refuses the send and logs the template name and its status
  (`ApprovedTemplateChooser.explainUnavailable()`); it does not crash.
- Templates belong to the **WABA, not the phone number** — create once on the brand WABA
  `2041802123192471` ("Kanha Furniture Company") and every number on it can use them. Templates on the
  old test WABA `1606315164548044` do **not** carry over.
- Nothing to approve inside Kanha: you **submit at Meta**, and the hourly `TemplateSyncService` flips the
  local row to `APPROVED`. Nothing to configure in code.

### Do these in this order

| Wave | What | Count | Why first |
|---|---|---|---|
| **0** | `login_otp` | 1 | Without it staff and customers cannot sign in by WhatsApp code. Strictest format. |
| **1** | Order + payment + quotation | 9 | Sent automatically the moment staff confirm an order, schedule a delivery, etc. |
| **1b** | `delivery_on_the_way` (§3.8) | 1 | Tells the customer the truck has left the gate; its "Received" tap is a delivery proof. Has buttons. |
| **2** | Complaints | 9 | Sent automatically on every complaint action. Silent failure today. |
| **3** | Account + staff + reminders | 10 | Welcome, password alert, visit reminders, missed-call apology, attendance reminders. |
| **4** | Post-sale (review, service) | 2 | Scheduled after delivery (+2 days) and after 6 months. |
| **5** | Marketing, plus 3 seeded templates nothing sends yet | 10 | Last — Meta looks harder at these, and a rejection here must not block waves 0–4. |
| **opt** | `product_carousel_N` (not in the 41) | 0–5 | Only if the bot is switched to carousel mode. Default is a list, which needs no template. |

(1 + 9 + 9 + 10 + 2 + 10 = **41** seeded, +1 for §3.8 = **42**. Three of wave 5 — `lead_acknowledgement`, `followup_nudge`,
`catalogue_followup` — have no automatic sender yet and can be skipped; see §3.6.)

---

## 2. How to create a template — the click path

Do this once per template. Plan on ~3 minutes each once you have done two.

### 2.1 In WhatsApp Manager (use this)

1. Open `https://business.facebook.com/latest/whatsapp_manager/message_templates`.
2. **Top-left, pick the account** "Kanha Furniture Company" (WABA `2041802123192471`). If you are on
   "Test WhatsApp Business Account" you are on the wrong one — stop.
3. Click **Create template**.
4. **Category** — choose exactly the category in the tables below (`Utility`, `Marketing` or
   `Authentication`). Do not pick the cheaper one; the *content* decides, and a mismatch is rejected or
   silently re-categorised.
5. **Name** — paste the name exactly (lowercase, underscores). The app finds templates by this name.
   You cannot rename later; a typo means delete and recreate.
6. **Language** — `English`, **not** Hindi. Kanha's copy is Hinglish in Latin script; choosing Hindi tells
   Meta to expect Devanagari and gets it rejected ([`08`](08-TEMPLATES-AND-ASSETS.md) §4.2).
   *Exception:* `login_otp` is `English (US)` (`en_US`).
7. **Header / Footer / Buttons** — leave empty. The seeded templates are **body-only**, and that is all
   the sender builds today (a header or button the app does not fill makes every send fail on a
   parameter mismatch — [`08`](08-TEMPLATES-AND-ASSETS.md) §2.3, §3.3).
8. **Body** — paste the body text from the tables below **exactly**, including `{{1}}`, `{{2}}`… and the
   dash signature where present. Do not "improve" the wording: the app's plain-text copy
   (`CustomerMessageText`) is written to match, and the **number of variables must equal what the code
   sends**.
9. **Variable samples** — Meta asks for an example for every `{{n}}`. Fill them with the realistic values
   in the *Samples* column. `test`, `abc`, `xxx` are a top rejection cause.
10. **Submit.** Status becomes *In review*; usually decided within 24 hours.
11. Next template.

### 2.2 Same thing through the API (use for the bulk)

Needs a token with `whatsapp_business_management` on the brand WABA (the production system-user token
in the vault — don't paste it into a doc or chat).

```bash
curl -X POST "https://graph.facebook.com/v23.0/2041802123192471/message_templates" \
  -H "Authorization: Bearer $WABA_TOKEN" -H "Content-Type: application/json" \
  -d '{
    "name": "order_confirmed",
    "language": "en",
    "category": "UTILITY",
    "components": [{
      "type": "BODY",
      "text": "Namaste {{1}}, aapka order {{2}} confirm ho gaya hai. Hum aapko delivery se pehle update karenge.",
      "example": { "body_text": [["Rahul Kumar", "KFO-000042"]] }
    }]
  }'
# → { "id": "...", "status": "PENDING", "category": "UTILITY" }
```

The `body_text` example is **one inner array** holding one sample per variable, in order. The Samples
column in §3 is written so you can lift it straight into that array. (`login_otp` is different — §3.1.)

### 2.3 After you submit

1. Wait for the decision (Manager shows *Approved* / *Rejected*; Meta also emails the business admin).
2. In Kanha, **Settings → WhatsApp → the number → Templates on this WABA → Sync templates now** — or just
   wait up to an hour. The sync adopts what Meta has, so a template made directly in Manager is picked
   up; nothing is "registered" by hand.
3. Confirm with the query in §5 that the row says `APPROVED`.
4. **Send one real message** to your own handset (section 6) and check the wording rendered correctly.

### 2.4 If Meta says no

| You see | Do |
|---|---|
| Rejected — *variable* / *sample* | Samples missing or fake. Edit, give realistic ones, resubmit. |
| Rejected — *category* / re-categorised to Marketing | Normal for borderline copy (`customer_registered`, `service_reminder`, `tried_calling_you`). A Marketing template still works but is **dearer and is blocked for customers who sent STOP** ([`08`](08-TEMPLATES-AND-ASSETS.md) §6) — accept it, or soften the wording. |
| Rejected — *language mismatch* | You picked Hindi. Recreate as English. |
| Rejected — body | Variable at the very start/end of the body, or two variables side by side. None of the seeded bodies do this, so it means the text was altered when pasted. |
| Stuck *In review* > 24 h | Wait a second day, then raise it in Business Support from the WABA page. |
| Approved but app still refuses | Sync didn't run: click *Sync templates now*. If the row is still `PENDING`, the name or language differs from the seed (`en` vs `en_US`) — compare in §5. |

---

## 3. The list

41 templates, numbered 1–41 across §3.1–§3.6.

Legend — **Samples** go in Meta's "variable sample" boxes in order ({{1}} first).
**Sent by** says what in the app triggers it. All are language `en` unless stated.

### 3.1 Wave 0 — Authentication (the one with a special shape)

| # | Name | Category | Language | Sent by |
|---|---|---|---|---|
| 1 | `login_otp` | **Authentication** | **`en_US`** | Every sign-in / verification code by WhatsApp (`WhatsappOtpDeliveryAdapter`) |

Authentication templates are **not free text.** In Manager choose category *Authentication*, then:

- Pick **Copy code** as the button type. Leave "Add security recommendation" on if you like — Meta
  generates the wording.
- Set the **code expiry** you actually want. It is baked into the template at creation — the app's
  setting and the template must agree (the 2026-09-11 reading found Meta's copy said 30 minutes and the
  code defaulted to 5; [`13`](13-ONBOARDING-TRACKER.md) §1.0b).
- Meta writes the body (`{{1}} is your verification code.`). The seeded body is only a placeholder of
  the same shape.
- API form: `components: [{"type":"BODY","add_security_recommendation":true},{"type":"BUTTONS","buttons":[{"type":"OTP","otp_type":"COPY_CODE","text":"Copy code"}]}]` with `"language":"en_US"`.

If you already have an approved OTP template (the test WABA had `login_otp` `en_US` and
`verification_code` `en`), the app tries an approved candidate in order — **create `login_otp` first** on
the brand WABA; it is the name every deployment is meant to use.

### 3.2 Wave 1 — Orders, payment, quotation (Utility, sent automatically)

| # | Name | Sent by | Body (paste exactly) | Samples |
|---|---|---|---|---|
| 2 | `order_confirmed` | Order confirmed (and again from the payment path) | Namaste {{1}}, aapka order {{2}} confirm ho gaya hai. Hum aapko delivery se pehle update karenge. | Rahul Kumar · KFO-000042 |
| 3 | `delivery_scheduled` | Delivery date/slot set | Namaste {{1}}, order {{2}} ki delivery {{3}} ko {{4}} slot mein hai. Koi badlav chahiye toh yahin reply kijiye. | Rahul Kumar · KFO-000042 · 25 Oct · Morning |
| 4 | `delivery_rescheduled` | Delivery date/slot changed | Namaste {{1}}, order {{2}} ki delivery ab {{3}} ko {{4}} slot mein hogi. Koi dikkat ho toh yahin reply kijiye. | Rahul Kumar · KFO-000042 · 27 Oct · Evening |
| 5 | `out_for_delivery` | Order leaves for delivery | Namaste {{1}}, aapka order {{2}} aaj delivery ke liye nikal chuka hai. Humari team aapse sampark karegi. | Rahul Kumar · KFO-000042 |
| 6 | `delivery_completed` | Delivery marked done | Namaste {{1}}, order {{2}} deliver ho gaya hai. Koi dikkat ho toh yahin reply kijiye. | Rahul Kumar · KFO-000042 |
| 7 | `installation_scheduled` | Installation job booked | Namaste {{1}}, order {{2}} ki installation {{3}} ko hogi. Technician aane se pehle call karega. | Rahul Kumar · KFO-000042 · 26 Oct |
| 8 | `order_cancelled` | Order cancelled | Namaste {{1}}, aapka order {{2}} cancel kar diya gaya hai ({{3}}). Koi sawaal ho toh batayein. | Rahul Kumar · KFO-000042 · customer request |
| 9 | `payment_received` | Payment recorded against an order | Namaste {{1}}, order {{2}} ke liye {{3}} ka payment mil gaya hai. Baaki: {{4}}. Dhanyavaad! | Rahul Kumar · KFO-000042 · ₹20,000 · ₹35,000 |
| 10 | `quotation_ready` | "Send quotation" in the inbox | Namaste {{1}}, aapka quotation {{2}} (total {{3}}) taiyaar hai. Poori details ke liye is message ka reply kijiye. | Rahul Kumar · QT-2026-0017 · ₹55,000 |

### 3.3 Wave 2 — Complaints (Utility, sent automatically)

These are the ones behind the 2026-10-02 "WhatsApp update is still not working" finding
(the cause was PENDING templates, not credentials). Bodies below are the rows **actually in the
database** (V26 / V2 / V46 — where V26 and V46 both define a name, V26 ran first and wins).

| # | Name | Sent by | Body (paste exactly) | Samples |
|---|---|---|---|---|
| 11 | `complaint_registered` | Complaint logged | Namaste {{1}}, aapki complaint darj ho gayi hai. Complaint number: {{2}}. Issue: {{3}}. Hum jald hi aapse sampark karenge. | Rahul Kumar · CMP-000123 · Wardrobe door loose |
| 12 | `complaint_resolved` | Complaint resolved | Namaste {{1}}, aapki complaint {{2}} resolve ho gayi hai. {{3}} Koi aur sawaal ho toh yahin reply kijiye. | Rahul Kumar · CMP-000123 · Hinge replaced. |
| 13 | `complaint_cancelled` | Complaint cancelled | Namaste {{1}}, aapki complaint {{2}} cancel kar di gayi hai. Agar yeh galti se hua hai toh please humein call karein ya yahin reply kijiye. — Kanha Furniture | Rahul Kumar · CMP-000123 |
| 14 | `complaint_status_update` | Staff "Send update" on a complaint | Namaste {{1}}, aapki complaint {{2}} ke baare mein update hai: {{3}}. Koi sawaal ho toh yahin reply kijiye. — Kanha Furniture | Rahul Kumar · CMP-000123 · Technician visit is planned for Saturday |
| 15 | `complaint_upload_link` | "Send photo link" | Namaste {{1}}, complaint {{2}} ke liye kripya problem ki photos yahan upload karein: {{3}}. Bill ki photo bhi upload kar sakte hain. Yeh link kuch dinon ke liye hi valid hai. | Rahul Kumar · CMP-000123 · https://kanhafurniture.com/c/AbC123xYz |
| 16 | `complaint_pickup_scheduled` | Pickup booked | Namaste {{1}}, complaint {{2}} ke product ka pickup {{3}} ko {{4}} slot mein hai. Humari team aapse sampark karegi. | Rahul Kumar · CMP-000123 · 25 Oct · Morning |
| 17 | `complaint_received` | Product reaches branch | Namaste {{1}}, complaint {{2}} ka product humare branch par mil gaya hai. Servicing ke baad hum aapko update karenge. | Rahul Kumar · CMP-000123 |
| 18 | `complaint_service_charge` | Service charge quoted | Namaste {{1}}, complaint {{2}} ki servicing ka anumaanit kharch Rs {{3}} hai. Aage badhne ke liye kripya haan reply karein ya humein call karein. | Rahul Kumar · CMP-000123 · 1500 |
| 19 | `complaint_return_scheduled` | Return delivery booked | Namaste {{1}}, complaint {{2}} ka product service hokar taiyaar hai. Delivery {{3}} ko {{4}} slot mein hogi. | Rahul Kumar · CMP-000123 · 28 Oct · Evening |

> **Sample for the link (#15):** use a realistic `https://kanhafurniture.com/…` path. URL shorteners
> (`bit.ly`) are a common rejection cause; and a variable that *is* the whole link is fine as long as
> the surrounding sentence gives Meta context, which this body does.
>
> **Side note for #13, #14:** the body mentions "— Kanha Furniture" as a sign-off; keep it, the
> variable count is what matters to the code.

### 3.4 Wave 3 — Account, staff, reminders (Utility)

| # | Name | Sent by | Body (paste exactly) | Samples |
|---|---|---|---|---|
| 20 | `customer_registered` | Customer account created (website/ops) | Namaste {{1}}, Kanha Furniture parivaar mein aapka swagat hai! Aapke naye furniture ke saath koi bhi sawaal ho toh yahin reply kijiye — hum madad karenge. — Kanha Furniture Company | Rahul Kumar |
| 21 | `staff_welcome` | Staff profile created | Namaste {{1}}, Kanha Furniture Company mein aapka profile ban gaya hai ({{2}}). Login details aapke manager se milenge. Koi dikkat ho toh yahin reply kijiye. — Kanha Furniture Company | Rahul Kumar · Sales Executive |
| 22 | `staff_activated` | Staff account activated | Namaste {{1}}, aapka Kanha Furniture Company account ab active hai — aap login kar sakte hain. Password bhool gaye hain toh 'password' likh kar bhejiye. — Kanha Furniture Company | Rahul Kumar |
| 23 | `password_changed` | Customer changes password (security alert) | Your Kanha Furniture account password was changed on {{1}}. If this was not you, call us immediately on {{2}}. | 11 Sep 2026, 3:04 PM · +91 98765 43210 |
| 24 | `visit_reminder` | Evening before a booked showroom visit | Namaste {{1}}, kal {{2}} store par aapka visit hai. Timing: {{3}}. | Rahul Kumar · Danapur · 4:30 pm |
| 25 | `visit_today` | ~2 hours before the visit | Namaste {{1}}, aaj {{2}} baje Kanha Furniture {{3}} showroom par aapka visit hai. Hum aapka intezaar kar rahe hain! Time badalna ho toh yahin reply kijiye. — Kanha Furniture Company | Rahul Kumar · 4:30 pm · Danapur |
| 26 | `missed_call_sorry` | After a missed call still owed a call-back | Namaste {{1}}, aapne Kanha Furniture ko call kiya tha par hum baat nahi kar paaye — maafi chahte hain. Humari team jald hi aapko call karegi, ya aap yahin apna sawaal likh dijiye. — Kanha Furniture Company | Rahul Kumar |
| 27 | `tried_calling_you` | After a call that went unanswered / busy / switched off | Namaste {{1}}, Kanha Furniture se humne aapko call kiya tha par baat nahi ho paayi. Aapko kab call karna theek rahega? Yahin reply kijiye. — Kanha Furniture Company | Rahul Kumar |
| 28 | `staff_clock_out_reminder` | Staff still clocked in after the day | Namaste {{1}}, aapne aaj {{2}} par clock-in kiya tha aur abhi tak clock-out nahi kiya hai. Kaam khatam ho gaya ho toh Kanha app mein My Attendance se clock-out kar dijiye. — Kanha Furniture Company | Rahul · 9:42 am |
| 29 | `staff_regularise_reminder` | Morning after a missed clock-out, or a manager's Remind | Namaste {{1}}, {{2}} ka aapka clock-out record nahi hua. Agar aapne us din kaam kiya tha toh 3 din ke andar Kanha app mein My Attendance → Regularise se Missed clock-out request daaliye, warna wo din kam ghante ka gina jayega. — Kanha Furniture Company | Rahul · 20 Oct |

> **Likely to be re-categorised by Meta:** `customer_registered` (reads as a welcome/promotional),
> `tried_calling_you` (asks the customer to respond about timing). If Meta moves one to Marketing, let
> it; the sync adopts Meta's category, and it will simply be subject to the opt-out gate.
> **#22** has a straight apostrophe pair around *password* in the body — type plain `'`, not curly quotes.

### 3.5 Wave 4 — Post-sale scheduled messages (Utility)

| # | Name | Sent by | Body (paste exactly) | Samples |
|---|---|---|---|---|
| 30 | `review_request` | Booked automatically 2 days after delivery | Namaste {{1}}, order {{2}} kaisa laga? Aapki raay humare liye important hai. | Rahul Kumar · KFO-000042 |
| 31 | `service_reminder` | Booked automatically 6 months after delivery | Namaste {{1}}, order {{2}} ko 6 mahine ho gaye. Free service check ke liye batayein. | Rahul Kumar · KFO-000042 |

These two read as promotional to Meta; if either is re-categorised to Marketing, accept it (see §2.4).

### 3.6 Wave 5 — Marketing, and seeded-but-unwired templates

Create these **last**. Marketing is billed higher and is **refused at send time for any customer who
opted out** — that is by design ([`10`](10-BROADCAST-AND-CONSENT.md)).

**Marketing, for customer engagement sequences** (seeded by V49 for that purpose; the sequence rows pick the template by name):

| # | Name | Category | Body (paste exactly) | Samples |
|---|---|---|---|---|
| 32 | `customer_welcome` | Marketing | Namaste {{1}}, Kanha Furniture parivaar mein aapka swagat hai! Aapke naye furniture ke saath koi bhi sawaal ho toh yahin reply kijiye — hum madad karenge. | Rahul Kumar |
| 33 | `customer_we_miss_you` | Marketing | Namaste {{1}}, bahut dino se aapse baat nahi hui. Hamare paas naye design aa gaye hain — dekhne ke liye showroom aaiye ya yahin reply kijiye. — Kanha Furniture | Rahul Kumar |
| 34 | `customer_birthday` | Marketing | Janamdin ki hardik shubhkamnaayein {{1}}! Aapke special din par Kanha Furniture ki taraf se dher saari shubhkamnaayein. | Rahul Kumar |
| 35 | `customer_anniversary` | Marketing | Namaste {{1}}, aapne {{2}} humse apna pehla furniture liya tha — us bharose ke liye dhanyavaad! Kuch naya chahiye toh yahin reply kijiye. — Kanha Furniture | Rahul Kumar · 2024 |

**Marketing / utility, chosen by staff by hand** (broadcast and the inbox template picker — no
automatic sender in code today):

| # | Name | Category | Body (paste exactly) | Samples |
|---|---|---|---|---|
| 36 | `festival_offer` | Marketing | Namaste {{1}}, {{2}} par Kanha Furniture mein special offer hai. Showroom aakar dekhiye! | Rahul Kumar · Diwali |
| 37 | `winback_quoted` | Marketing | Namaste {{1}}, aapne {{2}} ke liye quotation liya tha. Abhi bhi interested hain? Stock limited hai. | Rahul Kumar · 3-seater sofa |
| 38 | `crosssell_followup` | Marketing | Namaste {{1}}, aapke {{2}} ke saath ye bhi achha lagega. Dekhna chahenge? | Rahul Kumar · sofa set |
| 39 | `lead_acknowledgement` | Utility | Namaste {{1}}, aapki enquiry mil gayi hai. Humara consultant 2 ghante mein call karega. | Rahul Kumar |
| 40 | `followup_nudge` | Utility | Namaste {{1}}, aapne {{2}} ke baare mein pucha tha. Abhi bhi interested hain? | Rahul Kumar · dining table |
| 41 | `catalogue_followup` | Utility | Namaste {{1}}, {{2}} ka catalogue bheja tha. Koi design pasand aaya? | Rahul Kumar · bedroom |

`followup_nudge` is reserved for a re-engagement feature that is not built (`WhatsappConversation`
comment); `lead_acknowledgement` and `catalogue_followup` have no automatic sender either. They are
safe to **skip until a screen or sequence uses them** — nothing in waves 0–4 depends on them. #40 and
#41 will likely be re-categorised to Marketing.

(Count check: wave 0 = 1, wave 1 = 9, wave 2 = 9, wave 3 = 10, wave 4 = 2, wave 5 = 10 → **41**.)

### 3.7 Optional — the product carousel

Not seeded. Only needed if the bot setting `whatsapp.bot.product_display` is switched to `CAROUSEL`
(default `LIST` needs no template, and `CARDS` is free). Full recipe: [`08`](08-TEMPLATES-AND-ASSETS.md)
§12. In short — Marketing, English, name `product_carousel_2` (then `_3` … up to `_6`), body
`Here are our {{1}} — swipe to see them all 👉`, each card: image header, body `*{{1}}*` newline `{{2}}`,
two quick-reply buttons ("Learn more", "See in showroom"). **Not yet sent against a live account.**
Skip it until the core waves are live.

### 3.8 Wave 1b — `delivery_on_the_way` (built, not seeded, has buttons)

The transport module's door-proof message (`DeliveryNoticeSender`, owner decision 2026-10-20). **No
migration seeds it**, so it will not appear in the §5 query until the sync adopts it from Meta. The name
and language are settings: `whatsapp.delivery_notice.template` (default `delivery_on_the_way`) and
`whatsapp.delivery_notice.language` (default `en`).

| # | Name | Category | Sent by | Samples |
|---|---|---|---|---|
| 42 | `delivery_on_the_way` | Utility | Trip dispatched — one message per customer drop | Rahul Kumar · BR01AB1234 · 9876543210 · 4821 |

- **Body:** `Namaste {{1}}, your Kanha delivery is on its way in vehicle {{2}} (driver {{3}}). Your delivery code is {{4}} — give it to the driver only when your goods are in front of you.`
  ({{1}} customer name, {{2}} vehicle number, {{3}} driver's mobile, {{4}} 4-digit code.) This is the
  wording suggested in [transport/02-DESIGN.md](../transport/02-DESIGN.md) §9 — note it is **English, not
  Hinglish**, unlike the rest.
- **Buttons (2, Quick reply):** `Received` and `Not received` — type the labels exactly. A tap comes back
  as the button text and is matched against `whatsapp.delivery_notice.received_replies` /
  `not_received_replies`, which already list Latin and Devanagari phrases. Add the buttons in Manager under
  *Buttons → Quick reply*; the sender passes only the four body values.
- Until approved, the app sends the same text as free text with real buttons — but only inside the
  24-hour window; anyone else is skipped and the drop is proven by the code or the signed challan.
- The inbound handling of template quick-reply taps is described in
  [transport/03-AS-BUILT.md](../transport/03-AS-BUILT.md); I did not re-verify it live. Test it with your
  own phone before relying on it ([`08`](08-TEMPLATES-AND-ASSETS.md) §2.3 documents the older gap here).

---

## 4. Things that look like problems and are not

- **The same name appears in two migrations.** `complaint_pickup_scheduled`, `complaint_received`,
  `complaint_service_charge`, `complaint_return_scheduled` and `complaint_upload_link` are defined in
  both V26 and V46 with slightly different wording. V26 runs first and its rows win (V46 is
  `ON CONFLICT … DO NOTHING`). §3.3 uses the winning wording. The variable counts are identical, so
  either wording would work for sending; V46's header comment is out of date on this.
- **`customer_welcome` vs `customer_registered`** are two different templates on purpose: the first is
  Marketing (behind the opt-out gate), the second Utility so a new account is always told it exists
  (V102).
- **The local row says `en` but Meta approved `en_US`.** The app now sends in the language of the
  approved row, so this works — but the two must still exist as the *same name*.

---

## 5. Check what is pending right now

Run against the production database (read-only):

```sql
SELECT name, language, category, status, waba_id, updated_at
FROM   whatsapp_template
WHERE  deleted_at IS NULL
ORDER  BY (status = 'APPROVED'), category, name;
```

- Everything with `status <> 'APPROVED'` is outstanding — that is your remaining list from §3.
- A row with `waba_id IS NULL` is a seed row the sync has not matched to a WABA yet; a row with the
  WABA id is one the sync has seen at Meta.
- Names that are in §3 but **missing** from this result entirely were created at Meta but never seeded
  locally — fine, the sync adopts them.

In the app: **Settings → WhatsApp → the number → Templates on this WABA** lists the same thing straight
from Meta.

## 6. Prove each one works

For every template that flips to `APPROVED`:

1. Trigger its real event with **your own number** as the customer (e.g. confirm a test order, log a
   test complaint, send yourself a quotation).
2. On the handset check: it arrived from the right number, the wording is exactly the body, no `{{1}}`
   is left visible, the name and values are in the right slots.
3. Tick it off in [`13`](13-ONBOARDING-TRACKER.md) Phase C (C5).

If the send is refused, the log line names the template and its status — fix that, no code change.

## 7. Everything else, to date — Part B

Part A (§1–§6) is what to submit now. This section is every other WhatsApp template name that appears
anywhere in the code, the migrations, the plans or Meta's account readings, so nothing is left to memory.
**Not** in this list: the e-mail and in-app notification templates (`NotificationTemplateKey`,
`resources/templates/notification/email|inapp`) — those are Kanha's own and need no Meta approval. That
includes `VENDOR_PAPERS_DUE` (vendor papers expiring), which is one of them, not a WhatsApp template.

### 7.1 What each template is made of (its "items")

Everything in Part A is **body-only** except these:

| Template | Header | Body | Footer | Buttons |
|---|---|---|---|---|
| 40 body-only templates (#2–#41) | — | text with `{{n}}` | — | — |
| `login_otp` (#1) | — | Meta-written, one `{{1}}` code | optional security line / expiry | **Copy code** (OTP) |
| `delivery_on_the_way` (#42) | — | text, 4 variables | — | **Received**, **Not received** (quick reply) |
| `product_carousel_N` (optional) | — | one `{{1}}` | — | per card: image header, body `*{{1}}*` + `{{2}}`, 2 quick replies |

So **only #1, #42 and the carousel carry anything beyond a body.**

No template uses a **media header** (image, video, document). The sender builds body parameters only
([`08`](08-TEMPLATES-AND-ASSETS.md) §3.3, gap G-05), which is why every plan below that wants a PDF
attached needs code work first.

### 7.2 Optional now — the carousel family

| Name | Category | When it is needed |
|---|---|---|
| `product_carousel_2` … `product_carousel_6` (base name set by `whatsapp.bot.carousel_template`) | Marketing | Only if `whatsapp.bot.product_display` = `CAROUSEL`. Recipe in §3.7 and [`08`](08-TEMPLATES-AND-ASSETS.md) §12. Never sent against a live account. |

### 7.3 Planned, not built — do not submit yet

From the showroom-booking plan ([`03-HLD`](../showroom-booking/03-HLD.md) §8, sprint item BK-80). **No
code, migration or body text exists for these.** Submit nothing until the bodies are written and the
trigger is built; Meta approval is an owner task in sprint S0 once they are.

| Planned name | Category (planned) | Sent when | Notes |
|---|---|---|---|
| `booking_confirmed` | Utility | Booking confirmed | Plan attaches the booking slip **PDF** → needs a document-header template (gap G-05) |
| `payment_due_reminder` | Utility | Instalment due −2 d / on the day / +3 d | One template, three timings |
| `payment_bounced` | Utility | Cheque bounced | |
| `ready_for_dispatch` | Utility | Clearance ("NO DUES") | |
| `delivery_confirm_request` | Utility, button "Received" | At the door | **Overlaps `delivery_on_the_way` (#42)**, which already carries Received / Not received. Decide whether the booking flow reuses #42 before submitting a second one |

The same plan also reuses existing names — `payment_received` (add a receipt PDF), `delivery_scheduled`,
`out_for_delivery`, `delivery_completed` (add an invoice PDF), `review_request`, `order_cancelled` — so
those are already in Part A. Each new template is gated by a `notifications.booking.*.enabled` setting and
"must be APPROVED before its trigger is enabled".

### 7.4 Ideas that were named and never seeded

| Name | Where it appears | Status |
|---|---|---|
| `order_dispatched` | [`08`](08-TEMPLATES-AND-ASSETS.md) §3.2 / §5.3 and [`05`](05-CLOUD-API-REFERENCE.md) — **worked examples** of the API call | An example, not a real template. Superseded by `out_for_delivery`. Do not submit |
| `payment_reminder` (with a payment link) | [`08`](08-TEMPLATES-AND-ASSETS.md) §5.2 note | Proposed 2026-08, never seeded. The booking plan's `payment_due_reminder` is its successor |

### 7.5 What exists at Meta that is not Kanha's list

Readings from the docs — **not re-queried today**. Reference only; Kanha's code never depends on them
except `login_otp`.

| WABA | Template | Language | Category | Reading date and note |
|---|---|---|---|---|
| Test `1606315164548044` | `login_otp` | `en_US` | Authentication | 2026-09-11, Approved. The shape the brand-WABA `login_otp` should copy |
| Test | `verification_code` | `en` | Authentication | 2026-09-11, Approved. The OTP adapter's second candidate |
| Test | `catalogue_prod_1` | `en` | Marketing | 2026-09-11, Approved but **unusable**: named variable `{{name}}` + a product-list button that needs a Meta Commerce Catalogue; the sender only builds positional variables |
| Test | `hello_world`, `jaspers_market_*` (4) | — | Meta's stock demos | Earlier reading, in the test-WABA scorecard in [`13`](13-ONBOARDING-TRACKER.md). Ignore |
| Brand `2041802123192471` | `kfcsignup` | — | — | 2026-10-04. Created at Meta by hand; not used by the code. Ask whoever created it what it was for before deleting |
| Brand | `hello_world` | — | Meta's stock demo | 2026-10-04. Ignore |

### 7.6 The grand total

| Group | Count |
|---|---|
| Seeded, to submit now (Part A, #1–#41) | 41 |
| Built but set by setting, to submit now (#42) | 1 |
| Optional carousel family | up to 5 |
| Planned, not built (§7.3) | 5 |
| Examples / ideas, never real (§7.4) | 2 |
| At Meta already, not Kanha's (§7.5) | 3 Kanha-shaped (`login_otp` ×1, `verification_code`, `catalogue_prod_1`) + 7 stock/other |

---

# Part C — The Hindi (Devanagari) set

## 8. What it is, and who it is for

Today every template is Hinglish in Latin script, submitted as language `en` (§2.1). A **Hindi set** is
the *same names* again under language **Hindi (`hi`)**, written in Devanagari. Meta allows one name in
several languages, and the database already keys templates on (WABA, name, language), so a `hi` row sits
beside the `en` row with no clash.

**Who actually gets it.** The app knows a customer's language only if they have chatted with the bot:
`whatsapp_conversation.preferred_language` and `lead.preferred_language` hold `EN` / `HI` / `HINGLISH`
(`HI` = Devanagari, `HINGLISH` = Latin Hindi). A walk-in who bought at the counter and never messaged has
**no language on record**, so would get the `en` version regardless. Expect the Hindi set to reach
customers who wrote to the bot in Devanagari — useful, but a minority.

**Order of effort.** Customer-facing first (the order, complaint, reminder and account rows), marketing
second, staff rows last — staff already read the Latin text. `login_otp` needs no Hindi copy written by us:
if you want one, create it as category Authentication in language Hindi and Meta writes the body.

### 8.1 Rules for submitting the Hindi copy

- Language **Hindi** in Manager (`hi`); same **name**, same category, same variable count and order as the
  English row. The code fills `{{1}}…{{n}}` by position, so a sentence must be built so the same
  values still fit in the same order.
- **Write the brand in Devanagari** (कान्हा फर्नीचर) so the body is not a script mix; a handful of app
  labels stay Latin on purpose (`My Attendance`, `Regularise`, `Missed clock-out`, `password`) because
  staff tap or type exactly those words in the app and the bot.
- **Samples:** use the same sample values as the English table in §3, with the name as `राहुल कुमार`.
  Dates, slots, categories and vehicle numbers are supplied by the code **in English** (`25 Oct`,
  `Morning`), so the samples stay English — a Hindi sentence with an English slot word inside is expected.
- **Have a native Hindi reader check every line before you submit.** These were written by me, not
  reviewed by a native speaker; wording that is slightly off is a customer-trust problem long before it
  is a Meta problem. Once approved, edits send a template back through review.
- The bodies below deliberately mirror the English meaning one to one. Do not add promotional words to a
  Utility row.

## 8.2 The Hindi bodies — customer-facing

**Orders and payment** (Utility)

| Name | Body (paste exactly) |
|---|---|
| `order_confirmed` | नमस्ते {{1}}, आपका ऑर्डर {{2}} कन्फर्म हो गया है। डिलीवरी से पहले हम आपको अपडेट देंगे। |
| `delivery_scheduled` | नमस्ते {{1}}, ऑर्डर {{2}} की डिलीवरी {{3}} को {{4}} स्लॉट में है। कोई बदलाव चाहिए तो यहीं रिप्लाई कीजिए। |
| `delivery_rescheduled` | नमस्ते {{1}}, ऑर्डर {{2}} की डिलीवरी अब {{3}} को {{4}} स्लॉट में होगी। कोई दिक्कत हो तो यहीं रिप्लाई कीजिए। |
| `out_for_delivery` | नमस्ते {{1}}, आपका ऑर्डर {{2}} आज डिलीवरी के लिए निकल चुका है। हमारी टीम आपसे संपर्क करेगी। |
| `delivery_completed` | नमस्ते {{1}}, ऑर्डर {{2}} डिलीवर हो गया है। कोई दिक्कत हो तो यहीं रिप्लाई कीजिए। |
| `installation_scheduled` | नमस्ते {{1}}, ऑर्डर {{2}} की इंस्टॉलेशन {{3}} को होगी। टेक्नीशियन आने से पहले कॉल करेगा। |
| `order_cancelled` | नमस्ते {{1}}, आपका ऑर्डर {{2}} रद्द कर दिया गया है ({{3}})। कोई सवाल हो तो बताइए। |
| `payment_received` | नमस्ते {{1}}, ऑर्डर {{2}} के लिए {{3}} का भुगतान मिल गया है। बाकी: {{4}}। धन्यवाद! |
| `quotation_ready` | नमस्ते {{1}}, आपका कोटेशन {{2}} (कुल {{3}}) तैयार है। पूरी जानकारी के लिए इस मैसेज का रिप्लाई कीजिए। |
| `delivery_on_the_way` | नमस्ते {{1}}, आपकी कान्हा डिलीवरी गाड़ी नंबर {{2}} में निकल चुकी है (ड्राइवर {{3}})। आपका डिलीवरी कोड {{4}} है — यह कोड ड्राइवर को तभी बताइए जब आपका सामान आपके सामने हो। |

`delivery_on_the_way` buttons in Hindi: **मिल गया** and **नहीं मिला** (both are already in the app's
default reply lists, `whatsapp.delivery_notice.*_replies`). Its language setting
`whatsapp.delivery_notice.language` must then be changed to `hi` **if you want Hindi sent** — it is one
language for every customer, not per customer (§9).

**Complaints** (Utility)

| Name | Body (paste exactly) |
|---|---|
| `complaint_registered` | नमस्ते {{1}}, आपकी शिकायत दर्ज हो गई है। शिकायत नंबर: {{2}}। समस्या: {{3}}। हम जल्द ही आपसे संपर्क करेंगे। |
| `complaint_resolved` | नमस्ते {{1}}, आपकी शिकायत {{2}} का समाधान हो गया है। {{3}} कोई और सवाल हो तो यहीं रिप्लाई कीजिए। |
| `complaint_cancelled` | नमस्ते {{1}}, आपकी शिकायत {{2}} रद्द कर दी गई है। अगर यह गलती से हुआ है तो कृपया हमें कॉल करें या यहीं रिप्लाई कीजिए। — कान्हा फर्नीचर |
| `complaint_status_update` | नमस्ते {{1}}, आपकी शिकायत {{2}} के बारे में अपडेट है: {{3}}। कोई सवाल हो तो यहीं रिप्लाई कीजिए। — कान्हा फर्नीचर |
| `complaint_upload_link` | नमस्ते {{1}}, शिकायत {{2}} के लिए कृपया समस्या की फोटो यहाँ अपलोड करें: {{3}}। बिल की फोटो भी अपलोड कर सकते हैं। यह लिंक कुछ ही दिनों के लिए मान्य है। |
| `complaint_pickup_scheduled` | नमस्ते {{1}}, शिकायत {{2}} के प्रोडक्ट का पिकअप {{3}} को {{4}} स्लॉट में है। हमारी टीम आपसे संपर्क करेगी। |
| `complaint_received` | नमस्ते {{1}}, शिकायत {{2}} का प्रोडक्ट हमारी ब्रांच में पहुँच गया है। सर्विसिंग के बाद हम आपको अपडेट देंगे। |
| `complaint_service_charge` | नमस्ते {{1}}, शिकायत {{2}} की सर्विसिंग का अनुमानित खर्च {{3}} रुपये है। आगे बढ़ने के लिए कृपया हाँ रिप्लाई करें या हमें कॉल करें। |
| `complaint_return_scheduled` | नमस्ते {{1}}, शिकायत {{2}} का प्रोडक्ट सर्विस होकर तैयार है। डिलीवरी {{3}} को {{4}} स्लॉट में होगी। |

**Account, reminders, post-sale** (Utility)

| Name | Body (paste exactly) |
|---|---|
| `customer_registered` | नमस्ते {{1}}, कान्हा फर्नीचर परिवार में आपका स्वागत है! आपके नए फर्नीचर के साथ कोई भी सवाल हो तो यहीं रिप्लाई कीजिए — हम मदद करेंगे। — कान्हा फर्नीचर कंपनी |
| `password_changed` | आपके कान्हा फर्नीचर अकाउंट का पासवर्ड {{1}} को बदला गया है। अगर यह आपने नहीं किया, तो तुरंत {{2}} पर कॉल करें। |
| `visit_reminder` | नमस्ते {{1}}, कल {{2}} स्टोर पर आपका विज़िट है। समय: {{3}}। |
| `visit_today` | नमस्ते {{1}}, आज {{2}} पर कान्हा फर्नीचर {{3}} शोरूम में आपका विज़िट है। हम आपका इंतज़ार कर रहे हैं! समय बदलना हो तो यहीं रिप्लाई कीजिए। — कान्हा फर्नीचर कंपनी |
| `missed_call_sorry` | नमस्ते {{1}}, आपने कान्हा फर्नीचर को कॉल किया था पर हम बात नहीं कर पाए — माफ़ी चाहते हैं। हमारी टीम जल्द ही आपको कॉल करेगी, या आप यहीं अपना सवाल लिख दीजिए। — कान्हा फर्नीचर कंपनी |
| `tried_calling_you` | नमस्ते {{1}}, कान्हा फर्नीचर से हमने आपको कॉल किया था पर बात नहीं हो पाई। आपको कब कॉल करना ठीक रहेगा? यहीं रिप्लाई कीजिए। — कान्हा फर्नीचर कंपनी |
| `review_request` | नमस्ते {{1}}, ऑर्डर {{2}} कैसा लगा? आपकी राय हमारे लिए बहुत ज़रूरी है। |
| `service_reminder` | नमस्ते {{1}}, ऑर्डर {{2}} को 6 महीने हो गए हैं। फ्री सर्विस चेक के लिए बताइए। |
| `lead_acknowledgement` | नमस्ते {{1}}, आपकी पूछताछ हमें मिल गई है। हमारा कंसल्टेंट 2 घंटे में कॉल करेगा। |
| `followup_nudge` | नमस्ते {{1}}, आपने {{2}} के बारे में पूछा था। क्या आप अभी भी इच्छुक हैं? |
| `catalogue_followup` | नमस्ते {{1}}, {{2}} का कैटलॉग भेजा था। कोई डिज़ाइन पसंद आया? |

## 8.3 The Hindi bodies — marketing (Marketing)

| Name | Body (paste exactly) |
|---|---|
| `customer_welcome` | नमस्ते {{1}}, कान्हा फर्नीचर परिवार में आपका स्वागत है! आपके नए फर्नीचर के साथ कोई भी सवाल हो तो यहीं रिप्लाई कीजिए — हम मदद करेंगे। |
| `customer_we_miss_you` | नमस्ते {{1}}, काफ़ी दिनों से आपसे बात नहीं हुई। हमारे पास नए डिज़ाइन आ गए हैं — देखने के लिए शोरूम आइए या यहीं रिप्लाई कीजिए। — कान्हा फर्नीचर |
| `customer_birthday` | जन्मदिन की हार्दिक शुभकामनाएँ {{1}}! आपके खास दिन पर कान्हा फर्नीचर की ओर से ढेर सारी शुभकामनाएँ। |
| `customer_anniversary` | नमस्ते {{1}}, आपने {{2}} में हमसे अपना पहला फर्नीचर लिया था — इस भरोसे के लिए धन्यवाद! कुछ नया चाहिए तो यहीं रिप्लाई कीजिए। — कान्हा फर्नीचर |
| `festival_offer` | नमस्ते {{1}}, {{2}} पर कान्हा फर्नीचर में खास ऑफ़र है। शोरूम आकर देखिए! |
| `winback_quoted` | नमस्ते {{1}}, आपने {{2}} के लिए कोटेशन लिया था। क्या आप अभी भी इच्छुक हैं? स्टॉक सीमित है। |
| `crosssell_followup` | नमस्ते {{1}}, आपके {{2}} के साथ यह भी अच्छा लगेगा। देखना चाहेंगे? |

## 8.4 The Hindi bodies — staff (lowest priority)

| Name | Body (paste exactly) |
|---|---|
| `staff_welcome` | नमस्ते {{1}}, कान्हा फर्नीचर कंपनी में आपकी प्रोफ़ाइल बन गई है ({{2}})। लॉगिन की जानकारी आपके मैनेजर से मिलेगी। कोई दिक्कत हो तो यहीं रिप्लाई कीजिए। — कान्हा फर्नीचर कंपनी |
| `staff_activated` | नमस्ते {{1}}, आपका कान्हा फर्नीचर कंपनी अकाउंट अब चालू है — आप लॉगिन कर सकते हैं। पासवर्ड भूल गए हैं तो 'password' लिखकर भेजिए। — कान्हा फर्नीचर कंपनी |
| `staff_clock_out_reminder` | नमस्ते {{1}}, आपने आज {{2}} पर क्लॉक-इन किया था और अभी तक क्लॉक-आउट नहीं किया है। काम खत्म हो गया हो तो कान्हा ऐप में My Attendance से क्लॉक-आउट कर दीजिए। — कान्हा फर्नीचर कंपनी |
| `staff_regularise_reminder` | नमस्ते {{1}}, {{2}} का आपका क्लॉक-आउट रिकॉर्ड नहीं हुआ। अगर आपने उस दिन काम किया था तो 3 दिन के अंदर कान्हा ऐप में My Attendance → Regularise से Missed clock-out का रिक्वेस्ट डालिए, वरना वह दिन कम घंटों का गिना जाएगा। — कान्हा फर्नीचर कंपनी |

That is **41 Hindi bodies** — every Part A template except `login_otp`, which Meta writes itself:
30 in §8.2 (10 orders incl. `delivery_on_the_way` + 9 complaints + 11 account/reminders/post-sale),
7 in §8.3, 4 in §8.4. The carousel is not translated.

## 9. What the code needs before the Hindi set is used

Submitting the Hindi templates is safe on its own — nothing changes until a caller asks for `hi`. To
*use* them takes code, which is **not done**. Read from the code on 2026-10-22:

| Piece | Today | What Hindi needs |
|---|---|---|
| Choosing the row | `ApprovedTemplateChooser.firstApproved` prefers the caller's language, then `en_US`, then anything approved. If `hi` is approved and the caller says `hi`, it already picks `hi`; if the caller says `en`, it picks `en` | Nothing — ready |
| Automatic senders (orders, complaints, reminders, account, welcome) | Each publishes `WhatsappTemplateSendRequested` with the language **hardcoded `"en"`** | Each must look up the customer's language and pass `hi` for `HI`, `en` otherwise. Eleven call sites publish the event |
| Where the language comes from | `whatsapp_conversation.preferred_language` and `lead.preferred_language` (`EN`/`HI`/`HINGLISH`). **The customer record has none** | Resolve by phone: conversation → lead → else `en`. Counter-only customers stay `en` |
| Broadcasts | `BroadcastService.createDraft(…, templateName, language)` already takes a language per broadcast | Nothing — staff can pick `hi` for a segment today, once approved |
| `delivery_on_the_way` | One language for everyone via `whatsapp.delivery_notice.language` | Per-customer language, same as the senders above — or accept all-Hindi |
| The free-text fallback (while a template is unapproved) | `CustomerMessageText` is Hinglish only | A Hindi copy of each fallback line, or accept Hinglish while waiting |
| Fallback when `hi` is not approved yet | Chooser would fall through to the `en` row — good | Nothing |

Recommended order: submit the Hindi set **after** the English set is approved and live-tested (§6), then
do the caller change in one small pass. Doing it the other way round risks a customer who prefers Hindi
being sent nothing while the Hindi row is still in review (the chooser falls back to `en`, so this is
handled, but untested).

## 10. Changelog

- **2026-10-22 — v1.1.** Added `delivery_on_the_way` (#42, code-read: built, not seeded, has buttons);
  Part B (§7: every other template to date — what each is made of, carousel, planned booking set,
  never-real examples, what sits at Meta); Part C (§8 Hindi/Devanagari bodies for all 41 non-OTP Part A
  templates, §9 what the code needs). The Hindi was written by me and **has not been reviewed by a native
  speaker**.
- **2026-10-22 — v1.0.** Written. 41 templates from V2/V26/V46/V49/V102/V140/V141, ordered into
  submission waves; click path and API recipe; Meta-live status deliberately **not** claimed (last
  live reading 2026-10-04).
