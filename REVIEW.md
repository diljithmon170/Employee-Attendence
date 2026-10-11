<!-- # REVIEW.md

List every defect you found in the starter `app/main.py` (helpers and endpoints). For each one:

| # | Where (function / line) | What is wrong | How you'd notice it (test, input, or symptom) | How you fixed it |
|---|---|---|---|---|
| 1 | | | | |

Also note anything you looked at and decided was **not** a defect, and why. -->

# REVIEW.md

## Defects Found and Fixed in `app/main.py`

| #   | Where (function / line)  | What is wrong                                                                                                                                                                               | How you'd notice it (test, input, or symptom)                                                                                                    | How you fixed it                                                                                                                                                                                                   |
| --- | ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | `/health` endpoint       | The endpoint always returns HTTP 200 without verifying whether MongoDB is reachable.                                                                                                        | Stop MongoDB or configure an invalid MongoDB connection. The endpoint still reports a successful health status.                                  | Added a MongoDB ping using `client.admin.command("ping")`. Return HTTP 200 when successful and HTTP 503 when the database connection check fails.                                                                  |
| 2   | `compute_late_minutes()` | The function needs to enforce the 10-minute grace period correctly and return whole elapsed minutes.                                                                                        | For a 09:00 shift, test punch-in times at 09:10 and 09:11. Expected results are 0 and 11 minutes, respectively.                                  | Applied a 10-minute grace period. Return 0 when punch-in is within the grace period; otherwise, return whole elapsed minutes since the scheduled shift start.                                                      |
| 3   | `compute_work_hours()`   | Python's built-in `round()` uses round-half-to-even behavior, which can produce unexpected rounding results.                                                                                | Test durations that require rounding to two decimal places, including a halfway value.                                                           | Used `Decimal` and `ROUND_HALF_UP` to round calculated work hours consistently to two decimal places.                                                                                                              |
| 4   | `compute_overtime()`     | The function does not enforce the 30-minute minimum overtime threshold and does not reliably handle overnight shifts.                                                                       | Test overtime of 20 minutes (expected 0), 30 minutes (expected 30), and a punch-out after midnight for an overnight shift.                       | Added a 30-minute minimum threshold and overnight date handling to calculate overtime from the correct scheduled shift-end time.                                                                                   |
| 5   | `create_employee()`      | Duplicate employee codes may be inserted if uniqueness is not enforced at the database level.                                                                                               | Submit two employee creation requests with the same `emp_code`. Duplicate records may be created without a unique index.                         | Added a unique index on `emp_code` and handled `DuplicateKeyError` by returning HTTP 409 Conflict.                                                                                                                 |
| 6   | `list_employees()`       | The endpoint needs to validate pagination inputs and return an accurate count for the applied filters.                                                                                      | Request a page number below 1, an invalid page size, or filter employees by department and compare the returned total with the matching records. | Added pagination validation, applied filtering in MongoDB, used `count_documents()` for the matching total, and used `.skip()` and `.limit()` for pagination.                                                      |
| 7   | `punch_in()`             | The function accesses employee shift information without first verifying that the employee exists. It may also allow duplicate attendance records if uniqueness is not enforced by MongoDB. | Punch in using an unknown `emp_code`, or submit multiple punch-in requests for the same employee and date.                                       | Added employee existence validation with HTTP 404, used IST for the attendance date and timestamp, and handled duplicate attendance records with HTTP 409. Added a unique compound index on `emp_code` and `date`. |
| 8   | `list_attendance()`      | The function loads the entire attendance collection into Python memory, sorts records in Python, and performs pagination after loading all matching records.                                | Add many attendance records and request a single page. The endpoint still loads all matching records before returning a small result.            | Moved sorting and pagination into MongoDB using `.sort()`, `.skip()`, and `.limit()`. Used `count_documents()` to calculate the total number of matching records.                                                  |
| 9   | `create_indexes()`       | The attendance index creation statement is outside the startup function, so it executes when the module is imported rather than as part of the startup handler.                             | Inspect the indentation and execution order of the index creation statements.                                                                    | Moved both employee and attendance index creation statements inside `create_indexes()` so they run together during application startup.                                                                            |

## Items Reviewed but Not Considered Defects

1. **MongoDB client and database initialization:** Using `MongoClient` and selecting the database through environment variables is valid. However, creating a client does not guarantee that MongoDB is reachable. The health endpoint must explicitly ping the database.

2. **Work-hours return type:** Returning a `float` rounded to two decimal places is consistent with the existing function signature. Using `Decimal` internally improves rounding behavior without changing the public return type.

3. **Late-minute return type:** Returning an integer is appropriate because lateness is measured in whole minutes.

4. **Overtime return type:** Returning an integer is appropriate because overtime is measured in whole minutes. The 30-minute minimum is a business rule, not a requirement to change the return type.

5. **Existing function interfaces:** The function signatures were retained where possible to avoid breaking existing callers.

6. **MongoDB filtering and pagination:** Filtering, counting, sorting, and pagination in MongoDB is preferable to retrieving all matching records into Python memory, particularly as the attendance collection grows.

7. **Unique indexes:** Unique indexes are appropriate for preventing duplicate employee codes and duplicate daily attendance records. Existing duplicate data must be resolved before creating these indexes.

## Summary

Nine areas were reviewed and addressed:

- `/health`: MongoDB availability verification and HTTP 503 error handling.
- `compute_late_minutes()`: 10-minute grace period and whole-minute lateness calculation.
- `compute_work_hours()`: Decimal rounding with `ROUND_HALF_UP`.
- `compute_overtime()`: 30-minute minimum overtime and overnight shift handling.
- `create_employee()`: Duplicate employee prevention and HTTP 409 handling.
- `list_employees()`: MongoDB filtering, counting, and validated pagination.
- `punch_in()`: Employee existence validation, IST timestamp handling, and duplicate attendance prevention.
- `list_attendance()`: MongoDB-side filtering, sorting, counting, and pagination.
- `create_indexes()`: Centralized employee and attendance index creation during startup.

**Note:** These fixes should be validated against the application's actual requirements and automated tests. Pay particular attention to IST and UTC handling, overnight shifts, existing duplicate records, concurrent punch-in requests, pagination boundaries, and the expected behavior of the 10-minute grace period.
