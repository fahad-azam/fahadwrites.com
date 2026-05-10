---
title: "All You Need to Know About Cursors in Oracle"
description: "Deep dive into Oracle cursors, cursor state, PGA memory, SQL*Plus behavior, and ORA-01000 troubleshooting."
pubDate: 'May 05 2026'
---
 
# All You Need to Know About Cursors in Oracle
 
Cursors in Oracle are a very misunderstood topic, so let's have a clean discussion here.
 
---
 
## Cursor State
 
Tracks the context and position of an open SQL statement.
 
**Stores:**
- The SQL text
- The parsed execution plan
- Current position in result set
- Bind variable values
- Transaction context of that query
### How Cursor State Works
 
| Phase | What Happens |
|-------|-------------|
| `OPEN` | Cursor created in PGA, query executed, position = before row 1 |
| `FETCH` | Returns current row, advances position by 1, remembers new position |
| `CLOSE` | Cursor destroyed, PGA memory released, Library Cache entry freed |
 
### Where a Cursor Reads Data From
 
Cursor always reads from **Buffer Cache (RAM)** — never directly from disk.
 
```
Disk → Buffer Cache (once, only if block not already cached)
Buffer Cache → Cursor fetch (always, every fetch)
```
 
This is why Buffer Cache exists. RAM is ~100,000x faster than disk.
 
### Why Cursor State Lives in PGA (Not SGA)
 
Cursor state is **private** because each session has an independent position in its result set.
 
> You are at row 47. Another user running the same query may be at row 12. These positions are completely independent. Private memory is the correct design.
 
---
 
## Two Types of Cursors
 
### Implicit Cursor
 
Oracle opens and closes it automatically for every SQL statement. You never see it, but it is always there. Every `SELECT`, `INSERT`, `UPDATE`, and `DELETE` uses an implicit cursor internally.
 
### Explicit Cursor
 
You define and control it manually in PL/SQL. Used when you need to process result rows one by one.
 
**Basic example:**
```sql
DECLARE
  CURSOR emp_cur IS
    SELECT * FROM employees WHERE dept_id = 10;
  r emp_cur%ROWTYPE;
BEGIN
  OPEN emp_cur;          -- allocates memory, executes query
  FETCH emp_cur INTO r;  -- gets next row, advances pointer
  FETCH emp_cur INTO r;  -- gets next row again
  CLOSE emp_cur;         -- releases cursor memory from PGA
END;
```
 
**Cursor FOR loop (preferred — Oracle closes automatically):**
```sql
BEGIN
  FOR r IN (SELECT * FROM employees) LOOP
    DBMS_OUTPUT.PUT_LINE(r.first_name);
  END LOOP;  -- cursor opened and closed automatically
END;
```
 
---
 
## SQL\*Plus and Cursor Behaviour — Common Misconception
 
When you run a `SELECT` in SQL\*Plus, it looks like all rows appear at once. This is SQL\*Plus hiding the mechanics from you.
 
**What actually happens internally:**
1. SQL\*Plus opens a cursor internally
2. SQL\*Plus fetches rows in batches controlled by the `ARRAYSIZE` setting
3. SQL\*Plus collects **all** batches completely
4. Then SQL\*Plus displays everything at once
### ARRAYSIZE
 
Controls how many rows SQL\*Plus fetches per network round trip.
 
- **Default:** 15
- **Valid range:** 1 to 5000
- **Set it:** `SET ARRAYSIZE 100` (more efficient for large result sets)
**ARRAYSIZE affects:**
- Number of network round trips between SQL\*Plus and Oracle
- Memory used per fetch batch
- Overall query retrieval performance
- Consistent gets (logical I/O) on the database side
**ARRAYSIZE does NOT affect:**
- Display behaviour — ever
- What you see on screen
- When rows appear on screen
> SQL\*Plus **always** collects all rows first, then displays all at once. Table size, network speed, and ARRAYSIZE — none of these change display behaviour.
 
> **Risk of very large ARRAYSIZE:** If rows are wide (many columns, large data types), a single batch may exceed SQL\*Plus's internal buffer and cause an error mid-fetch. This is why the default is 15, not 5000.
 
### True Cursor Control
 
