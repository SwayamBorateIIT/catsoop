# Course Analytics Dashboard for CAT-SOOP

**Course:** Software Tools and Techniques for CSE

**Institute:** IIT Gandhinagar

**Project type:** Semester project — extending an existing open-source system
**Submission:** 1 — Problem understanding and proposed design

---

## 1. Summary

CAT-SOOP is the online course platform used for the Digital Systems course at IIT Gandhinagar. Students read the course material on it, solve exercises, and take quizzes that are graded automatically. Around 250 students use it today, and the professor is considering opening the course to the public.

The platform does a good job of delivering content and grading answers. What it does not do is tell the instructor what is actually happening across the class. There is no way to see how far students have progressed, which topics are causing trouble, or which students have quietly stopped working.

We propose to build a **course analytics dashboard** for instructors. It will read the activity data CAT-SOOP already produces, organise it into a form suited to analysis, and present it as a set of clear views — course overview, module-wise progress, question difficulty, individual student progress, and a list of students who may need attention.

The system will sit alongside CAT-SOOP rather than inside it. CAT-SOOP remains the authoritative source of all data, and we will change almost nothing in it.

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

### Summary of what is available

| Information | Status |
|---|---|
| Student list, roles, sections | Already available |
| Student names and emails | Already available |
| Course structure — modules and questions | Already available |
| Question details — points, type, grading mode | Already available |
| Submissions, with timestamps | Already available |
| Attempt counts | Already available |
| Scores (all three grading modes) | Already available |
| Answer-reveal events (a "giving up" signal) | Already available |
| **Module / page visits** | **Needs our plugin** |
| Time spent reading a page | Would need extra work — not planned |
| Module completion, class rank, difficulty | Calculated by us from the above |

The conclusion is encouraging: **almost everything is already there.** The project is mainly about organising and presenting it, not about instrumenting the platform.

---

## 5. Proposed design

### The overall idea

We will build a **separate application that reads from CAT-SOOP and presents analytics**, rather than building the analytics inside CAT-SOOP.

The reasoning is simple. Analytics questions like "average score across all modules for all students" require looking at a large amount of data at once. If we computed that inside CAT-SOOP while a user waits for a page to load, it would be slow — and we can already see the existing staff views suffering from exactly this problem. Instead, we compute the numbers ahead of time, store the results, and serve them instantly when the dashboard is opened.

### The pipeline

```
   CAT-SOOP              (unchanged — the source of truth)
      │
      │  reads activity records
      ▼
   Extractor             (Python; asks CAT-SOOP for its data)
      │
      │  cleans and organises
      ▼
   Analytics Database    (PostgreSQL; organised for analysis)
      │
      │  pre-calculated summaries
      ▼
   Analytics API         (serves numbers as data)
      │
      ▼
   Dashboard             (what the professor actually sees)
```

Alongside this, one small plugin sits inside the course folder in CAT-SOOP and records page visits into CAT-SOOP's own logs, where our extractor picks them up along with everything else.

### The five components

**1. Page-visit plugin**
A small piece of code placed in the course's folder. It notes when a student opens a module. It records one visit per module per session rather than every single page load, so the data stays meaningful and the volume stays low. This is the only thing we add inside CAT-SOOP.

**2. Extractor**
A Python program that runs on a schedule. It asks CAT-SOOP for the student list, the course structure, and all activity since the last time it ran. It remembers where it stopped, so each run only processes what is new instead of re-reading the entire history. It also matches up submissions with the scores that arrived later from the background grader.

**3. Analytics database**
A PostgreSQL database holding the organised data: students, modules, questions, attempts, visits, and pre-calculated summary tables. Kept deliberately separate from CAT-SOOP's own storage so that nothing we do can affect the running course.

**4. Analytics API**
A small web service that answers questions like "give me the module-wise summary for this course". It reads the pre-calculated summaries rather than recomputing anything, so responses are fast regardless of class size. It also checks that whoever is asking is actually an instructor — by asking CAT-SOOP, so that our idea of who is staff can never disagree with CAT-SOOP's.

**5. Dashboard**
The web interface the professor uses. Charts, tables, filters, search, and the ability to click from a summary down to the detail behind it. Data is fetched a page at a time, so the interface stays responsive whether the course has 250 students or 10,000.

### Why we keep CAT-SOOP as the source of truth

We never write anything back into CAT-SOOP, and we never modify its data. If our database were deleted entirely, we could rebuild it from scratch by re-running the extractor. This is an important safety property for a system that has to run alongside a live course.

### Where the dashboard lives — one website, not two

Although the dashboard is a separate program, it will appear to users as **part of the same website**. The professor should not have to remember a second address or log in a second time.

This is achieved with a **reverse proxy** — a standard piece of web server configuration that sits in front of both programs and decides which one handles each address:

```
                         iitgn.ac.in
                               │
                       ┌───────┴────────┐
                       │  reverse proxy │   (nginx or Caddy)
                       └───────┬────────┘
              ┌────────────────┴────────────────┐
              ▼                                 ▼
   anything ending in /analytics          everything else
              │                                 │
          Dashboard                          CAT-SOOP
```

### The address always names the course

A server can host many courses, so there can be no single "the dashboard". **Analytics live underneath the course they belong to**, mirroring how CAT-SOOP already organises everything:

| Address | What it is |
|---|---|
| `/digital-systems` | The Digital Systems course — CAT-SOOP |
| `/digital-systems/analytics` | Analytics for Digital Systems |
| `/digital-systems/analytics/modules` | Module breakdown for that course |
| `/signals-systems` | A different course — CAT-SOOP |
| `/signals-systems/analytics` | Analytics for *that* course |
| `/analytics` | Course picker, if a person teaches several |

This reads naturally, and it matches CAT-SOOP's own convention where the course name always comes first. It also means the menu link inside a course can simply point at `COURSE/analytics`, using CAT-SOOP's existing shorthand for "the current course" — so the same link works in every course without being edited.

Two consequences worth stating, because they follow from this choice rather than being extra work:

- **The course is never assumed.** There is no default course anywhere in the system. Every page, every stored figure and every request carries the course it belongs to.
- **Permission is checked per course.** A TA for Digital Systems opening `/signals-systems/analytics` is refused, because we ask CAT-SOOP whether *this* person is staff for *that* course. Teaching one course grants nothing anywhere else.

One practical detail: the proxy rule reserves the name `analytics` as the last part of a course address. A course would therefore not be able to have its own module called "analytics". This is a small, acceptable restriction, and we will note it in the deployment documentation rather than let a future instructor discover it by accident.

### Handling multiple runs of the same course

Courses repeat. Digital Systems in 2026 and in 2027 are the same course but different groups of students, and their data must never be mixed.

CAT-SOOP handles this by giving each run its own folder — so `/digital-systems-2026` and `/digital-systems-2027` are separate courses as far as the platform is concerned, and therefore separate as far as we are concerned. Our design keeps them separate in storage too, while still recording that they are two runs of the same underlying course. That makes it possible to compare one year against another later on — for example, whether a module became easier after the professor rewrote it — without ever mixing the two cohorts in a single figure.

### Why not the alternatives

We considered two other arrangements and rejected both. A **separate address** (`analytics.iitgn.ac.in`) would count as a different site to the browser, requiring extra configuration to share the login, and it would lose the natural course-first structure. Serving the dashboard **from inside CAT-SOOP itself** would mean every heavy analytics calculation running inside the course web server, competing with students trying to submit answers — exactly the performance problem we are trying to avoid.

### How the professor gets in

The route into the dashboard is a normal menu item inside CAT-SOOP:

1. The professor is already logged into CAT-SOOP as usual.
2. An **Analytics** link appears in that course's menu — visible only to staff of that course.
3. Clicking it hands over to the dashboard along with a short-lived pass confirming who they are, which course they are asking about, and that CAT-SOOP considers them staff there.
4. The dashboard checks that pass and opens on that course.

There is no separate account, no separate password, and no second login screen. CAT-SOOP stays the only place that decides who is an instructor, so our permissions can never disagree with the platform's.

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
| API | FastAPI (Python) | Same language as the extractor so we share code; produces documentation automatically, which helps when different team members build the frontend |
| Dashboard | React | The interface needs sorting, filtering, searching and drill-down across thousands of rows, which is more than a static page can do well |
| Scheduling | Built-in scheduler | We have three periodic jobs. A heavier job-queue system would add infrastructure without adding value at this size |
| Deployment | Docker Compose | One command brings up the whole system, so the professor or a future team can reproduce it easily |

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
- **Sharing one address does not mean sharing one permission.** Putting the dashboard and CAT-SOOP behind the same address makes signing in seamless, but each request to the dashboard is still authorised independently.
- **Permission is per course, not per server.** Being staff on one course grants no access to any other course's analytics. Every request names the course it concerns, and we ask CAT-SOOP about that specific course. This matters on a shared institute server where several courses run side by side.
- **We do not store answers.** The activity records contain the actual text students submitted. None of our metrics need it, so we discard it during extraction. This removes a large category of privacy risk for essentially no cost.
- **Small groups are suppressed.** Statistics for any group of fewer than five students are hidden, so section-level views cannot be used to work out an individual's results.
- **Credentials are kept out of the code** and supplied through the environment.
- **If the course becomes public**, it will include people who never agreed to be studied. We plan to use pseudonymous identifiers in aggregate views, with real identities visible only to that course's own instructors.

---

## 11. Proof of concept

