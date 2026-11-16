# E-Voting System (Local Simulation) — Project Report

> Educational simulation. It never connects to real election systems or voter databases and
> uses only fictional demo data.

## Contents
1. Project objectives
2. Project scope
3. Learning outcomes
4. Architecture and class design
5. OOP concepts used (with where to find them)
6. How each requirement is implemented
7. Data files
8. Design decisions: secret ballot, safe files, security notes
9. Test cases
10. Sample output
11. Future enhancements
12. Viva questions and answers

---

## 1. Project objectives
1. Build a complete multi-file C++17 console application using Object-Oriented Programming.
2. Model a real-world process (registration → authentication → voting → counting) with clear classes and responsibilities.
3. Enforce election rules in code: one person one vote, authentication, valid candidate, voting only while OPEN, confirmation before submission, results only after CLOSED.
4. Store all data permanently in text files with `fstream`, and recover safely from missing or corrupted files.
5. Practise defensive programming: validate every input, use exceptions, never crash on bad data.
6. Practise searching, sorting and STL containers on realistic data.
7. Produce a clean, colourful and user-friendly console interface.

## 2. Project scope
**Inside the scope**
* Voter registration, login, profile, candidate viewing, voting with confirmation, unique receipts, logout.
* Candidate management (add / edit / remove / view / search) by the admin.
* Voter management (add / remove / list / sort / search) by the admin.
* Election lifecycle (NOT STARTED → OPEN → CLOSED), statistics, turnout, results table, winner (including ties), ASCII bar chart.
* Persistent storage in six files, audit log, integrity check, reset and demo-data restore.

**Outside the scope (deliberately)**
* Any connection to a network, government database or real election.
* Real identity verification (Aadhaar/ID cards, biometrics) — never use real data.
* Production-grade cryptography, hardware security, multi-machine voting, graphical interface.
* Multiple simultaneous elections or ranked/approval voting systems (single-choice plurality only).

## 3. Learning outcomes
After completing this project a student can:
* Design a class hierarchy with an **abstract base class** and derived classes, and explain when inheritance is appropriate.
* Use **virtual functions** and base-class pointers to achieve run-time polymorphism.
* Apply **encapsulation** (private data, controlled public interface, invariants such as "a voter can vote only once").
* Write constructors with member-initialiser lists and call base-class constructors.
* Split a program into headers/sources and understand the layered design *model → logic → UI*.
* Read and write text files with `ifstream`/`ofstream`, parse delimited records, and write data safely (temp file + rename).
* Use `std::vector`, `std::map`, `std::set`, `std::optional`, `<algorithm>` (sort, copy_if, remove_if, shuffle) and lambdas.
* Implement and compare **linear search** and **binary search**.
* Create **custom exception classes** and handle them at the right layer.
* Validate user input and handle invalid menu choices gracefully.
* Test a program systematically (manual test cases + automated scripts).

## 4. Architecture and class design

```
                    ┌────────────────────────┐
                    │  main.cpp              │  command-line options, top-level try/catch
                    └───────────┬────────────┘
                                │ creates
                    ┌───────────▼────────────┐        PRESENTATION LAYER
                    │  ConsoleUI             │  menus, prompts, tables, bar chart
                    └───────────┬────────────┘
                                │ uses (reference)
                    ┌───────────▼────────────┐        BUSINESS-LOGIC LAYER
                    │  VotingSystem          │  all rules; throws exceptions on violations
                    └─┬────┬────┬────┬────┬──┘
        owns          │    │    │    │    │
   ┌──────────────────┘    │    │    │    └─────────────┐
   │                       │    │    │                  │
┌──▼──────────┐  ┌─────────▼┐ ┌─▼────▼───┐  ┌───────────▼──┐   DATA / MODEL LAYER
│ FileManager │  │ Election │ │ Candidate│  │  Vote        │
│ (fstream)   │  │ (state   │ │          │  │ (receipt,    │
│             │  │ machine) │ │          │  │ candidate,   │
└─────────────┘  └──────────┘ └──────────┘  │ date)        │
                                            └──────────────┘
          User  (abstract)           map<string, Voter>   Admin
           ├── Voter                 (voters_)            (admin_)
           └── Admin
```

