---
title: "All You Need to Know About Cursors in Oracle"
description: "Deep dive into Oracle cursors, cursor state, PGA memory, SQL*Plus behavior, and ORA-01000 troubleshooting."
pubDate: 'May 05 2026'
heroImage: '../../assets/ASM.jpg'
---

## All You need to know about Cursors in Oracle

Cursors in oracle are very misunderstood topic so lets have a clean discussion here

Cursor State:
    Tracks the context and position of an open SQL statement.
    Stores:
      * The SQL text
      * The parsed execution plan
      * Current position in result set
      * Bind variable values
      * Transaction context of that query

    How cursor state works:
      OPEN   → cursor created in PGA, query executed, position = before row 1
      FETCH  → returns current row, advances position by 1, remembers new position
      CLOSE  → cursor destroyed, PGA memory released, Library Cache entry freed

    Where cursor reads data from:
      Cursor always reads from BUFFER CACHE (RAM). Never directly from disk.
      Full flow:
        Disk → Buffer Cache (once, only if block not already cached)
        Buffer Cache → Cursor fetch (always, every fetch)
      This is why Buffer Cache exists.
      RAM is 100,000x faster than disk.

    Cursor state lives in PGA (private) not SGA (shared) because:
      You are at row 47 in your result set.
      Another user running the same query may be at row 12.
      These positions are completely independent.
      Private memory is the correct design.

  Stack Space:
    Local variables used in PL/SQL execution.
    Function call stack. Procedure parameters.




--------------------------------------------------------------------------------
TWO TYPES OF CURSORS
--------------------------------------------------------------------------------

Implicit Cursor:
  Oracle opens and closes automatically for every SQL statement.
  You never see it but it is always there.
  Every SELECT, INSERT, UPDATE, DELETE uses an implicit cursor internally.

Explicit Cursor:
  You define and control it manually in PL/SQL.
  Used when you need to process result rows one by one.

  Example:
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

  Cursor FOR loop (preferred — Oracle closes automatically):
    BEGIN
      FOR r IN (SELECT * FROM employees) LOOP
        DBMS_OUTPUT.PUT_LINE(r.first_name);
      END LOOP;  -- cursor opened and closed automatically
    END;




--------------------------------------------------------------------------------
SQL*PLUS AND CURSOR BEHAVIOUR — COMMON MISCONCEPTION
--------------------------------------------------------------------------------

When you run SELECT in SQL*Plus it looks like all rows appear at once.
This is SQL*Plus hiding the mechanics from you.

What actually happens internally:
  SQL*Plus opens a cursor internally.
  SQL*Plus fetches rows in batches controlled by ARRAYSIZE setting.
  SQL*Plus collects ALL batches completely.
  Then SQL*Plus displays everything at once.

ARRAYSIZE:
  Controls how many rows SQL*Plus fetches per network round trip.
  Default: 15. Valid range: 1 to 5000.
  SET ARRAYSIZE 100  (more efficient for large result sets)

  ARRAYSIZE affects:
    → Number of network round trips between SQL*Plus and Oracle
    → Memory used per fetch batch
    → Overall query retrieval performance
    → Consistent gets (logical I/O) on the database side

  ARRAYSIZE does NOT affect:
    → Display behaviour. Ever.
    → What you see on screen.
    → When rows appear on screen.

  SQL*Plus ALWAYS collects all rows first then displays all at once.
  Table size, network speed, ARRAYSIZE — none of these change display behaviour.

  Risk of very large ARRAYSIZE:
    If rows are wide (many columns, large data types)
    result per batch may exceed SQL*Plus internal buffer
    → error mid-fetch
    This is why default is 15 not 5000.

True cursor control is only visible in:
  → PL/SQL explicit cursors (you control OPEN, FETCH, CLOSE)
  → Application code (Java, Python, etc. — rs.next() = one FETCH)
  SQL*Plus is an admin tool. It hides cursor mechanics entirely.

JDBC / application fetch size:
  In Java, ResultSet.setFetchSize(n) = same concept as ARRAYSIZE.
  Controls how many rows come per network round trip.
  Application does NOT load all rows into memory at once.
  Fetches in batches. Cursor state tracks position each time.
  This is how Java applications handle million-row result sets
  without running out of memory.




--------------------------------------------------------------------------------
ORA-01000: MAXIMUM OPEN CURSORS EXCEEDED
--------------------------------------------------------------------------------

Parameter: OPEN_CURSORS (default: 300)
Meaning: maximum number of cursors one session can have open simultaneously.

When session tries to open cursor number 301 → ORA-01000.

Cause 1 — Cursor leak:
  Code opens cursors but never closes them.
  Cursors accumulate in the session until limit is hit.
  Common in Java, Python, and poorly written PL/SQL loops.

  Example of bad Java code:
    for (int i = 0; i < 10000; i++) {
      PreparedStatement ps = conn.prepareStatement(sql + i);
      ResultSet rs = ps.executeQuery();
      // forgot to close ps and rs
    }
  Each iteration opens a cursor. None are closed.
  Session hits 300 limit. ORA-01000.

Cause 2 — No bind variables:
  Each unique literal value = unique SQL text = unique cursor in Library Cache.
  Hundreds of similar queries with different literals flood Library Cache.
  Shared Pool exhaustion + cursor limit both at risk.

Affects two memory areas:
  PGA      → session's private cursor memory
  Shared Pool → Library Cache entry locked, cannot be reused or aged out
  A leaking application can degrade Shared Pool for ALL users. Not just itself.

Diagnosis:
  Find leaking session:
    SELECT s.username,
           s.sid,
           s.serial#,
           COUNT(c.cursor#) AS open_cursors
    FROM   v$session s,
           v$open_cursor c
    WHERE  s.sid = c.sid
    GROUP  BY s.username, s.sid, s.serial#
    ORDER  BY open_cursors DESC;

  Session with highest open_cursors = leak suspect.

  Check current parameter value:
    SHOW PARAMETER open_cursors;

Fix — three levels:

  Level 1 — Emergency only (not a real fix):
    ALTER SYSTEM SET OPEN_CURSORS = 1000;
    Buys time. If code is leaking it will hit 1000 too. Never the real solution.

  Level 2 — Application fix (correct fix):
    Fix the code. Always close cursors after use.
    Java: use try-with-resources (auto-closes even on exception)
      try (PreparedStatement ps = conn.prepareStatement(sql);
           ResultSet rs = ps.executeQuery()) {
        // process rows
      } // automatically closed here
    PL/SQL: always use CLOSE cursor_name explicitly
            or use cursor FOR loops (Oracle closes automatically)

  Level 3 — SQL fix:
    Replace literal values with bind variables.
    One cursor definition reused many times.
    Dramatically reduces unique cursor count in Library Cache.

Can DBA close individual cursors of another session?
  NO. Not possible. There is no command to surgically close
  individual cursors of a running session.

DBA options:
  Kill the entire session:
    ALTER SYSTEM KILL SESSION 'sid,serial#';
    Releases ALL cursors that session held.
    Rolls back uncommitted transactions.
    Application gets disconnection error.
    Nuclear option. Only for emergencies.

Interview trap:
  Wrong answer: "Just increase OPEN_CURSORS"
  Correct answer:
    "First diagnose using V$OPEN_CURSOR to identify the leaking session.
     Determine whether it is PL/SQL or application code causing the leak.
     Work with developers to fix cursor closure in application code.
     Use bind variables to reduce unique cursors in Shared Pool.
     Increasing OPEN_CURSORS is a temporary measure only while fix is implemented."


