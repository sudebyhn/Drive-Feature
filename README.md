# Drive Feature — Corporate Innovation Platform

An internal full-stack innovation platform built for Skoda Yuce Auto. Employees submit ideas anonymously, which pass through an AI similarity check and a 3-stage evaluation pipeline before reaching an executive Go/No-Go decision. The system includes a points engine, badge system, rewards module, and a real-time notification layer with email.

> **Status:** Live in production — used by 100+ employees at Skoda Yuce Auto

---

## Screenshots

![Drive Feature Screenshots](docs/Project%20pages%20Screenshots.png)

---

## What This System Does

Any employee can submit an idea anonymously. Before it goes public, an AI-powered NLP check screens it against existing ideas for duplicate detection. If it passes, it flows through a structured 3-stage pipeline. At each stage, the relevant board evaluates the idea, scores it, and decides whether to advance, revise, or reject it.

```
Employee submits idea (anonymous)
        |
AI similarity check (OpenAI)
        |
    passes             fails
        |                 |
  Published         Manager review
        |
Innovation Board review + Drive Score
        |
Executive Board -- Go / No-Go / Revise
        |
Rewards + Points + Badges
```

---

## Pages

```
/
|-- drive_future_anasayfa.html           # Landing / home page
|-- drive_future_fikirler.html           # Ideas feed -- submit and browse
|-- drive_future_inovasyon_temsilcileri.html  # Innovation Board panel
|-- drive_future_icra_kurulu.html        # Executive Board panel
|-- drive_future_panel.html              # Admin analytics panel
|-- drive_future_hakkimizda.html         # About page
|-- drive_future_oduller.html            # Rewards module
```

> Flask backend routes, DB schema, and business logic are not included (internal company code).
> config.py with DB credentials is excluded.

---

## Pages Overview

**Landing Page**
Animated hero with scroll-reveal, floating background orbs, and a rocket animation on idea submission.

**Ideas Feed**
Card grid with status filters, category filters, search, and tab views (all ideas / my ideas). Each card shows status, category tag, like count, comment count, and participation count. Author identity is hidden except for ideas that received Executive Board Go -- those are de-anonymized. Users cannot like, join, or interact with their own ideas.

**Innovation Board**
Role-restricted view. Displays ideas awaiting review (status 0) or revision (status 1). Each board member scores ideas independently using 3 metrics. Actions: approve to council, request revision, reject, or flag for re-evaluation.

**Executive Board**
Final decision layer. Shows Drive Score breakdown (applicability, impact, innovation), board notes, KPI summary (pending, Go, No-Go, revision pending). Actions: Go (status 4), No-Go (status 5), Request Revision (status 6).

**Admin Panel**
Analytics dashboard: idea volume by month (Chart.js), category distribution, total likes, comments, participation count, reward request management, news management, point multiplier settings. Includes PDF export via html2pdf.js and dark mode.

**Rewards**
Users browse available rewards, each with a point requirement and stock count. Claiming deducts points from the user's live balance. Admins manage reward status and stock from the Panel.

---

## AI Duplicate Detection

When a new idea is submitted, the full text (title + problem + solution) is sent to OpenAI for intent classification and similarity scoring against existing ideas.

- If similarity is below the threshold: idea is published immediately (status 0)
- If similarity is above the threshold: idea is held (status -1) and the admin receives an email with the similarity score and a link to the flagged idea
- If the AI service fails: idea is held and an error email is sent to the admin

The admin can then approve or reject the held idea. On approval, the idea is published and the user receives a notification. On rejection, the system fetches the most similar existing idea by ID and sends the user a message naming the idea it conflicted with.

---

## Drive Score

Drive Score is the numerical output of the Innovation Board evaluation. Each board member independently scores an idea across 3 criteria (0-5 scale each):

- Applicability (uygulanabilirlik)
- Expected Impact (beklenen_etki)
- Innovation Value (inovasyon_degeri)

The system averages each criterion across all members who have scored and sums them for the final Drive Score (max 15). Scores are stored per user in Fikirler_Kurul_Puan and aggregated live. When a revision is requested, the requesting member's scores are cleared so they can re-evaluate after the revision is submitted.

---

## Points Engine

Points are calculated dynamically using a time-based multiplier table (Drive_Future_Puan_Ayarlari). Each action's point value is looked up against the timestamp of the action, so historical records are preserved if the admin changes multipliers.

