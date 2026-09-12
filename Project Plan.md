# Course Analytics Dashboard for CAT-SOOP

**Course:** Software Tools and Techniques for CSE


**Project Name:** Extending an existing open-source system

**Submission:** 1 — Problem understanding and proposed design

---

## 1. Summary

CAT-SOOP is the online course platform used for the Digital Systems course at IIT Gandhinagar. Students read the course material on it, solve exercises, and take quizzes that are graded automatically. Around 250 students use it today, and the professor is considering opening the course to the public.

The platform does a good job of delivering content and grading answers. What it does not do is tell the instructor what is actually happening across the class. There is no way to see how far students have progressed, which topics are causing trouble, or which students have quietly stopped working.

We propose to build a **course analytics dashboard** for instructors. It will read the activity data CAT-SOOP already produces, organise it into a form suited to analysis, and present it as a set of clear views — course overview, module-wise progress, question difficulty, individual student progress, and a list of students who may need attention.

The dashboard will be built **into CAT-SOOP itself** as a new set of pages within the course, using the platform's existing login and permissions. Only the heavy calculation runs outside, on a schedule. CAT-SOOP remains the authoritative source of all data, and no file in the platform's own source code is modified.

---

## 2. Understanding the existing system

### What CAT-SOOP is

CAT-SOOP is an open-source learning platform originally built at MIT for their introductory EECS courses. It is used for courses where the assessment is mostly automatic — programming exercises, mathematical expressions, numerical answers.

It works differently from platforms like Moodle or Google Classroom. There is no admin panel where an instructor clicks through forms to build a course. Instead, **the course is a folder of files**. Each folder is a page on the website, and the instructor writes the content and the questions as text files inside those folders. This makes it very flexible for a technical course, but it also means the platform has no traditional database of courses, students and grades in the way most web applications do.

### How the course is organised

The structure mirrors the folders on disk:

```
Digital Systems (course)
├── Module 1: Number Systems
│   ├── learning material
│   └── exercises
├── Module 2: Boolean Algebra
├── Module 3: K-Maps
└── ...
```

Each module is a page. Each page can contain any number of questions. Each question has a type (multiple choice, numeric answer, symbolic expression, Python code, and so on), a point value, and rules such as how many attempts a student is allowed.

### How students use it

A student opens a module, reads the material, and answers the questions on the page. When they submit an answer, the platform checks it — sometimes immediately, sometimes by sending it to a separate grading process that runs in the background and reports the result back a few seconds later. Students can usually attempt a question several times.

### Who the users are

The course distinguishes three kinds of people:

| Role | What they can do |
|---|---|
| **Student** | Read content, attempt exercises, see their own results |
| **Guest** | Read content only |
| **Admin / Instructor** | Everything, plus edit the course material |

Permissions are attached to roles, and the platform already has a permission specifically meant for staff-level monitoring views. We will reuse it rather than inventing our own notion of "instructor".

---

## 3. The problem

The professor cannot currently answer basic questions about how the class is doing. For example:

- How many students have reached Module 4?
- How many have actually finished it?
- Which exercises are taking students the most attempts?
- Which students have not been active for two weeks?
- Is the class as a whole falling behind?
- Is a particular quiz too hard, or is the question simply badly worded?

CAT-SOOP does have two small staff views, but they only work for **one page at a time** and they recompute everything from scratch every time the page is opened. With 250 students they are already slow. There is nothing that gives a view of the whole course.

So the gap is not that the data does not exist. The gap is that **nothing collects it, organises it, or presents it.**

---

## 4. What we found in our investigation

Before designing anything, we studied how CAT-SOOP stores its data. The findings below shape every decision we made afterwards.

### Finding 1 — Student activity is already recorded

Every time a student submits an answer, checks an answer, saves work, or reveals a solution, CAT-SOOP writes a record of it. That record includes what the student did, when they did it, which questions were involved, and — depending on the question type — the score.

This is very good news. It means we do not have to modify the platform to start collecting most of what we need. **The data is already being written; it is simply never read for analysis.**

### Finding 2 — There is a record of "current state", but it has no history

Alongside the activity records, CAT-SOOP keeps a summary of where each student currently stands on each page: their latest score, how many attempts they have used, when they last submitted.

