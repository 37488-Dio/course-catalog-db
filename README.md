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