| Class | Responsibility | Key members |
|-------|----------------|-------------|
| `User` (abstract) | Common data & behaviour of anyone who logs in | `id_`, `name_`, `passwordHash_`, `authenticate()`, pure virtual `role()`, `displayProfile()`, `serialize()` |
| `Voter : User` | A registered voter | `age_`, `hasVoted_`, `markVoted()`, `resetVoteStatus()` |
| `Admin : User` | The administrator | overrides the three virtual functions |
| `Candidate` | One candidate | id, name, party, symbol, age, manifesto, `matches()`, `numericId()` |
| `Vote` | One anonymous ballot | receipt, candidateId, date |
| `Election` | State machine | `start()`, `stop()`, `reset()`, `stateName()` |
| `FileManager` | **All** file I/O | `loadVoters()`, `saveVoters()`, `appendVote()`, `appendAudit()` … |
| `VotingSystem` | **All** rules and data ownership | `castVote()`, `results()`, `registerVoter()` … |
| `ConsoleUI` | Menus & screens only | `run()`, `voterMenu()`, `adminMenu()` … |
| `Utils` (namespace) | Helpers | validation, hashing, padding, `readInt()`, `readValidated()` |
| `Exceptions.h` | Exception hierarchy | `VotingException` → `ValidationException`, `AuthenticationException`, `ElectionStateException`, `DuplicateVoteException`, `FileException` |

**Relationships:** `Voter` and `Admin` *are-a* `User` (inheritance). `VotingSystem` *has-a* `FileManager`, `Election`, `Admin`, a map of `Voter`s, a vector of `Candidate`s and a vector of `Vote`s (composition). `ConsoleUI` *uses-a* `VotingSystem` (association).

## 5. OOP concepts used (with where to find them)

| Concept | Where | Explanation |
|---------|-------|-------------|
| **Classes & objects** | every `.h/.cpp` pair | `Candidate c("C101", ...)`, `Voter v(...)`, `VotingSystem system(dataDir)` in `main.cpp` |
| **Encapsulation** | `Voter`, `Election`, `VotingSystem` | Data members are `private`/`protected`. `hasVoted_` can only change through `markVoted()`, which *refuses* a second call (throws `DuplicateVoteException`). The election state can change only through `start()/stop()/reset()`, which reject illegal transitions. Other classes cannot break these rules. |
| **Inheritance** | `Voter : public User`, `Admin : public User` | Shared ID/name/password handling is written once in `User`. |
| **Abstract class / pure virtual functions** | `User` (`= 0`) | `User` cannot be instantiated; subclasses *must* implement `role()`, `displayProfile()`, `serialize()`. |
| **Polymorphism (run-time)** | `ConsoleUI::showProfile(const User&)`, `ConsoleUI::showAccountOverview()`, `VotingSystem::allAccounts()` | A `vector<const User*>` holds the admin and all voters. The call `account->role()` runs `Voter::role()` or `Admin::role()` depending on the real object. The same `showProfile(user)` prints a voter card or an admin card. |
| **Virtual destructor** | `User::~User()` | Needed so a derived object is destroyed correctly through a base pointer. |
| **`override` keyword** | `Voter`, `Admin` | Compiler checks that we really override a base function. |
| **Constructors** | all classes | Member-initialiser lists; `Voter` calls `User(...)`; `explicit Election(std::string title = "...")`; `VotingSystem` seeds its random generator. |
| **Functions / modular design** | everywhere | Small single-purpose functions; `static` factory functions `deserialize()`. |
| **Operator-free value types** | `ResultRow`, `Statistics` | Simple `struct`s for data without behaviour. |
| **STL containers** | `std::map<string,Voter>` (fast lookup by ID, sorted iteration), `std::vector<Candidate>`, `std::vector<Vote>`, `std::set<string>` (duplicate receipt detection), `std::optional<Admin>` | Container choice is explained in the viva section. |
| **STL algorithms & lambdas** | `std::sort` (voters, candidates, results), `copy_if`, `remove_if`, `shuffle`, `transform` | e.g. sorting results by votes descending with a lambda comparator. |
| **Searching** | `VotingSystem::findCandidate()` = **binary search** on candidates kept sorted by numeric ID; `searchCandidates()`, `searchVoters()` = **linear search** (case-insensitive substring) | Binary search works because IDs only grow (C101, C102…) so the vector stays sorted. |
| **Sorting** | voters by ID / name / status; candidates by ID / name / party; results by votes then name | |
| **File handling** | `FileManager` | `ifstream` to read, `ofstream` to write/append; temp-file + rename for safe saves. |
| **Exception handling** | `Exceptions.h`, `ConsoleUI::attempt()`, `main()` | The logic layer *throws*; the UI layer *catches* `VotingException` and prints the message; `main()` catches everything else. |
| **Input validation** | `Utils::readInt/readValidated/isValid*` | Every prompt loops until valid; text is trimmed; numbers are parsed without `cin >>` pitfalls (no infinite loops on letters). |
| **Const-correctness, references, RAII** | getters are `const`; `std::ofstream` closes itself at scope end | |