However, this summary is **overwritten** each time, not appended to. It always shows the present moment and never the past. So it is useful for a quick check, but it cannot tell us how a student's score improved across attempts. For that we must use the activity records from Finding 1.

### Finding 3 — Automatically graded scores arrive separately

For most question types, submitting an answer does not immediately produce a score. The answer is queued, a separate grading process picks it up, and the result is written elsewhere with a reference number linking it back to the submission.

This means reconstructing a complete picture of "student X attempted question Y and scored Z" requires **combining two sources of information**. Our system must handle the case where a submission has been recorded but the score has not arrived yet, and must never treat "not graded yet" as "scored zero".

### Finding 4 — Page visits are not recorded at all

This is the one genuine gap. CAT-SOOP records what students *submit*, but not what they *open*. When a student loads a module and reads it without answering anything, nothing is written down.

This directly blocks the professor's most important question — *how many students have reached Module 4?* — because "reached" means "opened", and opening is invisible today.

### Finding 5 — We can fix that without modifying CAT-SOOP

CAT-SOOP supports **plugins**: small pieces of code that the platform runs automatically at defined points during a request, without any change to the platform's own source code. Plugins live in the course's own folder.

One of these points runs after a page has finished rendering — and, usefully, it runs only for real page views and not for the background submissions the browser sends while a student works. That is exactly the signal we need.

So we can record page visits by adding a small plugin to the course folder. **No file in CAT-SOOP itself has to be edited.** This was the most important thing we established, because it turns a risky "we need to modify the platform" project into a safe "we add something beside it" project.

### Finding 6 — The stored data can only be read by Python

CAT-SOOP stores its records in a Python-specific binary format, optionally compressed and optionally encrypted. It is not a normal database, and no external tool — a reporting tool, a different programming language, a database connector — can read it directly.

This settles a question we would otherwise have debated for weeks: our data extraction component **must be written in Python and must ask CAT-SOOP itself to read the data**, using the same functions the platform uses internally. We are not allowed to open its files ourselves, and we should not try.

---

## 5. Proposed design

### The overall idea

We will add the analytics dashboard **to CAT-SOOP as new pages inside the course**, rather than building a separate website beside it.

There is one difficulty to solve first. Analytics questions like "average score across all modules for all students" require looking at a large amount of data at once. Doing that while a user waits for a page to load would be slow, and would compete with students trying to submit answers — we can already see CAT-SOOP's two existing staff views suffering from exactly this problem.

The solution is to **separate calculating from displaying**. A background program calculates all the figures on a schedule and stores the results. The pages inside CAT-SOOP then only read those stored results, which is fast enough to serve like any other course page.

This gives us the best of both: the dashboard is genuinely part of the course site, with no separate login or address to maintain, and none of the expensive work happens while anyone is waiting.

### The pipeline

The analytics dashboard is **part of CAT-SOOP, not a separate website**. It is a set of new pages added to the course, reachable at ordinary CAT-SOOP addresses, using CAT-SOOP's own login and permissions.

What sits outside CAT-SOOP is only the part that does the heavy calculation, and that runs on a schedule in the background rather than while anyone is waiting:

```
  ┌──────────────────────── CAT-SOOP ────────────────────────┐
  │                                                          │
  │   Course pages          Analytics pages   ← we add these │
  │   /digital-systems      /digital-systems/analytics       │
  │        │                          ▲                      │
  │        │ activity records         │ reads summaries      │
  └────────┼──────────────────────────┼──────────────────────┘
           │                          │
           ▼                          │
     ┌───────────┐            ┌───────────────┐
     │ Extractor │──────────► │   Analytics   │
     │ (nightly, │            │   Database    │
     │  offline) │            │               │
     └───────────┘            └───────────────┘
```

The important separation is **when** work happens:

- **Slow work happens offline.** Reading every student's history and calculating averages, completion rates and difficulty is done by the extractor on a schedule, when nobody is waiting.
- **Fast work happens in the page.** When the professor opens the dashboard, the page reads a handful of already-calculated summary rows and draws them. It never recalculates anything.

This is what makes it safe to serve the dashboard from inside CAT-SOOP. The concern with putting analytics in the course web server is that a heavy query would block students trying to submit answers. Because the numbers are already computed, opening the dashboard is no more expensive than opening any other course page.

