# Project Overview 🚀
emiT is the full-scale implementation for my initial exploration of building a personalized time-tracking software, [emiHD](https://github.com/hector-ledesma/emiHD). This project is a culmination of the desire for a solution tailored to my specific workflow, and the desire to continuously improve my software engineering skills.
**Motivation and Learning**
emiT aims to be a centralized backend powering future client implementations. It provides a unique opportunity to combine precise time tracking with flexible tagging and data association for my specific workflow.

My primary long-term goal is to deepen my C++ expertise and how to build performant, lower-level software. This design document reflects my current understanding and serves as a learning tool in itself. As the project progresses, I’ll strive to improve my technical writing and planning skills alongside the technical implementation.

**Workflow Insights**
To better understand the project structure, here’s some background on my workflow:
- **Time Blindness and Fly-Over Days**: As someone who struggles with time blindness, I like to begin a stopwatch when I wake up. This stopwatch allows me to track my “logical day”, as my erratic sleep schedule sometimes leads me to effectively “skip” a calendar day from my perspective. The stopwatch allows me to keep a sense of elapsed time within my logical day, regardless of the calendar date.
- **Root Timer**: This is how emiT addresses these challenges. A Root Timer represents a “logical day” associated with a specific date. All other timers within emiT are nested within a Root Timer, ensuring a cohesive representation of my time usage, even across irregular schedules.


## A peep into my workflow
For the general structure the project will take to make sense, I’d like to share some insight into my workflow.

As someone who struggles with time blindness, I like to start a Stopwatch on my Apple Watch from the moment I wake up. This allows me to see what “hour into my day” I’m in. My sleeping schedule tends to be relatively erratic, sometimes leading into what I call “fly-over days”. A “fly-over” day happens when my sleeping schedule is so erratic, that I may wake up on Friday but by the time my day ends, I’ll next wake up on Sunday. Here, Saturday was simply part of my Friday, and it’s effectively a “fly-over” day.

This takes shape in the form of a **Root Timer**. A root timer is, in essence, a day-tracker timer associated with a specific “date”. All timers will automatically be children of a specific root timer.
# Design Document

## Tech Stack

| Functionality          | Technology                                                                                                                                                                                                                                                 |     |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --- |
| Programming Language   | The major goal of this project is to further my **C++** development abilities for the backend/business logic. As well as expose myself to GUI development using Python. I want to build performant, low-level code for the backend, and a Python frontend. |     |
| Database               | **SQLite**. I’d like for the database to be embedded into the process (at least initially).                                                                                                                                                                |     |
| Raw SQL                | Another major learning opportunity. Manually design/structure all queries.                                                                                                                                                                                 |     |
| IPC                    | **WebSocket** → **Beast (Boost)**. The need for updates to propagate across clients make WebSockets a great choice for communicating with clients.                                                                                                         |     |
| Logging                | **Boost.Log**. Logging for debugging and long-term analysis.                                                                                                                                                                                               |     |
| Testing                | **Catch2**. Header-only, easy-to-learn library for unit testing. **Gcov**  code coverage analysis to ensure test coverage on new functionality.                                                                                                            |     |
| CI/CD                  | Protected `main` branch. GitHub actions monitoring the `integration` branch will create pull requests if tests pass.                                                                                                                                           |     |
| Packaging/Installation | **WiX** for the backend. The backend will ultimately be structured as a Windows service.                                                                                                                                                                       |     |
| Profiling              | **perf** & **Valgrind(Memcheck)**. I’d like to learn to analyze program performance and work on improving performance long term.                                                                                                                           |     |
| Security               | Explore free TLS Certificates. Otherwise, implement a Public-Private key handshake layer to the process of establishing a connection.                                                                                                                      |     |
| Session management     | Maybe not users, but something akin to an Obsidian “vault”.                                                                                                                                                                                                |     |


## Database Logic

### Schemas and Structures
Timestamps will be stored following `ISO 8601` standard definition.[^1]

#### `Root_Timer`
| root_timer_id | date                                                      | start_time | end_time  |
| ------------- | --------------------------------------------------------- | ---------- | --------- |
| primary key   | Primary date this `ROOT_TIMER` is associated with. Index. | timestamp  | timestamp |

#### `Timer`
| timer_id    | root_timer_id               | title  | topic            | description      | start_timestamp | end_timestamp       | state                                         | deleted_at          | pause_duration_total                | duration                             | stopwatch                                      |
| ----------- | --------------------------- | ------ | ---------------- | ---------------- | --------------- | ------------------- | --------------------------------------------- | ------------------- | ----------------------------------- | ------------------------------------ | ---------------------------------------------- |
| primary key | foreign key to `ROOT_TIMER` | String | String, optional | String, optional | timestamp       | timestamp, optional | enum: `PLAY`, `PAUSED`, `STOPPED`, `DELETED` | timestamp, optional | `Integer representing milliseconds` | `Integer` representing milliseconds. | (Optional) INT, Foreign Key Stopwatch_Body(id) | 

##### `Pause_Event`
`TRIGGER` → on parent `Timer` `DELETE`, `DELETE` matching `PAUSE_EVENT`s.

| pause_event_id | timer_id                        | start_time       | end_time                   | Valid   |
| -------------- | ------------------------------- | ---------------- | -------------------------- | ------- |
| primary key,   | foreign key to `TIMER`, indexed | timestamp, index | timestamp, optional, index | boolean |

##### `Stopwatch_Body`

| stopwatch_id | lap_events_list              | target_time | alarm   |
| ------------ | ---------------------------- | ----------- | ------- |
| primary key  | foreign key to `LAP_EVENTS` | timestamp   | boolean | 

###### `Lap_Events`

| lap_id      | stopwatch_id                    | lapse_time |
| ----------- | ------------------------------- | ---------- |
| primary key | foreign key to `STOPWATCH_BODY` | timestamp  |

#### `Tags`

| tag_id      | name   | color              |
| ----------- | ------ | ------------------ |
| primary key | String | String (hex value) | 

##### `root_timer_tags`

| root_timer_id | tag_id      |
| ------------- | ----------- |
| foreign key   | foreign key | 

##### `timer_timer_tags`

| timer_id    | tag_id      |
| ----------- | ----------- |
| foreign key | foreign key |

##### `pause_event_tags`

| pause_event_id | tag_id      |
| -------------- | ----------- |
| foreign key    | foreign key |

##### `lap_event_tags`

| lap_event_id | tag_id      |
| ------------ | ----------- |
| foreign key  | foreign key |
### Database Triggers
`Timer.total_pause_duration` and `Timer.duration` should be automatically recalculated by the database when it receives a new pause event.

## Backend Structure Outline

### `Timer` Customization
- The client will hold the responsibility of sending a "valid" `Timer` over the wire. Meaning, that if the backend receives a `/create_timer` or `/update_timer` request, where the provided `start_time`, `end_time`, `duration`, and `pause_duration_total` don't make a lick of sense, the request will be rejected.
- This means that “editing” and “creation”, essentially boil down to the client creating a “valid” `Timer` object that the backend validates before passing it on to the database.
- **Levels of granularity**: The client should be able to request a fully detailed view of the desired `Timer`. Where it would be able to see and subsequently edit timer events manually. This would allow for extra detail on setting timers and their metadata. This also means the client is responsible for validation of the `Timer` and its related `Pause_Event`s.

### `Timer` validation
A `Timer` is valid if:
- `Timer.start_timestamp` < `Timer.end_timestamp`
- Every `Pause_Event` is valid (`end_time` > `start_time`)
- Every `Pause_Event.start_time` is after its parent `Timer`’s `start_time`.
- Every `Pause_Event.end_time` is before or equal to its parent’s `end_time`.
- No `Pause_Event`s overlap. Meaning, that there is to be a strict ordering where there can be no `Pause_Event` whose `start_time` is earlier than the preceding `Pause_Event`’s `end_time`.
- The previous rules apply to both `valid` and `invalidated` `Pause_Event`, but as separate _sets_. Meaning, we’ll only validate `valid` events against each other, and likewise for `invalid` events. With the exception of checking the `Pause_Event`’s times against its parent’s times for `invalid` `Pause_Event`s.
- Sum of all `valid` `Pause_Event`s = `total_pause_duration`.
- `Timer.duration` = `{end_time - start_time} - total_pause_duration` . Should `total_pause_duration` be an `Integer` representing milliseconds? This would then require casting before calculation.

With these validation heuristics, we can freely overhaul any given `Timer` to match a custom set of `duration` or `time` parameters. The client is responsible for invalidating any `Pause_Event`s that fall outside the validation range and creating new ones to make the Timer valid.

### Networking

#### Client Abstraction
Given the desire to serve to a variety of different clients, we’d benefit from adding a layer of abstraction. Each `Client` will: 
- Define how it will parse messages back over the line.
- Track what messages it’s subscribed to.

# Project Development Outline
## Phase 1 - VCS and CI/CD
### Stage 1 - Repository
- Create the base, empty mono repository for the project.
- Protect `main` to only receive code through `PR`s.
- Create `integration` branch.
- Set up the base directories for the laid-out servers and services and submit it as a `PR` into `main`.
### Stage 2 - GitHub Actions
- Create a default `.yml` configuration file.
- Test the basic CI/CD pipeline → Should be able of creating base directories and pushing them to `integration` and automatically creating the `PR` for `integration → main`. 

## Phase 2 - Containerization & MVP
### MVP Definition:
- **Connection**: WebSocket capabilities. This is how every client will communicate with the backend.
- **Timer**: Basic, `RegularTimer` creation, and stopping.
At its most basic, this is all we need to ensure we’re on the right track. Every single other piece of functionality will be built on top of this.
- **Testing**: This time around, I’ll leave the core functionality of this project for the very end. I want to establish **_from the very beginning_** a robust development structure. Beginning with Test Driven Development.
- **Error Handling**: Robust Error handling is to be implemented from the very beginning.
- **Electron Client**: At this point, the foundation for the Electron program will begin development alongside the backend. The development will be much more iterative than the backend.
### Stage 1 - Docker Images
- Define `Dockerfile` for our backend.
- Keep in mind we’ll have a client packaged together in the future, `docker-compose.yml` could be considered now.
### Stage 2 - CMake
- Basic CMake configuration file.
### Stage 3 - Catch2 Incorporation
- Download and incorporate `Catch2` library files.
- Set up directory structure.
- Write tests for all desired MVP functionality.
### Stage 4 - GitHub Actions
- Define `.yml` steps for GitHub Actions.
- Compose image.
- Build CMake project.
- Run tests.
### Stage 5 - MVP
- Develop MVP functionality.
### Stage 6 - Test Coverage
- Implement test coverage analysis with `Gcov`.
- Add a step to GitHub Actions to perform analysis. Failing 100% coverage should serve as just a warning. This allows us to have a “dumb” script that doesn’t check for comments, and the like for the time being. 

## Phase 3 - Logging
### Stage 1 - Due Diligence
- What are the common debugging levels (`DEBUG`, `INFO`, `ERROR`, `VERBOSE`)
- Read [The Twelve-Factor App](https://12factor.net/)
### Stage 2 - Setup and Tool Knowledge [^2]
- **Boost.Log** setup and tutorials.
### Stage 3 - Implementation
- Device a logging controller structure for integration within the software.
- Implement logging around existing functionality for all log levels.

## Phase 4 - Security
### Stage 1 - Education
- This step will require a “step back” (hehe) to read [Cryptography Engineering: Design Principles and Practical Applications: Ferguson, Niels, Schneier, Bruce, Kohno, Tadayoshi](https://www.amazon.com/Cryptography-Engineering-Principles-Practical-Applications/dp/0470474246). Extremely relevant for Backend and DevOps goals.
### Stage 2 - Implementation
- Implement the concepts learned into a security layer.
- Implement a security structure plan for future clients using different connections.

## Phase 5 - Profiling
### Stage 1 - Setup and Tool Knowledge [^2]
- Set up `gprof` ang `Valgrind`
### Stage 2 - Basic Profiling Structure
- Learn the basics of profiling
- Keep a log of analysis through feature development.
- A proper understanding of how to approach implementing, and actioning on profiling, will likely come after going through [Systems Performance: Enterprise and the Cloud](https://www.amazon.com/dp/0133390098/?coliid=I2ZNFV16I9MB3D&colid=1NW5Y3SM9AA07&psc=1&ref_=list_c_wl_lv_ov_lig_dp_it). So for now, learning how to build a log over time, and  catching the most obvious flaws will be enough.

## Phase 6 - Packaging & Installation
### Stage 1 - Setup and Tool Knowledge [^2]
- Learn basics of packaging and installations using **WiX**
### Stage 2 - Jenkins Consideration
- While GitHub Action is the core of this project’s CI/CD, I’d like to automate valid packaging, locally.

## Phase 7 - Full-Feature
At this point, the entire foundation has been laid out. All that’s left is implementing all the desired functionality, whilst maintaining full consideration for the previous concepts (**Testing**, **Logging**, **Error Handling**, **Security**, **Profiling**). The following table dictates the order of priority for the features.

| Feature                                        |
| ---------------------------------------------- |
| Play/Pause                                     |
| Stop                                           |
| Delete                                         |
| Description                                    |
| Tags                                           |
| Timer Update (Title, Topic, Description, Tags) |
| Custom Timer creation                          |
| Root Timer                                     |
| Stopwatch                                      |
| Discord Rich Presence Integration              |
| Windows Service Integration                    |

## Phase 8 - Clients
The backend is finally complete. All that’s left, is creating all the clients I envision using with this backend.

| Client          | Desired Workflow                                                                                                                                                                                                                                                                                            |
| --------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Electron        | The “face” of the application. This client will be the most feature-rich, as this is how  the back-end will be primarily managed. It will allow checking the entire database. But it will also implement the most important feature. An **always-on-top, translucent, miniature window for active timers**. |
| Obsidian plugin | Where all the learning happens. The final implementation should **automatically** start, pause, and stop timers based on the note that I’m working on. It should allow me to ignore certain notes (no need to track time spent on certain plan-structuring notes).                                          |
| Tmux plugin     | Akin to the pomodoro plugin. Would allow me to track Timers and stopwatched directly on my terminal workflow. It could also automatically associate to the session, and add metadata (through the Description and tags) on files and projects touched on.                                                   |