## 6. How each requirement is implemented

**Voter module** – `VotingSystem::registerVoter()` (auto ID `V1001, V1002 …` or a custom 4-12 char ID, unique check), `loginVoter()` (3 attempts), `Voter::displayProfile()`, `viewCandidatesFlow()`, `castVoteFlow()` (list → pick → card → **confirmation** → record), `castVote()`, unique receipt `RCP-YYYYMMDD-XXXXXXXX` from `generateReceipt()` (regenerated on collision), logout.

**Candidate module** – class `Candidate` (ID, name, party, symbol, age, manifesto). Admin: add (auto ID `C105 …`), edit (Enter keeps old value), remove, view (sortable), search. Only allowed while the election is NOT STARTED, so the ballot cannot change mid-election.

**Admin module** – default `admin / admin123` (stored hashed). Dashboard with 19 options: election control, voters, candidates, results, audit log, integrity check, account overview, password change, reset, restore demo data.

**Election module** – `Election` state machine. Illegal transitions throw `ElectionStateException`. Starting requires ≥ 2 candidates and ≥ 1 voter.

**Voting rules**

| Rule | Enforced in |
|------|-------------|
| One vote per voter | `Voter::markVoted()` + `VotingSystem::checkCanVote()` |
| Voter must be authenticated | `castVote()` takes a `Voter&` that can only be obtained from `loginVoter()` |
| Candidate must be valid | `findCandidate()` (binary search) |
| Voting must be open | `checkCanVote()` |
| Confirmation before recording | `castVoteFlow()` → `Utils::confirm()` |
| Permanent save | `FileManager::appendVote()` (append + flush) **before** the voter is marked as voted |
| Unique receipt | `generateReceipt()` + `findVote()` |
| Results only after CLOSED | `VotingSystem::results()` throws otherwise |

**Results** – table `Candidate Name | Party | Votes | Percentage`, total registered, total votes, votes not cast, turnout %, winner (or **TIE** message, or "no votes"), ASCII bar chart (scaled to 30 characters; `--ascii` uses `#`).

## 7. Data files (folder `data/`)

| File | One line looks like | Notes |
|------|--------------------|-------|
| `voters.txt` | `V1001\|Aarav Sharma\|25\|7e17ca84fb70a4ae\|0` | id, name, age, password hash, hasVoted |
| `candidates.txt` | `C101\|Arjun Mehta\|Progress Alliance\|Sun\|45\|Better roads...` | id, name, party, symbol, age, manifesto |
| `votes.txt` | `RCP-20261018-E4BDF51C\|C102\|2026-10-18` | receipt, candidate, date — **no voter ID** |
| `admin.txt` | `admin\|System Administrator\|2cf2ac5361ea3253` | id, name, password hash |
| `election.txt` | `OPEN\|General Election (Simulation)\|2026-10-18 10:02:44\|-` | state, title, start, end |
| `audit.txt` | `2026-10-18 10:03:15\|VOTE_CAST\|voter V1011` | timestamp, event, details |