### The four components

**1. Page-visit plugin**
A small piece of code placed in the course's folder. It notes when a student opens a module. It records one visit per module per session rather than every single page load, so the data stays meaningful and the volume stays low.

**2. Extractor**
A Python program that runs on a schedule, outside the web server. It asks CAT-SOOP for the student list, the course structure, and all activity since the last time it ran, then calculates the summaries. It remembers where it stopped, so each run only processes what is new instead of re-reading the entire history. It also matches up submissions with the scores that arrived later from the background grader.

**3. Analytics database**
Holds the organised data: students, modules, questions, attempts, visits, and the pre-calculated summary tables. Kept separate from CAT-SOOP's own storage so nothing we do can affect the running course. If it were deleted, re-running the extractor would rebuild it.

**4. Analytics pages inside CAT-SOOP**
New pages added to the course folder. They read the summary tables and present them as charts and tables. Because they are ordinary CAT-SOOP pages, they automatically get the site's appearance, navigation, login and permission checks without us building any of it.

### The routes

CAT-SOOP decides what page to show based on the folder structure of the course, so adding pages means adding folders. The analytics pages live in a folder called `analytics` inside the course:

| Address | Page |
|---|---|
| `/digital-systems` | Course home (existing) |
| `/digital-systems/analytics` | Overview — the whole class at a glance |
| `/digital-systems/analytics/modules` | Module-by-module breakdown |
| `/digital-systems/analytics/questions` | Question difficulty |
| `/digital-systems/analytics/students` | Student list, searchable |
| `/digital-systems/analytics/students/<id>` | One student in detail |
| `/digital-systems/analytics/attention` | Students who may need help |

Alongside these sit a few addresses that return data rather than a page, used by the charts to fetch numbers without reloading the whole page — for example when the professor changes a filter or a date range.

Because the address always begins with the course, **everything is automatically scoped to that course.** The page knows which course it belongs to from where it sits, so there is no default course anywhere in the system and no possibility of one course's figures appearing under another's.

A server can host many courses, and each simply gets its own copy of the folder:

```
courses/
├── digital-systems/
│   ├── 01.number-systems/
│   ├── 02.boolean-algebra/
│   └── analytics/          ← dashboard for Digital Systems
└── signals-systems/
    ├── 01.fourier/
    └── analytics/          ← dashboard for Signals & Systems
```

### How access is controlled

This is where integrating into CAT-SOOP pays off most.

CAT-SOOP works out who the visitor is and what they are allowed to do *before* it builds the page. Our analytics pages simply ask it. There is no separate login, no token to pass between systems, and no second list of who counts as an instructor.

We place a single configuration file in the `analytics` folder that checks the visitor's permissions. Because CAT-SOOP passes settings down from a folder to everything inside it, **that one file protects every analytics page**, including any we add later. A student or guest reaching any of these addresses is refused by CAT-SOOP itself before our code runs.

Permission is also naturally **per course**. A TA for Digital Systems opening `/signals-systems/analytics` is refused, because CAT-SOOP evaluates permissions for the course the address names. Teaching one course grants nothing anywhere else.

The link into the dashboard is an ordinary entry in the course menu, shown only to staff.

### Handling multiple runs of the same course

Courses repeat. Digital Systems in 2026 and 2027 are the same course but different groups of students, and their data must never be mixed.

CAT-SOOP already handles this by giving each run its own folder, so they are separate courses as far as the platform is concerned — and therefore separate as far as we are concerned. Our database keeps them separate too, while recording that they are two runs of the same underlying course. That makes it possible later to compare one year against another — for example, whether a module became easier after the professor rewrote it — without ever mixing two cohorts in a single figure.

### What we gain, and what we give up

Building inside CAT-SOOP rather than as a separate site is the right choice, but it is a genuine trade-off and we want to state both sides.

**What we gain**

- Login, permissions, appearance and navigation all come for free. This removes a substantial and error-prone piece of work.
- Our idea of who is an instructor cannot drift out of step with CAT-SOOP's, because we do not have one.
- No extra web server, no reverse proxy configuration, no second address to maintain.
- The result is a genuine CAT-SOOP extension. Another course can adopt it by copying a folder, which is a far better outcome than a tool that only works for one course.