Point-earning actions:
- Submitting an idea
- Liking an idea
- Commenting on an idea
- Joining an idea

Spent points (reward claims with status 0, 1, or 2) are deducted from the live total. Cancelled claims (status 3) trigger a stock refund and a points log entry for the reversal.

Admins can update point multipliers from the Panel. Changes take effect immediately or from the next day -- admin's choice.

---

## Badge System

Badges are awarded automatically based on user activity. Each badge has a badge_code that maps to a record in Fikirler_Rozetler. The system checks eligibility at key moments and awards the badge if not already earned.

Badge triggers:
- ilk_adim: first published idea
- kesifci: ideas submitted across 3 or more different categories
- uretken_beyin: 10 or more published ideas
- seri_uretici: 30 or more published ideas
- destekci: comments left on 10 or more other users' ideas

When a badge is earned, the user receives a push notification and email.

---

## Notification Architecture

All notifications go through a single send_df_notification function. It:
1. Looks up the user's ID and email from MySQL
2. Writes a push notification record to SQL Server (bell icon in the portal)
3. Optionally sends a branded HTML email via the internal SMTP sender

Email is optional per call (send_mail=True/False). Some events trigger only push (likes), others trigger both (idea status changes, badge awards, reward updates).

Notification triggers:
- Idea submitted and held for review (admin email + user push)
- AI error on submission (admin error email)
- Manager approves or rejects held idea (user email + push)
- Innovation Board action: revision, rejection, send to council (user email + push)
- Executive Board decision: Go, No-Go, revision request (user email + push)
- New like on your idea (push only)
- New comment on your idea (email + push)
- Badge earned (email + push)
- Reward claim status updated (email + push)

---

## Role System

3 roles are enforced via df_get_user_yetki:
- Regular employee: ideas feed, submit ideas, rewards, own profile
- inovasyon_yetkili: Innovation Board panel, Drive Score entry, batch actions
- icra_yetkili: Executive Board panel, Go/No-Go decisions

Pages redirect unauthorized users to the home page.

---

## DB Tables (referenced in code)

| Table | Purpose |
|---|---|
| Fikirler | Core idea records with status, score, timestamps |
| Fikirler_Likes | Per-user idea likes |
| Fikirler_Comments | Threaded comments with parent_id |
| Fikirler_Comments_Likes | Comment likes |
| Fikir_Katilim | Idea participation (join) |
| Fikirler_Saved | Saved/bookmarked ideas |
| Fikirler_Kurul_Puan | Per-member Drive Score entries |
| Fikirler_Temsilci_Aksiyon | Innovation Board action log |
| Fikirler_Temsilci_Oylama_Secim | Draft selections before batch submit |
| Fikirler_Icra_Kurulu_Aksiyon | Executive Board decision log |
| Fikirler_Benzerlik_Yonetici_Aksiyon | AI similarity manager action log |
| Fikirler_Revizyon_Aksiyon | Revision history log |
| Drive_Future_Puan_Ayarlari | Time-based point multiplier settings |
| Drive_Future_User_Puan_Log | Per-user point event log |
| Fikirler_Rozetler | Badge definitions |
| Fikirler_User_Rozet | Per-user earned badges |
| Fikirler_Odul | Reward catalog |
| Fikirler_Odul_Talep | Reward claim requests |
| Fikirler_Haber | News/announcements |
| Fikirler_Haber_View_Log | News view tracking |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Language | Python |
| Backend | Flask |
| AI | OpenAI API |
| Primary DB | MS SQL Server (pyodbc) |
| Auth DB | MySQL (flask-mysqldb) |
| Frontend | HTML, CSS, JavaScript |
| Charts | Chart.js |
| PDF export | html2pdf.js |
| Email | Internal SMTP |

---

## What Is Not Included

- Flask backend routes and business logic (internal company code)
- config.py -- DB connection strings
- Internal company data

---

## Author

Built by [Sude Bayhan](https://linkedin.com/in/sude-bayhan) at Skoda Yuce Auto.
Architecture, database design, all backend routes, business logic, points engine, notification system, and frontend implementation.
Frontend contributions on select pages provided by a colleague.

(c) 2026 Sude Bayhan. All rights reserved. This project is not licensed for use, modification, or distribution.