Rules: `|` separates fields; lines starting with `#` are comments; user text is sanitised (`|` → `/`).

**Missing/corrupted files:** a missing file is recreated. A bad line is skipped, a warning is shown at start-up, and the damaged file is copied to `<name>.corrupt.bak` so nothing is lost. A corrupted `admin.txt` recreates the default admin; a corrupted `election.txt` resets to NOT STARTED.

## 8. Design decisions

**Secret ballot.** A `Vote` contains *receipt + candidate + date* — never the voter. The voter record only has a yes/no `hasVoted` flag. The audit log records *who voted* but not *for whom* and not the receipt. When the election is stopped, the ballots are **shuffled** so the line order in `votes.txt` cannot be matched with the audit-log order. The voter sees the receipt once; `Verify a vote receipt` confirms it exists but intentionally does **not** show the choice (this prevents proving your vote to someone else, e.g. vote-buying).

**Crash safety.** Files are written to `name.tmp` and then renamed over the real file, so a crash cannot leave a half-written file. A vote is appended to `votes.txt` first; only then is the voter marked as voted. If the program stops between the two steps, **Admin → Verify data integrity** reports the mismatch.

**Integrity check** compares the number of voters flagged as voted with the ballots stored, looks for duplicate receipts, unknown candidates, more ballots than voters, and ballots while NOT STARTED.

**Locking rules.** Candidates can change and voters can be removed only while NOT STARTED, so the ballot and the electorate stay fixed during voting. Registration stays open until the election is CLOSED.

**Security notes (honest limitations).** Passwords are stored as a salted FNV-1a hash — good enough to avoid plain text in a classroom demo, but **not secure**; real systems use bcrypt/Argon2. Passwords are typed visibly (portable code). The data files are plain text that anyone with file access can edit. This is a *simulation*; see future enhancements.

## 9. Test cases

Manual test cases. Almost all of them are automated in `tests/smoke_test.sh`; run `make test` to execute the checks (all pass).