**What we give up**

- We cannot use a modern frontend framework. CAT-SOOP pages are built on the server and enhanced with plain JavaScript, so features like sorting a large table need to be written more manually than they would in a purpose-built application. We consider this acceptable, and it keeps the project's technology consistent with the platform we are extending.
- The database driver must be installed alongside CAT-SOOP. This is a small addition to the course server's setup, which we will document.
- We must be disciplined about keeping the pages fast. If a page ever needed to calculate something expensive, it would compete with students submitting answers. Our rule is that **analytics pages only read pre-calculated summaries** — never raw activity — and we will treat any violation of that rule as a defect.

### Different landing pages for different roles

**Yes — this is possible, and we are including it.** It was not in our original outline, and it is a sensible addition.

CAT-SOOP already supports this. When someone opens a course page, the platform works out who they are *before* deciding what to put on the page, so the same address can show different content to different people. The course already defines roles and what each is allowed to do — and a role of **Guest** is in fact the platform's default for anyone not explicitly enrolled.

What each role would land on:

| Role | Lands on | Sees |
|---|---|---|
| **Guest** | Course front page | Course description, syllabus, sample material, and an invitation to enrol. No exercises, no progress, no analytics. |
| **Student** | Personal course home | Their own progress through the modules, what to work on next, recent scores, upcoming due dates. |
| **Admin / Instructor** | Staff home | Class-wide summary, a link into the full analytics dashboard, and shortcuts to editing course content. |

Two things are worth separating here, because they are different pieces of work:

**Inside CAT-SOOP** — making the course home page show different things per role. This is done by editing the course's own content files to check the visitor's permissions and show the appropriate section. It is a small piece of work and needs no change to the platform. It is essentially course authoring.

**Inside the dashboard** — making sure only staff can reach it at all. This is a security requirement rather than a convenience, and is covered in Section 10.

We are treating the guest landing page as a **genuinely useful piece of the project rather than decoration**, because it becomes important if the professor makes the course public. At that point the front page is what a stranger sees first, and it needs to explain what the course is and how to join — a job nothing currently does.

One design point we want to flag: a student's landing page is effectively a *personal* analytics view — their own progress, their own weak spots, how they compare to the class average. That data is already being calculated for the instructor dashboard. Serving a small, private, individual version of it back to each student is a modest amount of extra work for a real benefit, and we are proposing it as a second-phase item rather than assuming it.

---

## 6. What the dashboard will show

We plan four levels of analysis, moving from the whole class down to a single question.

### Course overview

The starting screen. Total students, how many are active, overall average score, overall progress through the course, and a bar showing how many students have completed each module.

```
Digital Systems — Spring 2026

Students        250
Active          231
Average score    74%
Completion       68%

Module 1  ████████████████████░  94%
Module 2  ██████████████████░░░  87%
Module 3  ███████████████░░░░░░  75%
Module 4  ████████████░░░░░░░░░  61%
Module 5  ████████░░░░░░░░░░░░░  42%
```

### Module analysis

For each module: how many students opened it, how many started working on it, how many finished it, the average score, the spread of scores, the average number of attempts, and how many students appear to be stuck.

An important distinction we will maintain carefully:

- **Opened but never started** — students look at the module and back away. The material may look intimidating.
- **Started but never finished** — students engaged and struggled. The material is genuinely hard.

These call for different responses from the instructor, so we will never merge them into a single "did not complete" number.

### Question analysis

For each exercise: how many students attempted it, what fraction got it right on the first try, how the average score improves across attempts, how many attempts a typical student needs to succeed, and how many gave up.

The "improvement across attempts" view is particularly useful:

```
Question 17 — K-map reduction

Attempt 1 average   42%
Attempt 2 average   61%
Attempt 3 average   78%

Typical attempts to succeed: 3
```

This pattern suggests students eventually understand the concept but find the first encounter difficult — a different situation from a question where the average never improves, which usually means the question itself is unclear.

### Student view

For an individual student: overall score, which modules they have reached and completed, where they currently are, how many attempts they have made, when they were last active, and how they compare to the class average.

### Attention list

A list of students who may need help, with the reason clearly stated. For example:

```
Student 22110073 — attention required

  · No activity for 16 days
  · Currently at Module 3; most of the class is at Module 6
  · Overall score 41% (class average 68%)
  · 7 attempts on "K-map reduction", not yet passed
```

We deliberately avoid a machine-learning model here. With a few hundred students and no labelled data showing who genuinely needed help in the past, a model would be guessing, and the professor would have no way to judge whether a flag was trustworthy. Simple rules that state their own reasoning are more useful and more honest. If the system is later used across several years of course data, revisiting this becomes reasonable.

---

## 7. Measuring difficulty fairly

The professor wants to know which modules and questions are too hard. This sounds simple but has a trap in it, so it is worth explaining our approach.

**The trap:** a question can look difficult simply because of *who* attempted it. Late in a semester, strong students have already finished, so the students still working through the later modules are disproportionately the ones who are struggling. Judging the question by their scores alone makes it look harder than it is.

**Our approach** is to describe difficulty along four separate dimensions instead of collapsing it into one number:

| Dimension | What it tells the instructor |
|---|---|
| First-attempt success rate | How hard the question is on first encounter |
| Attempts needed to succeed | Whether it is hard, or just tedious and fiddly |
| Failure rate | Whether students eventually get past it at all |
| Abandonment rate | Whether students give up rather than persist |

We anchor on the **first-attempt success rate**, because it does not change depending on how many retries the course happens to allow — making it comparable across questions.

To deal with the selection-bias trap, we group students by their overall performance elsewhere in the course and report the success rate separately for each group. If a question is hard for the strongest students too, it is genuinely hard. If only the weakest students struggle with it, the cohort explains it.

We also plan a check for **badly designed questions**, which is a different thing from hard questions. If strong students do *worse* on a question than weak students, something is wrong with it — ambiguous wording, a wrong expected answer, or a misleading phrasing. This is a well-established measure in educational assessment and it is straightforward to calculate. We expect it to be one of the more valuable outputs of the project.

Finally, we will not report statistics for questions with very few attempts. A question attempted by six students tells us nothing, and showing a difficulty figure for it would invite the professor to act on noise.

---

## 8. Technology choices

| Part | Choice | Reason |
|---|---|---|
| Extractor | Python | Required — CAT-SOOP's data can only be read from Python |
| Database | PostgreSQL | Reliable, well understood, handles the joins and summaries this project needs, and comfortably handles the data volumes involved |
| Dashboard pages | CAT-SOOP's own page format | The dashboard is part of the course, so it is written the way every other CAT-SOOP page is written. This is what gives us login, permissions and site appearance for free |
| Charts and tables | Plain JavaScript with a charting library | CAT-SOOP pages do not use a frontend framework, so we follow the platform's approach. A small charting library loaded by the page covers the visualisations |
| Scheduling | A scheduled job (cron or equivalent) | We have three periodic tasks. A job-queue system would add infrastructure without adding value at this size |
| Deployment | Alongside the existing CAT-SOOP installation | No extra web server or proxy configuration. The database and the scheduled job are the only new pieces to install |

We deliberately did **not** choose a frontend framework such as React. It would be the natural choice for a standalone dashboard, but CAT-SOOP pages are built on the server and enhanced with plain JavaScript. Introducing a framework would mean fighting the platform rather than extending it, and would rule out the seamless integration that is the point of this approach.

We chose PostgreSQL over specialised analytics databases deliberately. Those are designed for hundreds of millions of rows; our largest realistic case is around ten million, which PostgreSQL handles without difficulty. Choosing a heavier system would mean spending the semester learning to operate it instead of building the analytics.

---

## 9. Designing for scale

The course has 250 students today but may become public. We are designing so that growth does not require a redesign:

- **Nothing is hard-coded.** Not the course name, not the number of students, not the semester, not the number of modules. The system reads all of it from CAT-SOOP.
- **Multiple courses and multiple semesters are supported from the start.** We separate the idea of a *course* from a *particular offering of that course*, so that Spring 2026 and Spring 2027 can be compared without their data mixing.
- **We calculate ahead of time.** Summaries are computed when data is extracted, not when the dashboard is opened. Opening the dashboard is fast no matter how much history exists.
- **We process only what is new.** Each run remembers where it stopped, so a routine update handles only recent activity rather than re-reading everything.
- **The browser never receives raw data.** Charts and tables are built from summaries computed on the server, so a class of 10,000 does not mean sending 10,000 records to the browser.