Before building the full system, we will prove the idea end to end on a local installation.

**Plan:**

1. Install CAT-SOOP locally and get it running.
2. Create a small test course of six modules with a mixture of question types and difficulties.
3. Create around 60 test students following five behaviour patterns.
4. Run a script that logs in as each student and works through the course over the web, exactly as a real student would.
5. Extract the resulting data, load it, and display it.

**The five student patterns:**

| Pattern | Behaviour | Should appear as |
|---|---|---|
| A | Completes everything, mostly first attempt | Top performer, no flags |
| B | Many attempts, scores steadily improve | Improvement visible across attempts |
| C | Stops entirely after Module 2 | Flagged as inactive and behind |
| D | Repeatedly fails one quiz, continues elsewhere | Flagged for repeated failure; makes that quiz look hard |
| E | Opens Module 4 but never attempts it | **Reached but not started** |

Pattern E is the important one. It is the only pattern that cannot be detected without our page-visit plugin. If the dashboard can tell student E apart from a student who never opened Module 4 at all, the instrumentation works.

We will use 60 students rather than 5 because five is not enough to produce meaningful averages or to test our rule about hiding statistics for small samples.

**Crucially, we will generate this data by actually using the platform over the web**, not by writing data files ourselves. Fabricated data would prove nothing. The whole point is to show that the data CAT-SOOP genuinely produces is sufficient.

---

## 12. Scope

### First phase — the working system

1. Local CAT-SOOP installation with a test course and test students
2. Page-visit plugin
3. Extractor pulling students, structure, activity and scores
4. Analytics database with summary tables
5. Progress tracking — reached, started, completed
6. Course overview, module, student and question views
7. Basic activity timeline
8. Reverse proxy setup so CAT-SOOP and the dashboard share one address
9. Role-based landing pages for guest, student and instructor
10. Sign-in handover from CAT-SOOP, with no second login

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

## 13. Plan of work

| Weeks | Focus | Completed when |
|---|---|---|
| 1 | Get CAT-SOOP running locally | We can find and read the record created by a submission |
| 2 | Test course and test students | The driver script runs the whole cohort through the course |
| 2–3 | Page-visit plugin | Visits are recorded; submissions do not create false visits |
| 3–4 | Extractor and database | Our attempt counts match CAT-SOOP's own for every student |
| 5–6 | Analytics calculations | Every summary can be rebuilt from scratch and matches hand-checked values |
| 5–7 | API and dashboard | The professor's account can open it; a student account cannot |
| 7 | Same-address setup and role-based landing pages | Guest, student and instructor each land somewhere different; the professor reaches analytics from the course menu without logging in again |
| 8–9 | Difficulty and attention list | The deliberately flawed question is correctly identified |
| 9 | Trends and export | Exported data matches what is shown on screen |
| 10 | Incremental updates and scale test | A 5,000-student test course updates in under a minute |
| 11 | Security and multi-course support | Two courses run side by side without interfering |
| 12 | Deployment and documentation | Someone outside the team can set it up from the README |

The order is deliberate. The two things that could sink the project — getting CAT-SOOP running, and proving the extracted data is correct — are handled first, so that if something is wrong we find out in week 3 rather than week 9.

---

## 14. Risks

| Risk | How we handle it |
|---|---|
| CAT-SOOP does not run on current Python versions | Use an older Python version in a dedicated environment; resolve in week 1 |
| The live installation stores data in encrypted form, making our fast update method unavailable | Confirm with the professor early; we have a fallback approach |
| Questions without explicit names could shift identity if the professor reorders a page | Detect this situation and warn, rather than silently reporting wrong history |
| Our plugin causes an error on a live page | Write it defensively and test thoroughly on our own installation first |
| Scope creep into unnecessary features | Google Classroom integration, account automation and machine learning are explicitly out of scope |

### Honest limitations

We want to state clearly what this system will *not* do:

- It cannot measure how long a student spends reading. We can record that a page was opened, not that it was read.
- It measures performance on graded exercises, not learning. A module can look easy because it is well taught or because its questions are trivial, and we cannot tell those apart.
- The attention list highlights students worth looking at. It does not predict who will fail, and we will label it that way in the interface so it is not over-trusted.

---

## 15. Expected outcome

At the end of the semester we expect to deliver:

1. A working analytics dashboard, running against a real CAT-SOOP installation
2. A data pipeline that keeps it up to date automatically
3. A small plugin that adds page-visit tracking to CAT-SOOP without modifying it
4. Documentation allowing the professor, or a future team, to deploy and extend it
5. This design report, together with our findings about how CAT-SOOP stores its data

The wider value is that none of this is specific to Digital Systems. Any course running on CAT-SOOP could use the same system by pointing it at a different course folder — which matters if the platform is adopted more widely at the institute.

---