| ID | Area | Steps | Expected result |
|----|------|-------|-----------------|
| TC01 | Start-up | Delete `data/`, run program | Folder and 6 files are created with demo data; no error |
| TC02 | Menu validation | At main menu type `abc`, `9`, empty line | "Invalid input. Please enter a whole number between 1 and 5." each time |
| TC03 | Registration – name | Name `123`, then `Maya Pillai` | First rejected, second accepted |
| TC04 | Registration – age | Age `abc`, `17`, `27` | Letters and 17 rejected; 27 accepted |
| TC05 | Registration – password | `abc` / mismatching confirmation | "Password must be 6-32…" / "Passwords do not match" |
| TC06 | Registration – ID | Choose `V1001` (existing) | Rejected: "not already in use"; empty input auto-generates next ID |
| TC07 | Voter login | Correct ID/password | Dashboard opens |
| TC08 | Wrong login | 3 wrong passwords | Message with attempt counter, then "Too many failed attempts" |
| TC09 | Case-insensitive ID | Log in as `v1001` | Works |
| TC10 | Vote before start | Election NOT STARTED, voter chooses "Cast my vote" | "Voting has not started yet" |
| TC11 | Start election | Admin → 2 → `y` | Status OPEN; start time stored |
| TC12 | Start with < 2 candidates | Remove candidates, try start | "At least 2 candidates are required" |
| TC13 | Valid vote | Choose `101`, confirm `y` | Receipt `RCP-…` displayed; `votes.txt` has one more line; voter flagged |
| TC14 | Candidate ID shortcuts | Type `c102` or `102` | Both accepted |
| TC15 | Invalid candidate | Type `999` | "Invalid candidate ID", prompt repeats |
| TC16 | Decline confirmation | Choose candidate, answer `n` | "Vote not submitted"; nothing recorded |
| TC17 | Cancel | Enter `0` | "Cancelled. No vote was recorded." |
| TC18 | Duplicate vote | Same voter votes again | "You have already voted" |
| TC19 | Receipt verification | Enter valid / fake receipt | "VALID … recorded on <date>" / "No ballot found" (choice never shown) |
| TC20 | Results while OPEN | Admin → 12 | "Results are available only after the election is CLOSED" |
| TC21 | Candidate edit while OPEN | Admin → 9 | "only allowed while the election is NOT STARTED" |
| TC22 | Stop election | Admin → 3 → `y` | Status CLOSED; ballots shuffled in `votes.txt` |
| TC23 | Vote after close | Voter tries to vote | "Voting is closed" |
| TC24 | Results | Admin → 12 after close | Table, totals, turnout, winner, bar chart; percentages add to 100 % |
| TC25 | Tie | Two candidates with equal top votes | "TIE between …" |
| TC26 | No votes | Close election with 0 votes | "No votes were cast - there is no winner." |
| TC27 | Persistence | Quit and restart after voting | State, votes and voter flags unchanged |
| TC28 | Candidate CRUD | Add, search `star`, edit (Enter keeps values), remove | Messages confirm; `candidates.txt` updated; IDs never reused downward |
| TC29 | Voter CRUD | Add voter, search by name, remove | Works; removal blocked when election is OPEN |
| TC30 | Sorting | Admin → 4 → options 1,2,3 | List ordered by ID / name / voted-first |
| TC31 | Corrupted file | Put garbage lines in `voters.txt` | Start-up warning, good records kept, `.corrupt.bak` created |
| TC32 | Corrupted election file | Write `NONSENSE` in `election.txt` | Warning; election reset to NOT STARTED |
| TC33 | Reset | Admin → 17, type `no` then `RESET` | First cancels; second clears votes and flags, state NOT STARTED |
| TC34 | Integrity check | Edit `voters.txt` to flag a voter as voted without a vote | Admin → 14 reports a mismatch |
| TC35 | End of input | Press Ctrl+D (Ctrl+Z on Windows) at any prompt | Program exits safely with a message |
| TC36 | Admin password | Change password, log in with new one | Works; old password rejected |