True cursor control is only visible in:
- **PL/SQL explicit cursors** — you control `OPEN`, `FETCH`, `CLOSE`
- **Application code** — e.g. Java/Python where `rs.next()` = one `FETCH`
SQL\*Plus is an admin tool. It hides cursor mechanics entirely.
 
### JDBC / Application Fetch Size
 
In Java, `ResultSet.setFetchSize(n)` is the same concept as `ARRAYSIZE`. It controls how many rows come per network round trip. The application does **not** load all rows into memory at once — it fetches in batches, and cursor state tracks the position each time.
 
This is how Java applications handle million-row result sets without running out of memory.
 
---
 
## ORA-01000: Maximum Open Cursors Exceeded
 
**Parameter:** `OPEN_CURSORS` (default: 300)  
**Meaning:** Maximum number of cursors one session can have open simultaneously.
 
When a session tries to open cursor number 301 → `ORA-01000`.
 
### Causes
 
**Cause 1 — Cursor leak:**  
Code opens cursors but never closes them. Cursors accumulate in the session until the limit is hit. Common in Java, Python, and poorly written PL/SQL loops.
 
```java
// BAD — leaking cursors
for (int i = 0; i < 10000; i++) {
  PreparedStatement ps = conn.prepareStatement(sql + i);
  ResultSet rs = ps.executeQuery();
  // forgot to close ps and rs ← cursor leak
}
```
 
Each iteration opens a cursor. None are closed. Session hits the 300 limit → `ORA-01000`.
 
**Cause 2 — No bind variables:**  
Each unique literal value = unique SQL text = unique cursor in Library Cache. Hundreds of similar queries with different literals flood the Library Cache, putting both Shared Pool exhaustion and the cursor limit at risk.
 
### Memory Impact
 
A cursor leak affects two memory areas:
 
| Area | Impact |
|------|--------|
| **PGA** | Session's private cursor memory grows unbounded |
| **Shared Pool** | Library Cache entry locked, cannot be reused or aged out |
 
> A leaking application can degrade the Shared Pool for **all** users — not just itself.
 
### Diagnosis
 
**Find the leaking session:**
```sql
SELECT s.username,
       s.sid,
       s.serial#,
       COUNT(c.cursor#) AS open_cursors
FROM   v$session s,
       v$open_cursor c
WHERE  s.sid = c.sid
GROUP  BY s.username, s.sid, s.serial#
ORDER  BY open_cursors DESC;
```
 
The session with the highest `open_cursors` count is the leak suspect.
 
**Check current parameter value:**
```sql
SHOW PARAMETER open_cursors;
```
 
### Fix — Three Levels
 
**Level 1 — Emergency only (not a real fix):**
```sql
ALTER SYSTEM SET OPEN_CURSORS = 1000;
```
Buys time. If the code is leaking, it will eventually hit 1000 too. Never the real solution.
 
**Level 2 — Application fix (the correct fix):**
 
Fix the code. Always close cursors after use.
 
*Java — use try-with-resources (auto-closes even on exception):*
```java
try (PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
  // process rows
} // automatically closed here
```
 
*PL/SQL — always use `CLOSE cursor_name` explicitly, or use cursor FOR loops (Oracle closes automatically).*
 
**Level 3 — SQL fix:**  
Replace literal values with bind variables. One cursor definition gets reused many times, dramatically reducing the unique cursor count in Library Cache.
 
### Can a DBA Close Individual Cursors of Another Session?
 
**No.** There is no command to surgically close individual cursors of a running session.
 
**DBA options:**
```sql
-- Kill the entire session (nuclear option — emergencies only)
ALTER SYSTEM KILL SESSION 'sid,serial#';
```
 
This releases **all** cursors that session held, rolls back uncommitted transactions, and disconnects the application with an error.
 
### Interview Trap
 
> ❌ **Wrong answer:** "Just increase `OPEN_CURSORS`."
 
> ✅ **Correct answer:** "First diagnose using `V$OPEN_CURSOR` to identify the leaking session. Determine whether it is PL/SQL or application code causing the leak. Work with developers to fix cursor closure in the application code. Use bind variables to reduce unique cursors in the Shared Pool. Increasing `OPEN_CURSORS` is a temporary measure only while the fix is implemented."
