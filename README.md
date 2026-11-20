# course-catalog-db

PostgreSQL schema for the course catalog (Lab 6, "Advanced Web Technologies").

## Constraint violation log

Commands were run with `\set VERBOSITY verbose` on PostgreSQL 16.15. The SQLSTATE comes from the first line of each error (`ERROR: 23505: ...`), the constraint from the `CONSTRAINT NAME` field.

| # | What we violate | SQLSTATE | Constraint | API response |
|---|-----------------|----------|------------|--------------|
| 1 | NULL in courses.title | 23502 | none: NOT NULL has no name, the message names the column `title` (`COLUMN NAME`) | 422; if Pydantic let an empty value through, that is a bug in the code (500) |
| 2 | courses.credits = 0 | 23514 | courses_credits_check | 422 |
| 3 | courses.likes = -1 | 23514 | courses_likes_check | 500: only our code changes the counter (`likes + 1`), so it must never go negative; if it does, there is a bug in the code |
| 4 | a one-letter student name | 23514 | students_name_check | 422 |
| 5 | a duplicate student email | 23505 | students_email_key | 409 Conflict |
| 6 | a duplicate course id | 23505 | courses_pkey | 409 Conflict |
| 7 | a repeated enrollment in a course | 23505 | enrollments_course_id_student_id_key | 409 Conflict |
| 8 | enrolling in a nonexistent course | 23503 | enrollments_course_id_fkey | 404: course not found |
| 9 | enrolling a nonexistent student | 23503 | enrollments_student_id_fkey | 404: student not found (422 would also work; I chose 404 because the id points to a missing resource) |
| 10 | deleting a course that has enrollments | 23503 on PostgreSQL 16 and 17 (23001 on 18) | enrollments_course_id_fkey | 409 Conflict: a course with enrollments cannot be deleted |
| 11 | deleting a student (`DELETE FROM students WHERE email = 'daniyar@example.com'`) | no error | enrollments_student_id_fkey (`ON DELETE CASCADE`) | the student's enrollments were deleted by cascade: 6 → 3 inside the transaction, 6 again after `ROLLBACK`; the API would answer 204 |
| 12 | (optional) a manual value in the identity column `students.id` | 428C9 | none: the column is `GENERATED ALWAYS` | 500: a bug in the code, not a user error |

After all the commands the tables still had 6, 3 and 6 rows: none of the failed commands changed the data. All the errors of experiments 1-10 start with `23`, the class "integrity constraint violation", so an application tells them apart by the code and the constraint name, not by the message text (the text differs between server versions).

Gaps in the numbers are not an error: in experiment 4 the failing row was `(4, A, a@example.com)`, so the number 4 was issued before the constraint was checked and is not returned to the counter. Experiments 4 and 5 each used up a number, so the next student got 6.

**Why check in the database if Pydantic already checks in code?** Pydantic (`Field(min_length=2)`) only protects the requests that come through our FastAPI application. The database is written to by others as well: SQL scripts and migrations, an admin typing in `psql`, a second service or a future version of the API, an import, or the same API with a bug that skips validation. Only the database sees every write, so it is the last line of defense. Some rules also cannot be checked correctly in code alone: two simultaneous requests can both see that an email is free and both insert it, and only the `UNIQUE` constraint prevents that atomically.

## Index: EXPLAIN before and after

Query: `SELECT * FROM enrollments WHERE student_id = 777;`

Database `idx_demo`: 500 courses, 20,000 students, 50,000 enrollments (`experiments/idx_data.sql`).

|  | Before the index | After the index |
|---|---|---|
| Access method | Seq Scan on enrollments | Bitmap Heap Scan + Bitmap Index Scan on enrollments_student_id_idx |
| cost | 0.00..1042.00 | 4.31..15.59 |
| Rows Removed by Filter | 49997 | not reported (the index finds the 3 rows directly) |
| Execution Time (the smallest of three runs) | 2.903 ms | 0.074 ms |

Before (`EXPLAIN ANALYZE`):

```
 Seq Scan on enrollments  (cost=0.00..1042.00 rows=3 width=34) (actual time=0.054..3.497 rows=3 loops=1)
   Filter: (student_id = 777)
   Rows Removed by Filter: 49997
 Planning Time: 0.378 ms
 Execution Time: 3.534 ms
```

After `CREATE INDEX enrollments_student_id_idx ON enrollments (student_id);`:

```
 Bitmap Heap Scan on enrollments  (cost=4.31..15.59 rows=3 width=34) (actual time=0.035..0.045 rows=3 loops=1)
   Recheck Cond: (student_id = 777)
   Heap Blocks: exact=3
   ->  Bitmap Index Scan on enrollments_student_id_idx  (cost=0.00..4.31 rows=3 width=0) (actual time=0.028..0.028 rows=3 loops=1)
         Index Cond: (student_id = 777)
 Planning Time: 0.429 ms
 Execution Time: 0.091 ms
```

The index is about 39 times faster here (3 rows are found without reading 50,000), but it is not free: `enrollments` takes 3336 kB and the new index 896 kB, and every insert, update and delete of an enrollment must also update the index.

Plans of the other two queries on the same data (no new index needed):

| Query | Plan |
|---|---|
| `WHERE email = 'student777@example.com'` | Index Scan using students_email_key (a `UNIQUE` constraint creates an index) |
| `WHERE course_id = 'course-7'` | Bitmap Index Scan on enrollments_course_id_student_id_key (the index on the pair also serves a search on its first column) |

In the main database, `EXPLAIN SELECT * FROM courses WHERE id = 'api-design';` gives `Seq Scan on courses (cost=0.00..1.07)`: six rows fit on one page.

Answers to the questions:

1. PostgreSQL creates indexes automatically only for `PRIMARY KEY` and `UNIQUE`, because it needs them to enforce uniqueness. A `FOREIGN KEY` does not get one. `email` has a `UNIQUE` constraint, and `course_id` is the first column of the `UNIQUE (course_id, student_id)` pair, so its index can be used for a search on `course_id` alone. An index on `(a, b)` does not help a search on `b` alone, so `student_id` had nothing to use.
2. The planner compares costs instead of always preferring an index. Six `courses` rows fit on a single data page, so reading the whole table (cost 1.07) is cheaper than walking the index and then fetching the row. On `enrollments` the full scan costs 1042 against about 15 through the index, so the index wins.
3. A search for one student's enrollments (`WHERE student_id = ...`, for example a "my courses" page) and the cascade delete of a student, where the database must find all enrollments with that `student_id`; without the index it would scan the whole table for every deleted student. It also helps a `JOIN students s ON s.id = e.student_id` that starts from a few students.