## 10. Sample output
```text
(Lines starting with ">>" show what the user typed. Colours are not shown here.)

############################  1. MAIN MENU  ############################
======================================================================
                           E-VOTING SYSTEM
                LOCAL SIMULATION - for education only
======================================================================
  Election status: NOT STARTED
----------------------------------------------------------------------
  1. Voter Login
  2. Voter Registration
  3. Admin Login
  4. Election Information
  5. Exit
----------------------------------------------------------------------
  Enter your choice (1-5): >> abc
  [ERROR] Invalid input. Please enter a whole number between 1 and 5.
  Enter your choice (1-5): >> 4

======================================================================
                         ELECTION INFORMATION
======================================================================
  Title              : General Election (Simulation)
  Status             : NOT STARTED
  Started at         : -
  Ended at           : -
  Candidates         : 4
  Registered voters  : 10
----------------------------------------------------------------------
  Rules: one vote per voter | voters must log in | voting only while OPEN
         results are shown only after the election is CLOSED

############################  2. VOTER REGISTRATION  ###################
======================================================================
                          VOTER REGISTRATION
======================================================================
  Demo only - do NOT enter real personal information.

  Full name            : >> 123
  [ERROR] 2-40 characters; letters, spaces, . - ' only.
  Full name            : >> Maya Pillai
  Age (18-120)         : >> 17
  [ERROR] Invalid input. Please enter a whole number between 18 and 120.
  Age (18-120)         : >> 27
  Password (6-32 chars): >> secret99
  Confirm password     : >> secret99
  Voter ID (Enter = auto-generate, or choose 4-12 letters/digits): >> (Enter)
  [OK] Registration complete!

  Your Voter ID is: V1011   (remember it - you need it to log in)

############################  3. CASTING A VOTE  #######################
======================================================================
                            CAST YOUR VOTE
======================================================================

  ID     | Name                   | Party/Group            | Symbol       | Age
  ------------------------------------------------------------------
  C101   | Arjun Mehta            | Progress Alliance      | Sun          |  45
  C102   | Lakshmi Devi           | Green Future Party     | Tree         |  52
  C103   | Sameer Khan            | Unity Front            | Bridge       |  38
  C104   | Neha Kapoor            | Citizens First         | Lamp         |  41

  Enter Candidate ID to vote for (0 = cancel): >> 999
  [ERROR] Invalid candidate ID. Please choose an ID from the list above.

  Enter Candidate ID to vote for (0 = cancel): >> 102
----------------------------------------------------------------------
  Candidate ID : C102
  Name         : Lakshmi Devi
  Party/Group  : Green Future Party
  Symbol       : Tree
  Age          : 52
  Manifesto    : Clean energy, urban forests and zero-waste
                 campuses.
----------------------------------------------------------------------
  Confirm your vote for Lakshmi Devi (Green Future Party)?  This CANNOT be changed. (y/n): >> y

  +----------------------------------------------------+
  |  VOTE RECORDED SUCCESSFULLY                        |
  |  Receipt No: RCP-20261018-E4BDF51C                 |
  +----------------------------------------------------+
  [INFO] Write this receipt down. It is shown only once and is not linked to your choice.

############################  4. RULE ENFORCEMENT  #####################
  (same voter tries again)
  [ERROR] You have already voted. Each voter may vote only once.

  (voting before the election starts)
  [ERROR] Voting has not started yet. Please come back when the election is OPEN.

  (admin asks for results while the election is OPEN)
  [ERROR] Results are available only after the election is CLOSED (now OPEN).

  (admin tries to add a candidate while OPEN)
  [ERROR] Adding candidates is only allowed while the election is NOT STARTED (now OPEN).

  (3 wrong passwords)
  [ERROR] Invalid Voter ID or password. (attempt 3 of 3)
  [ERROR] Too many failed attempts. Returning to the main menu.

############################  5. FINAL RESULTS  ########################
======================================================================
                        FINAL ELECTION RESULTS
                    General Election (Simulation)
======================================================================
  Candidate Name         | Party                  | Votes | Percentage
  ------------------------------------------------------------------
  Lakshmi Devi           | Green Future Party     |     4 |     44.44%
  Arjun Mehta            | Progress Alliance      |     3 |     33.33%
  Neha Kapoor            | Citizens First         |     1 |     11.11%
  Sameer Khan            | Unity Front            |     1 |     11.11%

  Total registered voters : 10
  Total votes cast        : 9
  Votes not cast          : 1
  Voter turnout           : 90.00%

  WINNER: Lakshmi Devi (Green Future Party) with 4 vote(s)

======================================================================
                           VOTES BAR CHART
======================================================================
  Lakshmi Devi           ██████████████████████████████ 4
  Arjun Mehta            ███████████████████████        3
  Neha Kapoor            ████████                       1
  Sameer Khan            ████████                       1

############################  6. AUDIT LOG (admin)  ####################
  2026-10-18 10:02:11  SYSTEM_START          first run - demo data created
  2026-10-18 10:02:30  VOTER_REGISTERED      V1011
  2026-10-18 10:02:41  LOGIN_OK              admin admin
  2026-10-18 10:02:44  ELECTION_STARTED      11 voters, 4 candidates
  2026-10-18 10:03:02  LOGIN_OK              voter V1011
  2026-10-18 10:03:15  VOTE_CAST             voter V1011
  2026-10-18 10:03:20  LOGIN_FAILED          voter login, id entered: V1001
  (note: the log records WHO voted, never WHAT they voted for)
```