Our realistic target is a few thousand students across several courses on a single ordinary server. We are not designing for millions of users, because nothing in this project justifies that and doing so would make it impossible to finish.

---

## 10. Privacy and access control

The dashboard contains students' academic records, so access control is a core requirement rather than an afterthought.

- **Only instructors can see it.** Access is decided by asking CAT-SOOP whether the person is staff for that course. We do not maintain our own list of who is an instructor, so our rules cannot drift out of step with the platform's.
- **Students and guests are refused.** They receive nothing, not a limited view.
- **Hiding the link is not the same as blocking access.** The Analytics menu item is hidden from students and guests, but that is only a convenience. The dashboard checks permissions on every single request, so typing the address directly gets a student nowhere. We mention this explicitly because hiding a link and calling it security is a common and serious mistake.
- **We inherit CAT-SOOP's checks rather than reimplementing them.** Because the analytics pages are CAT-SOOP pages, the platform performs its normal authentication and permission check before our code runs at all. There is no second login, no token passed between systems, and no separate list of who counts as staff that could fall out of date.
- **Permission is per course, not per server.** Being staff on one course grants no access to any other course's analytics. Every request names the course it concerns, and we ask CAT-SOOP about that specific course. This matters on a shared institute server where several courses run side by side.
- **We do not store answers.** The activity records contain the actual text students submitted. None of our metrics need it, so we discard it during extraction. This removes a large category of privacy risk for essentially no cost.
- **Small groups are suppressed.** Statistics for any group of fewer than five students are hidden, so section-level views cannot be used to work out an individual's results.
- **Credentials are kept out of the code** and supplied through the environment.
- **If the course becomes public**, it will include people who never agreed to be studied. We plan to use pseudonymous identifiers in aggregate views, with real identities visible only to that course's own instructors.

---



## 11. Scope

### First phase — the working system

1. Local CAT-SOOP installation with a test course and test students
2. Page-visit plugin
3. Extractor pulling students, structure, activity and scores
4. Analytics database with summary tables
5. Progress tracking — reached, started, completed
6. Course overview, module, student and question views
7. Basic activity timeline
8. Analytics pages added to the course, protected by CAT-SOOP's permission system
9. Role-based landing pages for guest, student and instructor
10. Staff-only Analytics link in the course menu

### Second phase — the valuable additions

11. Attention list with stated reasons
12. Difficulty analysis across the four dimensions
13. Badly-designed-question detection
14. Trends over time
15. Export to CSV
16. Advanced filtering and search
17. Incremental updates and performance testing at larger scale
18. Personal progress view for students on their own landing page

We consider the first phase the minimum that demonstrates the project works, and the second phase the part that makes it genuinely useful to the professor.

---

## 12. Plan of work

| Weeks | Focus | Completed when |
|---|---|---|
| 1 | Get CAT-SOOP running locally | We can find and read the record created by a submission |
| 2 | Test course and test students | The driver script runs the whole cohort through the course |
| 2–3 | Page-visit plugin | Visits are recorded; submissions do not create false visits |
| 3–4 | Extractor and database | Our attempt counts match CAT-SOOP's own for every student |
| 5–6 | Analytics calculations | Every summary can be rebuilt from scratch and matches hand-checked values |
| 5–7 | Analytics pages inside CAT-SOOP | The professor's account can open them from the course menu; a student typing the address directly is refused |
| 7 | Role-based landing pages | Guest, student and instructor each land somewhere different |
| 8–9 | Difficulty and attention list | The deliberately flawed question is correctly identified |
| 9 | Trends and export | Exported data matches what is shown on screen |
| 10 | Incremental updates and scale test | A 5,000-student test course updates in under a minute |
| 11 | Security and multi-course support | Two courses run side by side without interfering |
| 12 | Deployment and documentation | Someone outside the team can set it up from the README |

The order is deliberate. The two things that could sink the project — getting CAT-SOOP running, and proving the extracted data is correct — are handled first, so that if something is wrong we find out in week 3 rather than week 9.

---