## 11. Future enhancements
1. **Better security:** hidden password entry, bcrypt/Argon2 hashing, account lock-out saved to disk, encrypted data files, digital signatures on the audit log (hash chain).
2. **Cryptographic voting ideas:** blind signatures or homomorphic tallying so even the administrator cannot link a voter to a ballot; end-to-end verifiable receipts.
3. **Database back end:** replace text files with SQLite behind the same `FileManager` interface (make it an abstract `Storage` class and use polymorphism).
4. **Multiple elections / constituencies** and other voting methods (ranked choice, approval).
5. **Scheduling:** open/close automatically at set date-times.
6. **Reports:** export results to CSV/PDF, per-constituency turnout, age-group statistics.
7. **GUI or web front end** (Qt, or a REST API with the existing `VotingSystem` as the core).
8. **Unit tests** with Catch2/GoogleTest for every class, plus continuous integration.
9. **Smart pointers & templates:** a generic `Repository<T>` template for voters, candidates and votes.
10. **Localisation:** language files for menus and messages.
11. **Concurrency:** several polling booths with file locking.
12. **Undo/redo and backups:** automatic dated backups of the data folder.

## 12. Viva questions and answers

**Q1. What is the main idea of your project?**
A local, offline simulation of an election. Voters register and log in, the admin controls the election (NOT STARTED → OPEN → CLOSED), voters cast one confirmed vote each, and after closing the system shows results, turnout, winner and a bar chart. All data is stored in text files.

**Q2. Which OOP concepts did you use and where?**
Inheritance (`Voter`/`Admin` from `User`), abstract class and pure virtual functions (`User`), polymorphism (`vector<const User*>`, `showProfile(const User&)`), encapsulation (private `hasVoted_`, state machine in `Election`), constructors with initialiser lists, and exception classes. See section 5.

**Q3. What is an abstract class? Why is `User` abstract?**
A class with at least one pure virtual function (`= 0`); it cannot be instantiated. A plain `User` makes no sense — a person is either a voter or an admin — so the base class only defines the common interface and shared data.

**Q4. What is polymorphism? Show it in your code.**
One interface, many behaviours chosen at run time. In `showAccountOverview()` we loop over `const User*` and call `account->role()`; for the admin it returns "Administrator", for voters "Voter". `showProfile(const User&)` prints different profile screens depending on the real object type.

**Q5. Why did you write a virtual destructor in `User`?**
If a `Voter` is deleted through a `User*`, a non-virtual destructor would only run `~User()` and leak/skip derived cleanup (undefined behaviour). A virtual destructor makes the correct destructor run.

**Q6. What does `override` do?**
It asks the compiler to check that the function really overrides a virtual function in the base class. A typo in the signature becomes a compile error instead of a silent new function.

**Q7. Give an example of encapsulation.**
`Voter::hasVoted_` is private. Outside code cannot set it directly; it must call `markVoted()`, which throws `DuplicateVoteException` if the voter already voted. So the invariant "one vote per voter" cannot be broken by accident. Likewise `Election`'s state changes only via `start/stop/reset`.

**Q8. Why separate `VotingSystem` and `ConsoleUI`?**
Separation of concerns. `VotingSystem` contains the rules and has no `cout` menus, so it could be reused with a GUI or tested without a console. `ConsoleUI` only handles input/output.

**Q9. Why is there a separate `FileManager`?**
Single responsibility: only this class knows file names and formats. To switch to a database we change one class, not the whole program.

**Q10. Why `std::map` for voters and `std::vector` for candidates?**
Voters are looked up by ID all the time → `map` gives O(log n) lookup and sorted iteration. Candidates are few, are printed in order, and I need random access for binary search → `vector` kept sorted by numeric ID. Votes are only appended and counted → `vector`.

**Q11. Explain your binary search. What is its precondition?**
`findCandidate()` converts `"C102"` to the number 102 and repeatedly halves the range `[low, high)` of the vector, comparing with `numericId()`. The vector **must be sorted** by that key. It stays sorted because new IDs are always `max + 1` and loading sorts the list. Complexity O(log n) versus O(n) for linear search.

**Q12. Where did you use linear search and sorting?**
Linear search: `searchCandidates()` and `searchVoters()` (substring match on several fields, so sorting does not help). Sorting: `std::sort` with lambdas for voters (ID/name/status), candidates (ID/name/party) and results (votes descending, ties by name).

**Q13. How do you prevent duplicate voting?**
Three layers: `checkCanVote()` rejects a voter whose `hasVoted` is true; `Voter::markVoted()` throws if called twice; the flag is saved to `voters.txt`, so restarting the program does not reset it. The integrity check also compares flags with stored ballots.

**Q14. Why not store the voter ID inside the vote?**
To keep the ballot secret. `votes.txt` has receipt, candidate and date only. Ballots are shuffled when the election closes, and the audit log never records the choice or the receipt, so nobody can link a person to a candidate from the files.

**Q15. Why can the voter not see which candidate a receipt belongs to?**
If a receipt could prove a choice, someone could be forced or paid to show it (coercion/vote buying). The receipt only proves *that* a ballot was counted.

**Q16. What happens if the program crashes during a vote?**
The ballot is appended and flushed first, then the voter is marked and `voters.txt` is saved. A crash in between leaves a ballot without a flagged voter; Admin → *Verify data integrity* detects the mismatch. Files are written via temp file + rename, so no file is left half-written.

**Q17. How do you handle exceptions?**
Rule violations are reported with `throw` of specific classes derived from `VotingException`. `ConsoleUI::attempt()` catches `VotingException` and prints a friendly message so the menu continues. `InputClosedException` (Ctrl+D) is allowed to travel to `main()`, which exits cleanly. Unexpected `std::exception`s are caught in `main()`.

**Q18. How do you validate input?**
`readInt()` reads a whole line, accepts only digits in the given range and repeats otherwise; `readValidated()` takes a validation function (lambda) and an error message. Because I read lines with `getline`, typing letters can never put `cin` into an error state (no infinite loops).

**Q19. How does the program handle missing or corrupted files?**
`FileManager` returns empty data for a missing file, `VotingSystem::initialize()` recreates it (demo data on the very first run). Each line is parsed inside try/catch; bad lines are skipped, reported as warnings and the file is backed up as `.corrupt.bak`.

**Q20. How are passwords stored?**
As a 64-bit FNV-1a hash of `"salt:userId:password"`, printed as 16 hex digits. This hides the plain text and gives each user a different hash even for equal passwords, but FNV is fast and not secure; real systems use bcrypt/Argon2 — this is only a classroom simulation.

**Q21. What is the difference between `private` and `protected`?**
`private` members are visible only inside the class; `protected` members are also visible in derived classes. `User` keeps `id_`, `name_`, `passwordHash_` protected so `Voter` and `Admin` can use them in `displayProfile()`/`serialize()`, but the rest of the program cannot.

**Q22. Inheritance vs composition — where did you use each?**
Inheritance for "is-a" (`Voter` is a `User`). Composition for "has-a" (`VotingSystem` has an `Election`, a `FileManager`, voters, candidates and votes).

**Q23. Why does `Election` use a state machine?**
Voting is legal only in one state. Putting transitions in `start()/stop()/reset()` (which reject illegal ones) means every other part of the program can simply ask `isOpen()`; the rules are in one place.

**Q24. What is `std::optional` used for here?**
`FileManager::loadAdmin()` returns `std::optional<Admin>` — either an admin or "nothing" (file missing/corrupt). It avoids a magic value or a default-constructed `Admin` (which cannot exist because `Admin` has no default constructor).

**Q25. How do you pick the winner and handle ties?**
Results are sorted by votes descending. The top vote count is compared with every row: one leader → winner; several leaders → "TIE between …"; zero votes → "no winner". Percentages are relative to valid votes cast.

**Q26. Why `const` after member functions such as `int age() const`?**
It promises the function does not modify the object, so it can be called on `const` objects/references (for example in `showProfile(const User&)`).

**Q27. What would you improve first?**
Hidden password input, a stronger hash, and replacing the text files with SQLite behind an abstract `Storage` interface — which would show polymorphism even more clearly.
